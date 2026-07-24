# Fluxos Principais

## Objetivo

Descrever fluxos funcionais centrais do ecossistema MeDiS em nível macro, com base no comportamento atualmente implementado.

## Leitura desta página

Cada fluxo responde:

- quem inicia ação
- quais sistemas participam
- onde regra roda
- onde dado é persistido
- quais pontos merecem atenção arquitetural

---

## 1. Autenticação e manutenção de sessão

### Visão resumida

```text
Usuário -> Frontend -> Backend -> PostgreSQL
                    <- JWT + dados básicos do usuário
Frontend -> localStorage
```

### Objetivo do fluxo

Permitir acesso autenticado ao sistema e habilitar chamadas subsequentes à API principal.

### Sequência atual

1. usuário informa email e senha no frontend
2. frontend envia credenciais para `medis-back`
3. backend busca usuário no PostgreSQL
4. backend valida senha com Argon2
5. backend emite JWT
6. backend retorna `access_token` e dados básicos do usuário
7. frontend salva dados do usuário e token em `localStorage`
8. interceptor Axios passa a enviar header `Authorization: Bearer <token>`

### Sistemas envolvidos

- `medis-front`
- `medis-back`
- PostgreSQL

### Persistência

- credenciais e perfil do usuário: PostgreSQL
- token e sessão local: navegador (`localStorage`)

### Pontos de atenção

- sessão depende de armazenamento no cliente
- checagem de sessão ativa no frontend está simplificada
- proteção visual e navegação dependem de dados salvos localmente

---

## 2. Cadastro de usuário

### Visão resumida

```text
Usuário -> Frontend -> Backend -> PostgreSQL
```

### Objetivo do fluxo

Criar nova conta de usuário apta a executar triagens e manter histórico.

### Sequência atual

1. usuário preenche formulário de cadastro
2. frontend envia dados ao backend
3. backend separa senha e confirmação
4. backend gera hash da senha
5. backend persiste novo usuário no PostgreSQL
6. backend retorna dados básicos da conta criada

### Dados relevantes

Cadastro envolve dados pessoais e de endereço, usados depois também para geocodificação indireta das triagens.

### Pontos de atenção

- endereço do usuário impacta qualidade do dashboard geográfico quando coordenadas não são enviadas pelo cliente

---

## 3. Consulta de doenças e conteúdo clínico

### Visão resumida

```text
Usuário/Admin -> Frontend -> Backend -> PostgreSQL
```

### Objetivo do fluxo

Exibir doenças disponíveis e seus dados associados para navegação, triagem e administração.

### Sequência atual

1. frontend consulta lista de doenças
2. backend retorna identificadores e nomes
3. frontend consulta detalhe de doença específica quando necessário
4. backend retorna doença com relacionamentos principais:
   - sintomas
   - critérios de agravamento
   - perguntas
5. frontend monta tela conforme contexto de uso

### Sistemas envolvidos

- `medis-front`
- `medis-back`
- PostgreSQL

### Persistência

Sem escrita nesse fluxo. Apenas leitura de dados de domínio.

---

## 4. Triagem padrão baseada em regras

### Visão resumida

```text
Usuário -> Frontend -> Backend -> PostgreSQL
                         \-> Geocodificação externa (opcional)
```

### Objetivo do fluxo

Executar triagem com perguntas objetivas e pontuação definida por doença.

### Sequência atual

1. frontend carrega dados da doença e perguntas vinculadas
2. usuário responde questionário
3. frontend envia respostas e `userId` ao backend
4. backend valida existência da doença
5. backend valida se cada resposta corresponde a pergunta vinculada à doença
6. backend soma pontuação das respostas positivas
7. backend seleciona resultado compatível com faixa de pontuação da doença
8. backend tenta obter coordenadas:
   - usa latitude/longitude recebidas, se existirem
   - caso contrário, geocodifica endereço do usuário
9. backend persiste `RulesTriage`
10. backend persiste respostas associadas à triagem
11. backend retorna `diseaseId`, `score` e resultado

### Sistemas envolvidos

- `medis-front`
- `medis-back`
- PostgreSQL
- serviço externo de geocodificação via backend

### Regra de negócio principal

- cálculo de escore ocorre no backend
- resultado final depende das faixas configuradas na doença

### Persistência

- triagem: `RulesTriage`
- respostas: `Answer`
- coordenadas e acurácia: armazenadas junto da triagem

### Pontos de atenção

- qualidade do mapa depende de coordenadas fornecidas ou geocodificação bem-sucedida
- se faixa de resultado estiver mal configurada, backend não encontra saída válida

---

## 5. Triagem por modelo de predição

### Visão resumida

```text
Usuário -> Frontend -> medis-modelos
                   \-> Backend -> medis-modelos -> PostgreSQL
                         \-> Geocodificação externa (opcional)
```

### Objetivo do fluxo

Executar triagem dinâmica orientada por schema do modelo e registrar histórico oficial no backend.

### Etapa A — descoberta do modelo e montagem do formulário

1. frontend consulta modelos disponíveis
2. frontend identifica modelos associados à doença no backend
3. frontend consulta schema dinâmico do modelo no serviço Django
4. serviço Django retorna metadados com:
   - identificação do modelo
   - features esperadas
   - ordem dos campos
   - obrigatoriedade
   - opções e validações
   - interpretação de saída
5. frontend converte schema para formato interno e monta formulário dinâmico

### Etapa B — predição para feedback imediato

1. usuário preenche formulário dinâmico
2. frontend valida campos obrigatórios e limites básicos
3. frontend envia `params` para `medis-modelos`
4. serviço Django executa predição
5. frontend recebe resposta e prepara exibição do resultado

### Etapa C — persistência oficial no histórico

1. após predição bem-sucedida, frontend envia ao backend:
   - `slug`
   - `userId`
   - `diseaseId`
   - `params`
   - opcionalmente coordenadas
2. backend tenta obter latitude/longitude:
   - usa coordenadas recebidas, se houver
   - caso contrário, geocodifica endereço do usuário
3. backend chama novamente `medis-modelos` com `params`
4. backend recebe resposta do modelo
5. backend persiste `PredictionModelTriage` no PostgreSQL
6. backend salva payload da predição em campo JSON e texto interpretado em `result`
7. frontend informa sucesso ao usuário

### Sistemas envolvidos

- `medis-front`
- `medis-back`
- `medis-modelos`
- PostgreSQL
- serviço externo de geocodificação via backend

### Regra de negócio principal

- definição dos campos vem do metadata/schema do modelo
- execução algorítmica ocorre no serviço Django
- persistência oficial ocorre no backend

### Persistência

- triagem preditiva: `PredictionModelTriage`
- payload do modelo: salvo em JSON
- coordenadas e acurácia: armazenadas junto da triagem

### Pontos de atenção

- fluxo atual faz duas chamadas de predição ao serviço Django
- frontend e backend compartilham responsabilidade no fluxo
- consistência da experiência depende fortemente de `metadata.json`
- interpretação correta da classe positiva depende de `positive_class_index`

---

## 6. Histórico de triagens

### Visão resumida

```text
Usuário -> Frontend -> Backend -> PostgreSQL
```

### Objetivo do fluxo

Exibir ao usuário histórico unificado de triagens por regra e por modelo.

### Sequência atual

1. frontend solicita histórico por `userId`
2. backend busca triagens do tipo `RulesTriage`
3. backend busca triagens do tipo `PredictionModelTriage`
4. backend enriquece triagens por regra com perguntas/respostas
5. backend concatena ambos os tipos em estrutura unificada
6. backend ordena por data decrescente
7. frontend renderiza histórico consolidado

### Persistência

Sem escrita. Fluxo de leitura agregada.

### Pontos de atenção

- existem dois tipos de triagem com formatos internos diferentes
- backend assume papel de unificação para consumo da interface

---

## 7. Administração clínica e operacional

### Visão resumida

```text
Administrador -> Frontend -> Backend -> PostgreSQL
Administrador -> Frontend -> medis-modelos
```

### Objetivo do fluxo

Permitir manutenção das entidades clínicas e do catálogo de modelos usados no sistema.

### Subfluxos principais

#### 7.1 Gestão de doenças
- criar doença
- editar informações
- associar sintomas
- associar critérios de agravamento
- vincular perguntas e pontuação
- definir resultados por faixa
- definir modo ativo de triagem

#### 7.2 Gestão de perguntas, sintomas e critérios
- criar
- editar
- remover

#### 7.3 Gestão de unidades de saúde
- criar
- editar
- remover
- usar dados depois no dashboard geográfico

#### 7.4 Gestão de modelos
Fluxo atual dividido:
- CRUD físico do modelo ocorre em `medis-modelos`
- associação entre modelo e doença ocorre no backend

### Sequência de gestão de modelos

1. administrador envia upload de modelo ao serviço Django
2. Django valida arquivo `.pkl`/`.pk1`
3. Django gera ou atualiza `metadata.json`
4. Django recarrega módulo dinamicamente
5. frontend atualiza listagem de modelos
6. administrador vincula modelo à doença via backend
7. backend persiste associação `DiseasePredictionModel`

### Pontos de atenção

- autorização visual no frontend não substitui autorização no backend
- gestão de modelos é distribuída entre dois serviços
- metadados ruins podem tornar modelo importado inutilizável na interface

---

## 8. Dashboard geográfico e heatmaps

### Visão resumida

```text
Usuário/Admin -> Frontend -> Backend -> PostgreSQL
```

### Objetivo do fluxo

Exibir mapa com concentração geográfica de triagens e unidades de saúde relevantes.

### Sequência atual

1. frontend solicita dados geográficos ao backend
2. backend busca triagens por regra com coordenadas
3. backend busca triagens por modelo com coordenadas
4. backend aplica filtro temporal e, opcionalmente, filtro por doença
5. backend agrega coordenadas em grade geográfica
6. backend calcula intensidade relativa por doença
7. backend busca unidades de saúde ativas na área dos pontos ou retorna conjunto ativo geral
8. backend responde com:
   - pontos de heatmap por doença
   - conjunto agregado de heatmap
   - unidades de saúde
9. frontend renderiza mapa, heatmap e marcadores

### Regra de negócio principal

- dashboard combina dois tipos de triagem
- intensidade é normalizada por doença
- visão final depende de qualidade das coordenadas persistidas

### Pontos de atenção

- janela temporal influencia leitura epidemiológica
- geocodificação imprecisa reduz confiabilidade espacial
- fallback por cidade/capital introduz baixa precisão

---

## 9. Geocodificação de apoio

### Visão resumida

```text
Backend -> Nominatim/OpenStreetMap
Backend -> fallback local
```

### Objetivo do fluxo

Enriquecer triagens com coordenadas quando cliente não fornecer latitude/longitude.

### Sequência atual

1. backend recebe triagem sem coordenadas
2. backend busca endereço do usuário
3. backend consulta Nominatim com rate limit e cache
4. se consulta completa falhar, tenta por cidade
5. se ainda falhar, usa coordenadas pré-definidas de cidade ou capital
6. backend salva coordenadas e nível de acurácia na triagem

### Pontos de atenção

- não é fluxo de negócio principal, mas sustenta dashboard
- precisão pode variar muito conforme origem das coordenadas

---

## Resumo dos fluxos mais críticos

### Fluxos centrados no backend
- autenticação
- triagem padrão
- histórico
- dashboard
- administração de entidades clínicas

### Fluxos compartilhados com `medis-modelos`
- descoberta de modelos
- montagem de formulário dinâmico
- predição
- importação/atualização de modelos

### Fluxos com maior risco arquitetural atual
- triagem por modelo, por dividir responsabilidade entre frontend, backend e Django
- sessão/autorização no cliente
- dependência de metadados corretos para formulários dinâmicos

## Próximos detalhamentos recomendados

- transformar cada fluxo em diagrama de sequência
- documentar endpoints e payloads reais em `referencia/contratos.md`
- separar melhor fluxo de exibição imediata vs fluxo de persistência oficial
- explicitar regras de autorização por rota
