# Instalación de N8N con Docker, Traefik y VPS

> **Video de referencia:** [https://www.youtube.com/watch?v=lGzXhrVXFmo](https://www.youtube.com/watch?v=lGzXhrVXFmo)

---

## Referencia rápida de comandos

### 1. Comandos de Terminal (Linux y SSH)

#### Navegación y listado

| Comando | Timestamp | Descripción |
|---------|-----------|-------------|
| `cd [nombre_carpeta]` | 11:48 | Navega entre carpetas. |
| `ls` | 11:59 | Lista archivos y directorios dentro de una carpeta. |
| `pwd` | 12:03, 36:11, 37:41 | Muestra la ubicación actual. |

#### Conexión SSH

| Comando | Timestamp | Descripción |
|---------|-----------|-------------|
| `ssh root@[dirección IP]` | 17:58, 19:35, 36:45 | Se conecta al servidor VPS como usuario root. |
| `ssh-keygen -R [dirección IP]` | 19:10 | Elimina la entrada antigua del archivo `known_hosts` para resolver errores de autenticación SSH. |

#### Gestión de sesión

| Comando | Timestamp | Descripción |
|---------|-----------|-------------|
| `clear` | 19:30, 21:24, 23:36 | Limpia la ventana de la terminal. |
| `exit` | 36:29 | Sale de la sesión actual. |
| `su - n8nuser` | 37:10 | Inicia sesión como el superusuario "n8nuser". |

### 2. Comandos de Instalación y Configuración Inicial

| Comando / Acción | Timestamp | Descripción |
|-------------------|-----------|-------------|
| `sudo apt update` | 20:54 | Actualiza los paquetes del sistema. |
| `adduser n8nuser` | 21:31 | Crea el usuario "n8nuser". |
| `usermod -aG sudo n8nuser` | 21:31 | Agrega el usuario "n8nuser" al grupo sudo. |
| `su - n8nuser` | 21:31 | Cambia al nuevo usuario "n8nuser". |
| Instalar Docker y Docker Compose | 22:57 | Comandos para instalar Docker y Docker Compose. |
| Crear directorios | 23:47 | Comandos para crear los directorios necesarios. |

### 3. Comandos de Nano (Editor de Texto)

| Comando | Timestamp | Descripción |
|---------|-----------|-------------|
| `nano [nombre_archivo]` | 12:47, 23:58, 26:00, 30:04, 38:36 | Abre un archivo en el editor de texto Nano. |
| `Control + X` | 25:17 | Salir del editor Nano. |
| `Y` | 25:29 | Guardar los cambios al salir. |
| `Control + S` | 26:45, 38:52 | Guardar el archivo en Nano. |

### 4. Comandos de N8N y Docker Compose

| Comando / Acción | Timestamp | Descripción |
|-------------------|-----------|-------------|
| `docker compose up -d` | 12:11, 44:55 | Levanta los servicios (N8N, Traefik, PostgreSQL) en segundo plano. |
| `docker compose down` | 12:35, 44:32 | Baja los servicios (detiene N8N, Traefik, PostgreSQL). |
| `docker compose pull` | 44:26 | Obtiene las últimas versiones de las imágenes de Docker. |
| Permisos de Traefik y firewall | 39:17 | Comandos para dar permisos a Traefik y configurar el firewall. |
| Generar hash de contraseña | 27:21 | Comando para generar el hash de la contraseña. |

---

## Proceso de instalación paso a paso

### Paso 1: Conectarse al servidor VPS

```bash
ssh root@[dirección IP]
```

### Paso 2: Actualizar los paquetes del sistema

```bash
apt update && apt upgrade -y
```

### Paso 3: Crear usuario No-Root

```bash
# Crear usuario
adduser n8nuser

# Agregar a grupo sudo
usermod -aG sudo n8nuser

# Cambiar a nuevo usuario
su - n8nuser
```

### Paso 4: Instalar Docker y Docker Compose

```bash
# Instalar dependencias
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

# Agregar clave GPG de Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Agregar repositorio Docker
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instalar Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io

# Instalar Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Agregar usuario al grupo docker
sudo usermod -aG docker $USER
newgrp docker
```

### Paso 5: Crear directorios

```bash
# Crear estructura de directorios
mkdir -p ~/n8n-stack/{traefik,n8n,postgres}
cd ~/n8n-stack

# Crear directorios para datos persistentes
mkdir -p traefik/data
mkdir -p traefik/config
mkdir -p n8n/data
mkdir -p postgres/data
```

### Paso 6: Configuración de Traefik

#### 6.1 Crear archivo `traefik.yml`

```bash
nano traefik/traefik.yml
```

Contenido del archivo `traefik.yml`:

```yaml
# Traefik Configuration
global:
  checkNewVersion: false
  sendAnonymousUsage: false

api:
  dashboard: true
  debug: true
  insecure: false

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entrypoint:
          to: websecure
          scheme: https
          permanent: true
  websecure:
    address: ":443"

certificatesResolvers:
  letsencrypt:
    acme:
      tlsChallenge: {}
      email: tu-email@ejemplo.com  # CAMBIAR POR TU EMAIL
      storage: /letsencrypt/acme.json
      caServer: https://acme-v02.api.letsencrypt.org/directory

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
  file:
    filename: /config/dynamic.yml
    watch: true

log:
  level: INFO
```

#### 6.2 Crear configuración dinámica

```bash
nano traefik/config/dynamic.yml
```

Contenido del archivo `dynamic.yml`:

```yaml
# Dynamic Configuration
http:
  middlewares:
    secure-headers:
      headers:
        accessControlAllowMethods:
          - GET
          - OPTIONS
          - PUT
        accessControlMaxAge: 100
        hostsProxyHeaders:
          - "X-Forwarded-Host"
        referrerPolicy: "same-origin"
        sslRedirect: true
        forceSTSHeader: true
        stsIncludeSubdomains: true
        stsPreload: true
        stsSeconds: 31536000
```

---

### Configuracion de Docker Compose

### Paso 7: Generar Password Hash para Traefik

```bash
# Instalar herramientas
sudo apt install -y apache2-utils

# Generar hash (reemplazar 'tupassword' por tu password real)
htpasswd -nb admin tupassword
```

Ejemplo de salida:

```
admin:$apr1$n33ZsPF2$ibyAnDwOhZ3OCEnpgfMqn.
```

> **Nota:** Guarda este hash, lo necesitarás para la configuración de Docker Compose.
