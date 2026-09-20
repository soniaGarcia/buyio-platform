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