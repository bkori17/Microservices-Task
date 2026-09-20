# Microservices Architecture

## 1. Overview

This project is a small Node.js/Express microservices application containing four services:

1. **User Service** — manages and exposes user data.
2. **Product Service** — manages and exposes product data.
3. **Order Service** — creates and returns orders.
4. **Gateway Service** — provides a single API entry point and forwards requests to the backend services.

All services run as independent Docker containers and are orchestrated using Docker Compose.

The application is currently deployed and tested on an Ubuntu EC2 instance.

---

## 2. High-Level Architecture

```text
                         Client / curl
                              |
                              |
                    http://EC2:3003/api/*
                              |
                              v
                   +----------------------+
                   |    Gateway Service   |
                   |      Port 3003       |
                   +----------+-----------+
                              |
                    Docker Compose Network
                 _____________|______________
                /             |              \
               /              |               \
              v               v                v
    +----------------+ +----------------+ +----------------+
    |  User Service  | | Product Service| | Order Service  |
    |   Port 3000    | |   Port 3001    | |   Port 3002   |
    +----------------+ +----------------+ +----------------+
         /users           /products          /orders
                                              GET / POST
```

Docker Compose automatically creates a private network for the application.

The Gateway communicates with the backend services by using their Docker Compose service names:

```text
http://user-service:3000
http://product-service:3001
http://order-service:3002
```

Inside Docker, these service names are resolved through Docker's internal DNS.

---

## 3. Deployment Architecture

```text
AWS EC2 - Ubuntu
|
+-- Docker Engine
|
+-- Docker Compose
    |
    +-- user-service container
    |     Port 3000
    |
    +-- product-service container
    |     Port 3001
    |
    +-- order-service container
    |     Port 3002
    |
    +-- gateway-service container
          Port 3003
```

Each microservice runs in its own container and Node.js process.

---

## 4. Project Directory Structure

```text
microservices/
|
+-- Dockerfile
+-- docker-compose.yml
|
+-- user-service/
|   +-- app.js
|   +-- package.json
|
+-- product-service/
|   +-- app.js
|   +-- package.json
|
+-- order-service/
|   +-- app.js
|   +-- package.json
|
+-- gateway-service/
    +-- app.js
    +-- package.json
```

A **single common Dockerfile** is reused for all four services.

The Docker Compose `build.context` determines which service directory is passed to that Dockerfile.

---

## 5. Service Summary

| Service | Container Name | Port | Main Responsibility |
|---|---|---:|---|
| User Service | `user-service` | 3000 | Returns user information |
| Product Service | `product-service` | 3001 | Returns product information |
| Order Service | `order-service` | 3002 | Creates and returns orders |
| Gateway Service | `gateway-service` | 3003 | API entry point and service routing |

---

## 6. User Service

### Responsibility

The User Service provides user information.

### Port

```text
3000
```

### APIs

#### Health Check

```http
GET /health
```

Example:

```bash
curl http://localhost:3000/health
```

#### List Users

```http
GET /users
```

Example:

```bash
curl http://localhost:3000/users
```

Example response:

```json
[
  {
    "id": 1,
    "name": "John Doe"
  },
  {
    "id": 2,
    "name": "Jane Smith"
  }
]
```

The current implementation contains static in-memory user data.

---

## 7. Product Service

### Responsibility

The Product Service provides product information.

### Port

```text
3001
```

### APIs

#### Health Check

```http
GET /health
```

Example:

```bash
curl http://localhost:3001/health
```

#### List Products

```http
GET /products
```

Example:

```bash
curl http://localhost:3001/products
```

Example response:

```json
[
  {
    "id": 1,
    "name": "Laptop",
    "price": 999
  },
  {
    "id": 2,
    "name": "Phone",
    "price": 699
  }
]
```

The current implementation contains static in-memory product data.

---

## 8. Order Service

### Responsibility

The Order Service creates and returns orders.

### Port

```text
3002
```

### APIs

#### Health Check

```http
GET /health
```

Example:

```bash
curl http://localhost:3002/health
```

#### List Orders

```http
GET /orders
```

Example:

```bash
curl http://localhost:3002/orders
```

Initially the response is:

```json
[]
```

#### Create Order

```http
POST /orders
```

Example:

```bash
curl -X POST http://localhost:3002/orders \
  -H "Content-Type: application/json" \
  -d '{"userId":1,"productId":1}'
```

Example response:

```json
{
  "id": 1,
  "userId": 1,
  "productId": 1,
  "timestamp": "..."
}
```

### Important Design Note

Orders are currently stored in a JavaScript array:

```text
const orders = [];
```

This means order data exists only in the container's process memory.

If the Order Service container is restarted or recreated, the order data is lost.

For a production system, this should be replaced with persistent storage such as PostgreSQL, MySQL, DynamoDB, MongoDB, or another suitable database.

---

## 9. Gateway Service

### Responsibility

The Gateway Service acts as the application entry point.

Instead of clients needing to know all backend service locations, they can access the services through port `3003`.

### Port

```text
3003
```

### Gateway APIs

#### Gateway Health

```bash
curl http://localhost:3003/health
```

#### Users Through Gateway

```bash
curl http://localhost:3003/api/users
```

Internal call:

```text
Gateway
   |
   +--> http://user-service:3000/users
```

#### Products Through Gateway

```bash
curl http://localhost:3003/api/products
```

Internal call:

```text
Gateway
   |
   +--> http://product-service:3001/products
```

#### Orders Through Gateway

```bash
curl http://localhost:3003/api/orders
```

Internal call:

```text
Gateway
   |
   +--> http://order-service:3002/orders
```

#### Create Order Through Gateway

```bash
curl -X POST http://localhost:3003/api/orders \
  -H "Content-Type: application/json" \
  -d '{"userId":1,"productId":1}'
```

Internal flow:

```text
Client
   |
   | POST /api/orders
   v
Gateway Service :3003
   |
   | POST /orders
   v
Order Service :3002
   |
   v
Order created
```

---

## 10. Request Flow

### User Request Through Gateway

```text
1. Client sends:
   GET http://localhost:3003/api/users

2. Docker maps host port 3003 to the Gateway container.

3. Express in Gateway receives:
   GET /api/users

4. Gateway executes:
   axios.get("http://user-service:3000/users")

5. Docker DNS resolves:
   user-service -> User Service container IP

6. User Service receives:
   GET /users

7. User Service returns JSON.

8. Gateway receives the JSON.

9. Gateway sends it back to the client.
```

The same pattern is used for Product and Order requests.

---

## 11. Dockerfile

The project uses one common Dockerfile:

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install --omit=dev

COPY . .

CMD ["node", "app.js"]
```

### Dockerfile Explanation

#### Base Image

```dockerfile
FROM node:18-alpine
```

Uses the lightweight Alpine Linux Node.js 18 image.

#### Working Directory

```dockerfile
WORKDIR /app
```

All following commands operate inside `/app` in the image.

#### Copy Dependency Definition

```dockerfile
COPY package*.json ./
```

Copies the service's `package.json`.

#### Install Dependencies

```dockerfile
RUN npm install --omit=dev
```

Installs runtime dependencies.

Current dependencies are:

```text
express
axios
```

#### Copy Application

```dockerfile
COPY . .
```

Copies the selected service's application files into the container.

#### Start Application

```dockerfile
CMD ["node", "app.js"]
```

Starts the Node.js service.

---

## 12. Why One Dockerfile Works for Four Services

Docker Compose provides a different build context to the same Dockerfile.

Example:

```yaml
user-service:
  build:
    context: ./user-service
    dockerfile: ../Dockerfile
```

For this build:

```text
COPY . .
```

means:

```text
copy contents of ./user-service
```

For Product Service:

```yaml
context: ./product-service
```

the same Dockerfile copies the Product Service files.

Therefore one common Dockerfile can build four different images.

---

## 13. Docker Compose Configuration

```yaml
services:

  user-service:
    build:
      context: ./user-service
      dockerfile: ../Dockerfile
    container_name: user-service
    ports:
      - "3000:3000"

  product-service:
    build:
      context: ./product-service
      dockerfile: ../Dockerfile
    container_name: product-service
    ports:
      - "3001:3001"

  order-service:
    build:
      context: ./order-service
      dockerfile: ../Dockerfile
    container_name: order-service
    ports:
      - "3002:3002"

  gateway-service:
    build:
      context: ./gateway-service
      dockerfile: ../Dockerfile
    container_name: gateway-service
    ports:
      - "3003:3003"
    depends_on:
      - user-service
      - product-service
      - order-service
```

---

## 14. Docker Compose Networking

Docker Compose automatically creates a network similar to:

```text
microservices_default
```

All four containers join this network.

Conceptually:

```text
microservices_default
|
+-- user-service
|
+-- product-service
|
+-- order-service
|
+-- gateway-service
```

The Gateway does **not** use `localhost` to communicate with backend containers.

For example, this is correct:

```text
http://user-service:3000/users
```

This would be incorrect from inside the Gateway container:

```text
http://localhost:3000/users
```

Inside a container, `localhost` refers to that same container.

---

## 15. Docker Port Mapping

Example:

```yaml
ports:
  - "3000:3000"
```

Format:

```text
HOST_PORT:CONTAINER_PORT
```

Therefore:

```text
EC2 host port 3000
        |
        v
User container port 3000
```

Current mappings:

```text
Host 3000 -> User container 3000
Host 3001 -> Product container 3001
Host 3002 -> Order container 3002
Host 3003 -> Gateway container 3003
```

---

## 16. `depends_on`

Gateway configuration:

```yaml
depends_on:
  - user-service
  - product-service
  - order-service
```

This tells Docker Compose to start the backend service containers as dependencies of the Gateway.

Note that basic `depends_on` controls container start ordering; it does not by itself guarantee that an application is fully ready to accept requests.

For a more production-oriented solution, health checks and dependency readiness conditions can be added.

---

## 17. Build Process

From the project root:

```bash
cd ~/microservices
```

Validate Compose:

```bash
docker compose config
```

Build all images:

```bash
docker compose build
```

Or build and start:

```bash
docker compose up -d --build
```

Conceptual build flow:

```text
docker compose up -d --build
        |
        +--> Build User image
        |
        +--> Build Product image
        |
        +--> Build Order image
        |
        +--> Build Gateway image
        |
        +--> Create Compose network
        |
        +--> Create containers
        |
        +--> Start containers
```

---

## 18. Runtime Verification

Check all services:

```bash
docker compose ps
```

Check Docker containers:

```bash
docker ps
```

Check logs:

```bash
docker compose logs
```

Follow logs:

```bash
docker compose logs -f
```

Individual service:

```bash
docker logs user-service
docker logs product-service
docker logs order-service
docker logs gateway-service
```

---

## 19. Functional Test Commands

### Direct Backend Tests

```bash
curl http://localhost:3000/users
curl http://localhost:3001/products
curl http://localhost:3002/orders
```

### Gateway Tests

```bash
curl http://localhost:3003/api/users
curl http://localhost:3003/api/products
curl http://localhost:3003/api/orders
```

### Create Order

```bash
curl -X POST http://localhost:3003/api/orders \
  -H "Content-Type: application/json" \
  -d '{"userId":1,"productId":1}'
```

Then verify:

```bash
curl http://localhost:3003/api/orders
```

---

## 20. Health Check Commands

```bash
curl http://localhost:3000/health
curl http://localhost:3001/health
curl http://localhost:3002/health
curl http://localhost:3003/health
```

---

## 21. Container-to-Container Verification

To verify Docker DNS:

```bash
docker exec gateway-service ping -c 2 user-service
```

To test User Service from inside Gateway:

```bash
docker exec gateway-service wget -qO- \
  http://user-service:3000/users
```

Similarly:

```bash
docker exec gateway-service wget -qO- \
  http://product-service:3001/products
```

```bash
docker exec gateway-service wget -qO- \
  http://order-service:3002/orders
```

---

## 22. Start, Stop, and Rebuild

### Start

```bash
docker compose up -d
```

### Stop and remove project containers

```bash
docker compose down
```

### Rebuild after source changes

```bash
docker compose up -d --build
```

### Clean rebuild

```bash
docker compose down
docker compose build --no-cache
docker compose up -d
```

---

## 23. Important Design Characteristics

### Service Isolation

Each service:

- has its own Node.js process;
- runs in its own Docker container;
- owns its own application port;
- can be built independently;
- can be restarted independently.

### Synchronous Communication

The Gateway communicates with backend services synchronously using HTTP and Axios.

Example:

```text
Gateway -> HTTP -> User Service
```

There is no message queue in the current design.

### Service Discovery

Docker Compose provides service discovery using service names.

Example:

```text
user-service
product-service
order-service
```

No hard-coded container IP addresses are required.

### Data Storage

Current implementation:

```text
Users    -> hard-coded in application memory
Products -> hard-coded in application memory
Orders   -> runtime in-memory array
```

No external database is currently used.

---

## 24. Failure Behavior

If User Service is down and the client calls:

```text
GET /api/users
```

the Gateway catches the Axios error and returns:

```json
{
  "error": "Error fetching users"
}
```

The same pattern is used for Product and Order failures.

This is basic error handling. A production implementation could add retries, timeouts, circuit breakers, structured error responses, and centralized logging.

---

## 25. Security Considerations

For the current exercise, backend service ports are published to the EC2 host for direct testing.

A more production-oriented deployment would normally expose only the Gateway externally and keep ports `3000`, `3001`, and `3002` private inside the Docker network.

Conceptually:

```text
Internet
   |
   v
Gateway :3003
   |
Docker private network
   |
   +-- User :3000
   +-- Product :3001
   +-- Order :3002
```

Additional production security could include:

- TLS/HTTPS
- authentication and authorization
- API input validation
- secrets management
- restricted EC2 Security Group rules
- non-root container users
- vulnerability scanning
- dependency updates

These are future improvements and are not part of the current application implementation.

---

## 26. Current Limitations

The current implementation is suitable for demonstrating Dockerized microservices, but it is not a production architecture.

Known limitations include:

1. No persistent database.
2. Orders are lost after Order Service restart.
3. User and Product data are static.
4. No authentication or authorization.
5. No request validation.
6. No centralized logging.
7. No metrics or monitoring.
8. No distributed tracing.
9. No retries or circuit breaker.
10. No API rate limiting.
11. No Docker Compose health checks.
12. Backend ports are directly exposed for testing.
13. No service autoscaling.
14. No load balancer.
15. No orchestration platform such as ECS or Kubernetes.

---

## 27. Possible Production Evolution

A future architecture could look like:

```text
                       Internet
                           |
                           v
                  Application Load Balancer
                           |
                           v
                     API Gateway
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
     User Service     Product Service   Order Service
          |                |                |
          v                v                v
       Database         Database         Database
```

On AWS, the containers could later be moved to:

```text
ECR -> ECS/Fargate -> ALB
```

where:

- **ECR** stores Docker images.
- **ECS** runs and manages containers.
- **Fargate** can provide serverless container compute.
- **ALB** distributes incoming traffic.
- **CloudWatch** provides logs and monitoring.

This is a potential extension, not part of the current EC2 + Docker Compose implementation.

---

## 28. Troubleshooting Checklist

### Check Docker

```bash
docker --version
docker compose version
```

### Check running services

```bash
docker compose ps
```

### Check logs

```bash
docker compose logs --tail=100
```

### Check a specific service

```bash
docker logs gateway-service
```

### Validate Compose

```bash
docker compose config
```

### Test backend directly

```bash
curl http://localhost:3000/users
```

### Test through Gateway

```bash
curl http://localhost:3003/api/users
```

### Test Docker DNS

```bash
docker exec gateway-service ping -c 2 user-service
```

---

## 29. Quick Architecture Summary

```text
Technology:
  Node.js
  Express
  Axios
  Docker
  Docker Compose
  Ubuntu EC2

Services:
  User      :3000
  Product   :3001
  Order     :3002
  Gateway   :3003

Communication:
  REST/HTTP

Service Discovery:
  Docker Compose DNS

Container Network:
  Docker Compose default network

External Entry Point:
  Gateway Service

Data:
  In-memory only

Build:
  One shared Dockerfile
  Four build contexts

Orchestration:
  docker-compose.yml
```

---

## 30. Final End-to-End Flow

```text
curl http://localhost:3003/api/users
                  |
                  v
        EC2 Host Port 3003
                  |
                  v
        Gateway Container
                  |
                  | Docker DNS:
                  | user-service
                  v
        User Service Container
                  |
                  v
             GET /users
                  |
                  v
             JSON Response
                  |
                  v
        Gateway Container
                  |
                  v
              Client
```

This demonstrates the main microservices concepts used in the project:

- independent services,
- independent containers,
- API Gateway pattern,
- REST communication,
- Docker service discovery,
- Docker networking,
- container image creation,
- multi-container orchestration with Docker Compose.
