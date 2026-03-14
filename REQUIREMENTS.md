# E-Commerce Backend — Project Checklist

> Tick each box as you complete it. Work top to bottom — each section builds on the one before it.

---

## 📦 Week 1 — Monolith

### Step 1 · Project Setup

- [x] Create Spring Boot project via Spring Initializr with dependencies: `Spring Web`, `Spring Data JPA`, `Spring Security`, `PostgreSQL Driver`, `Flyway`, `Lombok`, `Validation`, `Docker Compose Support`, `Spring Cache`, `Redis`
- [ ] Set up `docker-compose.yml` with `postgres:15` service on port `5432`
- [ ] Set up `docker-compose.yml` with `redis:7` service on port `6379`
- [ ] Configure `application.yml` with datasource, JPA, and Redis properties
- [ ] Confirm app starts and connects to both PostgreSQL and Redis without errors

---

### Step 2 · Flyway Migrations

- [ ] Create `V1__create_users_table.sql` — columns: `id (UUID)`, `email`, `password_hash`, `first_name`, `last_name`, `role (ENUM)`, `created_at`, `updated_at`
- [ ] Create `V2__create_refresh_tokens_table.sql` — columns: `id (UUID)`, `user_id (FK)`, `token`, `expires_at`, `created_at`
- [ ] Create `V3__create_categories_table.sql` — columns: `id (UUID)`, `name`, `description`
- [ ] Create `V4__create_products_table.sql` — columns: `id (UUID)`, `name`, `description`, `price`, `stock_quantity`, `image_url`, `category_id (FK)`, `is_deleted`, `created_at`, `updated_at`
- [ ] Create `V5__create_orders_table.sql` — columns: `id (UUID)`, `user_id (UUID)`, `status (ENUM)`, `total_amount`, `created_at`, `updated_at`
- [ ] Create `V6__create_order_items_table.sql` — columns: `id (UUID)`, `order_id (FK)`, `product_id (UUID)`, `product_name`, `unit_price`, `quantity`
- [ ] Confirm all migrations run cleanly on app startup

---

### Step 3 · User Service

#### Entities & Repository
- [ ] Create `User` entity mapped to `users` table with all columns
- [ ] Create `RefreshToken` entity mapped to `refresh_tokens` table with `@ManyToOne` to `User`
- [ ] Create `Role` enum with values `USER` and `ADMIN`
- [ ] Create `UserRepository` extending `JpaRepository` with `findByEmail()` method
- [ ] Create `RefreshTokenRepository` with `findByToken()` and `deleteByUserId()` methods

#### Spring Security Configuration
- [ ] Configure `SecurityFilterChain` bean — permit `/api/auth/**`, secure everything else
- [ ] Create `JwtUtil` class — `generateToken()`, `validateToken()`, `extractClaims()` methods
- [ ] Create `JwtAuthenticationFilter` extending `OncePerRequestFilter`
- [ ] Register filter in security config before `UsernamePasswordAuthenticationFilter`
- [ ] Configure `BCryptPasswordEncoder` bean in `@Configuration` class
- [ ] Implement `UserDetailsService` loading user by email from DB

#### DTOs (use Java Records)
- [ ] `RegisterRequest` record — `email`, `firstName`, `lastName`, `password` with `@Valid` annotations
- [ ] `LoginRequest` record — `email`, `password`
- [ ] `AuthResponse` record — `accessToken`, `refreshToken`, `userId`, `email`, `role`
- [ ] `UserProfileResponse` record — `id`, `email`, `firstName`, `lastName`, `role`, `createdAt`

#### Service Layer
- [ ] `AuthService.register()` — validate unique email, hash password, save user, return tokens
- [ ] `AuthService.login()` — verify credentials, issue JWT (15 min expiry) + refresh token (7 days)
- [ ] `AuthService.refresh()` — validate refresh token from DB, issue new JWT
- [ ] `AuthService.logout()` — delete refresh token from DB
- [ ] `UserService.getProfile()` — return profile for authenticated user
- [ ] `UserService.updateProfile()` — update name or re-hash new password
- [ ] `UserService.listAllUsers()` — paginated, admin only

#### REST Endpoints
- [ ] `POST /api/auth/register` — public, returns `AuthResponse`
- [ ] `POST /api/auth/login` — public, returns `AuthResponse`
- [ ] `POST /api/auth/refresh` — public, returns new JWT
- [ ] `POST /api/auth/logout` — JWT required, invalidates refresh token
- [ ] `GET /api/users/me` — JWT required, returns `UserProfileResponse`
- [ ] `PUT /api/users/me` — JWT required, updates profile
- [ ] `GET /api/users` — JWT + ADMIN, paginated list of users
- [ ] `GET /api/users/{id}` — JWT + ADMIN, returns single user

#### Behaviour Checks
- [ ] Duplicate email registration returns `409 Conflict`
- [ ] Failed login returns `401` — does NOT reveal whether email exists
- [ ] Password minimum 8 characters enforced via `@Valid`
- [ ] JWT payload contains: `sub (userId)`, `email`, `role`, `iat`, `exp`
- [ ] Refresh token stored as hash in DB — never plain text
- [ ] `@PreAuthorize("hasRole('ADMIN')")` on admin endpoints

---

### Step 4 · Product Service

#### Entities & Repository
- [ ] Create `Category` entity mapped to `categories` table
- [ ] Create `Product` entity mapped to `products` table with `@ManyToOne` to `Category`
- [ ] Create `CategoryRepository` extending `JpaRepository`
- [ ] Create `ProductRepository` with custom `@Query` for search by name/description
- [ ] Add `findAllByIsDeletedFalse(Pageable pageable)` repository method
- [ ] Add `findByCategoryIdAndIsDeletedFalse(UUID categoryId, Pageable pageable)` method

#### Caching
- [ ] Configure `RedisCacheManager` bean with TTL of 10 minutes
- [ ] Add `@Cacheable("products")` on `getProductById()` service method
- [ ] Add `@CacheEvict("products")` on `updateProduct()` service method
- [ ] Add `@CacheEvict("products")` on `deleteProduct()` service method
- [ ] Confirm cache hit on second call to `GET /api/products/{id}` via logs

#### DTOs
- [ ] `CreateProductRequest` — `name`, `description`, `price`, `stockQuantity`, `categoryId` with `@Valid`
- [ ] `UpdateProductRequest` — same fields, all optional
- [ ] `ProductResponse` — all product fields including category name, `imageUrl`
- [ ] `CategoryResponse` — `id`, `name`, `description`
- [ ] `PagedResponse<T>` — generic wrapper with `content`, `page`, `size`, `totalElements`, `totalPages`

#### Service Layer
- [ ] `ProductService.listProducts()` — paginated, optional category filter, excludes soft-deleted
- [ ] `ProductService.getProductById()` — cached, throws `404` if not found or soft-deleted
- [ ] `ProductService.searchProducts()` — keyword search on name and description
- [ ] `ProductService.createProduct()` — admin only, validates category exists
- [ ] `ProductService.updateProduct()` — admin only, evicts cache
- [ ] `ProductService.deleteProduct()` — admin only, sets `is_deleted = true`, evicts cache
- [ ] `ProductService.decrementStock()` — internal, throws `409` if stock would go below zero
- [ ] `ProductService.generateUploadUrl()` — calls AWS S3Presigner, returns pre-signed PUT URL valid 5 min

#### REST Endpoints
- [ ] `GET /api/products` — public, paginated, optional `?category=` filter (default page=0, size=20)
- [ ] `GET /api/products/{id}` — public, cached response
- [ ] `GET /api/products/search` — public, `?q=` keyword parameter
- [ ] `POST /api/products` — JWT + ADMIN
- [ ] `PUT /api/products/{id}` — JWT + ADMIN
- [ ] `DELETE /api/products/{id}` — JWT + ADMIN, soft delete
- [ ] `PATCH /api/products/{id}/stock` — internal only, not exposed through gateway
- [ ] `GET /api/categories` — public
- [ ] `POST /api/categories` — JWT + ADMIN
- [ ] `GET /api/products/{id}/upload-url` — JWT + ADMIN, returns S3 pre-signed URL + image key

#### Behaviour Checks
- [ ] Soft-deleted products never appear in any listing or search result
- [ ] Stock cannot go below zero — `409 Conflict` with clear error message
- [ ] Pagination defaults: `page=0`, `size=20`, sorted by `created_at DESC`
- [ ] N+1 problem resolved — use `JOIN FETCH` or `@EntityGraph` for product + category query
- [ ] Image URL saved to product after separate S3 upload — not during product creation

---

### Step 5 · Order Service (Monolith Phase)

#### Entities & Repository
- [ ] Create `Order` entity mapped to `orders` table — note `user_id` is plain UUID, not a FK
- [ ] Create `OrderItem` entity mapped to `order_items` with `@ManyToOne` to `Order`
- [ ] Create `OrderStatus` enum: `PENDING`, `CONFIRMED`, `SHIPPED`, `DELIVERED`, `CANCELLED`
- [ ] Create `OrderRepository` with `findByUserIdOrderByCreatedAtDesc(UUID userId, Pageable pageable)`
- [ ] Create `OrderItemRepository`

#### DTOs
- [ ] `PlaceOrderRequest` — `List<OrderItemRequest>` where each item has `productId` and `quantity`
- [ ] `OrderItemRequest` — `productId (UUID)`, `quantity (int, min=1)`
- [ ] `OrderResponse` — `id`, `status`, `totalAmount`, `items`, `createdAt`
- [ ] `OrderItemResponse` — `productId`, `productName`, `unitPrice`, `quantity`, `subtotal`

#### Service Layer
- [ ] `OrderService.placeOrder()` — wrapped in `@Transactional`:
    - [ ] Extract `userId` from JWT claims — never from request body
    - [ ] For each item: call Product Service to get product details and validate stock
    - [ ] Call `PATCH /api/products/{id}/stock` to decrement stock for each item
    - [ ] Save `Order` and all `OrderItem` rows — snapshot `productName` and `unitPrice` at this moment
    - [ ] Calculate and store `totalAmount`
- [ ] `OrderService.getOrderHistory()` — paginated, only current user's orders
- [ ] `OrderService.getOrderById()` — validates order belongs to the requesting user, throws `403` otherwise
- [ ] `OrderService.cancelOrder()` — only allowed if status is `PENDING`, transitions to `CANCELLED`
- [ ] `OrderService.listAllOrders()` — admin only, paginated across all users
- [ ] `OrderService.updateOrderStatus()` — admin only, updates status

#### REST Endpoints
- [ ] `POST /api/orders` — JWT required, places new order
- [ ] `GET /api/orders` — JWT required, paginated order history for current user
- [ ] `GET /api/orders/{id}` — JWT required, must belong to requesting user
- [ ] `PATCH /api/orders/{id}/cancel` — JWT required, only cancels `PENDING` orders
- [ ] `GET /api/admin/orders` — JWT + ADMIN, all orders paginated
- [ ] `PATCH /api/admin/orders/{id}/status` — JWT + ADMIN, update any order's status

#### Behaviour Checks
- [ ] If stock decrement fails for any item, the entire order transaction rolls back — no partial orders saved
- [ ] `productName` and `unitPrice` in `order_items` are snapshots — changing the product later does not affect historical orders
- [ ] `userId` always comes from JWT claims, never the request body
- [ ] Attempting to view another user's order returns `403 Forbidden`
- [ ] Cancelling a non-PENDING order returns `409 Conflict`

---

### Step 6 · Cross-Cutting Concerns

#### Global Exception Handler
- [ ] Create `@ControllerAdvice` class with `@ExceptionHandler` methods
- [ ] Handle `ResourceNotFoundException` → `404` with message
- [ ] Handle `InsufficientStockException` → `409` with message
- [ ] Handle `AccessDeniedException` → `403` with message
- [ ] Handle `MethodArgumentNotValidException` → `400` with field-level error map
- [ ] Handle generic `Exception` → `500` with sanitised message (no stack trace in response)
- [ ] Consistent error response shape: `{ timestamp, status, error, message, path }`

#### Validation
- [ ] `@NotBlank`, `@Email`, `@Size`, `@Min`, `@NotNull` applied on all request DTOs
- [ ] Validation triggered via `@Valid` on all controller method parameters
- [ ] Validation errors return `400` with per-field messages

#### Unit Tests
- [ ] `AuthServiceTest` — register, login, duplicate email, bad credentials
- [ ] `ProductServiceTest` — create, update, delete, cache eviction, stock decrement
- [ ] `OrderServiceTest` — place order happy path, rollback on stock failure, ownership check
- [ ] Mockito used to mock all repository and external dependencies
- [ ] At least 1 `MockMvc` controller test per service verifying HTTP status and response shape

---

### Step 7 · AWS Deployment (Week 1)

#### Docker
- [ ] Write `Dockerfile` for the monolith — multi-stage build (build stage with Maven, runtime stage with JRE 17)
- [ ] Confirm `docker build` succeeds locally
- [ ] Confirm `docker run` connects to local PostgreSQL and Redis successfully

#### AWS Setup
- [ ] Create S3 bucket for product images — block public access ON
- [ ] Add bucket policy allowing `s3:GetObject` on `products/*` prefix publicly
- [ ] Configure CORS on S3 bucket — allow `PUT` method
- [ ] Create ECR repository — push Docker image via `docker tag` + `docker push`
- [ ] Provision RDS PostgreSQL `db.t3.micro` — free tier, same VPC as ECS
- [ ] Provision ElastiCache Redis `cache.t3.micro` — same VPC as ECS
- [ ] Create ECS Cluster — Fargate launch type
- [ ] Create ECS Task Definition — reference ECR image, set env vars via SSM Parameter Store
- [ ] Store secrets in SSM Parameter Store: `DB_URL`, `DB_PASSWORD`, `JWT_SECRET`, `REDIS_URL`
- [ ] Create ECS Service — 1 desired task, attach to VPC subnets
- [ ] Configure Security Groups — ECS task SG allows outbound to RDS SG on port `5432`
- [ ] Configure Security Groups — ECS task SG allows outbound to ElastiCache SG on port `6379`
- [ ] Confirm app is reachable via ECS public IP or Load Balancer DNS
- [ ] Confirm Flyway migrations run on first startup against RDS

---

## 🔀 Week 2 — Microservices Extraction

### Step 8 · Extract Order Service

- [ ] Create new Spring Boot project: `order-service` with its own `pom.xml` / `build.gradle`
- [ ] Create separate database in RDS for orders (or separate RDS instance)
- [ ] Move all Order entities, repositories, services, controllers, DTOs to new project
- [ ] Configure `order-service` `application.yml` — its own DB URL, port `8083`
- [ ] Remove Order code from monolith — monolith now only has User + Product services
- [ ] Confirm both apps start independently and connect to their respective DBs

---

### Step 9 · WebClient + Resilience4j

- [ ] Add `spring-boot-starter-webflux` dependency to `order-service` for `WebClient`
- [ ] Create `WebClient` bean configured with Product Service base URL
- [ ] Replace monolith-era `RestTemplate` calls with `WebClient` reactive calls
- [ ] Add `resilience4j-spring-boot3` dependency
- [ ] Configure `CircuitBreaker` for Product Service calls in `application.yml` — failure threshold 50%, wait 10s
- [ ] Annotate stock check method with `@CircuitBreaker(name = "productService", fallbackMethod = "...")`
- [ ] Implement fallback method — return `503` with `Retry-After: 5` header
- [ ] Add `@Retry(name = "productService")` — 3 attempts, 500ms wait
- [ ] Confirm circuit breaker trips when Product Service is stopped — fallback fires correctly

---

### Step 10 · API Gateway

- [ ] Create new Spring Boot project: `api-gateway` with `spring-cloud-starter-gateway` dependency
- [ ] Add Redis dependency for rate limiting
- [ ] Configure routes in `application.yml`:
    - [ ] `auth-route` → User Service, no JWT filter, path `/api/auth/**`
    - [ ] `users-route` → User Service, JWT required, path `/api/users/**`
    - [ ] `products-route` → Product Service, JWT optional, path `/api/products/**` and `/api/categories/**`
    - [ ] `orders-route` → Order Service, JWT required, path `/api/orders/**`
    - [ ] `admin-route` → respective services, JWT + ADMIN, path `/api/admin/**`
- [ ] Create `JwtAuthenticationFilter` as `GlobalFilter` — validates JWT, forwards `X-User-Id` and `X-User-Role` headers
- [ ] Configure `RequestRateLimiter` filter — `100` requests/min per IP via Redis
- [ ] Configure CORS in Gateway — remove CORS config from individual services
- [ ] Strip `X-Internal-Call` header from all incoming client requests
- [ ] Return `401` for missing or invalid JWT
- [ ] Return `429` for rate limit exceeded
- [ ] Confirm all service routes work end-to-end through the Gateway

---

### Step 11 · Kafka Integration

- [ ] Add `spring-kafka` dependency to `order-service` and create new `notification-service` project
- [ ] Add Kafka to `docker-compose.yml` — `confluentinc/cp-kafka:7.5.0` with Zookeeper
- [ ] Create Kafka topic `order.placed` with 3 partitions, 1 replica (local dev)
- [ ] Create Kafka topic `order.placed.DLT` for dead-letter messages

#### Order Service — Producer
- [ ] Create `OrderPlacedEvent` record — `orderId`, `userId`, `userEmail`, `items[]`, `totalAmount`, `timestamp`
- [ ] Configure `KafkaTemplate<String, OrderPlacedEvent>` bean with JSON serializer
- [ ] Call `kafkaTemplate.send("order.placed", event)` at end of `placeOrder()` — after successful DB commit
- [ ] Confirm event is published and visible in Kafka logs

#### Notification Service — Consumer
- [ ] Create `notification-service` Spring Boot project — no DB, no REST endpoints
- [ ] Configure `@KafkaListener(topics = "order.placed", groupId = "notification-service-group")`
- [ ] Implement listener — log structured message: `orderId`, `userId`, `totalAmount`, `timestamp`
- [ ] Configure manual acknowledgement — `AckMode.MANUAL`
- [ ] Configure `DeadLetterPublishingRecoverer` — after 3 retry failures, route to `order.placed.DLT`
- [ ] Use `Executors.newVirtualThreadPerTaskExecutor()` for async processing (Java 21)
- [ ] Expose `GET /actuator/health` and `GET /actuator/metrics`
- [ ] Confirm end-to-end: place an order → event logged in notification service console

---

### Step 12 · Integration Tests

- [ ] Add `testcontainers` BOM and `postgresql`, `kafka`, `redis` modules to each service
- [ ] Create `@SpringBootTest` integration test for User Service — full register + login flow against real PostgreSQL container
- [ ] Create `@SpringBootTest` integration test for Product Service — CRUD + cache verification against real PostgreSQL + Redis containers
- [ ] Create `@SpringBootTest` integration test for Order Service — full order placement flow, verify stock decremented, Kafka event published
- [ ] Create `@SpringBootTest` integration test for Notification Service — consume test Kafka event, verify listener fires
- [ ] Run all tests via `./mvnw test` — confirm green

---

### Step 13 · Deploy Microservices to AWS

- [ ] Write separate `Dockerfile` for each service: `user-service`, `product-service`, `order-service`, `notification-service`, `api-gateway`
- [ ] Create ECR repository for each service
- [ ] Build and push all Docker images to their respective ECR repositories
- [ ] Create ECS Task Definition for each service with correct environment variables from SSM
- [ ] Create ECS Service for each Task Definition
- [ ] Configure Security Groups to allow inter-service communication within VPC
- [ ] Set up Application Load Balancer in front of `api-gateway` service only — all traffic enters via gateway
- [ ] Deploy Kafka via Amazon MSK (managed) or a single EC2 `t3.small` for dev
- [ ] Update Notification Service to consume from MSK/EC2 Kafka
- [ ] Confirm end-to-end flow through Load Balancer → Gateway → Services → RDS → Kafka works

---

## ✨ Week 3 — Polish

### Step 14 · Observability

- [ ] Add `spring-boot-starter-actuator` and `micrometer-registry-cloudwatch` to all services
- [ ] Enable Actuator endpoints: `health`, `metrics`, `info` in `application.yml`
- [ ] Configure Micrometer to push metrics to CloudWatch every 60 seconds
- [ ] Add `correlation-id` MDC logging — generate UUID per request in a filter, include in all log lines
- [ ] Configure Logback to output structured JSON logs
- [ ] Confirm logs appear in CloudWatch Log Groups per service
- [ ] Create CloudWatch Dashboard — panels for request count, error rate, JVM heap, DB connection pool
- [ ] Set CloudWatch Alarm on error rate > 5% for Order Service

---

### Step 15 · CI/CD Pipeline

- [ ] Create `.github/workflows/deploy.yml`
- [ ] Pipeline triggers on push to `main` branch
- [ ] Stage 1 — Build: `./mvnw clean package -DskipTests`
- [ ] Stage 2 — Test: `./mvnw test` (Testcontainers spin up automatically)
- [ ] Stage 3 — Docker Build: `docker build -t {service}:{sha} .`
- [ ] Stage 4 — Push to ECR: authenticate with `aws ecr get-login-password`, push image
- [ ] Stage 5 — Deploy to ECS: `aws ecs update-service --force-new-deployment`
- [ ] Store AWS credentials as GitHub Actions secrets: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`
- [ ] Confirm pipeline runs green end-to-end on a test push

---

### Step 16 · S3 Image Upload

- [ ] Add AWS SDK v2 `s3` and `s3-transfer-manager` dependencies to Product Service
- [ ] Configure `S3Presigner` bean using ECS Task IAM role (no hardcoded credentials)
- [ ] Implement `generateUploadUrl()` — produces pre-signed `PUT` URL valid for 5 minutes
- [ ] URL targets key pattern: `products/{productId}/{uuid}.jpg`
- [ ] ECS Task IAM Role policy — `s3:PutObject` and `s3:GetObject` on bucket `products/*`
- [ ] Test full flow via Postman:
    - [ ] Call `GET /api/products/{id}/upload-url` → receive `uploadUrl` and `imageKey`
    - [ ] Call `PUT {uploadUrl}` with image binary — direct to S3, no backend involved
    - [ ] Call `PUT /api/products/{id}` with `{ imageUrl: "https://bucket.s3.region.amazonaws.com/products/..." }`
    - [ ] Call `GET /api/products/{id}` — confirm `imageUrl` is returned in response

---

### Step 17 · Documentation & Interview Prep

#### README
- [ ] Architecture diagram in `README.md` (can embed the one from this project plan)
- [ ] Tech stack table — language, framework, DB, cache, messaging, cloud
- [ ] How to run locally — `docker compose up` + `./mvnw spring-boot:run` instructions
- [ ] How to run tests — single command
- [ ] AWS deployment architecture description

#### Decision Log
- [ ] Document why RDS over EC2-hosted PostgreSQL
- [ ] Document why pre-signed URLs over backend-proxied S3 uploads
- [ ] Document why WebClient over RestTemplate
- [ ] Document why Kafka over synchronous REST for notifications
- [ ] Document why monolith-first before microservice extraction
- [ ] Document why Redis for caching and rate limiting

#### Interview Practice
- [ ] Walk through the entire project end-to-end out loud — as if explaining to an interviewer
- [ ] Explain `@Transactional` using your order placement flow as the example
- [ ] Explain the circuit breaker — what happens when Product Service is down
- [ ] Explain the JWT flow — issue, validate at gateway, forward headers to services
- [ ] Explain the Kafka at-least-once guarantee and why the notification handler must be idempotent
- [ ] Explain the N+1 problem using your Product + Category query as the example
- [ ] Load test with k6 — 50 virtual users placing orders — watch circuit breaker trip in logs
- [ ] Answer: "Why did you choose this architecture?" — practice a 2-minute answer

---

## 🏁 Done Criteria

You are interview-ready when:

- [ ] All services run locally via `docker compose up` + individual `spring-boot:run` commands
- [ ] All unit and integration tests pass with `./mvnw test`
- [ ] Full flow works on AWS: register → login → browse products → place order → notification logged
- [ ] CI/CD deploys automatically on every push to `main`
- [ ] You can explain every architectural decision in the decision log without referring to notes
- [ ] You can draw the architecture diagram from memory in under 5 minutes