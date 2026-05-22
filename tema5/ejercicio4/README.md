# Docker Compose: Escenarios Multicontenedor

## ¿Por qué Docker Compose?

Las aplicaciones reales rara vez corren en un único contenedor. Una app web típica necesita al menos un servidor de aplicaciones y una base de datos; una arquitectura de microservicios puede requerir docenas de servicios independientes. Levantar y conectar todo eso a mano es tedioso y propenso a errores.

**Docker Compose** resuelve esto permitiendo declarar toda la infraestructura en un fichero `docker-compose.yaml` y gestionarla con un único comando. Sus principales ventajas:

- **Declarativo**: el fichero describe el estado deseado, no los pasos para llegar a él.
- **Orden de arranque garantizado**: `depends_on` asegura que un servicio espera a sus dependencias.
- **Red automática**: Compose crea una red compartida entre contenedores con resolución DNS por nombre de servicio y nombre de contenedor.
- **Ciclo de vida unificado**: levantar, parar, destruir y monitorizar todos los servicios con un solo comando.

> Desde julio de 2023, **Compose V1** (`docker-compose`) está en fin de vida. Usa siempre **Compose V2** integrado en Docker CLI: `docker compose`.

---

## El fichero docker-compose.yaml

Cada aplicación tiene su propio directorio con su `docker-compose.yaml`. Los comandos `docker compose` deben ejecutarse desde ese directorio.

Estructura básica con la aplicación **Let's Chat** como ejemplo:

```yaml
version: '3.1'
services:
  app:
    container_name: letschat
    image: sdelements/lets-chat
    restart: always
    environment:
      LCB_DATABASE_URI: mongodb://mongo/letschat
    ports:
      - 80:8080
    depends_on:
      - db
  db:
    container_name: mongo
    image: mongo:4
    restart: always
    volumes:
      - mongo:/data/db
volumes:
  mongo:
```

Parámetros clave:
| Parámetro | Función |
|-----------|---------|
| `services` | Define cada contenedor del escenario |
| `restart: always` | Reinicia el contenedor si se detiene inesperadamente |
| `depends_on` | Garantiza el orden de arranque entre servicios |
| `volumes` | Monta volúmenes o bind mounts en el contenedor |

---

## Comandos esenciales

| Comando | Descripción |
|---------|-------------|
| `docker compose up -d` | Crea y arranca los contenedores en segundo plano |
| `docker compose ps` | Lista el estado de los contenedores del escenario |
| `docker compose stop` | Detiene los contenedores sin eliminarlos |
| `docker compose start` | Reanuda contenedores parados |
| `docker compose restart` | Reinicia los servicios (útil tras cambios de config) |
| `docker compose pause / unpause` | Pausa/reanuda los contenedores |
| `docker compose logs -f` | Muestra los logs en tiempo real |
| `docker compose exec <svc> bash` | Abre una shell en el contenedor del servicio |
| `docker compose rm` | Elimina los contenedores parados |
| `docker compose down` | Para y elimina contenedores y redes |
| `docker compose down -v` | Ídem, incluyendo los volúmenes |
| `docker compose build` | Construye las imágenes definidas con `Dockerfile` |
| `docker compose top` | Muestra los procesos activos en cada contenedor |

### Ciclo de vida de Let's Chat

```bash
# Arrancar
$ docker compose up -d
[+] Running 4/4
 ✔ Network letschat_default         Created   0.1s
 ✔ Volume "letschat_mongo_letschat" Created   0.0s
 ✔ Container mongo                  Started   0.3s
 ✔ Container letschat               Started   0.2s

# Verificar
$ docker compose ps
NAME       IMAGE                  COMMAND        SERVICE   PORTS
letschat   sdelements/lets-chat   "npm start"    app       0.0.0.0:80->8080/tcp
mongo      mongo:4                "mongod"       db        27017/tcp

# Destruir
$ docker compose down
[+] Running 3/3
 ✔ Container letschat       Removed   10.4s
 ✔ Container mongo          Removed    0.4s
 ✔ Network letschat_default Removed    0.1s
```

---

## Almacenamiento

### Volúmenes Docker

```yaml
version: '3.1'
services:
  db:
    container_name: contenedor_mariadb
    image: mariadb
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: asdasd
    volumes:
      - mariadb_data:/var/lib/mysql
volumes:
  mariadb_data:
```

Compose crea automáticamente el volumen `mariadb_mariadb_data`. Para destruirlo junto con el escenario:

```bash
$ docker compose down -v
```

### Bind Mount

```yaml
volumes:
  - ./data:/var/lib/mysql
```

El directorio `./data` se crea automáticamente al arrancar. **Importante**: `docker compose down -v` no elimina directorios de bind mount, solo volúmenes Docker.

---

## Ejemplos prácticos

### Ejemplo 1 – Guestbook (app Python + Redis)

```yaml
version: '3.1'
services:
  app:
    container_name: guestbook
    image: iesgn/guestbook
    restart: always
    environment:
      REDIS_SERVER: redis
    ports:
      - 8080:5000
  db:
    container_name: redis
    image: redis
    restart: always
    command: redis-server --appendonly yes
    volumes:
      - redis:/data
volumes:
  redis:
```

> `REDIS_SERVER` puede ser el nombre del contenedor (`redis`) o el del servicio (`db`), ya que Compose resuelve ambos por DNS.

---

### Ejemplo 2 – Temperaturas (frontend + backend)

```yaml
version: '3.1'
services:
  frontend:
    container_name: temperaturas-frontend
    image: iesgn/temperaturas_frontend
    restart: always
    ports:
      - 8081:3000
    environment:
      TEMP_SERVER: temperaturas-backend:5000
    depends_on:
      - backend
  backend:
    container_name: temperaturas-backend
    image: iesgn/temperaturas_backend
    restart: always
```

---

### Ejemplo 3 – WordPress + MariaDB

**Con volúmenes Docker:**

```yaml
version: '3.1'
services:
  wordpress:
    container_name: servidor_wp
    image: wordpress
    restart: always
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: user_wp
      WORDPRESS_DB_PASSWORD: asdasd
      WORDPRESS_DB_NAME: bd_wp
    ports:
      - 80:80
    volumes:
      - wordpress_data:/var/www/html/wp-content
  db:
    container_name: servidor_mysql
    image: mariadb
    restart: always
    environment:
      MYSQL_DATABASE: bd_wp
      MYSQL_USER: user_wp
      MYSQL_PASSWORD: asdasd
      MYSQL_ROOT_PASSWORD: asdasd
    volumes:
      - mariadb_data:/var/lib/mysql
volumes:
  wordpress_data:
  mariadb_data:
```

**Con bind mount** (sustituir los `volumes` de cada servicio):

```yaml
volumes:
  - ./wordpress:/var/www/html/wp-content   # en el servicio wordpress
  - ./mysql:/var/lib/mysql                  # en el servicio db
```

---

### Ejemplo 4 – Tomcat + nginx (proxy inverso)

```yaml
version: '3.1'
services:
  aplicacionjava:
    container_name: tomcat
    image: tomcat:9.0
    restart: always
    volumes:
      - ./sample.war:/usr/local/tomcat/webapps/sample.war:ro
  proxy:
    container_name: nginx
    image: nginx
    ports:
      - 80:80
    volumes:
      - ./default.conf:/etc/nginx/conf.d/default.conf:ro
```

El directorio del proyecto debe contener `sample.war` y `default.conf`. nginx recibe el tráfico en el puerto 80 y lo reenvía a Tomcat internamente.

```bash
$ docker compose up -d
$ docker compose ps
```

Accede a la aplicación desde el navegador en el puerto 80 de tu máquina.
