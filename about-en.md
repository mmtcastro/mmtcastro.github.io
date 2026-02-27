---
layout: page
title: About
permalink: /about-en/
---

English \| [Versao em portugues](/about/)

## About this blog

This blog was born from the need to document and share the lessons learned from a real-world project to modernize enterprise applications built on **HCL Domino/Notes**.

Over more than 20 years, we developed a complete ERP in Lotus Notes/Domino with modules for sales, purchasing, finance, inventory, services, HR, and business intelligence. Thousands of documents, hundreds of views, dozens of agents, and a solid foundation of business rules that support the company's daily operations.

The decision to modernize does not mean discarding what was built. On the contrary: it means evolving the architecture to current technologies while preserving data, processes, and — most importantly — the security model that Domino provides.

## The modernization stack

| Layer | Technology |
|---|---|
| **Backend** | Java 17+, Spring Boot, Spring Data JPA, Spring Security |
| **Frontend** | Vaadin Flow (server-side Java components) |
| **Integration** | HCL Domino REST API (DRAPI) |
| **Authentication** | LDAP (Domino Directory) + JWT (DRAPI tokens) |
| **Authorization** | ACL based on the HCL Domino security model |
| **Database** | PostgreSQL |

## Why share

The HCL Domino community is active, but documentation on integration with modern Java frameworks is still scarce. Every discovery we make — whether configuring JWT authentication via DRAPI, mapping Domino views to JPA queries, or replicating the ACL model in Spring Security — represents hours of research and experimentation.

By publishing these discoveries here, we hope to:

- **Reduce the learning curve** for those on the same path
- **Create a practical reference** for Domino + Spring Boot + Vaadin integration
- **Encourage discussion** about modernization strategies in the community

## About the author

**Marcelo Castro** — Technology professional with experience in development and architecture of enterprise solutions on the HCL Domino/Notes platform. Currently leading the modernization of a complete ERP to Java/Spring Boot with integration via Domino REST API.

- GitHub: [mmtcastro](https://github.com/mmtcastro)
