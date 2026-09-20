# buyio-platform

```mermaid
graph TD
    %% Capa Cliente
    subgraph Client_Layer ["Capa de Presentación (Frontend)"]
        ReactApp["buyio-frontend (React)<br/>• Form Validations (Email, Fechas)<br/>• Storage de JWT Bearer Token"]
    end

    %% Capa Gateway
    subgraph Gateway_Layer ["Capa de Gateway & Entrada"]
        Gateway["buyio-api-gateway<br/>(Spring Cloud Gateway / Reverse Proxy)"]
        JwtFilter["JWT Authentication Filter<br/>• Intercepta Headers HTTP<br/>• Extrae & Extrae Token Bearer"]
    end

    %% Capa Seguridad
    subgraph Auth_Layer ["Servicio de Autenticación"]
        AuthService["buyio-auth-service<br/>(Spring Security + JWT Issuer/Validator)"]
        AuthDB[("PostgreSQL<br/>(buyio_auth_db)")]
    end

    %% Capa de Negocio
    subgraph Business_Layer ["Servicios de Dominio de Negocio"]
        subgraph Order_Service ["buyio-order-service"]
            OrderCtrl["OrderController<br/><i>REST API Endpoints</i>"]
            OrderSvc["OrderService<br/><i>Lógica de Negocio + Soft Delete</i>"]
            OrderRepo["OrderRepository<br/><i>JPA Data Access Layer</i>"]
        end

        subgraph Catalog_Service ["buyio-catalog-service"]
            CatalogCtrl["CatalogController"]
            CatalogSvc["CatalogService"]
            CatalogRepo["CatalogRepository"]
        end

        OrderDB[("PostgreSQL<br/>(buyio_order_db)<br/><i>Tables: orders, order_items</i>")]
        CatalogDB[("PostgreSQL<br/>(buyio_catalog_db)<br/><i>Tables: products, suppliers</i>")]
    end

    %% Capa Event-Driven & Auditoría
    subgraph Audit_Layer ["Asincronía & Bitácora de Auditoría"]
        Kafka["Apache Kafka Event Bus<br/><i>Topics: OrderEvents, CatalogEvents, UserEvents</i>"]
        AuditService["buyio-audit-service<br/><i>Kafka Consumer @KafkaListener</i>"]
        AuditDB[("PostgreSQL<br/>(buyio_audit_db)<br/><i>Table: audit_logs</i>")]
    end

    %% Flujos y Conexiones
    ReactApp -->|"1. Petición HTTP (POST/PUT/GET) + Bearer JWT"| Gateway
    Gateway --> JwtFilter
    JwtFilter -->|"2. Invocación / Validar Firma JWT & Claims"| AuthService
    AuthService -.->|"Verifica Usuario / Roles"| AuthDB

    JwtFilter -->|"3. Petición Autenticada y Ruteada"| OrderCtrl
    JwtFilter -.->|"Ruteo secundario"| CatalogCtrl

    OrderCtrl --> OrderSvc
    OrderSvc --> OrderRepo
    OrderRepo -->|"4. Persistencia Relacional<br/>(created_at, updated_at, estado)"| OrderDB

    OrderSvc -->|"5. Publica OrderCreated / OrderCancelled"| Kafka

    CatalogCtrl --> CatalogSvc
    CatalogSvc --> CatalogRepo
    CatalogRepo --> CatalogDB
    CatalogSvc -.->|"Publica CatalogUpdated"| Kafka

    Kafka -->|"6. Suscripción Asíncrona de Eventos"| AuditService
    AuditService -->|"7. Registro Inmutable en Bitácora"| AuditDB

    %% Estilos de Nodos
    classDef clientStyle fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#fff;
    classDef gatewayStyle fill:#1e3a8a,stroke:#60a5fa,stroke-width:2px,color:#fff;
    classDef authStyle fill:#4c1d95,stroke:#c084fc,stroke-width:2px,color:#fff;
    classDef businessStyle fill:#064e3b,stroke:#34d399,stroke-width:2px,color:#fff;
    classDef auditStyle fill:#831843,stroke:#f472b6,stroke-width:2px,color:#fff;
    classDef dbStyle fill:#1c1917,stroke:#a8a29e,stroke-width:2px,color:#fff;

    class ReactApp clientStyle;
    class Gateway,JwtFilter gatewayStyle;
    class AuthService authStyle;
    class OrderCtrl,OrderSvc,OrderRepo,CatalogCtrl,CatalogSvc,CatalogRepo businessStyle;
    class Kafka,AuditService auditStyle;
    class AuthDB,OrderDB,CatalogDB,AuditDB dbStyle;
```


## Flujo Técnico de la Arquitectura

1. **Intercepción y Validación JWT:** Toda solicitud originada en el cliente React ingresa por `buyio-api-gateway`. El filtro customizado `JwtAuthenticationFilter` extrae el token del header `Authorization: Bearer <token>` e interactúa con `buyio-auth-service` para verificar su firma, vigencia y autoridades antes de dar paso a los servicios internos. Si el token es inválido o no está presente, se retorna de forma inmediata una respuesta `401 Unauthorized`.

2. **Procesamiento de Órdenes de Compra:** Una vez autorizada la solicitud, el Gateway la enruta hacia `OrderController` dentro de `buyio-order-service`. La capa de servicio (`OrderService`) ejecuta la lógica transaccional, aplicando anulación lógica (*soft delete*) cambiando el campo de estado de la orden en lugar de realizar una eliminación física (`DELETE`).

3. **Persistencia Relacional y Auditoría de Fechas:** Las entidades JPA gestionadas mediante `OrderRepository` mapean las relaciones clave con productos/proveedores y ejecutan el guardado en la base de datos PostgreSQL correspondiente (`buyio_order_db`). Cada tabla garantiza la bitácora obligatoria mediante las anotaciones de auditoría JPA (`@CreatedDate` / `@LastModifiedDate`) asignando automáticamente `created_at` y `updated_at`.

4. **Trazabilidad Asíncrona vía Kafka:** Tras confirmar la transacción relacional, el servicio emite un evento de dominio (`OrderEvents`) hacia Apache Kafka. El microservicio `buyio-audit-service` consume estos mensajes de forma descolada y asíncrona, registrando el historial unificado y no mutable de cambios en `buyio_audit_db`.


## Modelo de Datos (Entidad-Relación)

```mermaid
erDiagram
    %% Microservicio: buyio-auth-service
    USERS {
        BIGINT id PK
        VARCHAR username UK
        VARCHAR email UK
        VARCHAR password
        VARCHAR role
        TIMESTAMP created_at "BITÁCORA"
        TIMESTAMP updated_at "BITÁCORA"
    }

    %% Microservicio: buyio-catalog-service
    SUPPLIERS {
        BIGINT id PK
        VARCHAR name
        VARCHAR email
        VARCHAR phone
        VARCHAR status "ESTADO (ACTIVE/INACTIVE)"
        TIMESTAMP created_at "BITÁCORA"
        TIMESTAMP updated_at "BITÁCORA"
    }

    CATEGORIES {
        BIGINT id PK
        VARCHAR name UK
        VARCHAR description
        TIMESTAMP created_at "BITÁCORA"
        TIMESTAMP updated_at "BITÁCORA"
    }

    PRODUCTS {
        BIGINT id PK
        VARCHAR code UK
        VARCHAR name
        TEXT description
        NUMERIC price
        BIGINT supplier_id FK
        BIGINT category_id FK
        VARCHAR status "ESTADO (ACTIVE/INACTIVE)"
        TIMESTAMP created_at "BITÁCORA"
        TIMESTAMP updated_at "BITÁCORA"
    }

    %% Microservicio: buyio-order-service
    ORDERS {
        BIGINT id PK
        VARCHAR order_number UK
        BIGINT supplier_id "FK Logica -> SUPPLIERS"
        NUMERIC total_amount
        VARCHAR status "ESTADO (CREATED/CANCELLED/COMPLETED)"
        TIMESTAMP created_at "BITÁCORA"
        TIMESTAMP updated_at "BITÁCORA"
    }

    ORDER_ITEMS {
        BIGINT id PK
        BIGINT order_id FK
        BIGINT product_id "FK Logica -> PRODUCTS"
        INTEGER quantity
        NUMERIC unit_price
        NUMERIC subtotal
        TIMESTAMP created_at "BITÁCORA"
        TIMESTAMP updated_at "BITÁCORA"
    }

    %% Microservicio: buyio-audit-service
    AUDIT_LOGS {
        BIGINT id PK
        VARCHAR entity_name
        VARCHAR entity_id
        VARCHAR action
        VARCHAR performed_by
        TEXT payload
        TIMESTAMP created_at "BITÁCORA"
    }

    %% Relaciones Fisica dentro de buyio-catalog-service
    SUPPLIERS ||--o{ PRODUCTS : "provee"
    CATEGORIES ||--o{ PRODUCTS : "clasifica"

    %% Relacion Fisica dentro de buyio-order-service (1 a Muchos)
    ORDERS ||--|{ ORDER_ITEMS : "contiene"

    %% Relaciones Logicas entre dominios
    SUPPLIERS ..o{ ORDERS : "referencia_logica (supplier_id)"
    PRODUCTS ..o{ ORDER_ITEMS : "referencia_logica (product_id)"
```
