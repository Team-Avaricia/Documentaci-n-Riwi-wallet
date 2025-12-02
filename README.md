# Documentación DevOps - Configuración Inicial del Proyecto

## Información General

**Proyecto**: Proyecto Académico - Riwi Wallet  
**Fecha de Inicio**: Diciembre 2024  
**Servidor**: VPS proporcionado para el proyecto

---

## 1. Configuración Inicial del Servidor

### 1.1 Gestión de Usuarios

Como primera medida de seguridad y organización del equipo, se crearon cuentas de usuario para todos los integrantes del proyecto con permisos administrativos.

**Usuarios creados**: 6 integrantes del equipo 

**Comandos utilizados**:
```bash
# Crear usuario
sudo adduser <nombre_usuario>

# Agregar permisos sudo
sudo usermod -aG sudo <nombre_usuario>
```

---

## 2. Instalación de Herramientas Base

### 2.1 Docker

Se instaló Docker como plataforma de contenedorización para facilitar el despliegue y gestión de servicios.

```bash
# Actualizar repositorios
sudo apt update

# Instalar Docker
sudo apt install docker.io -y

# Habilitar Docker al inicio
sudo systemctl enable docker
sudo systemctl start docker

# Agregar usuario al grupo docker
sudo usermod -aG docker $USER
```

### 2.2 Nginx

Nginx se configuró como servidor web reverso para gestionar el tráfico HTTP/HTTPS.

```bash
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

### 2.3 Certbot

Certbot se instaló para la gestión automática de certificados SSL mediante Let's Encrypt.

```bash
sudo apt install certbot python3-certbot-nginx -y
```

### 2.4 Docker Compose

Se instaló Docker Compose para orquestar múltiples contenedores.

```bash
sudo apt install docker-compose -y
```

---

## 3. Configuración de DNS y SSL

### 3.1 Dominio Configurado

**URL**: https://docs.avaricia.crudzaso.com/

### 3.2 Configuración Nginx

Se configuró Nginx como proxy reverso para dirigir el tráfico al contenedor de Wiki.js.

**Archivo de configuración**: `/etc/nginx/sites-available/docs.avaricia.crudzaso.com`

### 3.3 Certificado SSL

Se generó un certificado SSL con Certbot:

```bash
sudo certbot --nginx -d docs.avaricia.crudzaso.com
```

---

## 4. Despliegue de Servicios con Docker Compose

### 4.1 Arquitectura de Contenedores

Se desplegaron dos servicios principales:

1. **PostgreSQL** (Base de datos)
2. **Wiki.js** (Sistema de documentación)

### 4.2 Archivo docker-compose.yml

```yaml
version: '3.8'

services:
  db:
    image: postgres:16-alpine
    container_name: riwi_db
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    ports:
      - "5432:5432"
    volumes:
      - riwi_data:/var/lib/postgresql/data
      - ./config/postgresql.conf:/etc/postgresql/postgresql.conf
    command: postgres -c config_file=/etc/postgresql/postgresql.conf
    deploy:
      resources:
        limits:
          memory: 1500m
          cpus: '0.75'
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U riwi_user -d riwiwallet_db"]
      interval: 10s
      timeout: 5s
      retries: 5

  wiki:
    image: ghcr.io/requarks/wiki:2
    container_name: riwi_wiki
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      DB_TYPE: postgres
      DB_HOST: db
      DB_PORT: 5432
      DB_USER: ${WIKI_DB_USER}
      DB_PASS: ${WIKI_DB_PASSWORD}
      DB_NAME: ${WIKI_DB_NAME}
    volumes:
      - riwi_wiki_assets:/wiki/assets
    depends_on:
      db:
        condition: service_healthy
    deploy:
      resources:
        limits:
          memory: 512m
          cpus: '0.5'

volumes:
  riwi_data:
  riwi_wiki_assets:
```

### 4.3 Variables de Entorno

Se creó un archivo `.env` en el mismo directorio del `docker-compose.yml` con las credenciales.

### 4.4 Comandos de Despliegue

```bash
# Levantar los contenedores
docker-compose up -d

# Ver logs
docker-compose logs -f

# Ver estado de los contenedores
docker-compose ps

# Detener los contenedores
docker-compose down
```

---

## 5. Optimización de PostgreSQL

### 5.1 Archivo de Configuración

Se creó un archivo de configuración personalizado para optimizar el rendimiento de PostgreSQL.

**Ruta**: `./config/postgresql.conf`

```conf
# Configuración de memoria
shared_buffers = 512MB
work_mem = 4MB
maintenance_work_mem = 64MB
wal_buffers = 16MB

# Configuración de rendimiento
effective_io_concurrency = 10

# Configuración de conexiones
max_connections = 50

# Permitir conexiones externas
listen_addresses = '*'
```

### 5.2 Justificación de Parámetros

- **shared_buffers (512MB)**: Memoria compartida para cachear datos de tablas e índices
- **work_mem (4MB)**: Memoria para operaciones de ordenamiento y hash por consulta
- **maintenance_work_mem (64MB)**: Memoria para operaciones de mantenimiento como VACUUM
- **wal_buffers (16MB)**: Buffer para el write-ahead log
- **effective_io_concurrency (10)**: Número de operaciones de I/O concurrentes
- **max_connections (50)**: Límite de conexiones simultáneas
- **listen_addresses**: Configurado para aceptar conexiones desde cualquier IP

---

## 6. Limitación de Recursos

### 6.1 PostgreSQL
- **Memoria**: 1.5GB
- **CPU**: 75% de un núcleo

### 6.2 Wiki.js
- **Memoria**: 512MB
- **CPU**: 50% de un núcleo

Estas limitaciones previenen que un servicio consuma todos los recursos del servidor.

---

## 7. Health Checks

Se implementó un health check para PostgreSQL que verifica:
- **Comando**: `pg_isready -U riwi_user -d riwiwallet_db`
- **Intervalo**: Cada 10 segundos
- **Timeout**: 5 segundos
- **Reintentos**: 5 intentos antes de marcar como unhealthy

Esto asegura que Wiki.js solo inicie cuando la base de datos esté completamente operativa.

---

## 8. Volúmenes y Persistencia

### 8.1 Volúmenes Configurados

1. **riwi_data**: Almacena los datos de PostgreSQL
2. **riwi_wiki_assets**: Almacena los archivos estáticos de Wiki.js

Estos volúmenes garantizan que los datos persistan incluso si los contenedores se eliminan.

---

## 9. Próximos Pasos

- [ ] Configurar backups automáticos de la base de datos
- [ ] Implementar monitoreo con Prometheus/Grafana
- [ ] Configurar firewall (UFW) para restringir acceso
- [ ] Documentar procedimientos de rollback
- [ ] Implementar CI/CD para despliegues automatizados

---

## 10. Recursos y Referencias

- [Docker Documentation](https://docs.docker.com/)
- [Wiki.js Documentation](https://docs.requarks.io/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Nginx Documentation](https://nginx.org/en/docs/)
- [Certbot Documentation](https://certbot.eff.org/docs/)

---

**Documento creado por**: DevOps Team  
**Última actualización**: Diciembre 2024