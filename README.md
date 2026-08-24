# Personal Microservices Project

This project is a personal learning project designed to explore microservices architecture and develop practical backend skills with Spring Boot and Spring Cloud.

The system is composed of independent services for customers, products, billing, service discovery, and API routing. Each business service owns its own database and communicates with other services through REST APIs.

## Architecture

```text
                       ┌──────────────────────┐
                       │      API Gateway      │
                       │        :8080          │
                       └──────────┬───────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
              ▼                   ▼                   ▼
      Customer Service     Inventory Service    Billing Service
          :8081                 :8082               :8083
          MySQL                 MySQL               MySQL
              │                   │                   │
              └───────────────────┴───────────────────┘
                                  │
                       ┌──────────▼───────────┐
                       │   Eureka Discovery    │
                       │        :8761          │
                       └──────────────────────┘
```

Billing Service communicates with Customer Service and Inventory Service using OpenFeign. The API Gateway uses Eureka and Spring Cloud LoadBalancer to discover service instances dynamically.

## Services

| Service | Responsibility | Port | Database |
|---|---|---:|---|
| `discovery-server` | Eureka service registry | `8761` | None |
| `customer-service` | Customer management | `8081` | `customer_db` |
| `inventory-service` | Product management | `8082` | `inventory_db` |
| `billing-service` | Invoice management | `8083` | `billing_db` |
| `api-gateway` | Single entry point and routing | `8080` | None |

## Technologies

- Java 21
- Spring Boot 3.5.6
- Spring Cloud 2025.0.0
- Spring Data JPA
- MySQL 8
- Spring Cloud Gateway Server WebFlux
- Netflix Eureka
- Spring Cloud OpenFeign
- Maven
- JUnit 5 and Mockito

## Project Structure

```text
micro-services project/
├── customer-service/
├── inventory-service/
├── billing-service/
├── discovery-server/
├── api-gateway/
├── class-diag.png
└── arch.png
```

## Prerequisites

- Java 21 or later
- Maven
- MySQL 8 or Docker Desktop
- Git, if you want to contribute or push changes to GitHub

## Database Configuration

Each business service uses a separate database:

```text
customer_db
inventory_db
billing_db
```

By default, the services use:

```text
username: root
password: root
host: localhost
port: 3306
```

If your MySQL password is different, configure it with environment variables before starting the corresponding service.

Example for Customer Service:

```powershell
$env:CUSTOMER_DB_USERNAME="root"
$env:CUSTOMER_DB_PASSWORD="your-password"
```

Example for Billing Service:

```powershell
$env:BILLING_DB_USERNAME="root"
$env:BILLING_DB_PASSWORD="your-password"
```

The databases are created automatically when the configured MySQL user has sufficient permissions.

## Start the Application

Start the services in this order:

### 1. Start Eureka

```powershell
cd discovery-server
mvn spring-boot:run
```

Open the Eureka dashboard:

```text
http://localhost:8761
```

### 2. Start Customer Service

```powershell
cd customer-service
mvn spring-boot:run
```

### 3. Start Inventory Service

```powershell
cd inventory-service
mvn spring-boot:run
```

### 4. Start Billing Service

```powershell
cd billing-service
mvn spring-boot:run
```

### 5. Start API Gateway

```powershell
cd api-gateway
mvn spring-boot:run
```

All client requests should go through the Gateway:

```text
http://localhost:8080
```

If port `8080` is already used by another application, use another port temporarily:

```powershell
$env:SERVER_PORT="8085"
mvn spring-boot:run
```

## API Examples

### Create a customer

```http
POST http://localhost:8080/api/customers
Content-Type: application/json

{
  "name": "Ahmed Alaoui",
  "email": "ahmed@example.com"
}
```

### Create a product

```http
POST http://localhost:8080/api/products
Content-Type: application/json

{
  "name": "Mechanical Keyboard",
  "price": 249.90
}
```

### Create a bill

```http
POST http://localhost:8080/api/bills
Content-Type: application/json

{
  "customerId": 1,
  "items": [
    {
      "productId": 1,
      "quantity": 2
    }
  ]
}
```

### Read resources

```text
GET http://localhost:8080/api/customers
GET http://localhost:8080/api/products
GET http://localhost:8080/api/bills
```

## Health Checks

Each service exposes health information through Actuator:

```text
GET http://localhost:8081/actuator/health
GET http://localhost:8082/actuator/health
GET http://localhost:8083/actuator/health
GET http://localhost:8761/actuator/health
GET http://localhost:8080/actuator/health
```

## Testing

Run the tests inside each service directory:

```powershell
mvn test
```

The project currently includes unit tests for the Customer, Inventory, and Billing services, as well as startup tests for the Discovery Server and API Gateway.

## Current Status

Implemented:

- Customer Service
- Inventory Service
- Billing Service
- Eureka Discovery Server
- API Gateway with dynamic service discovery
- OpenFeign communication between Billing, Customer, and Inventory
- MySQL persistence for business services

Planned improvements:

- Angular frontend
- Authentication and authorization with JWT
- Docker Compose for the complete system
- Circuit breakers and resilience patterns
- Centralized configuration
- Monitoring and distributed tracing
- CI/CD with GitHub Actions

## Learning Goals

This project is intended to practice:

- Microservice boundaries and independent databases
- REST API design
- Service discovery
- API Gateway routing
- Synchronous service-to-service communication
- Data consistency and historical invoice prices
- Testing distributed application components
