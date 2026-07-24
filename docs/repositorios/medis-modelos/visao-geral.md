# `medis-modelos` — Visão Geral

## Papel no ecossistema

Serviço Django responsável por disponibilizar e executar modelos de predição usados nas triagens.

## Responsabilidades principais

- listar modelos disponíveis
- expor metadados de entrada
- executar predição
- devolver resposta interpretável ao ecossistema
- reutilizar modelos carregados

## Stack principal

- Python
- Django
- NumPy
- Pandas
- scikit-learn

## Conteúdo esperado desta seção

Futuras páginas devem detalhar:

- formato de `metadata.json`
- ciclo de vida de modelos
- cache de carregamento
- contratos REST
- papel de `positive_class_index`
