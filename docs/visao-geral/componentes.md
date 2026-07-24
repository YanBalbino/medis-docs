# Componentes do Ecossistema

## Sumário

Esta página descreve componentes principais e papéis no sistema.

## `medis-front`

Frontend em Next.js responsável por experiência do usuário.

### Responsabilidades principais
- autenticação no cliente
- navegação por fluxos de doença e triagem
- dashboard geográfico
- área administrativa
- integração com backend e serviço de modelos

## `medis-back`

Backend em NestJS responsável por domínio transacional.

### Responsabilidades principais
- autenticação JWT
- persistência em PostgreSQL via Prisma
- CRUD de entidades clínicas
- registro de triagens
- integração com predição
- suporte a mapas de calor e geocodificação

## `medis-modelos`

Serviço Django responsável por catálogo e execução de modelos.

### Responsabilidades principais
- listar modelos
- expor schema/metadados de entrada
- executar predição
- padronizar resposta dos modelos
- reutilizar modelos carregados

## Banco de dados

PostgreSQL usado pelo backend como base principal de persistência.

## Documentação central

`medis-docs` consolida visão transversal do ecossistema para reduzir dependência de conhecimento tácito.
