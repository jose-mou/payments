# Payments App

A payments management API built as a set of event-driven microservices.

## Goal

The goal of this project is to provide a backend platform for managing payments end to end: initiating payments, processing them, and keeping a consistent record of every transaction. The system is decomposed into independent microservices that communicate primarily through events (Apache Kafka), with REST APIs exposed for client-facing operations.

## Repository Structure

Each top-level directory is an independent microservice — its own Spring Boot application, its own database, its own deployment unit.

> **Note:** Ideally each microservice would live in its own repository, with independent versioning, CI/CD pipelines, and access control. For simplicity, this project keeps all microservices in a single repository (monorepo), treating each directory as if it were a separate repository.

## Technology Stack

- Java 25 + Spring Boot 4
- Gradle
- PostgreSQL (one database per service)
- Apache Kafka (event-driven communication)
- JUnit 6 + Testcontainers

## Documentation

- [Architecture](doc/ARCHITECTURE.md)
- [Development practices (CLAUDE.md)](CLAUDE.md)
