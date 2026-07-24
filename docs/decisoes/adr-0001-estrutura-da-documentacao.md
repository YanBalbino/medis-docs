# ADR-0001 — Estrutura da Documentação

- **Status**: Aceito
- **Data**: 2026-07-03

## Contexto

Projeto MeDiS envolve múltiplos repositórios e múltiplas stacks. Conhecimento arquitetural não pode depender apenas de READMEs locais ou memória de mantenedores.

## Decisão

Adotar documentação central em `medis-docs` com organização inicial por:

- visão geral do sistema
- repositórios
- operação
- referência
- decisões arquiteturais/documentais

## Consequências

### Positivas
- visão transversal do ecossistema
- onboarding mais simples
- base para evolução incremental
- menor risco de conhecimento tácito

### Negativas
- exige disciplina de atualização
- pode divergir do código se não houver revisão contínua

## Próximos ADRs sugeridos

- arquitetura de integração entre serviços
- estratégia de configuração por ambiente
- convenção de contratos do serviço de modelos
