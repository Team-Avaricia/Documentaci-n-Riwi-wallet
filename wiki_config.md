# Documentación de Configuración del Proyecto Wiki

## 1. 📂 Estructura del Directorio Base

Todos los archivos de configuración residen en el directorio principal del proyecto:

```
/home/juan_david/wiki/
├── docker-compose.yaml
├── .env
└── config/
    └── postgresql.conf
```

## 2. 🐳 Configuración de Docker Compose (`docker-compose.yaml`)

Se utiliza una arquitectura de microservicios con una red interna (`riwi-net`).

### **Network y Volumes**

La red interna `riwi-net` aísla los servicios. El volumen `wiki_data` se utiliza para la persistencia de PostgreSQL.

```yaml
networks:
  riwi-net:
    driver: bridge

volumes:
  wiki_data:
```

---

### **Servicio `wiki-db` (PostgreSQL)**

Configurado con un límite de memoria de 1.5GB y persistencia.

| Parámetro | Valor | Razón de la Configuración |
|----------|--------|---------------------------|
| POSTGRES_USER | riwiwiki | Usuario de BD para Wiki.js. |
| POSTGRES_DB | riwi_wiki_db | Nombre de la BD. |
| POSTGRES_PASSWORD | ${WIKI_DB_PASSWORD} | Obtenida del `.env` (requiere comillas simples si contiene `$`). |
| volumes | ./config/postgresql.conf | Se monta el archivo de configuración personalizado. |
| healthcheck | pg_isready | Asegura que la BD esté lista antes de que Wiki.js se conecte. |

#### **Archivo `postgresql.conf` (Modificación Clave)**

Para permitir la conexión desde la red interna de Docker:

```
listen_addresses = '*'
```

---

### **Servicio `wiki` (Wiki.js)**

Configurado para el acceso seguro mediante Reverse Proxy.

| Parámetro | Valor | Razón |
|----------|--------|-------|
| ports | "127.0.0.1:3000:3000" | Limita el acceso interno del VPS para Nginx. |
| DB_HOST | wiki-db | Nombre del servicio Docker. |
| WIKI_HOST | 0.0.0.0 | Permite escuchar en todas las interfaces internas del contenedor. |
| WIKI_BASE_URL | https://docs.avaricia.crudzaso.com | Evita errores de `Connection reset`. |
| command | Script con `netcat` | Soluciona el race condition en el arranque esperando a que la BD esté lista. |

---

## 3. ⚙️ Configuración de Nginx (Reverse Proxy)

La configuración de Nginx permite recibir tráfico HTTP/HTTPS y redirigirlo hacia el contenedor de Wiki.js.

| Configuración | Valor | Descripción |
|---------------|--------|-------------|
| Archivo | `/etc/nginx/sites-available/wiki.conf` | Enlace simbólico a `sites-enabled`. |
| server_name | docs.avaricia.crudzaso.com | Dominio manejado por Nginx. |
| proxy_pass | http://127.0.0.1:3000 | Redirección al contenedor Wiki.js. |
| Certificado | Let's Encrypt | HTTPS automático con redirección desde HTTP. |

---

