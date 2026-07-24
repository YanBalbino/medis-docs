# `medis-back` — Estrutura

## Estrutura principal atual

```text
src/
  auth/
  criteria/
  dashboard/
  diseases/
  geocoding/
  health-units/
  prediction/
  questions/
  symptoms/
  triage/
  users/
prisma/
  migrations/
```

## Leitura inicial da estrutura

- `auth/` — autenticação e autorização
- `users/` — usuários
- `diseases/`, `questions/`, `symptoms/`, `criteria/` — domínio clínico
- `triage/` — execução e registro de triagens
- `prediction/` — integração com serviço de modelos
- `dashboard/` — dados agregados para visualização
- `health-units/` — unidades de saúde
- `geocoding/` — apoio geográfico
- `prisma/` — schema, migrações e seed

## Próximos detalhamentos

- mapa de módulos e dependências
- entidades persistidas
- fluxos por endpoint
- convenções Prisma
