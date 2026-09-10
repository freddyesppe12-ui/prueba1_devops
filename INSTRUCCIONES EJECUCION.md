# Instrucciones de ejecución — Proyecto Biblioteca (Spring Boot + MySQL)

Guía paso a paso para dejar el proyecto corriendo en VS Code usando **MySQL** como base de datos.

---

## 1. Requisitos previos

Instala esto antes de empezar:

1. **JDK 21** (`java -version` debe mostrar 21).
2. **MySQL Server 8.x** (necesario por los `CHECK` de las tablas, requieren MySQL 8.0.16+).
3. **MySQL Workbench** (o el cliente `mysql` de línea de comandos) para crear las bases de datos.
4. **Visual Studio Code**, con estas extensiones:
   - `Extension Pack for Java` (Microsoft)
   - `Spring Boot Extension Pack` (VMware/Microsoft)

---

## 2. Configurar la contraseña de MySQL

El proyecto usa por defecto:
- Usuario: `root`
- Contraseña: `alone15`

Si tu MySQL local tiene otra contraseña de `root`, ábrela y cámbiala en estos 3 archivos (busca la línea `password:`):

- `ms-usuarios/src/main/resources/application.yml`
- `ms-catalogo/src/main/resources/application.yml`
- `ms-recursos/src/main/resources/application.yml`

---

## 3. Crear las bases de datos y las tablas

Los scripts SQL ya están listos en la carpeta `init-multi-db/`. Debes ejecutarlos **en este orden exacto**:

1. `00-create_dbs.sql` → crea todas las bases de datos.
2. `01-usuarios.sql` → crea las tablas del microservicio `usuarios` + datos de prueba.
3. `02-catalogo.sql` → crea las tablas del microservicio `catalogo` + datos de prueba.
4. `03-recursos.sql` → crea las tablas del microservicio `recursos` + datos de prueba.

### Opción A — MySQL Workbench (recomendada, más simple)

1. Abre MySQL Workbench y conéctate a tu servidor local (`root` / tu contraseña).
2. Abre `init-multi-db/00-create_dbs.sql` (File → Open SQL Script) y ejecútalo completo (rayo ⚡).
3. Repite lo mismo, en orden, con `01-usuarios.sql`, `02-catalogo.sql` y `03-recursos.sql`.

### Opción B — Línea de comandos

Desde la carpeta raíz del proyecto, ejecuta uno por uno (te pedirá la contraseña de `root`):

```bash
mysql -u root -p < init-multi-db/00-create_dbs.sql
mysql -u root -p < init-multi-db/01-usuarios.sql
mysql -u root -p < init-multi-db/02-catalogo.sql
mysql -u root -p < init-multi-db/03-recursos.sql
```

> ✅ Al terminar deberías tener 3 bases con datos: `usuarios`, `catalogo`, `recursos` (las demás bases que crea el script 00 quedan vacías, reservadas para futuros microservicios).

---

## 4. Abrir el proyecto en VS Code

1. Abre VS Code → `File > Open Folder...` → selecciona la carpeta raíz `Biblioteca-Springboot` (la que contiene el `pom.xml` principal).
2. Espera a que la extensión de Java termine de importar los módulos Maven (ícono de carga en la barra inferior).

---

## 5. Compilar todo el proyecto

Abre una terminal integrada en VS Code (`Terminal > New Terminal`) en la raíz del proyecto y ejecuta:

```bash
mvn clean install -DskipTests
```

Esto compila `common`, `eureka`, `ms-usuarios`, `ms-catalogo`, `ms-recursos`, `api-gateway` y `ms-vehiculos` en el orden correcto (Maven resuelve las dependencias entre módulos automáticamente).

---

## 6. Orden de ejecución de los servicios

⚠️ **Importante:** los microservicios no se levantan todos a la vez ni en cualquier orden. Deben iniciarse **uno por uno, en este orden**, esperando ~15-20 segundos entre cada uno para que se registren en Eureka:

| Orden | Servicio | Clase principal | Puerto |
|---|---|---|---|
| 1️⃣ | Eureka Server | `eureka/src/main/java/cl/triskeledu/eureka/BibliotecaEurekaApplication.java` | 8761 |
| 2️⃣ | MS Usuarios | `ms-usuarios/src/main/java/cl/triskeledu/usuarios/BibliotecaUsuariosApplication.java` | 9001 |
| 3️⃣ | MS Catálogo | `ms-catalogo/src/main/java/cl/triskeledu/catalogo/BibliotecaCatalogoApplication.java` | 9002 |
| 4️⃣ | MS Recursos | `ms-recursos/src/main/java/cl/triskeledu/recursos/BibliotecaRecursosApplication.java` | 9003 |
| 5️⃣ | API Gateway | `api-gateway/src/main/java/cl/triskeledu/gateway/BibliotecaGatewayApplication.java` | 9000 |

### Cómo iniciar cada uno en VS Code

Para cada clase de la tabla, en ese orden:

1. Abre el archivo `...Application.java` correspondiente.
2. Haz clic en **`Run`** (aparece justo encima del método `main`), o clic derecho → **`Run Java`**.
3. Espera a ver en la consola el mensaje `Started ...Application` antes de iniciar el siguiente.

No hace falta ejecutar nada del módulo `common` (es solo una librería compartida, ya se compiló en el paso 5).

---

## 7. Verificar que todo quedó funcionando

- **Eureka Dashboard** (deben aparecer los 4 servicios registrados): http://localhost:8761
- **Swagger de cada microservicio**:
  - Usuarios: http://localhost:9001/swagger-ui.html
  - Catálogo: http://localhost:9002/swagger-ui.html
  - Recursos: http://localhost:9003/swagger-ui.html
- **API Gateway** (punto de entrada único): http://localhost:9000

Puedes probar los endpoints con la colección de Postman incluida en la carpeta `postman/`.

---

## 8. Microservicio extra: Vehículos (standalone)

`ms-vehiculos` es independiente del resto: **no usa Eureka, no pasa por el API Gateway y no tiene seguridad JWT**. Es un CRUD simple con Swagger que se levanta solo.

1. Crea su base de datos (solo hace falta crearla vacía, las tablas las crea Hibernate solo al arrancar):
   ```sql
   CREATE DATABASE vehiculos;
   ```
   (con MySQL Workbench o `mysql -u root -p -e "CREATE DATABASE vehiculos;"`)
2. Si tu contraseña de `root` es distinta a `alone15`, ajústala en `ms-vehiculos/src/main/resources/application.yml`.
3. Abre `ms-vehiculos/src/main/java/cl/triskeledu/vehiculos/BibliotecaVehiculosApplication.java` y dale **Run**. No depende de ningún otro servicio, se puede iniciar en cualquier momento.
4. Swagger: http://localhost:9004/swagger-ui.html

Endpoints disponibles bajo `/api/v1/vehiculos`: `GET` (listar), `GET /{id}`, `POST`, `PUT /{id}`, `DELETE /{id}`.

---

## 9. Problemas comunes

- **`Access denied for user 'root'@'localhost'`** → la contraseña en `application.yml` no coincide con tu MySQL. Revisa el paso 2.
- **`Unknown database 'usuarios'`** → no ejecutaste (o falló) `00-create_dbs.sql`. Repite el paso 3.
- **Un microservicio no aparece en Eureka** → asegúrate de haber iniciado `eureka` primero y de esperar unos segundos antes de levantar los demás.
- **Error de `CHECK constraint`** → verifica que tu versión de MySQL sea 8.0.16 o superior (`SELECT VERSION();`).
