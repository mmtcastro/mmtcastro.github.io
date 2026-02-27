---
layout: page
title: Sobre
permalink: /about/
---

## Sobre este blog

Este blog nasceu da necessidade de documentar e compartilhar os aprendizados de um projeto real de modernizacao de aplicacoes corporativas construidas sobre **HCL Domino/Notes**.

Ao longo de mais de 20 anos, desenvolvemos um ERP completo em Lotus Notes/Domino com modulos de vendas, compras, financeiro, estoque, servicos, RH e business intelligence. Sao milhares de documentos, centenas de views, dezenas de agentes e uma base sólida de regras de negocio que sustentam a operacao diaria da empresa.

A decisao de modernizar nao significa descartar o que foi construido. Pelo contrario: significa evoluir a arquitetura para tecnologias atuais, preservando dados, processos e, principalmente, o modelo de seguranca que o Domino oferece.

## A stack de modernizacao

| Camada | Tecnologia |
|---|---|
| **Backend** | Java 17+, Spring Boot, Spring Data JPA, Spring Security |
| **Frontend** | Vaadin Flow (componentes server-side em Java) |
| **Integracao** | HCL Domino REST API (DRAPI) |
| **Autenticacao** | LDAP (Domino Directory) + JWT (tokens DRAPI) |
| **Autorizacao** | ACL baseada no modelo de seguranca do HCL Domino |
| **Banco de dados** | PostgreSQL |

## Por que compartilhar

A comunidade HCL Domino e ativa, mas a documentacao sobre integracao com frameworks Java modernos ainda e escassa. Cada descoberta que fazemos — seja configurar autenticacao JWT via DRAPI, mapear views do Domino para consultas JPA, ou reproduzir o modelo de ACL em Spring Security — representa horas de pesquisa e experimentacao.

Ao publicar essas descobertas aqui, esperamos:

- **Reduzir a curva de aprendizado** para quem esta no mesmo caminho
- **Criar uma referencia pratica** de integracao Domino + Spring Boot + Vaadin
- **Estimular a discussao** sobre estrategias de modernizacao na comunidade

## Sobre o autor

**Marcelo Castro** — Profissional de tecnologia com experiencia em desenvolvimento e arquitetura de solucoes corporativas sobre a plataforma HCL Domino/Notes. Atualmente conduzindo a modernizacao de um ERP completo para Java/Spring Boot com integracao via Domino REST API.

- GitHub: [mmtcastro](https://github.com/mmtcastro)
