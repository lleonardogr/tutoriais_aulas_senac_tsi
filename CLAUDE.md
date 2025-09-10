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

### Tutorial Structure

The course is organized in a numbered sequence through `Writerside/in.tree`:

0. **00-troubleshooting-java-quarkus.md** - Troubleshooting guide for Java/Quarkus development setup
1. **01-intro-webservices-apis.md** - Introduction to WebServices and RESTful APIs
2. **02-consuming-apis-with-curl.md** - API consumption basics with curl  
3. **03-first-api-with-quarkus.md** - Creating first API with Quarkus
4. **04-crud-with-validation.md** - CRUD operations with validation
5. **05-filtering-sorting-pagination.md** - Advanced query features
6. **06-hateoas-with-quarkus.md** - HATEOAS implementation for self-descriptive APIs
7. **07-swagger-with-quarkus.md** - API documentation with Swagger
8. **08-api-key-quarkus.md** - API key authentication
9. **09-Rate-Limit-No-Quarkus.md** - Rate limiting implementation
10. **10-Idempotencia-no-quarkus.md** - Idempotency in APIs
11. **11-projeto-final-avaliacao.md** - Final project requirements and evaluation criteria

The learning path is defined in `starter.topic` with spotlight, primary, secondary, and misc sections for content organization.

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

## Development Commands

This repository is focused on documentation management rather than code compilation. Key commands:

### Writerside Documentation
- **Build documentation**: Automated via GitHub Actions (`.github/workflows/build-docs.yml`)
- **Local preview**: Use Writerside IDE or plugins for live preview
- **Topic structure**: Managed through `Writerside/in.tree` XML configuration
- **Categories and variables**: Defined in `Writerside/c.list` and `Writerside/v.list`

### Git Workflow
- Documentation is automatically built and deployed to GitHub Pages on push to `main` branch
- The build process uses JetBrains Writerside Docker builder
- Testing is performed via `writerside-checker-action` to validate documentation structure

## Content Guidelines

When editing tutorial content:
- **Language consistency**: All content must be in Portuguese (Brazilian Portuguese)
- **Progressive difficulty**: Follow the numbered sequence and maintain appropriate complexity escalation
- **Quarkus focus**: Preserve code examples that demonstrate Quarkus best practices and framework-specific features
- **Educational alignment**: Keep practical examples focused on TSI curriculum objectives
- **Writerside compliance**: Ensure markdown syntax is compatible with Writerside documentation system

## Architecture Overview

This is a **documentation-only repository** using JetBrains Writerside:
- No application code or build systems (Maven/Gradle)
- Automated documentation pipeline via GitHub Actions
- Tutorial content structured as progressive learning modules
- Focus on practical Quarkus API development education