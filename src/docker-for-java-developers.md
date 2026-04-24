---
title: "Docker for Java Developers"
description: "A practical guide to containerizing Java and Spring Boot applications with Docker — covering Dockerfiles, multi-stage builds, Docker Compose, and production tips."
date: 2026-04-14
tags: ["Java", "Docker", "DevOps"]
cover: "images/docker-cover.svg"
---

## Why Docker?

Running a Java app locally is easy. Getting it to behave the same way on a teammate's machine, a CI server, and production is not. Docker solves this by packaging your application and everything it needs — JDK, configs, dependencies — into a single portable container.

At Amdocs, every microservice in our billing platform ships as a Docker image. Here's what I've learned about doing it right.

## Your First Dockerfile

A basic Dockerfile for a Spring Boot app:

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY target/billing-service.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Build and run it:

```bash
docker build -t billing-service:1.0 .
docker run -p 8080:8080 billing-service:1.0
```

## Multi-Stage Builds

Don't ship the JDK and Maven in your production image. Use a multi-stage build to keep the final image lean:

```dockerfile
# Stage 1: build
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# Stage 2: run
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=builder /app/target/billing-service.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

This cuts image size significantly — a typical Spring Boot image goes from ~600MB down to ~200MB.

## Docker Compose for Local Development

Running your service alongside a database locally:

```yaml
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/billing
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: secret
    depends_on:
      - db

  db:
    image: postgres:16
    environment:
      POSTGRES_DB: billing
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

Start everything with one command:

```bash
docker compose up
```

## Useful Commands

```bash
# list running containers
docker ps

# view logs
docker logs -f billing-service

# open a shell inside a running container
docker exec -it billing-service sh

# remove stopped containers and unused images
docker system prune
```

## Key Takeaways

1. **Always use multi-stage builds** — never ship build tools in production images
2. **Use specific base image tags** — `eclipse-temurin:21-jre` not `latest`
3. **Externalize config** — pass environment variables, never bake secrets into images
4. **Add a `.dockerignore`** — exclude `target/`, `.git`, and test files to speed up builds

> A container that works on your laptop will work in production — that's the whole point.

Docker is now a baseline skill for any backend developer. Once you're comfortable with single containers, the natural next step is Kubernetes for orchestrating them at scale.
