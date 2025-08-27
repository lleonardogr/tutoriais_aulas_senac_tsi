# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a Portuguese educational repository containing Quarkus API development tutorials for TSI (Technical Systems for Internet) courses at SENAC. The content is organized as a structured learning path covering REST API development with the Quarkus framework.

## Documentation Structure

This repository uses **JetBrains Writerside** for documentation management:

- **Documentation source**: `Writerside/topics/` contains all tutorial markdown files
- **Configuration**: `Writerside/writerside.cfg` and `Writerside/in.tree` define the documentation structure
- **Images**: `Writerside/images/` contains tutorial screenshots and diagrams
- **Topic tree**: `in.tree` defines the hierarchical organization of tutorials

### Key Tutorial Topics

The course follows a progressive learning structure:

1. **intro-webservices-apis.md** - Introduction to WebServices and RESTful APIs
2. **consuming-apis-with-curl.md** - API consumption basics with curl
3. **first-api-with-quarkus.md** - Creating first API with Quarkus
4. **crud-with-validation.md** - CRUD operations with validation
5. **swagger-with-quarkus.md** - API documentation with Swagger
6. **api-key-quarkus.md** - API key authentication
7. **7-Rate-Limit-No-Quarkus.md** - Rate limiting implementation
8. **8-Idempotencia-no-quarkus.md** - Idempotency in APIs
9. **projeto-final-avaliacao.md** - Final project requirements and evaluation criteria

## Content Language and Context

- **Language**: All content is in Portuguese (Brazilian Portuguese)
- **Target audience**: TSI (Technical Systems for Internet) students at SENAC
- **Framework focus**: Quarkus (Java framework for cloud-native development)
- **Educational level**: Intermediate programming course covering REST API development

## Development Environment

Based on the tutorials, the expected development setup includes:
- **JDK 11+** (recommended JDK 17)
- **Maven 3.8.1+** or Gradle as build tools
- **Quarkus framework** - primary focus of the curriculum
- **H2 Database** for development and learning
- **Hibernate ORM with Panache** for data persistence

## Final Project Requirements

The course culminates in a comprehensive API project that must include:
- Minimum 3 entities with relationships (One-to-One, One-to-Many, Many-to-Many)
- Minimum 15 REST endpoints total (5 per entity)
- Bean Validation implementation
- Swagger documentation
- JWT authentication or API key security
- Rate limiting and idempotency features

## Writerside Commands

When working with documentation:
- Documentation builds and previews are handled through Writerside IDE or plugins
- Topic structure is defined in `in.tree` with XML format
- Categories and variables can be managed through `c.list` and `v.list` files

## File Editing Guidelines

When editing tutorial content:
- Maintain Portuguese language consistency
- Follow the established tutorial progression and difficulty curve
- Preserve code examples that demonstrate Quarkus best practices
- Keep practical examples focused on the educational objectives