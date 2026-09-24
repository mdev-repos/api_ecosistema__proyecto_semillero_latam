<div align="center">

# 🌱 ECOSistema API

![Java](https://img.shields.io/badge/Java-17-blue)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.3.1-brightgreen)
![MariaDB](https://img.shields.io/badge/MariaDB-BBDD-003545)
![JWT](https://img.shields.io/badge/Auth-JWT%20%2B%20OAuth2-yellow)
![Docker](https://img.shields.io/badge/Docker-Dockerfile-2496ED)
![Status](https://img.shields.io/badge/Estado-Simulación%20finalizada-lightgrey)

API REST para **ECOSistema**, una plataforma que conecta productores y prestadores de servicios con impacto ambiental positivo con personas comprometidas con el cuidado del medioambiente. Los proveedores publican sus productos, un equipo de administradores los modera, y cualquier visitante puede descubrirlos — incluso ordenados por cercanía geográfica.

</div>

<br/>

## Índice
- [Contexto del proyecto](#contexto-del-proyecto)
- [Mi rol en el equipo](#mi-rol-en-el-equipo)
- [Stack técnico](#stack-técnico)
- [Arquitectura](#arquitectura)
- [Módulos y funcionalidades](#módulos-y-funcionalidades)
- [Cómo correrlo en local](#cómo-correrlo-en-local)
- [Configuración](#configuración)
- [Autor](#autor)

<br/>

## Contexto del proyecto

Este backend se desarrolló durante **Semillero LATAM**, la propuesta formativa práctica de **Quinto Impacto** (Mendoza) — un programa de simulación laboral real que hoy ya no está activo. Fue mi primera experiencia de desarrollo colaborativo (2024): arrancamos siendo un equipo de 8 personas y terminamos trabajando 5-6 hasta el cierre del proyecto, con 4 desarrolladores en el equipo de backend.

Trabajamos bajo metodología **SCRUM**, organizando el trabajo por sprints en un tablero **Trello** al estilo **Kanban** para el seguimiento de tareas.

<br/>
<hr>
<br/>

## Mi rol en el equipo

Con el correr del proyecto terminé ejerciendo, de forma no oficial, como referente del equipo de backend. En lo técnico, mi aporte más específico fue el **sistema de envío de emails automatizados**: investigué la documentación de Spring Mail y Thymeleaf de forma autónoma para implementarlo de punta a punta — [`EmailSenderService`](./ecosistemas/ecosistemas/src/main/java/com/semillero/ecosistemas/service/EmailSenderService.java) arma y envía un resumen semanal en HTML (con logo embebido) a administradores y proveedores registrados en la newsletter, usando **Thymeleaf** para construir el cuerpo del correo y **JavaMailSender** para el envío, disparado por [`ScheduledEmailService`](./ecosistemas/ecosistemas/src/main/java/com/semillero/ecosistemas/service/ScheduledEmailService.java).

<br/>
<hr>
<br/>

## Stack técnico

| Categoría | Tecnología |
|---|---|
| Lenguaje | Java 17 |
| Framework | Spring Boot 3.3.1 (Web, Data JPA, Security, Validation) |
| Base de datos | MariaDB |
| Autenticación | OAuth2 (Google Login) + JWT (jjwt) |
| Emails automatizados | Spring Mail (JavaMailSender) + Thymeleaf |
| Imágenes | Cloudinary |
| Geolocalización | Nominatim (OpenStreetMap) — cálculo de distancias |
| Documentación de API | Springdoc OpenAPI (Swagger UI) |
| Build | Maven |
| Reducción de boilerplate | Lombok |
| Contenedores | Docker |

<br/>
<hr>
<br/>

## Arquitectura

Arquitectura en capas clásica de Spring Boot:

```
Cliente (JSON / multipart)
   │
   ▼
Controller     → recibe la request, valida autorización por rol (@PreAuthorize)
   │
   ▼
Service        → lógica de negocio (moderación, matching por cercanía, envío de mails)
   │
   ▼
Repository (Spring Data JPA)
   │
   ▼
MariaDB
```

**Autenticación**: login con Google (OAuth2) → `AuthenticationService` da de alta o recupera al usuario → se emite un JWT propio (`JwtService`) que protege el resto de los endpoints mediante `JwtAuthenticationFilter`, con autorización por rol (`ADMIN` / `SUPPLIER`) a nivel de método.

**Geolocalización**: `DistanceService` toma la ubicación del visitante, geocodifica la provincia/país de cada producto contra la API de **Nominatim** (`GeocodingService`) y calcula la distancia con `DistanceCalculator`, devolviendo los 5 productos más cercanos.

<br/>
<hr>
<br/>

## Módulos y funcionalidades

**Usuarios y roles**
- `Admin`: modera productos, gestiona catálogos y administradores.
- `Supplier`: se registra, publica hasta 3 productos con imágenes, y puede editar/eliminar los propios.
- Visitante sin login: navega el catálogo público de productos aceptados y las publicaciones.

**Productos y flujo de moderación**

Cada producto que carga un proveedor pasa por un circuito de revisión antes de quedar visible al público:

```
REVISION_INICIAL → (Admin revisa) ─┬─▶ ACEPTADO
                                     ├─▶ DENEGADO
                                     └─▶ REQUIERE_CAMBIOS → Supplier corrige → CAMBIOS_REALIZADOS → (vuelve a revisión)
```

El admin puede dejar feedback textual en cada cambio de estado, y solo los productos `ACEPTADO` son visibles públicamente.

**Publicaciones**: contenido tipo blog/noticias, con conteo de vistas — la base del ranking de "más vistas" que se ve en el dashboard.

**Geolocalización**: búsqueda de los productos más cercanos a la ubicación del visitante (ver [Arquitectura](#arquitectura)).

**Preguntas frecuentes**: catálogo de preguntas y respuestas organizadas por categoría, con CRUD completo (documentado en el código como `ChatBot`, aunque funcionalmente es un módulo de FAQ, no un chatbot conversacional).

**Dashboard administrativo**: métricas para el equipo de administradores — productos nuevos por estado, proveedores por categoría, últimas 5 publicaciones y las 5 más vistas.

**Emails automáticos**: resumen semanal por correo a administradores (productos nuevos/pendientes de revisión) y proveedores (productos propios aceptados) — ver [Mi rol en el equipo](#mi-rol-en-el-equipo).

**Imágenes**: subida de hasta 3 imágenes por producto a Cloudinary.

**Documentación de API**: Swagger UI autogenerado con springdoc-openapi sobre todos los controllers.

<br/>
<hr>
<br/>

## Cómo correrlo en local

Prerrequisitos: JDK 17, Maven, MariaDB corriendo en local (o Docker para levantar solo la app).

```bash
# 1. Crear la base de datos en MariaDB
# 2. Configurar las variables de entorno (ver tabla de Configuración)
# 3. Levantar la aplicación
cd ecosistemas/ecosistemas
./mvnw spring-boot:run
```

O buildeando la imagen con el Dockerfile incluido:

```bash
cd ecosistemas/ecosistemas
./mvnw clean package -DskipTests
docker build -t ecosistema-api .
docker run -p 8080:8080 --env-file .env ecosistema-api
```

<br/>
<hr>
<br/>

## Configuración

Ninguna credencial vive en el código: `application.properties` lee todo desde variables de entorno.

| Variable | Uso |
|---|---|
| `SERVER_PORT` | Puerto del servidor |
| `DB_URL` / `DB_USER` / `DB_PASSWORD` | Conexión a MariaDB |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | Login con Google (OAuth2) |
| `JWT_SECRET` / `JWT_EXPIRATION` | Firma y expiración del token propio |
| `CLOUDINARY_API_KEY` / `CLOUDINARY_API_SECRET` / `CLOUDINARY_CLOUD_NAME` | Subida de imágenes |
| `EMAIL_USER` / `EMAIL_PASSWORD` | Envío de emails automáticos (SMTP Gmail) |

La geolocalización usa la API pública de Nominatim y no requiere API key.

<br/>
<hr>
<br/>

## Autor

**Matías Mazzitelli** — Backend Developer (Java / Spring Boot)

[GitHub](https://github.com/mdev-repos) · [LinkedIn](https://www.linkedin.com/in/mnm-dev)
