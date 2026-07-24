# Documentação do MeDiS

Documentação central do projeto **MeDiS**.

## Objetivo

Estabelecer visão geral durável do sistema para manutenção futura, onboarding técnico e evolução controlada.

## Escopo inicial

Esta primeira versão prioriza:

- contexto do sistema
- arquitetura geral
- componentes do ecossistema
- fluxos principais
- estrutura inicial dos três repositórios
- operação local em alto nível

## Repositórios do ecossistema

- `medis-front` — frontend Next.js
- `medis-back` — backend NestJS + Prisma + PostgreSQL
- `medis-modelos` — serviço Django para gerenciamento e execução de modelos de predição
- `medis-docs` — documentação central em MkDocs

## Como navegar

- **Visão Geral**: entendimento macro do sistema
- **Repositórios**: descrição resumida de cada código-base
- **Operação**: execução e dependências
- **Referência**: contratos e convenções
- **Decisões**: registros de arquitetura e documentação

## Próximos incrementos

- detalhamento de módulos
- contratos de API com exemplos reais
- diagramas de fluxo por caso de uso
- decisões arquiteturais adicionais
