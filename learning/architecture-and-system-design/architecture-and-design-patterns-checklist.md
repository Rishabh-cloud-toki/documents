# Architecture & Design Patterns — The High-Impact 20%

> The areas that come up in almost every architect-level discussion.
> Tick items off once you can explain the concept *and* point to where you've used it.

## Contents

- [Part 1 — Architecture](#part-1--architecture)
  - [Architectural Styles & Patterns](#1-architectural-styles--patterns)
  - [System Design Essentials](#2-system-design-essentials)
  - [Cloud & DevOps Integration](#3-cloud--devops-integration)
  - [Security Architecture](#4-security-architecture)
  - [Observability & Non-Functionals](#5-observability--non-functionals)
- [Part 2 — Design Patterns](#part-2--design-patterns-the-core-20-to-know-cold)
  - [Creational](#1-creational-patterns)
  - [Structural](#2-structural-patterns)
  - [Behavioral](#3-behavioral-patterns)
  - [Concurrency](#4-concurrency-patterns)
  - [Enterprise Integration (EIP)](#5-enterprise-integration-patterns-eip)
- [Bonus — Spring Boot Patterns & Practices](#bonus--spring-boot-specific-patterns--practices)

---

# Part 1 — Architecture

## 1. Architectural Styles & Patterns

- [ ] Monolith vs. Microservices vs. Modular Monolith
- [ ] Hexagonal (Ports & Adapters)
- [ ] Layered Architecture
- [ ] Event-Driven Architecture (EDA)
- [ ] Service-Oriented Architecture (SOA)
- [ ] Serverless / cloud-native architectures

## 2. System Design Essentials

- [ ] High Availability (HA), scalability (horizontal vs. vertical), and fault tolerance
- [ ] CAP theorem and its trade-offs
- [ ] Load balancing, caching, and rate limiting
- [ ] Async messaging (Kafka, RabbitMQ)
- [ ] Distributed systems basics — network partitions, consistency models

## 3. Cloud & DevOps Integration

- [ ] 12-Factor App principles
- [ ] Containers (Docker) and orchestration (Kubernetes)
- [ ] CI/CD (Jenkins, GitHub Actions) and deployment strategies (blue/green, canary)

## 4. Security Architecture

- [ ] Authentication & authorization (OAuth2, JWT)
- [ ] API gateway, CORS, SSL/TLS
- [ ] Data encryption at rest and in transit

## 5. Observability & Non-Functionals

- [ ] Monitoring, logging, tracing (ELK, Prometheus, Grafana)
- [ ] SLAs, SLOs, and SLIs
- [ ] Performance tuning and profiling

---

# Part 2 — Design Patterns: The Core 20% to Know Cold

Most frequently discussed in system design, interviews, and real-world implementations.

## 1. Creational Patterns

| Pattern | Why it matters |
| --- | --- |
| **Singleton** | Know the thread-safety pitfalls (double-checked locking, enum, holder idiom) |
| **Factory Method** | Decoupling object creation from usage |
| **Builder** | Object construction, especially around Spring beans and immutable DTOs |

## 2. Structural Patterns

| Pattern | Why it matters |
| --- | --- |
| **Adapter** | Legacy integration |
| **Decorator** | Dynamic behavior — e.g. Spring Security filter chains |
| **Facade** | Simplifying complex APIs — e.g. service aggregators |

## 3. Behavioral Patterns

| Pattern | Why it matters |
| --- | --- |
| **Strategy** | Business-logic decision trees |
| **Observer** | Pub/sub, especially in event-driven architectures |
| **Template Method** | Used throughout Spring — `JdbcTemplate`, `RestTemplate` |

## 4. Concurrency Patterns

- [ ] Thread Pool
- [ ] Future / `CompletableFuture`
- [ ] Producer–Consumer

## 5. Enterprise Integration Patterns (EIP)

- [ ] Message Channel
- [ ] Message Router
- [ ] Content Enricher

> Especially relevant for microservice intercommunication via Spring Integration / Spring Cloud Stream.

---

## Bonus — Spring Boot-Specific Patterns & Practices

- [ ] Spring Cloud Config and service discovery (Eureka, Consul)
- [ ] Circuit breaker (Resilience4j)
- [ ] Centralized logging & tracing (Micrometer Tracing / Sleuth, Zipkin)
- [ ] Reactive architectures (WebFlux, Project Reactor)

---

## Notes

<!-- Concrete examples from your own projects go here — one per pattern is enough. -->