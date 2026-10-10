# RotaViva API: contrato para o front-end

Documentação do contrato da API REST da plataforma **RotaViva**, desenvolvida para a **Vértice Logística Integrada Ltda.** (empresa fictícia criada para o PIM VII do curso de Análise e Desenvolvimento de Sistemas da UNIP).

Este repositório não contém o código do backend. Ele reúne apenas o que o time de front-end (aplicativo Flutter) precisa para começar a integração: rotas, formatos de requisição e resposta, regras de autenticação e exemplos de JSON.

> **Status do contrato:** rascunho acadêmico. Os formatos abaixo podem sofrer ajustes durante o desenvolvimento. Qualquer mudança deve ser combinada entre backend e front-end e registrada neste arquivo.

---

## Sumário

1. [Sobre o projeto](#1-sobre-o-projeto)
2. [Visão geral da arquitetura](#2-visão-geral-da-arquitetura)
3. [Convenções gerais da API](#3-convenções-gerais-da-api)
4. [Autenticação e perfis](#4-autenticação-e-perfis)
5. [Lista de endpoints](#5-lista-de-endpoints)
6. [Contratos e exemplos de JSON](#6-contratos-e-exemplos-de-json)
7. [Sincronização offline](#7-sincronização-offline)
8. [Tratamento de erros](#8-tratamento-de-erros)
9. [Dicas para a integração no Flutter](#9-dicas-para-a-integração-no-flutter)
10. [Autoria e contexto acadêmico](#10-autoria-e-contexto-acadêmico)

---

## 1. Sobre o projeto

A **Vértice Logística Integrada** é uma empresa fictícia de transporte rodoviário de cargas fracionadas, com sede em Jacareí (SP) e cerca de 380 colaboradores. Ela controla entregas, ocorrências e checklists de veículos em planilhas e papel, e não possui um canal estruturado para denúncias de discriminação.

A **RotaViva** resolve esses dois problemas com uma plataforma em três frentes:

| Frente | Tecnologia | Público |
| --- | --- | --- |
| Aplicativo mobile | Flutter | Motoristas e operadores |
| API REST (este contrato) | ASP.NET Core | Consumida pelo aplicativo |
| Painel administrativo | ASP.NET Core MVC | Gestores, RH e auditoria |

Principais funcionalidades expostas pela API:

- autenticação por matrícula e senha, com renovação de sessão;
- lista de entregas do dia e atualização de status;
- registro de ocorrências com foto;
- envio de checklist do veículo;
- trilhas de capacitação, incluindo conteúdo sobre relações étnico-raciais;
- canal de denúncia anônima;
- sincronização de operações realizadas sem internet.

## 2. Visão geral da arquitetura

```text
Aplicativo Flutter
      |  HTTPS + Authorization: Bearer <accessToken>
      v
API REST (ASP.NET Core)
      |
      v
Services --> Repositories --> Entity Framework Core --> SQL Server
```

O backend é organizado em camadas (Controllers, Services, Repositories e Models). Para o front-end, o que importa é o contrato HTTP descrito neste documento.

## 3. Convenções gerais da API

| Item | Convenção |
| --- | --- |
| URL base | `https://<host-da-api>/api/v1` (o host será informado quando houver ambiente publicado) |
| Formato | JSON (`application/json`), exceto upload de fotos (`multipart/form-data`) |
| Nomes de propriedades | `camelCase` |
| Datas e horas | ISO 8601 em UTC, por exemplo `2026-10-09T14:32:00Z`. A conversão para o fuso local é feita pelo app |
| Enumerações | Enviadas e recebidas como **texto**, por exemplo `"EmRota"` |
| Versionamento | Na URL (`/api/v1/...`) |
| Paginação | Parâmetros `page` (começa em 1) e `pageSize` nas listagens extensas |
| Erros | Padrão Problem Details (seção 8) |
| Limitação de taxa | O login aceita 5 tentativas por minuto. Excedido, a API responde `429` |

### Valores das enumerações

**StatusEntrega:** `Pendente`, `EmRota`, `Concluida`, `Falhou`

Transições permitidas:

| De | Para |
| --- | --- |
| `Pendente` | `EmRota` |
| `EmRota` | `Concluida` ou `Falhou` |

Qualquer outra transição retorna `400`.

**Perfil:** `Motorista`, `Operador`, `Gestor`, `RH`, `Auditor`, `Administrador`

## 4. Autenticação e perfis

A API usa tokens **JWT** do tipo Bearer.

| Token | Validade | Uso |
| --- | --- | --- |
| `accessToken` | 15 minutos | Enviado em todas as requisições protegidas |
| `refreshToken` | 7 dias | Usado somente para obter um novo `accessToken` |

Fluxo:

1. O app chama `POST /auth/login` com matrícula e senha.
2. A API devolve `accessToken` e `refreshToken`.
3. O app guarda os dois em armazenamento seguro do dispositivo.
4. Cada requisição envia o cabeçalho `Authorization: Bearer <accessToken>`.
5. Ao receber `401`, o app chama `POST /auth/refresh`. A cada renovação o `refreshToken` anterior é revogado e um novo é emitido (**rotação**), então o app deve sempre substituir o token guardado.
6. Se o refresh também falhar com `401`, o app leva o usuário de volta à tela de login.

O `identificador` do usuário vem dentro do token. O app **nunca** envia o próprio ID nas requisições.

## 5. Lista de endpoints

| Verbo | Rota | Finalidade | Perfis autorizados | Respostas |
| --- | --- | --- | --- | --- |
| POST | `/auth/login` | Autenticar por matrícula e senha | Anônimo | 200, 401, 429 |
| POST | `/auth/refresh` | Renovar o token de acesso | Anônimo com refresh token | 200, 401 |
| GET | `/entregas` | Listar as entregas do dia do motorista | Motorista, Gestor | 200 |
| GET | `/entregas/{id}` | Detalhar uma entrega | Motorista, Gestor | 200, 404 |
| PATCH | `/entregas/{id}/status` | Atualizar o status da entrega | Motorista | 204, 400, 404 |
| POST | `/entregas/{id}/ocorrencias` | Registrar ocorrência com foto | Motorista, Operador | 201, 400 |
| POST | `/checklists` | Enviar checklist do veículo | Motorista, Operador | 201, 400 |
| GET | `/treinamentos` | Listar trilhas de capacitação | Todos os perfis autenticados | 200 |
| POST | `/denuncias` | Registrar denúncia anônima | Todos os perfis autenticados | 202, 400 |
| POST | `/sincronizacao` | Enviar operações realizadas offline | Motorista, Operador | 200, 400 |

Todas as rotas acima são relativas à URL base (`/api/v1`). Rotas protegidas retornam `401` sem token válido e `403` quando o perfil não tem permissão ou quando o recurso pertence a outro usuário.

## 6. Contratos e exemplos de JSON

> Os contratos de **login**, **entregas** (leitura e atualização de status) e **sincronização** seguem o que foi definido no projeto. Os contratos de **refresh**, **ocorrências**, **checklists**, **treinamentos** e **denúncias** são uma **proposta inicial** para o front-end começar e podem ser ajustados.

### 6.1 Login

`POST /auth/login`

Requisição:

```json
{
  "matricula": "123456",
  "senha": "minhaSenhaSegura"
}
```

Resposta `200`:

```json
{
  "tipoToken": "Bearer",
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "q8Zk3...base64...",
  "expiraEm": "2026-10-09T14:47:00Z",
  "perfil": "Motorista"
}
```

Respostas de erro: `401` para credenciais inválidas (a mensagem é genérica e não informa se o erro foi na matrícula ou na senha), `429` quando o limite de tentativas é excedido. Após 5 tentativas consecutivas malsucedidas, a conta é bloqueada temporariamente.

### 6.2 Renovar token

`POST /auth/refresh`

Requisição:

```json
{
  "refreshToken": "q8Zk3...base64..."
}
```

Resposta `200`: mesmo formato da resposta de login, com um **novo** `refreshToken`.

### 6.3 Listar entregas do dia

`GET /entregas`

Resposta `200`:

```json
[
  {
    "id": 1042,
    "codigoRastreio": "VLI-2026-001042",
    "status": "Pendente",
    "criadaEm": "2026-10-09T11:05:00Z"
  },
  {
    "id": 1043,
    "codigoRastreio": "VLI-2026-001043",
    "status": "EmRota",
    "criadaEm": "2026-10-09T11:20:00Z"
  }
]
```

DTO de saída (`EntregaResponse`):

| Campo | Tipo | Descrição |
| --- | --- | --- |
| `id` | inteiro | Identificador da entrega |
| `codigoRastreio` | texto | Código único de rastreio |
| `status` | texto | Um dos valores de `StatusEntrega` |
| `criadaEm` | data e hora (UTC) | Momento de criação |

`GET /entregas/{id}` retorna um único objeto no mesmo formato, ou `404` se a entrega não existir.

### 6.4 Atualizar status da entrega

`PATCH /entregas/{id}/status`

Requisição:

```json
{
  "novoStatus": "Concluida",
  "observacao": "Entregue ao responsável no portão."
}
```

DTO de entrada (`AtualizarStatusRequest`):

| Campo | Tipo | Obrigatório | Descrição |
| --- | --- | --- | --- |
| `novoStatus` | texto | Sim | Um dos valores de `StatusEntrega` |
| `observacao` | texto | Não | Comentário livre |

Respostas: `204` sem corpo em caso de sucesso, `400` para transição inválida, `404` se a entrega não existir.

### 6.5 Registrar ocorrência (proposta)

`POST /entregas/{id}/ocorrencias`, enviado como `multipart/form-data`.

| Campo | Tipo | Obrigatório | Descrição |
| --- | --- | --- | --- |
| `tipo` | texto | Sim | Por exemplo `Avaria`, `Atraso`, `EnderecoNaoEncontrado` |
| `descricao` | texto | Sim | Descrição da ocorrência |
| `latitude` | número | Não | Localização no momento do registro |
| `longitude` | número | Não | Localização no momento do registro |
| `foto` | arquivo | Não | Imagem JPEG ou PNG, tamanho máximo a definir |

Resposta `201`:

```json
{
  "id": 88,
  "entregaId": 1043,
  "tipo": "Avaria",
  "registradaEm": "2026-10-09T15:10:00Z"
}
```

### 6.6 Enviar checklist do veículo (proposta)

`POST /checklists`

Requisição:

```json
{
  "veiculoId": 12,
  "odometroKm": 84210,
  "itens": [
    { "codigo": "PNEUS", "conforme": true, "observacao": null },
    { "codigo": "FREIOS", "conforme": false, "observacao": "Ruído ao frear." }
  ]
}
```

Resposta `201`:

```json
{
  "id": 530,
  "registradoEm": "2026-10-09T10:40:00Z"
}
```

### 6.7 Listar treinamentos (proposta)

`GET /treinamentos`

Resposta `200`:

```json
[
  {
    "id": 3,
    "titulo": "Diversidade e combate à discriminação no trabalho",
    "descricao": "Trilha obrigatória sobre inclusão e relações étnico-raciais.",
    "obrigatorio": true,
    "concluido": false
  }
]
```

### 6.8 Registrar denúncia anônima (proposta)

`POST /denuncias`

Requisição:

```json
{
  "categoria": "Discriminacao",
  "relato": "Descrição do ocorrido, com data, local e pessoas envolvidas, se o denunciante desejar informar."
}
```

Resposta `202`:

```json
{
  "protocolo": "DEN-2026-000217"
}
```

O endpoint exige autenticação para evitar abuso, mas o identificador do usuário **não é gravado** junto à denúncia. O app deve exibir o protocolo ao usuário e não deve registrar o conteúdo da denúncia em logs locais.

## 7. Sincronização offline

Motoristas trabalham em trechos sem sinal. O app registra as operações localmente e as envia em lote quando a conexão volta.

`POST /sincronizacao`

Requisição:

```json
{
  "dispositivoId": "a3f1c2d0-8b5e-4c1a-9f27-5d6e7a8b9c01",
  "operacoes": [
    {
      "idCliente": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "tipo": "AtualizarStatusEntrega",
      "entregaId": 1042,
      "novoStatus": "Concluida",
      "ocorridoEm": "2026-10-09T14:32:00Z"
    }
  ]
}
```

| Campo | Tipo | Descrição |
| --- | --- | --- |
| `dispositivoId` | GUID | Identificador do aparelho, gerado e guardado pelo app |
| `operacoes[].idCliente` | GUID | Identificador da operação, **gerado pelo app no momento em que ela é criada** |
| `operacoes[].tipo` | texto | Tipo da operação. Hoje: `AtualizarStatusEntrega` (novos tipos serão adicionados aqui) |
| `operacoes[].ocorridoEm` | data e hora (UTC) | Momento em que a ação aconteceu no dispositivo |

Os demais campos de cada operação dependem do `tipo`. Para `AtualizarStatusEntrega`: `entregaId` e `novoStatus`.

### Regra de idempotência

A sincronização é **idempotente**: se o app reenviar uma operação cujo `idCliente` já foi processado (por exemplo, porque a resposta anterior se perdeu), a API a descarta sem duplicar o efeito. Por isso, o app deve:

- gerar o `idCliente` uma única vez, ao criar a operação, e **nunca regenerar** em um reenvio;
- manter a operação na fila local até receber confirmação da API;
- ao reenviar, usar sempre o mesmo `idCliente`.

### Resposta (proposta)

`200`:

```json
{
  "resultados": [
    {
      "idCliente": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "resultado": "Aplicada",
      "motivo": null
    }
  ]
}
```

Valores possíveis de `resultado`: `Aplicada`, `Duplicada` (já processada anteriormente, deve ser removida da fila) e `Rejeitada` (com o `motivo`, por exemplo transição de status inválida).

## 8. Tratamento de erros

Os erros seguem o padrão **Problem Details** para APIs HTTP, com corpo JSON uniforme:

```json
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.1",
  "title": "Requisição inválida",
  "status": 400,
  "detail": "Transição de status inválida."
}
```

| Status | Quando acontece | O que o app deve fazer |
| --- | --- | --- |
| 400 | Dados inválidos ou regra de negócio violada | Exibir o `detail` ao usuário ou corrigir a requisição |
| 401 | Token ausente, inválido ou expirado | Tentar o refresh. Se falhar, voltar ao login |
| 403 | Perfil sem permissão ou recurso de outro usuário | Exibir mensagem de acesso negado |
| 404 | Recurso inexistente | Atualizar a lista local |
| 409 | Conflito de edição simultânea do mesmo registro | Recarregar o registro e decidir como resolver |
| 429 | Muitas tentativas de login | Aguardar antes de tentar de novo |
| 5xx | Erro interno | Tentar novamente depois, mantendo a operação na fila local |

## 9. Dicas para a integração no Flutter

- **Cliente HTTP:** crie um interceptor que adicione o cabeçalho `Authorization` em todas as chamadas protegidas e trate o `401` fazendo o refresh uma única vez antes de repetir a requisição.
- **Armazenamento dos tokens:** use armazenamento seguro do dispositivo (por exemplo, o pacote `flutter_secure_storage`), nunca preferências comuns.
- **Fila offline:** guarde as operações em um banco local (por exemplo, SQLite) com o `idCliente`, o tipo e o estado de envio, e esvazie a fila com `POST /sincronizacao` quando houver conexão.
- **Datas:** converta sempre as datas recebidas de UTC para o fuso do aparelho na hora de exibir.
- **Enumerações:** mapeie os textos recebidos para `enum` no Dart e trate valores desconhecidos de forma segura, caso a API ganhe novos valores.
- **Segurança:** não registre senhas, tokens nem o conteúdo de denúncias em logs.

## 10. Autoria e contexto acadêmico

Projeto desenvolvido para o **PIM VII** do curso de Análise e Desenvolvimento de Sistemas da UNIP. A Vértice Logística Integrada Ltda. é uma empresa **fictícia**, criada exclusivamente para fins acadêmicos.

Este documento descreve o contrato da API em nível conceitual. Os trechos de código do projeto são simplificados e servem para fins de estudo.
