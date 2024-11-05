# GopherSocial API

GopherSocial is a social networking API designed for gophers, featuring a modular design, scalability, and essential tools like rate-limiting, caching, and email notifications. This guide covers setup, configuration, and usage.

## Features

- **User Authentication**: Secure token-based authentication.
- **Database and Caching**: PostgreSQL for relational data and Redis for caching.
- **Rate Limiting**: Protects the API from excessive requests.
- **Email Support**: SendGrid integration for email notifications.
- **Swagger Documentation**: API documentation at `/v1/swagger/index.html#/`.

## Requirements

- **Go** 1.23.2+
- **PostgreSQL** (for relational data)
- **Redis** (optional, for caching)
- **SMTP Server** (for email functionality)
- **SendGrid API Key** ([Get a SendGrid API Key](https://sendgrid.com/docs/ui/account-and-settings/api-keys/))
- **Docker and Docker Compose** (recommended for setting up dependencies)

## Project Structure

```plaintext
.
├── Dockerfile              # Docker configuration for building the API
├── docker-compose.yml      # Docker Compose setup for PostgreSQL and Redis
├── cmd/api                 # Main API application and routing
├── cmd/migrate             # Migration files
├── docs                    # Swagger documentation files
├── internal                # Internal packages (auth, db, env, mailer, ratelimiter, store)
├── scripts                 # Additional scripts
└── .envrc                  # Environment configuration file
```

## Setup Instructions

### 1. Clone the Repository

```bash
git clone https://github.com/yourusername/gophersocial.git
cd gophersocial
```

### 2. Environment Configuration

Use `.envrc` to set up environment variables. Here’s an example:

```bash
export ADDR=":8080"
export DB_ADDR="postgres://admin:adminpassword@localhost/socialnetwork?sslmode=disable"
export SENDGRID_API_KEY="your_sendgrid_api_key"
export AUTH_TOKEN_SECRET="your_auth_token_secret"
export FROM_EMAIL="your_email@example.com"
```

### 3. Run the Containers

Use Docker Compose to set up and run the required services:

```yaml
services:
    db:
        image: postgres:16.3
        container_name: postgres-db
        environment:
            POSTGRES_DB: socialnetwork
            POSTGRES_USER: admin
            POSTGRES_PASSWORD: adminpassword
        networks:
            - backend
        volumes:
            - db-data:/var/lib/postgresql/data
        ports:
            - "5432:5432"

    redis:
        image: redis:6.2-alpine
        container_name: redis
        ports:
            - "6379:6379"
        command: redis-server --save 60 1 --loglevel warning

    redis-commander:
        image: rediscommander/redis-commander
        ports:
            - "127.0.0.1:8081:8081"
        environment:
            REDIS_HOST: redis
        depends_on:
            - redis

volumes:
    db-data:
networks:
    backend:
        driver: bridge
```

Run the containers:

```bash
docker-compose up -d
```

### 4. Database Initialization

To initialize the PostgreSQL database, create a database named `social`:

```bash
createdb social
```

### 5. Run Migrations

Apply database migrations:

```bash
make migrate-up
```

### 6. Schema Diagram

![Database Schema](/schema.png)



## Running the Application

Use `air` for live reloading (recommended) or run the server manually.

### Running with Air

```bash
air
```

### Running Manually

```bash
go run cmd/api/main.go
```

The server should be accessible at [http://localhost:8080](http://localhost:8080).

## API Documentation

Swagger provides an interactive API documentation interface:

- **Swagger Documentation**: [http://localhost:8080/v1/swagger/index.html#/](http://localhost:8080/v1/swagger/index.html#/)

## Testing

### Rate Limiting

To test the rate limiter, use `autocannon`. Install it globally if needed:

```bash
npm install -g autocannon
```

Run the test to simulate a burst of 22 requests within 1 second:

```bash
npx autocannon -r 22 -d 1 -c 1 --renderStatusCodes http://localhost:8080/v1/health
```

### Health Check

You can verify the health of the API with:

```bash
curl http://localhost:8080/v1/health
```

## Docker Deployment

This Dockerfile builds the application and runs it in a minimal container:

```dockerfile
# Build Stage
FROM golang:1.22 as builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o api cmd/api/*.go

# Run Stage
FROM scratch
WORKDIR /app
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/api .
EXPOSE 8080
CMD ["./api"]
```

To build and run the Docker image:

```bash
docker build -t gophersocial-api .
docker run -p 8080:8080 gophersocial-api
```
