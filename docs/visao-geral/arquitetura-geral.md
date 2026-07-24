# Arquitetura Geral

## Objetivo

Descrever arquitetura macro do ecossistema MeDiS de forma estável, suficiente para manutenção futura, onboarding técnico e evolução dos três repositórios principais.

## Visão estrutural

Ecossistema MeDiS é composto por quatro blocos principais:

1. **`medis-front`** — aplicação web em Next.js
2. **`medis-back`** — API principal em NestJS
3. **`medis-modelos`** — serviço Django para catálogo e execução de modelos de predição
4. **PostgreSQL** — base transacional principal do sistema

## Diagrama lógico

```text
[ Usuário ]
    |
    v
[ medis-front | Next.js ]
    |                  \
    | HTTP              \ HTTP
    v                    v
[ medis-back | NestJS ]  [ medis-modelos | Django + ML ]
    |
    | Prisma
    v
[ PostgreSQL ]
```

## Leitura do diagrama

- usuário interage apenas com frontend
- frontend consome backend para domínio transacional
- frontend também consome serviço de modelos para metadados dinâmicos e predição
- backend persiste dados clínicos, administrativos e históricos no PostgreSQL
- backend integra com `medis-modelos` para associação de modelos e persistência de triagens preditivas

## Papel de cada componente

### `medis-front`

Camada de experiência do usuário.

Responsabilidades atuais confirmadas no código:

- autenticação e manutenção de sessão no cliente
- navegação por rotas públicas, de paciente e administrativas
- consumo da API NestJS via `BASE_URL`
- consumo direto do serviço Django via `ML_API_URL`
- renderização dinâmica de formulários de predição a partir do schema do modelo
- submissão de triagem padrão
- submissão de predição e posterior gravação no histórico
- visualização de dashboard geográfico

Características relevantes:

- usa **Next.js Pages Router**
- token JWT salvo em `localStorage`
- `Authorization: Bearer <token>` injetado por interceptor Axios
- proteção administrativa também existe no cliente por leitura do papel do usuário salvo localmente
- URLs de integração hoje estão fixas em código (`http://localhost:3030` e `http://localhost:8000`)

### `medis-back`

Núcleo transacional do sistema.

Responsabilidades atuais confirmadas no código:

- autenticação de usuários
- emissão de JWT
- proteção global de rotas via `AuthGuard`, com exceções marcadas por `@Public()`
- CRUD e relacionamento de entidades de domínio:
  - usuários
  - doenças
  - perguntas
  - sintomas
  - critérios de agravamento
  - unidades de saúde
- execução da triagem baseada em regras/pontuação
- gravação de histórico de triagens por regra e por modelo
- associação entre doenças e modelos de predição
- agregação de dados para heatmap e dashboard
- geocodificação de endereços para enriquecer triagens com latitude/longitude

Características relevantes:

- expõe API HTTP na porta `3030`
- habilita CORS para `localhost:3000` e `localhost:3030`
- usa `PrismaService` para persistência em PostgreSQL
- usa `HttpModule` para chamadas ao serviço de modelos e a serviços externos de geocodificação

### `medis-modelos`

Serviço especializado em modelos de aprendizado de máquina.

Responsabilidades atuais confirmadas no código:

- descoberta automática de modelos por diretório
- leitura e exposição de `metadata.json`
- introspecção e validação de arquivos `.pkl`/`.pk1`
- geração automática de metadados quando ausentes
- importação, atualização e remoção de modelos
- execução de predição por slug
- exposição de schema dinâmico para montagem de formulário

Características relevantes:

- expõe rotas sob `/modelos/`
- carrega módulos dinamicamente a partir de subdiretórios em `modelos/`
- mantém registro em memória dos modelos carregados
- usa artefatos de modelo e metadados como fonte de verdade da interface dinâmica
- depende de metadados corretos para interpretação de saída, incluindo `positive_class_index`

### PostgreSQL

Base transacional principal do ecossistema.

Responsabilidades atuais confirmadas no schema Prisma:

- persistência de usuários
- persistência de doenças e entidades relacionadas
- persistência de triagens por regra (`RulesTriage`)
- persistência de triagens por modelo (`PredictionModelTriage`)
- persistência de respostas da triagem padrão
- persistência de associação entre doença e modelo (`DiseasePredictionModel`)
- persistência de unidades de saúde e surtos

## Módulos de domínio mais importantes

### Domínio clínico

Centro do sistema em torno de `Disease`.

Cada doença pode se relacionar com:
- sintomas
- critérios de agravamento
- perguntas de triagem
- resultados por faixa de pontuação
- modelos de predição associados
- modo ativo de triagem

### Domínio de triagem

Sistema mantém dois tipos principais de triagem:

#### 1. Triagem por regras

Executada no backend.

Fluxo atual:
- frontend envia respostas ao backend
- backend busca perguntas da doença
- backend soma pontuações das respostas verdadeiras
- backend seleciona faixa de resultado
- backend persiste triagem e respostas no banco
- backend adiciona coordenadas quando disponíveis ou geocodifica endereço do usuário

#### 2. Triagem por modelo de predição

Executada em duas etapas separadas.

Fluxo atual confirmado no frontend:
- frontend busca schema do modelo no serviço Django
- frontend monta formulário dinâmico
- frontend envia parâmetros ao Django para obter predição
- frontend envia dados também ao backend para persistir histórico da triagem preditiva
- backend chama novamente o Django para obter resposta oficial e persisti-la no banco
- backend enriquece registro com geolocalização explícita ou geocodificada

Observação importante:

> Arquitetura atual divide predição dinâmica entre frontend e backend. Frontend usa `medis-modelos` para experiência imediata; backend usa `medis-modelos` para gravação oficial no histórico.

## Padrão de comunicação entre serviços

### Frontend -> Backend

Usado para:
- login e registro
- perfil de usuário
- doenças e triagens por regra
- histórico de triagens
- dashboard geográfico
- administração de sintomas, perguntas, critérios, doenças e unidades
- associação de modelos a doenças
- persistência de triagens por modelo

### Frontend -> Serviço de modelos

Usado para:
- listar modelos
- obter detalhes de modelo
- obter schema dinâmico
- importar, atualizar e remover modelos
- executar predição para feedback imediato na interface

### Backend -> Serviço de modelos

Usado para:
- listar modelos disponíveis
- obter detalhes de modelo
- filtrar modelos vinculados a uma doença
- executar predição para persistência de histórico

### Backend -> serviços externos

Usado para:
- geocodificação via Nominatim/OpenStreetMap
- fallback para coordenadas conhecidas em caso de falha

## Decisões arquiteturais implícitas no código

### Backend como núcleo transacional

Mesmo quando frontend conversa diretamente com `medis-modelos`, backend continua sendo fonte de verdade para:
- autenticação
- regras de negócio
- histórico persistido
- agregações analíticas
- relacionamento entre entidades clínicas

### Serviço de modelos como contexto especializado

`medis-modelos` não mantém base transacional clínica. Papel dele é:
- gerenciar catálogo de modelos
- expor metadados
- encapsular execução de predição
- validar compatibilidade técnica dos artefatos

### Frontend como orquestrador de experiência

Frontend concentra adaptação visual dos fluxos:
- triagem padrão com perguntas fixas
- triagem dinâmica baseada em schema
- área administrativa
- dashboard com mapas e heatmaps

## Dados e persistência

### Dados persistidos no backend

- usuários
- doenças
- perguntas e pontuações
- respostas
- critérios e sintomas
- unidades de saúde
- surtos
- histórico de triagens
- vínculo entre doença e modelo

### Dados mantidos no serviço de modelos

- artefatos `.pkl`/`.pk1`
- `metadata.json`
- módulos Python gerados/carregados dinamicamente
- registro em memória dos modelos ativos

### Dados mantidos no cliente

- JWT e dados de usuário em `localStorage`
- estado temporário de formulários dinâmicos
- cache de queries no React Query

## Fluxos arquiteturais prioritários

### Autenticação

```text
Usuário -> Frontend -> Backend -> PostgreSQL
```

- backend valida credenciais
- backend emite JWT
- frontend armazena token e dados do usuário localmente
- requisições seguintes enviam bearer token

### Triagem padrão

```text
Usuário -> Frontend -> Backend -> PostgreSQL
```

- frontend busca doença e perguntas
- usuário responde formulário
- backend calcula escore e resultado
- backend salva triagem, respostas e geodados

### Triagem por modelo

```text
Usuário -> Frontend -> medis-modelos
                   \-> Backend -> medis-modelos -> PostgreSQL
```

- frontend usa schema do modelo para montar formulário
- frontend obtém resultado de predição para exibição
- backend executa chamada de predição para persistência oficial
- backend salva resultado e coordenadas

### Dashboard geográfico

```text
Frontend -> Backend -> PostgreSQL
```

- backend combina triagens por regra e por modelo
- backend agrega pontos geográficos por doença
- backend retorna heatmap e unidades de saúde

## Aspectos transversais

### Segurança

Estado atual:
- JWT emitido pelo backend
- proteção global de rotas no backend
- algumas rotas públicas explicitamente marcadas
- verificação administrativa também existe no frontend
- sessão ainda depende de armazenamento no cliente

### Geolocalização

Estado atual:
- triagens podem receber latitude/longitude do cliente
- se cliente não enviar coordenadas, backend tenta geocodificar endereço do usuário
- serviço de geocodificação usa cache, rate limit e fallback por cidade/capital
- dados geográficos alimentam heatmaps e dashboard

### Dinamismo de modelos

Estado atual:
- schema do formulário vem do modelo
- interface pode se adaptar sem mudança estrutural no frontend
- validade funcional depende de metadados consistentes
- `positive_class_index` é crítico para interpretação correta da probabilidade

## Pontos de atenção atuais

### 1. Integração direta do frontend com `medis-modelos`

Vantagem:
- feedback rápido para formulário dinâmico

Trade-off:
- contrato distribuído entre frontend e Django
- parte da lógica de integração escapa do backend

### 2. Configuração acoplada ao ambiente local

Estado atual:
- URLs de backend e modelos fixas no frontend
- reduz portabilidade entre ambientes

### 3. Autorização parcialmente espelhada no cliente

Estado atual:
- proteção administrativa visual existe no frontend
- backend ainda deve permanecer fonte final de autorização

### 4. Dependência forte de metadados dos modelos

Estado atual:
- erro em `metadata.json` compromete formulário, validação e interpretação
- geração automática ajuda, mas não substitui revisão semântica

### 5. Dupla chamada de predição no fluxo dinâmico

Estado atual:
- frontend chama Django para mostrar resultado
- backend chama Django de novo para persistir histórico
- arquitetura futura pode optar por centralização maior no backend

## Princípios para evolução futura

- manter backend como fonte de verdade transacional
- explicitar contratos HTTP entre serviços
- reduzir acoplamento de configuração por ambiente
- tratar metadados de modelo como contrato versionado
- documentar decisões arquiteturais via ADR
- separar claramente fluxo de exibição imediata e fluxo de persistência oficial

## Relação com restante da documentação

Esta página responde **como ecossistema é organizado**.

Próximas páginas complementares:
- `visao-geral/componentes.md` — resumo de responsabilidades por sistema
- `visao-geral/fluxos-principais.md` — fluxos funcionais em maior detalhe
- `referencia/contratos.md` — endpoints, payloads e respostas
- `repositorios/*` — implementação e estrutura de cada código-base
