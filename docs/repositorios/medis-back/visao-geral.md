# `medis-back` — Visão Geral

## Papel no ecossistema

API principal do sistema. Centraliza regras de negócio, persistência e integrações.

## Responsabilidades principais

- autenticação
- CRUD de entidades clínicas
- gerenciamento de triagens
- integração com modelos de predição
- dados de dashboard
- persistência em PostgreSQL

## Stack principal

- NestJS
- TypeScript
- Prisma
- PostgreSQL
- JWT

## Conteúdo esperado desta seção

Futuras páginas devem detalhar:

- módulos Nest
- DTOs e entidades
- schema Prisma
- endpoints por domínio
- integração com `medis-modelos`
