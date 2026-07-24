# Contrato de `metadata.json`

## Objetivo

Definir formato oficial para descrever modelos de predição no MeDiS.

O metadata é consumido por:

- serviço Django, durante carga e predição
- frontend, para montar formulários dinamicamente
- administradores, para identificar e versionar modelos
- documentação e ferramentas de validação

> Esta página formaliza contrato atual e proposta de formato canônico. Ela não altera automaticamente modelos ou código.

## Estrutura mínima

Cada modelo deve possuir diretório próprio:

```text
modelos/
  meu_modelo/
    __init__.py
    metadata.json
    modelo.pkl ou modelo.pk1
```

`medis-modelos` descobre subdiretórios contendo `metadata.json` e carrega o módulo Python correspondente.

## Formato canônico

```json
{
  "model": {
    "slug": "covid_predicao_mortalidade",
    "name": "Covid Predição Mortalidade",
    "version": "1.0.0",
    "description": "Modelo de predição usando GradientBoostingClassifier",
    "algorithm": "GradientBoostingClassifier",
    "library": "sklearn",
    "prediction_type": "mortalidade",
    "n_features": 8,
    "created_at": "2026-05-29T18:06:19.594080",
    "updated_at": "2026-05-29T18:06:19.594080"
  },
  "features": [
    {
      "name": "faixaetaria",
      "label": "Faixa etária",
      "type": "int",
      "field_type": "select",
      "required": true,
      "order": 1,
      "description": "Categoria etária do paciente",
      "options": [
        {
          "label": "0-11 anos",
          "value": 0
        },
        {
          "label": "12-17 anos",
          "value": 1
        }
      ]
    }
  ],
  "output": {
    "type": "mortalidade",
    "positive_class_index": 0,
    "thresholds": {
      "low": 0.15,
      "high": 0.4
    },
    "messages": {
      "low": "Risco baixo de mortalidade",
      "moderate": "Risco moderado de mortalidade",
      "high": "Risco alto de mortalidade"
    }
  }
}
```

---

## 1. Seção `model`

Identifica modelo e fornece informações para interface, administração e rastreabilidade.

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---:|---|
| `slug` | string | sim | Identificador único usado nas URLs |
| `name` | string | sim | Nome exibido ao usuário |
| `version` | string | recomendado | Versão do modelo/metadados |
| `description` | string | recomendado | Descrição funcional |
| `algorithm` | string | recomendado | Algoritmo usado |
| `library` | string | recomendado | Biblioteca de treinamento/execução |
| `prediction_type` | string | recomendado | Resultado previsto, por exemplo `mortalidade` |
| `n_features` | integer | recomendado | Número esperado de features |
| `created_at` | ISO 8601 string | recomendado | Data de criação |
| `updated_at` | ISO 8601 string | recomendado | Data da última atualização |

### Regras

- `slug` deve estar em snake_case
- usar somente letras minúsculas, números e `_`
- `slug` deve corresponder ao diretório do modelo
- `name` e `description` são nomes canônicos
- `nome` e `descricao` pertencem ao formato legado e não devem ser usados em novos modelos

Exemplo válido:

```text
covid_predicao_mortalidade
```

Exemplos inválidos:

```text
Covid Predição Mortalidade
covid-predicao-mortalidade
covid/predicao
```

---

## 2. Seção `features`

`features` é lista ordenada de campos que modelo espera receber.

### Campos de feature

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---:|---|
| `name` | string | sim | Nome técnico enviado ao modelo |
| `label` | string | recomendado | Nome exibido na interface |
| `type` | string | sim | Tipo do valor enviado ao modelo |
| `field_type` | string | sim | Tipo de controle visual |
| `required` | boolean | sim | Indica se campo deve ser preenchido |
| `order` | integer | sim | Ordem de exibição |
| `description` | string | recomendado | Explicação do campo |
| `options` | array | condicional | Opções para campos enumerados |
| `validation` | object | condicional | Limites e regras numéricas |
| `unit` | string | opcional | Unidade de medida |

### Valores aceitos em `type`

Valores atuais observados:

- `int`
- `float`
- `number`
- `bool`
- `string`

O valor deve corresponder ao tipo realmente aceito pelo artefato de ML.

### Valores aceitos em `field_type`

Valores usados pelo frontend:

- `select` — seleção entre opções
- `radio` — escolha única visível
- `checkbox` — seleção booleana ou múltipla, conforme implementação
- `number` — entrada numérica

### `options`

Usar lista de objetos, não mapa:

```json
"options": [
  {
    "label": "Sim",
    "value": 1
  },
  {
    "label": "Não",
    "value": 0
  }
]
```

Regras:

- `label` é texto de apresentação
- `value` é valor enviado ao modelo
- valores devem ser compatíveis com `type`
- ordem da lista deve ser preservada

### `validation`

Para campos numéricos:

```json
"validation": {
  "min": 0,
  "max": 120
}
```

Campos possíveis:

- `min`
- `max`

Frontend usa esses valores para validação visual. Serviço de modelos deve continuar validando requisitos essenciais no backend do modelo.

---

## 3. Seção `output`

Descreve resultado, interpretação e classe positiva.

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---:|---|
| `type` | string | recomendado | Tipo de resultado previsto |
| `positive_class_index` | integer | sim para classificadores | Índice da classe positiva em `predict_proba` |
| `thresholds.low` | number | recomendado | Limite superior de risco baixo |
| `thresholds.high` | number | recomendado | Limite superior de risco moderado |
| `messages.low` | string | recomendado | Mensagem para risco baixo |
| `messages.moderate` | string | recomendado | Mensagem para risco moderado |
| `messages.high` | string | recomendado | Mensagem para risco alto |

### Thresholds

Thresholds são representados entre `0` e `1` no metadata:

```json
"thresholds": {
  "low": 0.15,
  "high": 0.4
}
```

O serviço converte esses valores para percentual ao produzir interpretação.

Regra atual:

- `probability <= low` — baixo
- `low < probability <= high` — moderado
- `probability > high` — alto

### `positive_class_index`

Define qual posição de `predict_proba` representa classe positiva.

Exemplo:

```python
probabilities = model.predict_proba(data)
positive_probability = probabilities[0][positive_class_index]
```

Valor não pode ser escolhido por convenção genérica. Deve corresponder à ordem real de `model.classes_`.

> Valor incorreto pode inverter interpretação clínica do resultado.

---

## 4. Contrato de request de predição

O request para Django deve usar nomes técnicos de features:

```json
{
  "faixaetaria": 5,
  "qntVacinas": 2,
  "dorDeCabeca": 1,
  "diabetes": 0
}
```

Regras:

- enviar valores compatíveis com cada feature
- incluir todos os campos `required: true`
- não usar labels como chaves
- não incluir dados de contexto clínico que não sejam features
- contexto de usuário, doença e geolocalização pertence ao backend, não ao modelo

## 5. Contrato de response de predição

Resposta atual típica:

```json
{
  "model": {
    "slug": "covid_predicao_mortalidade",
    "name": "Covid Predição Mortalidade"
  },
  "request": {
    "faixaetaria": 5,
    "diabetes": 0
  },
  "message": "Risco baixo de mortalidade",
  "probability": 12.4
}
```

Campos:

- `model` — resumo do modelo usado
- `request` — parâmetros recebidos
- `message` — interpretação textual
- `probability` — probabilidade percentual

## 6. Compatibilidade legada

Formato antigo documentado em README usava:

```json
{
  "model": {
    "slug": "modelo_1",
    "nome": "Modelo 1",
    "descricao": "Descrição"
  },
  "params": {
    "param1": {
      "title": "Parâmetro 1",
      "options": {
        "Sim": 1,
        "Não": 0
      }
    }
  }
}
```

Esse formato não deve ser usado para novos modelos.

### Diferenças

| Legado | Canônico |
|---|---|
| `nome` | `name` |
| `descricao` | `description` |
| `params` | `features` |
| `options` como mapa | `options` como lista |
| sem `output` estruturado | `output` com thresholds e classe positiva |

Se compatibilidade for necessária, ela deve existir em adaptador explícito, não em formatos ambíguos misturados.

## 7. Validação antes de disponibilizar modelo

Checklist:

- [ ] slug válido e único
- [ ] `metadata.json` é JSON válido
- [ ] `model.slug` corresponde ao diretório
- [ ] arquivo `.pkl`/`.pk1` abre corretamente
- [ ] modelo possui features identificáveis
- [ ] `n_features` corresponde às features declaradas
- [ ] cada feature possui `name`, `type`, `field_type`, `required` e `order`
- [ ] opções correspondem ao tipo do campo
- [ ] thresholds estão entre `0` e `1`
- [ ] `positive_class_index` corresponde a `model.classes_`
- [ ] predição de teste retorna probabilidade válida
- [ ] mensagem de cada faixa foi revisada
- [ ] metadata e artefato têm mesma versão lógica

## 8. Inconsistências atuais a corrigir

### Índice padrão da classe positiva

Há diferença entre código de geração e execução:

- geração automática informa `positive_class_index = 0` como padrão
- `MLModel.predict()` usa fallback `1` quando campo não existe

Decisão necessária: exigir campo explicitamente ou definir um único padrão validado.

### Campos opcionais no frontend

Frontend espera alguns campos modernos, como `model_info`, `features` e `output`. O contrato canônico deve usar `model`, `features` e `output`, mantendo transformação centralizada se houver legado.

### Validação do schema

O importador valida compatibilidade do arquivo de modelo, mas metadata também precisa de validação estrutural e semântica completa.

## 9. Versionamento do contrato

Quando formato mudar de modo incompatível, adicionar versão explícita:

```json
{
  "schema_version": "1.0",
  "model": {},
  "features": [],
  "output": {}
}
```

Até versão ser implementada no código, esta documentação considera formato canônico como `1.0` conceitual.

## Referências de implementação

- `medis-modelos/modelos/base_model.py`
- `medis-modelos/modelos/api.py`
- `medis-modelos/modelos/services/models_crud.py`
- `medis-modelos/modelos/introspection.py`
- `medis-modelos/scripts/generate_metadata.py`
- `medis-front/src/utils/queries.ts`
- `medis-front/src/components/DynamicPredictionForm.tsx`
