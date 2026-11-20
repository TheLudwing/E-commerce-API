# 🛍️  E-commerce API

Backend REST para un e-commerce de productos de peluquería, construido con **Spring Boot 3**, **MySQL** y **JWT**. Implementa flujo completo de catálogo → carrito → orden, con foco en **integridad transaccional**, **concurrencia** e **idempotencia**.

---

## 📋 Tabla de contenidos

- [Stack](#-stack)
- [Características](#-características)
- [Arquitectura](#-arquitectura)
- [Empezar](#-empezar)
- [Variables de entorno](#-variables-de-entorno)
- [Endpoints principales](#-endpoints-principales)
- [Migraciones y seed](#-migraciones-y-seed)
- [Estructura del proyecto](#-estructura-del-proyecto)
- [Decisiones técnicas destacadas](#-decisiones-técnicas-destacadas)
- [Roadmap](#-roadmap)
- [Contribuir](#-contribuir)
- [Licencia](#-licencia)

---

## 🧰 Stack

| Categoría | Tecnología |
|-----------|-----------|
| Lenguaje | Java 21 |
| Framework | Spring Boot 3.x |
| Persistencia | Spring Data JPA + Hibernate |
| Base de datos | MySQL 8 |
| Seguridad | Spring Security + JWT (JJWT) |
| Migraciones | Flyway |
| Documentación | SpringDoc OpenAPI (Swagger UI) |
| Validación | Jakarta Bean Validation |
| Utilidades | Lombok, MapStruct |
| Testing | JUnit 5, Mockito, AssertJ |
| Build | Maven |

---

## ✨ Características

### Autenticación y autorización
- Registro e inicio de sesión con **JWT** (access token)
- Contraseñas hasheadas con **BCrypt**
- Roles: `ADMIN`, `WORKER`, `CLIENT`
- Protección por endpoint con `@PreAuthorize`

### Catálogo
- CRUD de **categorías** (admin)
- CRUD de **productos** con categorías (**ManyToMany**)
- **Búsqueda dinámica** con filtros combinables (`Specification` de JPA)
- **Borrado lógico** (`deleted = true`)
- Paginación y ordenamiento vía `Pageable`
- Snapshot del precio en el momento de la compra

### Carrito
- Un carrito activo por usuario (`@OneToOne` con `users`)
- Añadir, actualizar y eliminar ítems
- Validación de stock al añadir
- Suma automática si el producto ya existe en el carrito
- Cálculo dinámico del total

### Órdenes
- **Checkout transaccional** carrito → orden
- **Máquina de estados** validada:
  ```
  CREATED → PAID → SHIPPED → DELIVERED
     │        │
     ▼        ▼
  CANCELLED  REFUNDED
  ```
- Descuento **atómico** de stock con bloqueo optimista (`@Version`)
- **Snapshot** del nombre, talla, color y precio por ítem
- Números de orden legibles y únicos (`ORD-20241108-0001`)
- Listado del cliente (summary) vs. listado admin (con datos del cliente)

### Integridad e idempotencia
- **Lock pesimista** en checkout → evita órdenes duplicadas por doble clic o reintentos
- Header `Idempotency-Key` opcional para reintentos seguros (devuelve la misma orden)
- Tabla `processed_webhooks` lista para webhooks idempotentes (Stripe)
- Transiciones de estado **idempotentes** (`PAID → PAID` devuelve 200, no 409)
- **Manejo unificado de errores** con `ProblemDetail` (RFC 7807)

---

## 🏗️ Arquitectura

```
┌──────────────┐       ┌────────────────────┐      ┌─────────────┐
│  Cliente     │──JWT─▶│  Controllers       │─────▶│  Services   │
│ (SPA/mobile) │◀──────│ (REST + Swagger)   │◀─────│             │
└──────────────┘       └────────────────────┘      └──────┬──────┘
                                                          │
                                              ┌───────────▼──────────┐
                                              │  Repositories (JPA)  │
                                              └───────────┬──────────┘
                                                          │
                                              ┌───────────▼──────────┐
                                              │       MySQL          │
                                              │   (Flyway)           │
                                              └──────────────────────┘
```

**Capas:**
- **Controller** → validación de entrada, DTOs, autorización
- **Service** → reglas de negocio, transacciones, máquina de estados
- **Repository** → acceso a datos, `Specification` para filtros
- **Entity** → modelo de persistencia (nunca se expone en la API)

---

## 🚀 Empezar

### Requisitos
- Java 21+
- Maven 3.9+
- MySQL 8+

### Instalación

```bash
# 1. Clonar
git clone https://github.com/<tu-usuario>/ecommerce-peluqueria.git
cd ecommerce-peluqueria

# 2. Crear la base de datos
mysql -u root -p
```

```sql
CREATE DATABASE peluqueria_db
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
```

```bash
# 3. Configurar variables de entorno (ver sección siguiente)
cp .env.example .env
# Editar .env con tus credenciales

# 4. Ejecutar
./mvnw spring-boot:run
```

La app arranca en `http://localhost:8080`.
Swagger UI: `http://localhost:8080/swagger-ui.html`

---

## 🔐 Variables de entorno

Crea un archivo `.env` (o exporta las variables) con:

```env
# Base de datos
DB_URL=jdbc:mysql://localhost:3306/peluqueria_db?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
DB_USERNAME=root
DB_PASSWORD=tu_password

# JWT
JWT_SECRET=<cadena-de-al-menos-256-bits-en-base64>
JWT_ACCESS_EXPIRATION=900000   # 15 min

# Stripe (cuando se integre)
STRIPE_API_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
```

> ⚠️ **Nunca** subas el archivo `.env` al repositorio. Está incluido en `.gitignore`.

Ejemplo de `application.yml`:

```yaml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: true
  flyway:
    enabled: true

jwt:
  secret: ${JWT_SECRET}
  access-expiration: ${JWT_ACCESS_EXPIRATION}
```

---

## 🌐 Endpoints principales

### Autenticación (`/api/v1/auth`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| `POST` | `/register` | Registro de usuario |
| `POST` | `/login` | Login, devuelve JWT |

### Catálogo (`/api/v1/product`, `/api/v1/category`)
| Método | Ruta | Rol | Descripción |
|--------|------|-----|-------------|
| `GET` | `/product/index` | público | Productos activos (paginados) |
| `GET` | `/product/filter?name=&color=&size=` | público | Búsqueda con filtros |
| `GET` | `/product/{id}` | ADMIN | Detalle por id |
| `POST` | `/product/create` | ADMIN | Crear producto |
| `PUT` | `/product/update/{id}` | ADMIN | Actualizar producto |
| `DELETE` | `/product/{id}` | ADMIN | Borrado lógico |
| `GET` | `/category` | público | Listar categorías |
| `POST` | `/category` | ADMIN | Crear categoría |

### Carrito (`/api/v1/cart`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| `GET` | `/` | Ver carrito actual (lo crea si no existe) |
| `POST` | `/items` | Añadir producto |
| `PUT` | `/items/{itemId}` | Actualizar cantidad |
| `DELETE` | `/items/{itemId}` | Eliminar ítem |
| `DELETE` | `/` | Vaciar carrito |

### Órdenes (`/api/v1/orders`)
| Método | Ruta | Rol | Descripción |
|--------|------|-----|-------------|
| `POST` | `/checkout` | CLIENT | Crear orden desde el carrito |
| `GET` | `/my-orders` | CLIENT | Listar mis órdenes |
| `GET` | `/{orderNumber}` | CLIENT | Detalle de mi orden |
| `PATCH` | `/{orderNumber}/cancel` | CLIENT | Cancelar (solo si `CREATED`) |
| `GET` | `/admin/all` | ADMIN | Listar todas las órdenes |
| `PATCH` | `/admin/{id}/status` | ADMIN | Cambiar estado |

**Ejemplo de request:**
```bash
curl -X POST http://localhost:8080/api/v1/orders/checkout \
  -H "Authorization: Bearer <JWT>" \
  -H "Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000"
```

---

## 🗄️ Migraciones y seed

Las migraciones viven en `src/main/resources/db/migration/` y se ejecutan automáticamente al arrancar con **Flyway**:

| Archivo | Propósito |
|---------|-----------|
| `V1__init_schema.sql` | Tablas base (users, category, products, product_category) |
| `V2__seed_data.sql` | Datos de prueba (18 productos, 6 categorías) |
| `V3__cart.sql` | Tablas carts y cart_items |
| `V4__orders.sql` | Tablas orders y order_items |
| `V5__order_sequence.sql` | Secuencia de números de orden |
| `V6__product_version.sql` | Columna `version` para bloqueo optimista |
| `V7__idempotency.sql` | Tablas idempotency_keys y processed_webhooks |

**Reparar estado si una migración falla:**
```sql
DELETE FROM flyway_schema_history WHERE success = 0;
```

> 💡 **Importante:** `ddl-auto: validate`. Hibernate **no** modifica el esquema, solo valida que las entidades coincidan con la BD. Flyway es la única fuente de verdad del esquema.

---

## 📁 Estructura del proyecto

```
src/main/java/com/api/peluqueria/
├── controller/          # REST controllers
├── entity/
│   ├── dto/            # Request DTOs (record)
│   ├── response/       # Response DTOs (record)
│   ├── request/        # Request bodies
│   └── specification/  # Specifications de filtros JPA
├── exception/          # Excepciones de dominio + handler
├── repository/         # Spring Data JPA repos
├── service/            # Lógica de negocio
├── util/               # JWT, mappers, generadores
└── Application.java
```

---

## 🧠 Decisiones técnicas destacadas

Algunas decisiones que hacen que este proyecto sea algo más que un CRUD:

### Integridad transaccional
- **`@Version`** en `Product` previene sobreventa cuando dos usuarios compran la última unidad
- **Lock pesimista** (`SELECT FOR UPDATE`) en el checkout evita órdenes duplicadas
- **`REQUIRES_NEW`** al generar número de orden → nunca se reutilizan números aunque el checkout falle

### Snapshot de datos
- `OrderItem` guarda **nombre, precio, talla y color** del producto en el momento de la compra
- Si el producto cambia de precio o se borra, la orden sigue mostrando lo que el cliente pagó
- **Obligatorio** para facturación y auditoría

### Máquina de estados
- Transiciones de `Order.status` validadas en un único lugar (`OrderStateMachine`)
- No se puede saltar de `CREATED` a `DELIVERED` sin pasar por `PAID` y `SHIPPED`
- **Tests exhaustivos** cubren transiciones válidas, inválidas y estados terminales

### Idempotencia
- **`Idempotency-Key` header** devuelve la misma orden si se reintenta la petición
- **`processed_webhooks`** prepara el terreno para Stripe (o cualquier webhook)
- `PAID → PAID` es válido (retry) pero `PAID → CREATED` no (transición ilegal real)

### Seguridad
- `userId` **siempre** se extrae del JWT (`SecurityContextHolder`), nunca del body
- Endpoints del cliente validan que el recurso le pertenezca (**prevención de IDOR**)
- Contraseñas con **BCrypt**, nunca en texto plano
- JWT con expiración corta

---

## 🗺️ Roadmap

### ✅ Completado
- [x] Autenticación JWT con roles
- [x] CRUD de categorías y productos
- [x] Búsqueda con filtros dinámicos
- [x] Carrito con validación de stock
- [x] Checkout transaccional
- [x] Máquina de estados de orden
- [x] Idempotencia en checkout y cambio de estado
- [x] Manejo unificado de errores con `ProblemDetail`

### 🚧 En progreso
- [ ] Integración con **Stripe** (checkout session + webhook idempotente)
- [ ] **Caché con Redis** para el catálogo
- [ ] **Tareas asíncronas** con RabbitMQ (envío de emails)

### 📅 Planificado
- [ ] **Testcontainers** para tests de integración
- [ ] **Docker Compose** (app + MySQL + Redis + RabbitMQ)
- [ ] **CI/CD** con GitHub Actions
- [ ] **Observabilidad** (Actuator + Prometheus + Grafana)
- [ ] **Elasticsearch** para búsqueda full-text
- [ ] Rate limiting en endpoints públicos
- [ ] Refresh tokens con rotación

---

## 🤝 Contribuir

Este es un proyecto de aprendizaje personal. Si encuentras un bug o tienes una sugerencia:

1. Abre un **issue** describiendo el problema o la idea
2. Si quieres contribuir código, abre un **PR** desde una rama feature:
   ```bash
   git checkout -b feature/nombre-descriptivo
   ```

---

## 📝 Licencia

Este proyecto está bajo la licencia **MIT**. Consulta el archivo [LICENSE](LICENSE) para más detalles.

---

## 📬 Contacto

**Desarrollador:** [Tu nombre]
**LinkedIn:** [linkedin.com/in/tu-perfil](https://linkedin.com/in/tu-perfil)
**Email:** tu.email@ejemplo.com

---

⭐ Si este proyecto te resulta útil, considera darle una estrella.
