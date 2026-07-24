# `medis-modelos` — Estrutura

## Estrutura principal atual

```text
modelos/
  covid_predicao_internacao/
  covid_predicao_mortalidade/
  services/
prediction_models/
scripts/
manage.py
```

## Leitura inicial da estrutura

- `modelos/` — diretórios de modelos e serviços associados
- `modelos/services/` — suporte à carga e execução
- `prediction_models/` — componentes auxiliares do serviço
- `scripts/` — automações locais
- `manage.py` — entrada do Django

## Próximos detalhamentos

- estrutura interna de cada modelo
- convenções de metadados
- endpoints reais
- fluxo de carga e cache
