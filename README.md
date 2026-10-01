# BarrioDigital

BarrioDigital es una plataforma basada en arquitectura de microservicios orientada a la gestión digital de trámites y solicitudes comunales.

La solución separa las distintas responsabilidades del sistema en microservicios independientes, utiliza Apache Kafka para comunicación mediante eventos, bases de datos separadas por servicio, autenticación con Microsoft Entra ID mediante MSAL y exposición de servicios mediante AWS API Gateway.

---
## PASOS PARA LEVANTAR EL SISTEMA
1- Crear una carpeta en el escritorio y luego entrar en ella (se usara para clonar todos los repositorios)
 
2- Clonar el repositorio de infraestructura
Clona únicamente el repositorio /infra que acabas de configurar (es el que contiene tu cerebro de automatización) y accede a su carpeta:
git clone https://github.com/TU_ORGANIZACION_O_USUARIO/infra.git
 luego entrar en la carpeta infra con cd infra

3- Ejecutar el script restaurador
Ejecuta tu script de automatización para que descargue de forma simultánea los otros 8 repositorios del proyecto:
powershell -ExecutionPolicy Bypass -File .\restaurar-workspace.ps1

Este comando clonará automáticamente en la carpeta de tu Workspace los repositorios de frontend-barriodigital, ms-barriodigital-bff, los 5 microservicios de dominio (requests, catalog, notify, report, audit) y la carpeta de documentación /docs.
Paso 5: Levantar toda la infraestructura con un solo comando
Dado que renombraste el archivo maestro a docker-compose.yml en la raíz de /infra (tal como se aprecia en tu captura de pantalla), solo debes ejecutar el comando estándar de Docker:
docker compose up -d


## Arquitectura general

El flujo general de la aplicación es:

```text
Usuario
  ↓
Frontend React
  ↓
Microsoft Entra ID + MSAL
  ↓
Access Token JWT
  ↓
AWS API Gateway
  ↓
Microservicios
  ↓
Bases de datos independientes
```

La aplicación frontend autentica al usuario mediante Microsoft Entra ID utilizando MSAL.

Una vez autenticado, el usuario obtiene un Access Token que se envía en las solicitudes HTTP.

AWS API Gateway recibe las peticiones, valida el token mediante un JWT Authorizer y posteriormente redirige la solicitud al microservicio correspondiente.

---

# Microservicios

La solución se divide en distintos microservicios, cada uno con una responsabilidad específica.

Cada microservicio mantiene su propia base de datos y no accede directamente a la base de datos de otro servicio.

Esto permite mantener independencia entre componentes y facilita la escalabilidad, mantenimiento y evolución del sistema.

---

## Catalog Service

Responsable de la gestión del catálogo de trámites disponibles.

Puerto local:

```text
8081
```

Ruta general:

```text
/api/catalog/*
```

---

## Requests Service

Responsable de la creación y gestión de solicitudes asociadas a los trámites.

Puerto local:

```text
8082
```

Ruta general:

```text
/api/requests/*
```

Cuando una solicitud es creada o cambia de estado, este microservicio publica un evento en Kafka.

Ejemplo conceptual de evento:

```json
{
  "requestId": 3001,
  "procedureId": 101,
  "oldStatus": "NONE",
  "newStatus": "INGRESADO",
  "timestamp": "2026-09-12T19:00:00"
}
```

Este evento puede ser consumido por otros microservicios de forma independiente.

---

## Audit Service

Responsable de registrar la trazabilidad de los cambios realizados sobre las solicitudes.

Puerto local:

```text
8084
```

Base de datos:

```text
PostgreSQL
Neon Tech
```

Audit consume eventos provenientes de Kafka y almacena información relacionada con los cambios de estado de las solicitudes.

Entre los datos registrados se encuentran:

- ID de solicitud
- ID de trámite
- Estado anterior
- Estado nuevo
- Tipo de evento
- Fecha y hora del evento
- Información de usuario cuando corresponde
- Identificadores de correlación

### Endpoints Audit

Obtener todos los eventos:

```http
GET /api/audit
```

Obtener eventos por solicitud:

```http
GET /api/audit/request/{requestId}
```

Obtener eventos por usuario:

```http
GET /api/audit/user/{userId}
```

Obtener eventos por tipo:

```http
GET /api/audit/type/{eventType}
```

Audit también permite realizar consultas relacionadas con rangos de fechas.

### Función principal

Audit permite responder preguntas como:

```text
¿Qué ocurrió con una solicitud?
¿Qué estado tenía anteriormente?
¿A qué estado cambió?
¿Cuándo ocurrió el cambio?
```

Esto permite mantener trazabilidad histórica de los trámites.

---

## Report Service

Responsable de generar métricas y reportería utilizando los eventos generados dentro del sistema.

Puerto local:

```text
8085
```

Base de datos:

```text
PostgreSQL
Neon Tech
```

Report consume eventos desde Kafka y almacena la información necesaria para calcular indicadores.

### Endpoints Report

KPIs:

```http
GET /api/report/kpis?range=last24h
```

También puede utilizar otros rangos implementados:

```http
GET /api/report/kpis?range=last7d
```

Respuesta conceptual:

```json
{
  "range": "last24h",
  "totalEvents": 10,
  "createdRequests": 5,
  "completedRequests": 3
}
```

Trámites más solicitados:

```http
GET /api/report/top-procedures?range=last7d
```

Este endpoint permite obtener la cantidad de solicitudes asociadas a cada procedimiento.

---

# Comunicación entre microservicios

Los microservicios no comparten directamente sus bases de datos.

La comunicación de eventos se realiza mediante Kafka.

Flujo conceptual:

```text
Requests
   │
   │ publica evento
   ▼
 Kafka
   │
   ├───────────────┐
   │               │
   ▼               ▼
 Audit           Report
   │               │
   ▼               ▼
BD Audit        BD Report
```

Cuando Requests genera un cambio:

1. Requests actualiza su propia base de datos.
2. Requests publica un evento en Kafka.
3. Audit consume el evento.
4. Audit registra la trazabilidad en su propia base.
5. Report consume el mismo evento.
6. Report registra información útil para generar métricas.

---

# ¿Por qué existen bases de datos diferentes?

Cada microservicio es propietario de sus propios datos.

Por ejemplo:

```text
Requests
→ mantiene el estado operacional de las solicitudes

Audit
→ mantiene el historial de cambios

Report
→ mantiene información para métricas y análisis
```

Algunos datos pueden existir en más de un servicio, pero con propósitos diferentes.

Esta duplicación controlada permite evitar dependencias directas entre bases de datos.

---

# Apache Kafka

Kafka se utiliza como mecanismo de comunicación asíncrona entre microservicios.

Esto permite que un servicio publique un evento sin conocer directamente qué otros servicios lo consumirán.

En desarrollo local:

```text
Kafka interno Docker:
kafka:9092

Kafka desde Windows:
localhost:29092
```

Kafka UI:

```text
http://localhost:8090
```

Topics utilizados durante el proyecto:

```text
requests.events
audit.timeline
```

---

# Frontend

El frontend está desarrollado utilizando:

```text
React
Vite
Axios
MSAL
Recharts
```

Puerto local aproximado:

```text
http://localhost:5173
```

o:

```text
http://localhost:5174
```

dependiendo de la disponibilidad del puerto.

---

# Pantallas principales

## Catálogo

Ruta:

```text
/catalog
```

Permite visualizar los trámites disponibles.

---

## Solicitudes

Ruta:

```text
/requests
```

Permite gestionar las solicitudes asociadas a los trámites.

---

## Dashboard

Ruta:

```text
/dashboard
```

Consume:

```http
GET /api/report/kpis?range=last24h
```

Muestra indicadores como:

```text
Eventos procesados
Trámites ingresados
Trámites finalizados
```

---

## Reportes

Ruta:

```text
/reports
```

Consume:

```http
GET /api/report/top-procedures?range=last7d
```

Los datos se representan utilizando gráficos generados con Recharts.

---

## Auditoría

Ruta:

```text
/audit
```

Consume:

```http
GET /api/audit
```

y:

```http
GET /api/audit/request/{requestId}
```

Permite visualizar el historial cronológico de cambios asociados a una solicitud.

---

# Microsoft Entra ID y MSAL

La autenticación del frontend se realiza utilizando Microsoft Entra ID.

La integración con React se realiza mediante:

```text
@azure/msal-react
@azure/msal-browser
```

El flujo de autenticación es:

```text
Usuario
  ↓
React
  ↓
MSAL
  ↓
Microsoft Entra ID
  ↓
Inicio de sesión
  ↓
Access Token
```

Una vez autenticado, MSAL mantiene la sesión del usuario y permite obtener tokens de acceso.

---

# Roles

La aplicación contempla roles como:

```text
Admin
Funcionario
Vecino
Auditor
```

Estos roles permiten definir qué funcionalidades puede utilizar cada usuario.

---

# JWT

El Access Token entregado por Microsoft Entra ID utiliza formato JWT.

Este token contiene información como:

```text
Usuario
Issuer
Audience
Fecha de expiración
Roles
Claims
```

El token se envía en cada solicitud protegida:

```http
Authorization: Bearer <access_token>
```

---

# AWS API Gateway

AWS API Gateway funciona como punto de entrada para los microservicios.

Flujo:

```text
Frontend React
      ↓
Access Token
      ↓
AWS API Gateway
      ↓
JWT Authorizer
      ↓
Microservicio correspondiente
```

API Gateway valida el JWT antes de permitir el acceso.

---

# JWT Authorizer

Configuración conceptual:

```text
Issuer:
https://login.microsoftonline.com/{tenant-id}/v2.0

Audience:
{client-id}
```

El Authorizer verifica:

- firma del token
- emisor
- audiencia
- expiración

---

# Rutas API Gateway

Las rutas generales del proyecto son:

```text
/api/catalog/*
/api/requests/*
/api/audit/*
/api/report/*
```

Utilizando rutas proxy:

```text
ANY /api/catalog/{proxy+}
ANY /api/requests/{proxy+}
ANY /api/audit/{proxy+}
ANY /api/report/{proxy+}
```

---

# Integración mediante Ngrok

Durante la etapa de integración, los microservicios locales pueden exponerse mediante Ngrok.

Ejemplo:

```text
Audit
localhost:8084
↓
Ngrok
↓
API Gateway
```

```text
Report
localhost:8085
↓
Ngrok
↓
API Gateway
```

API Gateway utiliza las URLs públicas generadas por Ngrok como integración HTTP.

---

# Flujo completo del sistema

El flujo completo de una operación puede representarse así:

```text
Usuario
  ↓
React
  ↓
MSAL
  ↓
Microsoft Entra ID
  ↓
Access Token JWT
  ↓
AWS API Gateway
  ↓
Requests Service
  ↓
BD Requests
  ↓
Kafka
  ├──────────────────┐
  ↓                  ↓
Audit               Report
  ↓                  ↓
BD Audit            BD Report
  ↓                  ↓
API Audit           API Report
  └─────────┬────────┘
            ↓
          React
```

---

# Flujo de una solicitud

Ejemplo:

```text
1. El usuario inicia sesión.
2. Microsoft Entra ID autentica al usuario.
3. MSAL obtiene un Access Token.
4. El usuario realiza una operación desde React.
5. React envía la solicitud hacia API Gateway.
6. API Gateway valida el JWT.
7. La petición llega a Requests.
8. Requests guarda la solicitud.
9. Requests publica un evento en Kafka.
10. Audit consume el evento.
11. Report consume el evento.
12. Audit registra la trazabilidad.
13. Report actualiza sus métricas.
14. React puede consultar Dashboard, Reportes o Auditoría.
```

---

# Desarrollo local

Durante desarrollo local el frontend puede utilizar el proxy de Vite.

Ejemplo:

```javascript
server: {
  proxy: {
    '/api/report': {
      target: 'http://localhost:8085',
      changeOrigin: true
    },

    '/api/audit': {
      target: 'http://localhost:8084',
      changeOrigin: true
    }
  }
}
```

De esta manera:

```text
React
↓
/api/report/*
↓
localhost:8085
```

y:

```text
React
↓
/api/audit/*
↓
localhost:8084
```

---

# Integración final

En la integración final, el frontend deja de llamar directamente a los puertos locales.

En su lugar utiliza la URL entregada por AWS API Gateway.

Ejemplo conceptual:

```text
https://xxxxxxxx.execute-api.us-east-1.amazonaws.com
```

Por lo tanto:

```text
React
↓
API Gateway
↓
Microservicios
```

---



# Docker

Los microservicios pueden ejecutarse utilizando Docker.

Ejemplo conceptual:


docker build -t ms-barriodigital-report .


Kafka puede encontrarse dentro de la red Docker como:


kafka:9092


Mientras que desde aplicaciones ejecutadas directamente en Windows:

localhost:29092


---

# Persistencia

Audit y Report utilizan PostgreSQL alojado en Neon Tech.

Las credenciales y datos sensibles no deben escribirse directamente en el código fuente.

Las propiedades utilizan variables de entorno:

```properties
spring.datasource.url=${DB_URL}
spring.datasource.username=${DB_USERNAME}
spring.datasource.password=${DB_PASSWORD}
spring.kafka.bootstrap-servers=${KAFKA_BOOTSTRAP_SERVERS:localhost:29092}
```


# Resumen de tecnologías

| Área | Tecnología |
|---|---|
| Frontend | React |
| Build frontend | Vite |
| HTTP Client | Axios |
| Gráficos | Recharts |
| Autenticación | Microsoft Entra ID |
| Cliente de autenticación | MSAL |
| API Gateway | AWS API Gateway |
| Backend | Spring Boot |
| Lenguaje backend | Java 17 |
| Mensajería | Apache Kafka |
| Base de datos | PostgreSQL |
| Base de datos cloud | Neon Tech |
| Contenedores | Docker |
| Túneles de desarrollo | Ngrok |
| Control de versiones | Git + GitHub |


# Responsabilidades principales

## Audit

Trazabilidad
Historial de estados
Auditoría de solicitudes


## Report

KPIs
Métricas
Top de trámites
Reportería


## Requests
Gestión operacional de solicitudes
Publicación de eventos


## Catalog


Gestión de trámites disponibles


# Explicación corta para presentación

Una forma resumida de explicar la arquitectura es:

BarrioDigital utiliza una arquitectura de microservicios donde cada servicio posee una responsabilidad específica y su propia base de datos. Requests administra las solicitudes y publica eventos mediante Kafka. Audit consume estos eventos para mantener la trazabilidad histórica, mientras que Report los utiliza para generar métricas y reportes. El frontend está desarrollado en React y utiliza MSAL con Microsoft Entra ID para autenticar a los usuarios. Las solicitudes protegidas incorporan un Access Token JWT y pasan a través de AWS API Gateway, que valida el token antes de redirigir la petición al microservicio correspondiente.



# Repositorios

La solución se organiza en distintos repositorios asociados a:


Frontend

Catalog Service

Requests Service

Audit Service

Report Service

Infraestructura

Documentación


# Clonación de repositorios

Para poder clonar uno o los 9 repositorios primero se debe crear una carpeta en tu escritorio (con cualquier nombre), después debes abrir PowerShell e ingresar los siguientes comandos:

cd Desktop

cd (el nombre de la carpeta que se ha creado)

git clone https://github.com/(nombre del proyecto)/(repositorio que se quiere clonar).git

docker compose up -d

cd Desktop permite mover o cambiar la ubicicavión 

el git clone como dicta su función clona el o los repositorios con todos sus archivos del proyecto a la nueva carpeta que se había creado con anterioridad, esto puede ser usado ya sea para traspasar el proyecto a un nuevo espacio de trabajo más comodo o tener respaldos por si se llega a perder todo el progreso que se realizo.
