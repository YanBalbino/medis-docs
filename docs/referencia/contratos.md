# Contratos Entre Serviços

## Objetivo

Registrar contratos HTTP atualmente implementados entre `medis-front`, `medis-back` e `medis-modelos`.

Esta página descreve o estado do código. Não representa necessariamente o contrato desejado para produção.

## Endereços locais

| Serviço | Endereço local | Base |
|---|---:|---|
| Frontend | `http://localhost:3000` | — |
| Backend | `http://localhost:3030` | `/` |
| Serviço de modelos | `http://localhost:8000` | `/modelos/` |

O frontend define esses endereços em `src/utils/consts.ts`.

## Regras gerais

### Backend

- API NestJS escuta na porta `3030`
- `ValidationPipe` global aplica transformação de tipos
- CORS aceita atualmente `http://localhost:3000` e `http://localhost:3030`
- autenticação usa `Authorization: Bearer <JWT>`
- `AuthGuard` é global
- rotas marcadas com `@Public()` não exigem autenticação no guard atual

### Serviço de modelos

- API Django é montada sob `/modelos/`
- endpoints usam JSON, exceto importação/atualização de arquivos
- slug é normalizado para minúsculas e substitui hífen por underscore
- respostas de erro nem sempre seguem um único formato histórico

### Frontend

- usa Axios para chamadas
- chamadas ao backend passam pelo cliente `api`
- chamadas ao serviço de modelos usam Axios diretamente
- chamadas ao backend incluem JWT pelo interceptor
- chamadas diretas ao Django não passam pelo interceptor do backend

---

# 1. Frontend -> Backend

Base local:

```text
http://localhost:3030
```

## 1.1 Autenticação

### Login

```http
POST /auth/login
Content-Type: application/json
```

Request:

```json
{
  "email": "usuario@example.com",
  "password": "senha"
}
```

Response de sucesso:

```json
{
  "access_token": "<jwt>",
  "user": {
    "id": "uuid",
    "email": "usuario@example.com",
    "name": "Nome do usuário",
    "role": "USER"
  }
}
```

Regras de validação:

- `email` deve ser válido
- `password` é obrigatório

### Cadastro

```http
POST /auth/register
Content-Type: application/json
```

Request atual:

```json
{
  "name": "Nome do usuário",
  "email": "usuario@example.com",
  "password": "senha",
  "confirm_password": "senha",
  "address": "Rua Exemplo",
  "phone": "84999999999",
  "city": "Mossoró",
  "cpf": "00000000000",
  "birthdate": "2000-01-01",
  "number": "100",
  "complement": "Apto 1",
  "state": "Rio Grande do Norte",
  "cep": "59600000",
  "uf": "RN",
  "gender": ""
}
```

Response de sucesso contém somente:

```json
{
  "id": "uuid",
  "email": "usuario@example.com",
  "name": "Nome do usuário",
  "createdAt": "2026-01-01T00:00:00.000Z"
}
```

### Dados do usuário

```http
GET /auth/me/:id
Authorization: Bearer <jwt>
```

Retorna usuário identificado por `id`, sem senha.

> O frontend possui função legada que chama `/auth/me` sem `id`. Endpoint implementado atualmente exige `:id`.

## 1.2 Doenças e triagem padrão

### Listar doenças

```http
GET /diseases
```

Resposta resumida:

```json
[
  {
    "id": "uuid",
    "name": "Nome da doença"
  }
]
```

### Buscar doença principal

```http
GET /diseases/main
```

Retorna identificador da primeira doença encontrada. A seleção atual não explicita critério de prioridade.

### Buscar doença

```http
GET /diseases/:id
```

Retorna doença com relações clínicas, incluindo sintomas, critérios e perguntas ativas.

### Criar doença

```http
POST /diseases
Authorization: Bearer <jwt>
Content-Type: application/json
```

Payload base depende de `CreateDiseaseDto`.

### Atualizar doença

```http
PATCH /diseases/:id
Authorization: Bearer <jwt>
Content-Type: application/json
```

Payload atual pode conter:

- dados básicos da doença
- `symptomsInDisease`
- `aggravationCriteriasInDisease`
- `questionsInDisease`
- `results`
- modo de triagem

### Remover doença

```http
DELETE /diseases/:id
Authorization: Bearer <jwt>
```

### Executar triagem padrão

```http
POST /diseases/triage/:id
Authorization: Bearer <jwt>
Content-Type: application/json
```

Request:

```json
{
  "userId": "uuid",
  "answers": [
    {
      "questionId": "uuid",
      "answer": true
    },
    {
      "questionId": "uuid",
      "answer": false
    }
  ],
  "latitude": -5.188,
  "longitude": -37.3444,
  "accuracy": 20
}
```

`latitude`, `longitude` e `accuracy` são opcionais. Sem coordenadas, backend tenta geocodificar endereço do usuário.

Response:

```json
{
  "diseaseId": "uuid",
  "score": 4,
  "result": {
    "content": "Resultado interpretado",
    "minPontuation": 0,
    "maxPontuation": 5
  }
}
```

Comportamento:

- respostas verdadeiras somam `pontuation` da pergunta vinculada
- resultado é encontrado em `Disease.results`
- triagem é persistida em `RulesTriage`
- respostas são persistidas em `Answer`

### Histórico de triagens

```http
GET /diseases/triageHistory/:userId
Authorization: Bearer <jwt>
```

Retorna coleção unificada de:

- triagens por regra, com `type: "rules"`
- triagens por modelo, com `type: "predictionModel"`

Resultado é ordenado por data decrescente.

## 1.3 Modelos via backend

### Listar modelos

```http
GET /prediction/models
```

Backend consulta `GET <PREDICTION_API_URL>/modelos` e repassa lista.

### Listar modelos associados a doença

```http
GET /prediction/diseases/:id
```

Backend combina:

- modelos disponíveis no Django
- vínculos persistidos em `DiseasePredictionModel`

### Buscar detalhes de modelo

```http
GET /prediction/models/:slug
```

Backend consulta:

```text
GET <PREDICTION_API_URL>/modelos/:slug
```

### Associar modelo à doença

```http
POST /prediction/models/:slug/add-to-disease/:diseaseId
Authorization: Bearer <jwt>
```

Sem body. Persiste vínculo por `diseaseId` + `predictionModelSlug`.

### Remover associação

```http
DELETE /prediction/models/:slug/remove-from-disease/:diseaseId
Authorization: Bearer <jwt>
```

### Executar e persistir predição

```http
POST /prediction/models/:slug/predict
Authorization: Bearer <jwt>
Content-Type: application/json
```

Request:

```json
{
  "userId": "uuid",
  "diseaseId": "uuid",
  "params": {
    "idade": 42,
    "febre": 1
  },
  "latitude": -5.188,
  "longitude": -37.3444,
  "accuracy": 20
}
```

Backend envia somente `params` ao Django:

```http
POST /modelos/:slug/predict/
Content-Type: application/json
```

Depois persiste resposta em `PredictionModelTriage`.

> Rotas de `prediction` estão marcadas com `@Public()` no controller atual, incluindo predição e associação de modelos. Isso precisa ser revisado antes de considerar contrato de segurança definitivo.

## 1.4 Dashboard e geodados

### Geodados agregados

```http
GET /dashboard/geodata?diseasesIds=uuid1,uuid2
Authorization: Bearer <jwt>
```

Também aceita parâmetro repetido:

```http
GET /dashboard/geodata?diseasesIds=uuid1&diseasesIds=uuid2
```

Response conceitual:

```json
{
  "diseases": {
    "uuid1": [
      [-5.188, -37.344, 0.8]
    ]
  },
  "heatmapData": [
    {
      "latitude": -5.188,
      "longitude": -37.344,
      "intensity": 0.8
    }
  ],
  "healthUnits": []
}
```

O backend combina `RulesTriage` e `PredictionModelTriage`, limita dados aos últimos 30 dias no fluxo de dashboard e agrega pontos por grade.

### Heatmap por doença

```http
GET /diseases/:id/heatmap
```

Parâmetros opcionais:

- `startDate`: data ISO
- `endDate`: data ISO
- `maxAccuracy`: precisão máxima em metros
- `limit`: limite de registros

### Heatmap geral

```http
GET /diseases/heatmap
```

Aceita também `diseaseId` nos parâmetros.

## 1.5 Entidades CRUD

Os seguintes recursos seguem padrão NestJS semelhante:

| Recurso | Base | Operações |
|---|---|---|
| Perguntas | `/questions` | `GET`, `POST`, `GET /:id`, `PATCH /:id`, `DELETE /:id` |
| Sintomas | `/symptoms` | `GET`, `POST`, `GET /:id`, `PATCH /:id`, `DELETE /:id` |
| Critérios | `/criteria` | `GET`, `POST`, `GET /:id`, `PATCH /:id`, `DELETE /:id` |
| Usuários | `/users` | `GET`, `GET /:id`, `PATCH /:id`, `DELETE /:id` |
| Unidades de saúde | `/health-units` | `GET`, `POST`, `GET /:id`, `PATCH /:id`, `DELETE /:id` |

Regras de autenticação devem ser confirmadas por rota no código antes de uso em documentação pública detalhada.

---

# 2. Frontend -> `medis-modelos`

Base local:

```text
http://localhost:8000/modelos
```

## 2.1 Listar modelos

```http
GET /modelos/
```

Resposta baseada em `summary()` de cada modelo:

```json
[
  {
    "slug": "covid_predicao_mortalidade",
    "name": "Modelo de mortalidade",
    "description": "Descrição do modelo"
  }
]
```

Os campos exatos dependem do `metadata.json` de cada modelo. Versões antigas podem usar `nome` e `descricao`.

## 2.2 Detalhes do modelo

```http
GET /modelos/:slug/
```

Retorna metadata completo do modelo. Formato atual esperado:

```json
{
  "model": {
    "slug": "modelo_exemplo",
    "name": "Modelo exemplo",
    "version": "1.0.0",
    "description": "Descrição",
    "algorithm": "RandomForest",
    "prediction_type": "risco"
  },
  "features": [
    {
      "name": "idade",
      "label": "Idade",
      "field_type": "number",
      "type": "int",
      "required": true,
      "order": 1,
      "validation": {
        "min": 0,
        "max": 120
      }
    }
  ],
  "output": {
    "thresholds": {
      "low": 0.3,
      "high": 0.6
    },
    "positive_class_index": 1
  }
}
```

## 2.3 Schema para formulário dinâmico

```http
GET /modelos/schema/:slug/
```

Retorna detalhes do modelo. Frontend usa principalmente:

- `features`
- `field_type`
- `required`
- `order`
- `options`
- `validation`
- `output.thresholds`
- dados de identificação em `model`

O frontend converte resposta da API para formato interno antes de renderizar `DynamicPredictionForm`.

## 2.4 Executar predição

```http
POST /modelos/:slug/predict/
Content-Type: application/json
```

Request:

```json
{
  "idade": 42,
  "febre": 1
}
```

Response típica:

```json
{
  "model": {
    "slug": "modelo_exemplo",
    "name": "Modelo exemplo"
  },
  "request": {
    "idade": 42,
    "febre": 1
  },
  "message": "Risco moderado: probabilidade estimada de 45.2%.",
  "probability": 45.2
}
```

Comportamento do serviço:

- valida features obrigatórias contra metadata
- carrega modelo pickle, usando cache após primeira carga
- chama `predict_proba`
- usa `output.positive_class_index` para selecionar classe positiva
- calcula interpretação a partir de `output.thresholds` e `output.messages`

Erro de parâmetros ausentes:

```json
{
  "error": "Missing required parameters: idade",
  "missing": ["idade"],
  "expected_features": ["idade", "febre"]
}
```

## 2.5 Importar modelo

```http
POST /modelos/import/
Content-Type: multipart/form-data
```

Campos:

- `slug`: obrigatório
- `model_file`: arquivo `.pkl` ou `.pk1`, obrigatório
- `metadata_file`: JSON opcional

Limites e validações atuais:

- slug em snake_case
- arquivo não pode estar vazio
- tamanho máximo: 200 MB
- modelo precisa ser carregável e compatível
- metadata pode ser enviada ou gerada automaticamente

Response de sucesso:

```json
{
  "message": "Model imported successfully.",
  "model": {
    "slug": "modelo_exemplo"
  }
}
```

## 2.6 Atualizar modelo

```http
POST /modelos/:slug/update/
Content-Type: multipart/form-data
```

Campos opcionais, mas pelo menos um deve ser fornecido:

- `model_file`
- `metadata_file`

Após atualização, módulo é recarregado e registry é reconstruído.

## 2.7 Remover modelo

```http
DELETE /modelos/:slug/delete/
```

Remove diretório do modelo, exclui módulo do registry e atualiza listagem em memória.

---

# 3. Backend -> `medis-modelos`

URL configurada por:

```env
PREDICTION_API_URL=http://localhost:8000
```

Chamadas atuais do `PredictionService`:

| Operação backend | Método | URL Django |
|---|---:|---|
| listar modelos | `GET` | `/modelos` |
| detalhes | `GET` | `/modelos/:slug` |
| predição | `POST` | `/modelos/:slug/predict/` |

## Contrato de predição do backend

O backend recebe envelope de negócio próprio:

```json
{
  "userId": "uuid",
  "diseaseId": "uuid",
  "params": {
    "feature": 1
  },
  "latitude": -5.188,
  "longitude": -37.3444,
  "accuracy": 20
}
```

Mas envia ao Django somente:

```json
{
  "feature": 1
}
```

Isso separa:

- contexto de persistência, pertencente ao backend
- parâmetros algorítmicos, pertencentes ao modelo

## Resposta consumida pelo backend

Backend espera resposta compatível com `PredictionResponse`, principalmente:

- `model`
- `message`
- payload da predição

O objeto completo é armazenado como JSON em `PredictionModelTriage.predictionModel`; `message` é copiada para `PredictionModelTriage.result`.

---

# 4. Contrato de modelo e metadados

## Arquivos por modelo

Cada modelo é descoberto em subdiretório de `modelos/` e deve conter, no mínimo:

```text
modelo_slug/
  __init__.py
  metadata.json
  modelo.pkl ou modelo.pk1
```

## Interface pública do módulo

Módulo carregado deve expor:

```python
summary()
details()
predict(params)
```

Modelos gerados pelo sistema delegam essas funções para `MLModel`.

## Features

Cada feature deve permitir ao frontend descobrir:

- nome técnico
- label
- tipo de campo
- obrigatoriedade
- ordem
- opções, quando aplicável
- validações

## Saída

A seção `output` deve documentar pelo menos:

- limiares de interpretação
- mensagens por nível
- `positive_class_index`

Exemplo:

```json
{
  "output": {
    "thresholds": {
      "low": 0.3,
      "high": 0.6
    },
    "messages": {
      "low": "Risco baixo: {probability}%",
      "moderate": "Risco moderado: {probability}%",
      "high": "Risco alto: {probability}%"
    },
    "positive_class_index": 1
  }
}
```

> `positive_class_index` não é detalhe cosmético. Índice incorreto pode inverter interpretação de probabilidade e resultado clínico.

---

# 5. Tratamento de erros

## Backend

Erros NestJS podem incluir:

- `400` — payload inválido ou regra de validação
- `401` — não autenticado ou credencial inválida
- `404` — entidade/modelo não encontrado
- `409` — conflito de unicidade ou recurso existente
- `500` — falha interna ou integração

## Serviço de modelos

Erros observados no código incluem:

```json
{
  "error": "Model not found."
}
```

```json
{
  "error": "Invalid metadata_file JSON."
}
```

```json
{
  "error": "Missing required parameters: feature_name",
  "missing": ["feature_name"],
  "expected_features": ["feature_name"]
}
```

Importação também pode retornar `warnings` quando compatibilidade é válida, mas há ressalvas.

## Regra para clientes

Clientes devem:

- verificar status HTTP
- não assumir que toda resposta contém `message`
- exibir `error` quando presente
- tratar respostas de modelo como contrato versionável

---

# 6. Divergências e riscos contratuais conhecidos

## 6.1 Rota legada `/models`

Existe hook frontend que consulta:

```text
GET http://localhost:8000/models/:slug/
```

Mas rotas Django atuais estão sob:

```text
GET http://localhost:8000/modelos/:slug/
```

Esse hook parece legado ou incompatível e deve ser corrigido/removido após confirmação de uso.

## 6.2 Nomes de campos antigos e novos

README antigo usa campos como:

- `nome`
- `descricao`

Implementação mais recente usa:

- `name`
- `description`
- `model_info`
- `features`
- `output`

É necessário escolher formato canônico e, se necessário, documentar versão de compatibilidade.

## 6.3 Autorização de endpoints

O controller de predição marca como públicas operações que alteram estado:

- associação de modelo a doença
- remoção de associação
- predição persistida pelo backend

Isso deve ser revisado antes de expor ambiente além do desenvolvimento.

## 6.4 Contrato duplicado de predição

Predição dinâmica passa pelo Django duas vezes:

1. frontend para exibição
2. backend para persistência

Isso pode produzir divergência se modelo ou metadata mudar entre chamadas.

---

# 7. Próxima evolução do contrato

Recomendações:

1. definir formato canônico de metadata
2. versionar contrato do serviço de modelos
3. corrigir ou remover rota frontend `/models`
4. proteger endpoints administrativos
5. centralizar predição no backend, se essa for decisão arquitetural
6. adicionar testes de contrato
7. documentar schemas OpenAPI ou JSON Schema
8. padronizar payloads de erro
