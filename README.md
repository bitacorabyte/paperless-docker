# Paperless-ngx con Docker & Traefik

Despliegue automatizado y seguro de **Paperless-ngx** (gestor documental) utilizando Docker Compose, integrado con **Redis** como gestor de caché/cola de tareas y enrutado mediante **Traefik** como proxy reverso.

---

## 📋 Precondiciones

Antes de desplegar este contenedor, asegúrate de cumplir con los siguientes requisitos en tu host Docker:

1. **Docker y Docker Compose** instalados en el sistema.
2. **Redes externas creadas**: Este `docker-compose.yml` utiliza dos redes externas que deben existir previamente en tu entorno Docker:
   - `priv-service`: Red privada para la comunicación interna entre servicios y bases de datos.
   - `traefik`: Red pública/interna a través de la cual Traefik gestiona el tráfico web.
   
   Puedes crearlas ejecutando:
   ```bash
   docker network create priv-service
   docker network create traefik
   ```
3. **Base de datos externa**: El archivo de configuración asume el uso de una base de datos externa (como PostgreSQL o MariaDB) accesible mediante las variables de entorno definidas (puedes levantar una base de datos en tu stack o usar una ya existente).
4. **Proxy Reverso Traefik**: Tener operativo un contenedor de Traefik configurado para escuchar en la red `traefik`.

---

## 📂 Volúmenes y Persistencia

El contenedor mapea los siguientes directorios locales del host para garantizar la persistencia de datos y la importación de documentos:

* `/opt/docker/paperless/data`: Base de datos SQLite interna (si aplica) y metadatos del sistema.
* `/opt/docker/paperless/media`: Almacenamiento de los documentos PDF procesados y sus previsualizaciones.
* `/opt/docker/paperless/export`: Directorio para copias de seguridad e importación/exportación masiva.
* `/opt/docker/paperless/consume`: **Carpeta de consumo automático**. Deposita aquí tus documentos escaneados o en PDF para que Paperless los importe de forma automática.

---

## 🔒 Consideraciones de Seguridad

Al desplegar aplicaciones de gestión documental que contienen información sensible o personal, es vital tener en cuenta las siguientes medidas de seguridad:

1. **Credenciales por Defecto**: Nunca dejes las credenciales de administración por defecto (`PAPERLESS_ADMIN_USER` y `PAPERLESS_ADMIN_PASSWORD`). Cámbialas inmediatamente en tu archivo `.env`.
2. **Uso de Variables de Entorno (`.env`)**: No expongas contraseñas ni claves de API directamente en el `docker-compose.yml`. Utiliza siempre un archivo `.env` protegido con permisos restrictivos (`chmod 600 .env`).
3. **Aislamiento en Redes**: El contenedor de Redis se encuentra exclusivamente en la red privada (`priv-service`) sin puertos expuestos al exterior, evitando accesos no autorizados. Paperless se comunica con Redis de forma interna.
4. **Exposición mediante Traefik**: 
   - El puerto interno `8000` de Paperless está comentado (`# ports: ...`) para **impedir el acceso directo** sin pasar por el proxy.
   - Todo el tráfico pasa obligatoriamente por Traefik (`traefik.http.routers.paperless.rule=Host(\`paperless.home.arpa\`)`). 
   - **Recomendación**: Es altamente aconsejable habilitar certificados TLS/SSL en Traefik (HTTPS) en lugar de utilizar texto plano (`entrypoints=web`) si el servicio es accesible fuera de una red local de confianza.

---

## ⚙️ Configuración de Etiquetas (Labels) de Traefik

El servicio de Paperless incluye las siguientes etiquetas (*labels*) de Docker para que Traefik detecte y enrute el tráfico automáticamente:

* `traefik.enable=true`: Habilita la gestión de este contenedor por parte de Traefik.
* `traefik.http.routers.paperless.rule=Host(\`paperless.home.arpa\`)`: Define el dominio o subdominio por el cual se accederá a la aplicación (en este ejemplo, `paperless.home.arpa`).
* `traefik.http.routers.paperless.entrypoints=web`: Especifica el punto de entrada (puerto de escucha) configurado en Traefik.
* `traefik.http.services.paperless.loadbalancer.server.port=8000`: Indica a Traefik que debe redirigir el tráfico hacia el puerto interno `8000` del contenedor de Paperless.
* `traefik.docker.network=traefik`: Fuerza a Traefik a utilizar la red Docker especificada para comunicarse con el contenedor.

---

## 🚀 Puesta en Marcha

1. Clona este repositorio o crea la estructura de directorios en tu servidor.
2. Crea un archivo `.env` en la misma ruta basándote en las variables requeridas (puedes consultar la documentación oficial de Paperless-ngx para ver todas las variables de entorno compatibles).
3. Asegúrate de que las redes de Docker existen.
4. Ejecuta el despliegue con el siguiente comando:

```bash
docker compose up -d
```

5. Accede a tu navegador a través de la URL configurada en Traefik (ej: `http://paperless.home.arpa` o la ruta segura que hayas definido).
