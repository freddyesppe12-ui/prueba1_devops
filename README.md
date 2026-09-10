# Biblioteca-Springboot

Sistema de microservicios para la gestión de una biblioteca, desplegado en AWS (EC2 + Aurora) con un pipeline de CI/CD automatizado vía GitHub Actions.

## 1. Descripción del proyecto y arquitectura

El proyecto está compuesto por los siguientes módulos:

- **eureka** — Servidor de descubrimiento de servicios (Service Discovery), permite que los microservicios se registren y se encuentren entre sí.
- **api-gateway** — Punto de entrada único de la aplicación, enruta las peticiones hacia el microservicio correspondiente.
- **ms-usuarios** — Microservicio encargado de la gestión de usuarios.
- **ms-catalogo** — Microservicio encargado del catálogo de libros/recursos disponibles.
- **ms-recursos** — Microservicio encargado de la gestión de recursos/préstamos.
- **common** — Módulo con clases y utilidades compartidas entre los microservicios.
- **init-multi-db** — Scripts SQL para la creación inicial de las bases de datos y tablas.

Los microservicios ms-usuarios, ms-catalogo y ms-recursos se conectan cada uno a su propia base de datos (`usuarios`, `catalogo`, `recursos`) dentro de un mismo clúster de **Amazon Aurora (MySQL-Compatible)**. Eureka y el API Gateway no tienen base de datos propia.

Toda la infraestructura corre en una instancia **EC2 (Ubuntu 24.04)**, donde cada microservicio se ejecuta como un servicio **systemd** independiente.

## 2. Estrategia de ramificación

Este proyecto trabaja **únicamente con la rama `main`**, sin ramas `develop`, `feature` o `hotfix`.

**Justificación:** al tratarse de un proyecto académico individual, de alcance acotado y con un único desarrollador trabajando sobre el código en un periodo corto, mantener múltiples ramas agregaba complejidad de gestión. Sumado a lo dicho en clases, que solamente usemos rama main

## 3. Convenciones de commits

Se utiliza un estilo inspirado en *Conventional Commits*, con prefijos que indican el tipo de cambio:

| Prefijo | Uso |
|---|---|
| `chore:` | Tareas de mantenimiento, estructura inicial, configuración |
| `ci:` | Cambios relacionados al pipeline de CI/CD |
| `fix:` | Corrección de errores |
| `docs:` | Cambios en documentación (README, comentarios) |
| `feat:` | Nuevas funcionalidades |

Ejemplos usados en este repositorio:
- `chore: estructura inicial del proyecto`
- `ci: agrega pipeline de build y deploy con systemd a EC2`
- `fix: agrega permiso de ejecucion a mvnw en el workflow`
- `fix: se elimina modulo ms-vehiculos que no pertenece a este proyecto`

## 4. Estructura de carpetas

```
Biblioteca-Springboot/
├── api-gateway/
├── eureka/
├── ms-usuarios/
├── ms-catalogo/
├── ms-recursos/
├── common/
├── init-multi-db/
│   ├── 00-create_dbs.sql
│   ├── 01-usuarios.sql
│   ├── 02-catalogo.sql
│   └── 03-recursos.sql
├── .github/
│   └── workflows/
│       └── ci.yml
├── pom.xml
└── README.md
```

## 5. Flujo de trabajo colaborativo

Comandos usados en el ciclo normal de trabajo:

```bash
git add .
git commit -m "tipo: descripcion breve del cambio"
git push origin main
```

Para revisar el estado y el historial:

```bash
git status
git log --oneline
```

Al ser un único desarrollador sobre `main`, no se usan Pull Requests; cada commit se sube directo y el pipeline de GitHub Actions valida automáticamente que el build y el despliegue sigan funcionando.

## 6. Pipeline CI/CD

El workflow (`.github/workflows/ci.yml`) se dispara automáticamente con cada `push` a `main` y ejecuta:

1. **Checkout** del código del repositorio.
2. **Configuración de JDK 21** (Temurin).
3. **Compilación** del proyecto con Maven (`./mvnw clean package -DskipTests`).
4. **Copia de los `.jar` compilados** a la instancia EC2 vía SCP (`appleboy/scp-action`).
5. **Actualización y reinicio de los servicios systemd** en la EC2 vía SSH (`appleboy/ssh-action`): reemplaza cada `.jar` en `/home/ubuntu/micros/` y reinicia los servicios en orden (`eureka` → `ms-usuarios` → `ms-catalogo` → `ms-recursos` → `api-gateway`), con pausas entre cada uno para dar tiempo de registro en Eureka.

Las credenciales de acceso a la EC2 (`EC2_HOST`, `EC2_USER`, `EC2_SSH_KEY`) están guardadas como **Secrets** en GitHub y nunca quedan expuestas en el código ni en los logs del pipeline.

## 7. Infraestructura y despliegue

- **Base de datos:** Amazon Aurora (MySQL-Compatible), clúster `biblioteca-aurora`, con tres bases (`usuarios`, `catalogo`, `recursos`) creadas mediante los scripts de `init-multi-db`.
- **Servidor de aplicación:** instancia EC2 Ubuntu 24.04, con Java 21 instalado.
- **Seguridad de red:** security groups configurados para que la EC2 pueda conectarse a Aurora por el puerto 3306, y para exponer únicamente los puertos necesarios (Eureka, Gateway, microservicios).
- **Gestión de procesos:** cada microservicio corre como un servicio `systemd` independiente (`eureka.service`, `ms-usuarios.service`, `ms-catalogo.service`, `ms-recursos.service`, `api-gateway.service`), con reinicio automático ante fallos (`Restart=on-failure`).

**Manejo de credenciales:** las credenciales de conexión a Aurora (`DB_URL`, `DB_USERNAME`, `DB_PASSWORD`) **nunca se escriben en el código ni suben a GitHub**. Los microservicios usan un perfil `prod` en `application.yml` que las lee desde variables de entorno:

```yaml
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

Estas variables se definen directamente en los archivos `.service` de systemd en la EC2 (sección `Environment=`), por lo que viven únicamente en el servidor y no forman parte del pipeline de despliegue ni del repositorio.

## 8. Declaración de uso de IA

Se usó Claude para la redacción de este README, depuración de errores del pipeline de GitHub Actions, explicación de conceptos de systemd/Aurora, y un montón de otros conceptos que se desconocían.

## 9. Reflexiones individuales

Personalmente entendí mejor cómo funciona el tema de CI/CD, entendí mejor cómo funcionan los commits, los git push, git add (prácticamente no los conocía). Me costó mucho si hilar las ideas vistas en clases, es un montón de información que jamás había visto y de golpe, entonces era muy complejo armar el rompecabezas en mi mente para tratar de entender todo y poder avanzar en la prueba. A pesar de ver las clases para poder avanzar, seguía complicandome un poco todo.