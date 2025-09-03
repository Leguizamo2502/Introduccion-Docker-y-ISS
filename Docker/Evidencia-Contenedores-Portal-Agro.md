# Evidencia de Contenedores – Portal Agro‑comercial del Huila

> **Alumno:** Sergio Andrés Leguizamo Vargas  
> **Fecha:** 3 de septiembre de 2025  
> **Entregable:** Dockerfiles (backend y frontend), `nginx.conf`, `docker-compose.yml` y video de pruebas

---

## 1. Objetivo
Documentar la contenedorización del proyecto (backend .NET 8 y frontend Angular 17/18/19 con NGINX), describiendo configuración, comandos de build/run, verificación con healthchecks y pruebas ejecutadas en video.

---

## 2. Requisitos previos
- Docker Desktop o Docker Engine + Docker Compose v2
- Puertos libres en el host: **1433**, **5433**, **3307**, **8080**, **4300**
- Espacio en disco para volúmenes: `sql_data`, `postgres_data`, `mysql_data`

---

## 3. Estructura relevante del proyecto
```
.
├─ Portal-Agro-comercial-del-Huila/
│  ├─ Web/               # API .NET 8 (Dockerfile backend)
│  ├─ Business/
│  ├─ Custom/
│  ├─ Data/
│  ├─ Entity/
│  └─ Utilities/
├─ Frontend/
│  └─ Portal-Agro/
│     ├─ Dockerfile      # Dockerfile frontend
│     └─ nginx.conf
└─ docker-compose.yml
```

---

## 4. Backend – Dockerfile (`Portal-Agro-comercial-del-Huila/Web/Dockerfile`)

```dockerfile
# Consulte https://aka.ms/customizecontainer para aprender a personalizar su contenedor de depuración y cómo Visual Studio usa este Dockerfile para compilar sus imágenes para una depuración más rápida.

# Esta fase se usa cuando se ejecuta desde VS en modo rápido (valor predeterminado para la configuración de depuración)
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
USER $APP_UID
WORKDIR /app
EXPOSE 8080
#EXPOSE 8081

# Esta fase se usa para compilar el proyecto de servicio
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src

COPY Portal-Agro-comercial-del-Huila/Web/Web.csproj Web/
COPY Portal-Agro-comercial-del-Huila/Business/Business.csproj Business/
COPY Portal-Agro-comercial-del-Huila/Custom/Custom.csproj Custom/
COPY Portal-Agro-comercial-del-Huila/Data/Data.csproj Data/
COPY Portal-Agro-comercial-del-Huila/Entity/Entity.csproj Entity/
COPY Portal-Agro-comercial-del-Huila/Utilities/Utilities.csproj Utilities/
#COPY Portal-Agro-comercial-del-Huila/Portal-Agro.sln .
COPY . .

RUN dotnet restore "./Web/Web.csproj"

# Copiar resto del código
COPY Portal-Agro-comercial-del-Huila/ .

WORKDIR "/src/Web"
RUN dotnet build "./Web.csproj" -c $BUILD_CONFIGURATION -o /app/build

# Esta fase se usa para publicar el proyecto de servicio que se copiará en la fase final.
FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "./Web.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

# Esta fase se usa en producción o cuando se ejecuta desde VS en modo normal (valor predeterminado cuando no se usa la configuración de depuración)
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "Web.dll"]
```

**Notas técnicas del backend**
- Expone **8080** y fija `ASPNETCORE_URLS=http://+:8080` desde `docker-compose.yml`.
- Se incluyen múltiples proyectos para restaurar dependencias y compilar la solución.
- Publica self-contained deshabilitado (`UseAppHost=false`) para reducir tamaño.

---

## 5. Frontend – Dockerfile (`Frontend/Portal-Agro/Dockerfile`)

```dockerfile
# Build Angular
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build -- --configuration=production

# Runtime NGINX
FROM nginx:alpine
# Sobrescribe el default server
COPY nginx.conf /etc/nginx/conf.d/default.conf
# COPIA DESDE /browser (clave)
COPY --from=build /app/dist/portal-agro/browser /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**Razón del `.../browser`**: En Angular moderno (builder `@angular-devkit/build-angular:application`), el `outputPath` genera `dist/<app>/browser`. NGINX debe servir esa carpeta.

---

## 6. NGINX – `nginx.conf` (en `Frontend/Portal-Agro/nginx.conf`)

```nginx
server {
  listen 80;
  server_name localhost;

  root /usr/share/nginx/html;
  index index.html;

  location / {
    try_files $uri $uri/ /index.html;
  }

  error_page 404 /index.html;
}
```

**Notas técnicas del frontend**
- `try_files` enruta SPA al `index.html` para rutas de Angular.
- Se usa `default.conf` sobrescrito en `/etc/nginx/conf.d/`.

---

## 7. Orquestación – `docker-compose.yml` (raíz del repositorio)

```yaml
services:
  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: sqlserver
    ports: ["1433:1433"]
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=Admin123.
    volumes:
      - sql_data:/var/opt/mssql
    healthcheck:
      test: ["CMD-SHELL", "/opt/mssql-tools18/bin/sqlcmd -C -S localhost -U SA -P $$SA_PASSWORD -Q "SELECT 1" || exit 1"]
      interval: 5s
      timeout: 3s
      retries: 30
      start_period: 20s

  postgres:
    image: postgres:16-alpine
    container_name: postgres
    environment:
      - POSTGRES_DB=PortalAgro
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=Admin123.
    ports: ["5433:5432"]
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d PortalAgro"]
      interval: 5s
      timeout: 3s
      retries: 30
      start_period: 10s

  mysql:
    image: mysql:8.3
    container_name: mysql
    environment:
      - MYSQL_ROOT_PASSWORD=Admin123.
      - MYSQL_DATABASE=Port
    ports: ["3307:3306"]
    volumes:
      - mysql_data:/var/lib/mysql
    command: ["mysqld", "--default-authentication-plugin=mysql_native_password"]
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h localhost -uroot -p$$MYSQL_ROOT_PASSWORD || exit 1"]
      interval: 5s
      timeout: 3s
      retries: 30
      start_period: 15s

  backend:
    build:
      context: .
      dockerfile: Portal-Agro-comercial-del-Huila/Web/Dockerfile
    container_name: backend
    ports: ["8080:8080"]
    environment:
      - ASPNETCORE_URLS=http://+:8080
      - ConnectionStrings__SqlServer=Server=sqlserver,1433;Database=PortalAgro;User Id=sa;Password=Admin123.;Encrypt=False;TrustServerCertificate=True
      - ConnectionStrings__Postgres=Host=postgres;Port=5432;Database=PortalAgro;Username=postgres;Password=Admin123.
      - ConnectionStrings__MySql=Server=mysql;Port=3306;Database=PortalAgro;User ID=root;Password=Admin123.;SslMode=Preferred;AllowPublicKeyRetrieval=True;CharSet=utf8mb4;DefaultCommandTimeout=60;Connection Timeout=60;Pooling=true;MinimumPoolSize=0;MaximumPoolSize=50;
      # toggles por entorno
      - MIGRATE_SQLSERVER=true
      - MIGRATE_POSTGRES=true
      - MIGRATE_MYSQL=true
    depends_on:
      sqlserver:
        condition: service_healthy
      postgres:
        condition: service_healthy
      mysql:
        condition: service_healthy

  frontend:
    build:
      context: ./Frontend/Portal-Agro
      dockerfile: Dockerfile
    container_name: frontend
    ports: ["4300:80"]
    depends_on:
      - backend

volumes:
  sql_data:
  postgres_data:
  mysql_data:
```

**Puntos clave del `compose`**
- Healthchecks en **SQL Server**, **PostgreSQL** y **MySQL** para arranque ordenado.
- El backend espera a las BBDD (via `depends_on.condition: service_healthy`).  
- Conexiones internas por nombre de servicio (`sqlserver`, `postgres`, `mysql`) y puertos del contenedor.

---

## 8. Comandos para construir y ejecutar

```bash
# 1) Construcción sin cache
docker compose build --no-cache

# 2) Levantar en segundo plano
docker compose up -d

# 3) Ver estado/health
docker compose ps

# 4) Ver logs de un servicio (ejemplos)
docker compose logs -f backend
docker compose logs -f sqlserver

# 5) Parar y eliminar (manteniendo volúmenes)
docker compose down

# 6) Parar y eliminar TODO (incluye volúmenes)
docker compose down -v
```

---

## 9. Verificación rápida de servicios
- **Backend API**: `http://localhost:8080` (endpoints de salud o Swagger si está habilitado).  
- **Frontend Angular**: `http://localhost:4300` (la SPA debe cargar; rutas internas deben funcionar por `try_files`).  
- **BBDD**:
  - SQL Server (host): `localhost,1433` (usuario `sa` / pass `Admin123.`)
  - PostgreSQL (host): `localhost:5433` (usuario `postgres` / pass `Admin123.`)
  - MySQL (host): `localhost:3307` (usuario `root` / pass `Admin123.`)

---


## 10. Pruebas en video
Evidencia de ejecución y pruebas funcionales en contenedores:  
**Video:** https://youtu.be/3ZY-uEP5gF4

---



## 11. Conclusión
La solución contenedorizada permite levantar entorno completo (tres BBDD para pruebas, API .NET 8 y SPA Angular con NGINX) mediante un único `docker compose up -d`, con healthchecks que garantizan el orden de arranque y configuración estándar para desarrollo local.
