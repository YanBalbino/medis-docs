# Ambiente e Dependências

## Serviços do ecossistema

Para operação local completa, ambiente depende de:

- `medis-front`
- `medis-back`
- `medis-modelos`
- PostgreSQL

## Portas usuais

- frontend: `3000`
- backend: `3030`
- modelos: `8000`

## Dependências cruzadas

- frontend depende de backend e serviço de modelos
- backend depende de PostgreSQL e serviço de modelos
- serviço de modelos pode operar isolado para testes de predição

## Próximos detalhamentos

- matriz de dependências
- ordem recomendada de subida
- requisitos de runtime por stack
