# Amazon E-Commerce Backend -- Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Clients["Clients"]
        WEB["Web Browser"]
        MOBILE["Mobile App"]
    end

    subgraph Gateway["API Layer"]
        APIGW["API Gateway<br/>(Auth, Rate Limit, Routing)"]
        LB["Load Balancer"]
    end

    subgraph CatalogSearch["Product Discovery"]
        CATALOG["Product Catalog Service<br/>(Sharded MySQL + Redis Cache)"]
        SEARCH["Search Service<br/>(Elasticsearch, 50+ nodes)"]
        RECO["Recommendation Service<br/>(Collaborative Filtering)"]
        PRICING["Pricing Service<br/>(Dynamic Pricing + Promos)"]
    end

    subgraph CartCheckout["Cart & Checkout"]
        CART["Cart Service<br/>(DynamoDB)"]
        ORDER["Order Service<br/>(Saga Orchestrator)"]
        INVENTORY["Inventory Service<br/>(Sharded PostgreSQL)"]
        PAYMENT["Payment Service<br/>(PCI-compliant, Gateway Integrations)"]
    end

    subgraph Fulfillment["Fulfillment & Delivery"]
        FULFILL["Fulfillment Service<br/>(Warehouse Routing)"]
        SHIPPING["Shipping Service<br/>(Carrier Integration)"]
        NOTIFY["Notification Service<br/>(Email, SMS, Push)"]
    end

    subgraph DataLayer["Data Layer"]
        MYSQL["Sharded MySQL<br/>(Product Catalog)"]
        PG["Sharded PostgreSQL<br/>(Orders, Inventory)"]
        DYNAMO["DynamoDB<br/>(Cart)"]
        REDIS["Redis / Memcached<br/>(Product Cache, Sessions)"]
        ES["Elasticsearch<br/>(Search Index)"]
    end

    subgraph EventBus["Event Bus"]
        KAFKA["Kafka<br/>(Order Events, Inventory Events, CDC)"]
    end

    subgraph Analytics["Analytics & ML"]
        FLINK["Flink<br/>(Real-time Aggregation)"]
        DW["Data Warehouse<br/>(Redshift / BigQuery)"]
        FRAUD["Fraud Detection<br/>(ML Scoring)"]
    end

    WEB --> LB --> APIGW
    MOBILE --> LB

    APIGW -->|"browse"| CATALOG
    APIGW -->|"search"| SEARCH
    APIGW -->|"recommendations"| RECO
    APIGW -->|"cart ops"| CART
    APIGW -->|"checkout"| ORDER

    CATALOG --> MYSQL
    CATALOG --> REDIS
    SEARCH --> ES
    PRICING --> REDIS

    ORDER -->|"1. reserve stock"| INVENTORY
    ORDER -->|"2. authorize payment"| PAYMENT
    ORDER -->|"3. create order"| PG
    ORDER -->|"4. publish event"| KAFKA

    INVENTORY --> PG
    INVENTORY --> REDIS
    CART --> DYNAMO

    KAFKA --> FULFILL
    KAFKA --> NOTIFY
    KAFKA --> FLINK
    FLINK --> DW
    DW --> RECO

    FULFILL --> SHIPPING
    SHIPPING --> NOTIFY

    ORDER -->|"fraud check"| FRAUD

    KAFKA -->|"CDC"| ES
```

## 2. Deep-Dive: Checkout Flow (Saga Pattern)

```mermaid
flowchart TB
    subgraph UserAction["User Action"]
        CLICK["Click 'Place Order'<br/>(idempotency_key: uuid)"]
    end

    subgraph Validation["Step 0: Validate"]
        VALIDATE_CART["Validate Cart<br/>(Items exist, prices current)"]
        VALIDATE_ADDR["Validate Shipping Address"]
        VALIDATE_PAY["Validate Payment Method"]
    end

    subgraph InventoryStep["Step 1: Reserve Inventory"]
        subgraph PerItem["For Each Item"]
            CHECK_STOCK["Check available_quantity >= requested"]
            DECR["Atomic UPDATE:<br/>available -= qty,<br/>reserved += qty<br/>(SELECT FOR UPDATE)"]
        end
        INV_SUCCESS{"All items<br/>reserved?"}
        INV_FAIL["Return: Item X out of stock"]
    end

    subgraph PaymentStep["Step 2: Authorize Payment"]
        CALC_TOTAL["Calculate Total<br/>(items + tax + shipping)"]
        AUTH["Payment Gateway:<br/>Authorize $total"]
        PAY_SUCCESS{"Authorized?"}
        PAY_FAIL["Compensate: Release inventory"]
    end

    subgraph OrderStep["Step 3: Create Order"]
        CREATE_ORDER["Insert Order Record<br/>(status: CONFIRMED)"]
        CREATE_ITEMS["Insert Order Items<br/>(per-item details)"]
        ORDER_SUCCESS{"Order created?"}
        ORDER_FAIL["Compensate: Void payment<br/>+ Release inventory"]
    end

    subgraph PublishStep["Step 4: Publish & Respond"]
        PUBLISH["Publish OrderConfirmed<br/>event to Kafka"]
        RESPOND["Return 201 Created<br/>{order_id, status: CONFIRMED}"]
    end

    subgraph Compensation["Compensation Handlers"]
        RELEASE_INV["Release Inventory:<br/>available += qty,<br/>reserved -= qty"]
        VOID_PAY["Void Payment<br/>Authorization"]
    end

    subgraph Background["Background Jobs"]
        RESERVATION_TTL["Reservation TTL Monitor<br/>(Release after 15 min)"]
        CAPTURE["Payment Capture<br/>(After fulfillment confirms)"]
    end

    CLICK --> VALIDATE_CART
    VALIDATE_CART --> VALIDATE_ADDR
    VALIDATE_ADDR --> VALIDATE_PAY

    VALIDATE_PAY --> CHECK_STOCK
    CHECK_STOCK --> DECR
    DECR --> INV_SUCCESS
    INV_SUCCESS -->|"yes"| CALC_TOTAL
    INV_SUCCESS -->|"no"| INV_FAIL

    CALC_TOTAL --> AUTH
    AUTH --> PAY_SUCCESS
    PAY_SUCCESS -->|"yes"| CREATE_ORDER
    PAY_SUCCESS -->|"no"| PAY_FAIL
    PAY_FAIL --> RELEASE_INV

    CREATE_ORDER --> CREATE_ITEMS
    CREATE_ITEMS --> ORDER_SUCCESS
    ORDER_SUCCESS -->|"yes"| PUBLISH
    ORDER_SUCCESS -->|"no"| ORDER_FAIL
    ORDER_FAIL --> VOID_PAY
    ORDER_FAIL --> RELEASE_INV

    PUBLISH --> RESPOND

    RESERVATION_TTL -.->|"expired reservations"| RELEASE_INV
    PUBLISH -.->|"after shipping"| CAPTURE
```

## 3. Critical Path Sequence: Search to Order Delivery

```mermaid
sequenceDiagram
    participant U as User
    participant API as API Gateway
    participant SRCH as Search Service
    participant CAT as Catalog Service
    participant CART as Cart Service
    participant ORD as Order Service
    participant INV as Inventory Service
    participant PAY as Payment Service
    participant K as Kafka
    participant FF as Fulfillment Service
    participant SHIP as Shipping Service
    participant NOTIF as Notification Service

    U->>API: GET /search?q=wireless headphones&max_price=50
    API->>SRCH: Query Elasticsearch
    SRCH->>SRCH: BM25 + LTR ranking, facet computation
    SRCH-->>U: Results: [{product_id, title, price, rating, ...}]

    U->>API: GET /products/PROD_123
    API->>CAT: Fetch product details (cache hit: Redis)
    CAT-->>U: {title, price, variants, reviews, delivery_estimate}

    U->>API: POST /cart/items {product_id: PROD_123, qty: 1}
    API->>CART: Add to DynamoDB cart
    CART-->>U: {cart: {items: [...], subtotal: $39.99}}

    U->>API: POST /orders {cart_id, address_id, payment_id}
    API->>ORD: Begin checkout saga (idempotency_key: uuid)

    ORD->>ORD: Validate cart items and prices

    ORD->>INV: Reserve inventory (PROD_123, qty=1, FC=warehouse-east)
    INV->>INV: SELECT FOR UPDATE; available -= 1, reserved += 1
    INV-->>ORD: Reserved (reservation_id: RES_456)

    ORD->>PAY: Authorize $42.49 (items + tax + shipping)
    PAY->>PAY: Call payment gateway (Stripe)
    PAY-->>ORD: Authorized (auth_id: AUTH_789)

    ORD->>ORD: Create order record (status: CONFIRMED)
    ORD->>K: Publish OrderConfirmed event
    ORD-->>U: {order_id: ORD_001, status: CONFIRMED, delivery: "Mar 9"}

    K->>FF: OrderConfirmed event
    FF->>FF: Route to warehouse-east (nearest with stock)
    FF->>FF: Generate pick list

    K->>NOTIF: OrderConfirmed event
    NOTIF-->>U: Email: "Order confirmed! Delivery by Mar 9"

    Note over FF: Warehouse picks and packs order

    FF->>SHIP: Create shipment (carrier: UPS)
    SHIP-->>FF: Tracking number: 1Z999AA1
    FF->>K: OrderShipped event

    K->>PAY: OrderShipped -> Capture payment
    PAY->>PAY: Capture $42.49

    K->>NOTIF: OrderShipped event
    NOTIF-->>U: Email: "Your order has shipped! Track: 1Z999AA1"

    SHIP->>K: OrderDelivered event
    K->>NOTIF: OrderDelivered
    NOTIF-->>U: "Your order has been delivered!"
```
