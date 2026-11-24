# 0010 Technology stack: Spring Boot 4, Java 21, Angular 21

- Status: accepted
- Date: 2026-11-02

## Context
The system must be modern and supported for years. Spring Boot 3.5's open-source support ended in mid-2026.

## Decision

| Layer | Choice |
|---|---|
| Language | Java 21: records, sealed types, pattern matching, virtual threads (`spring.threads.virtual.enabled`) |
| Framework | Spring Boot 4.1, Spring Modulith 2.1, Hibernate 7 (JPA 3.2), Jackson 3 |
| Database | PostgreSQL 17, Flyway |
| API docs | springdoc 3 (OpenAPI 3.1) |
| Build | Maven multi-module, Spotless with Google Java Format, ArchUnit |
| Web | Angular 21: standalone components, signals, built-in control flow, functional guards and interceptors, lazy routes |
| Local / tests | Embedded PostgreSQL (zonky), with the production roles |

## Update, 24 November 2026

The stack has moved up its own release lines since this decision, which stands:

| Layer | Now |
|---|---|
| Language | Java 21 bytecode (`java.version` 21), run on a Java 25 runtime (Eclipse Temurin 25 in CI and the image) |
| Framework | Spring Boot 4.2, Spring Modulith 2.2, springdoc 3.2 |
| Database | PostgreSQL 18 (embedded PostgreSQL 18 locally and in tests) |
| Web | Angular 22, built and served with Node 26 |

## Consequences
- Long support windows and current idioms.
- Boot 4 moved packages (Jackson 3, modular auto-configuration), so some older examples on the web won't apply as written.
