# 8. API e Contrato

> Referência da API REST **ProjetoZup** — o contrato que o backend expõe e o frontend consome. A
> documentação **canônica** é a especificação **OpenAPI 3.0** (`openapi.json`, 37 operações) na raiz
> do repositório do backend, importável em Swagger UI, Redoc, Postman ou Insomnia. Esta seção resume
> os endpoints, a autenticação, o formato de erros e como o front se conecta a tudo isso.

- **Fonte da verdade do contrato:** <https://github.com/Grogww/ProjetoZup> · `openapi.json`.
- **Base:** todas as rotas são prefixadas por **`/api`** (exceto o servir estático `/uploads`).
- **Servidor padrão documentado:** `http://localhost:3000`.

Legenda de acesso: 🟢 público · 🔵 autenticado · 🟠 admin · ⚪ auth opcional · ⏱️ rate-limited.

## 8.1 Endpoints

### Health

| Método | Rota | Acesso | Descrição |
| --- | --- | --- | --- |
| GET | `/health` | 🟢 | Healthcheck (`{ "status": "ok" }`) |

### Autenticação

| Método | Rota | Acesso | Descrição |
| --- | --- | --- | --- |
| POST | `/auth/register` | 🟢 | Cadastra usuário (`name, email, cpf, password`, `neighborhood_id?`) |
| POST | `/auth/login` | 🟢 | Login por `cpf` + `password` → sessão com tokens |
| POST | `/auth/refresh` | 🟢 | Troca `refresh_token` por nova sessão |
| POST | `/auth/forgot-password` | 🟢 ⏱️ | Solicita e-mail de reset (resposta genérica; rate-limited) |
| POST | `/auth/reset-password` | 🟢 ⏱️ | Redefine a senha com `token` + nova `password` |

### Usuários

| Método | Rota | Acesso | Descrição |
| --- | --- | --- | --- |
| GET | `/users/me` | 🔵 | Perfil do usuário autenticado |
| PATCH | `/users/me` | 🔵 | Atualiza o próprio perfil (`name, email, password, avatar_url, neighborhood_id`) |
| GET | `/users` | 🟠 | Lista usuários |
| GET | `/users/:id` | 🟠 | Detalha um usuário |
| PATCH | `/users/:id/role` | 🟠 | Altera a role (`citizen \| agent \| admin`) |

### Ocorrências

| Método | Rota | Acesso | Descrição |
| --- | --- | --- | --- |
| GET | `/occurrences` | 🟢 | Lista com filtros: `status, category_id, subcategory_id, neighborhood_id, author_id, assigned_organization_id, limit (≤200), offset` |
| GET | `/occurrences/nearby` | 🟢 | Busca por proximidade: `lat`, `lng`, `radius` (m, ≤50000; padrão 500) |
| GET | `/occurrences/:id` | ⚪ | Detalha a ocorrência (inclui `media` e, se autenticado, `voted_user`) |
| POST | `/occurrences` | 🔵 | Cria ocorrência (anti-duplicidade + geofencing); pode retornar **409** duplicidade |
| PATCH | `/occurrences/:id` | 🔵 | Edita campos `title/description/address/latitude/longitude` (autor/admin, janela de 24 h) |
| PATCH | `/occurrences/:id/status` | 🔵 | Transição de status (validada pela máquina de estados; **409** se inválida) |
| DELETE | `/occurrences/:id` | 🔵 | Remove a ocorrência e suas mídias (autor com janela / admin) |
| GET | `/occurrences/:id/media` | 🟢 | Lista mídias |
| POST | `/occurrences/:id/media` | 🔵 | Envia mídias (`multipart`, campo `media`) |
| DELETE | `/occurrences/:id/media/:mediaId` | 🔵 | Remove uma mídia |
| GET | `/occurrences/:id/reopens` | 🟢 | Histórico de recorrência (cadeia de reaberturas) |
| POST | `/occurrences/:id/reopen` | 🔵 | Reabre (cria nova ocorrência encadeada; `reason` obrigatório) |
| GET | `/occurrences/:id/status-history` | 🟢 | Trilha de auditoria de status |

### Avaliações (votos)

| Método | Rota | Acesso | Descrição |
| --- | --- | --- | --- |
| GET | `/occurrences/:id/evaluations` | 🔵 | Lista votos da ocorrência |
| POST | `/occurrences/:id/upvote` | 🔵 | Vota a favor (idempotente por usuário) |
| POST | `/occurrences/:id/downvote` | 🔵 | Vota contra |
| DELETE | `/occurrences/:id/vote` | 🔵 | Remove o próprio voto |

> Não é possível votar em ocorrências `closed` (**409 `OCCURRENCE_CLOSED`**). Contadores e `score`
> são recalculados de forma transacional.

### Bairros

| Método | Rota | Acesso | Descrição |
| --- | --- | --- | --- |
| GET | `/neighborhoods` | 🟢 | Lista bairros (sem geometria) |
| GET | `/neighborhoods/locate` | 🟢 | Geofencing: bairro que contém `lat`/`lng` |
| GET | `/neighborhoods/:id` | 🟢 | Detalha (com `boundary` e `center_point` em GeoJSON) |
| GET | `/neighborhoods/:id/occurrences` | 🟢 | Ocorrências do bairro |

### Categorias / Subcategorias

| Método | Rota | Acesso | Descrição |
| --- | --- | --- | --- |
| GET | `/categories` · `/categories/:id` | 🟢 | Lista / detalha |
| POST | `/categories` | 🟠 | Cria (`name`, `slug?` derivado, `description?, icon?, color (#RRGGBB)?, is_active?`) |
| PATCH | `/categories/:id` | 🟠 | Atualiza |
| DELETE | `/categories/:id` | 🟠 | Remove (409 se em uso) |
| GET | `/subcategories` · `/subcategories/:id` | 🟢 | Lista / detalha |
| POST | `/subcategories` | 🟠 | Cria (`category_id`, `name`, `slug?`, `description?, icon?, is_active?`) |
| PATCH | `/subcategories/:id` | 🟠 | Atualiza |
| DELETE | `/subcategories/:id` | 🟠 | Remove (409 se em uso) |

### Órgãos

| Método | Rota | Acesso | Descrição |
| --- | --- | --- | --- |
| GET | `/organizations` | 🟢 | Lista órgãos responsáveis (somente leitura) |

### Analytics / Transparência

Filtros comuns (query string): `from`, `to` (datas), `category_id`, `subcategory_id`,
`neighborhood_id`, e `status` (quando aplicável).

| Método | Rota | Acesso | Descrição |
| --- | --- | --- | --- |
| GET | `/analytics/overview` | 🟢 | KPIs globais: totais, por grupo de status, taxa de resolução, tempos médios, recorrência, top categorias |
| GET | `/analytics/by-neighborhood` | 🟢 | Pendências/indicadores por bairro (inclui per capita) |
| GET | `/analytics/heatmap` | 🟢 | Pontos `[{ lat, lng, weight }]` para mapa de calor (`bbox`, `status`, `category_id`, `limit ≤ 50000`) |
| GET | `/analytics/response-time` | 🟢 | Tempo de resposta/resolução (média e mediana), `group_by`: `category \| neighborhood \| month` |
| GET | `/analytics/by-organization` | 🟠 | Eficiência por órgão (backlog, taxa de resolução, reincidência) |

> Os endpoints públicos de analytics expõem apenas **agregados** (sem PII). Os tempos de resposta
> levam em conta o fuso de Brasília e expõem `sample_size` como ressalva de completude.

### Estático

| Método | Rota | Acesso | Descrição |
| --- | --- | --- | --- |
| GET | `/uploads/occurrences/<arquivo>` | 🟢 | Bytes das mídias (somente leitura) |

## 8.2 Autenticação

- O cadastro (`/auth/register`) cria um usuário `citizen` com senha protegida por **bcrypt**.
- O login (`/auth/login`) é por **CPF + senha** e retorna:
  ```json
  {
    "user": { "id": 1, "name": "...", "email": "...", "role": "citizen", "...": "..." },
    "access_token": "<JWT>",
    "refresh_token": "<JWT>",
    "token_type": "Bearer",
    "expires_in": "3h"
  }
  ```
- Envie o access token nas rotas protegidas via header `Authorization: Bearer <access_token>`.
- O **access token** carrega `sub, name, email, role, type: "access"`. O **refresh token**
  (`type: "refresh"`) é persistido no usuário e trocável em `/auth/refresh`; é **revogado** ao
  resetar a senha. Validades por `JWT_EXPIRES_IN` (ex.: `3h`) e `JWT_REFRESH_EXPIRES_IN` (`7d`).
- Dados sensíveis (`password_hash`, `cpf`, tokens) **nunca** são retornados (RN-14).
- **Mock auth:** o backend tem um modo mock ativado **só** por `USE_MOCK_AUTH=true`
  (`middlewares/auth.js`); por padrão roda **JWT real**. Nunca usar em produção.

Detalhes de autorização por papel em [04-perfis-e-permissoes.md](./04-perfis-e-permissoes.md).

## 8.3 Cliente HTTP do frontend

Toda chamada do front passa por `src/lib/api.ts`:

- Usa `import.meta.env.VITE_API_BASE_URL` (ex.: `http://localhost:3000/api`) — **nunca** URL fixa;
  remove barras finais.
- Injeta `Authorization: Bearer <access>` quando `auth !== false`; tokens em `localStorage`
  (`zup_access_token`, `zup_refresh_token`).
- **Refresh automático em 401:** chama `POST /auth/refresh`, repete a requisição **uma vez**
  (`_noRetry`, single-flight para evitar tempestade de refresh) e limpa tokens se o refresh falhar.
  `normalizeSession()`/`doRefresh()` aceitam resposta com ou sem envelope `session` (tolerância a
  variações do contrato de auth).
- Suporta **multipart** (upload de mídia, campo `media`) e montagem de query string.
- Erros não-OK viram `ApiError { message, status, data }` (lê `error`/`message` do corpo).

Módulos por domínio em `src/lib/`: `auth-api.ts`, `occurrences-api.ts`, `categories-api.ts`,
`organizations-api.ts`, `neighborhoods-api.ts`, `evaluations-api.ts`, `analytics-api.ts`,
`validators.ts`. Decisões de contrato no front:

| Decisão | Escolha | Justificativa |
|---------|---------|---------------|
| Formato de coordenadas | **GeoJSON Point `[lng, lat]`** | Padrão GeoJSON/PostGIS; o front mapeia `coordinates[1]→lat`, `coordinates[0]→lng` (`occurrences-api.ts`) |
| Identidade de taxonomia | **id numérico** na fronteira da API, **slug** na lógica do front | id é o que o backend espera; slug é chave estável para mapeamentos (ex.: slug→órgão) |
| Origem de mídia relativa | Prefixar com a **origem do backend** | URLs `/uploads/...` viram absolutas via `resolveMediaUrl` |
| Contorno de bairros | **N+1 deliberado** (lista + detalhe por bairro) | A listagem não traz geometria; só `GET /:id` traz `boundary`. Aceitável para ~15 bairros com cache longo. Ideal futuro: backend expor `boundary` na listagem ou um `GET /neighborhoods/boundaries` |
| Status desconhecido | Normaliza para `pending` | UI resiliente a status fora do enum conhecido |

## 8.4 Formato de erros

Erros retornam JSON com a chave `error` e, quando útil, `details`:

```json
{ "error": "status must be one of: pending, awaiting_validation, ..." }
```

```json
{
  "error": "Cannot change status from 'pending' to 'resolved'",
  "details": { "from": "pending", "to": "resolved", "allowed": ["awaiting_validation", "closed"] }
}
```

```json
{
  "error": "OCCURRENCE_DUPLICATE",
  "details": { "duplicate_id": 42, "distance_m": 18.4 }
}
```

Códigos HTTP usados: **200/201/204** (sucesso), **400** (validação), **401** (não autenticado),
**403** (sem permissão / `EDIT_WINDOW_EXPIRED`), **404** (não encontrado), **409** (conflito:
`OCCURRENCE_DUPLICATE`, `INVALID_STATUS_TRANSITION`, `OCCURRENCE_CLOSED`, `OCCURRENCE_NOT_REOPENABLE`,
`OCCURRENCE_ALREADY_REOPENED`, `OCCURRENCE_IN_USE`), **413/415** (upload), **429** (rate limit).

## 8.5 Upload de mídias

- Endpoint: `POST /api/occurrences/:id/media` (autenticado), `multipart/form-data` com o campo
  **`media`** (até `MAX_UPLOAD_FILES` arquivos).
- Validação por mimetype (allowlist sem SVG), tamanho máximo por arquivo e **nome aleatório gerado
  pelo servidor** (extensão derivada do mimetype; o nome original nunca entra no caminho de disco).
- Os bytes ficam em `UPLOAD_DIR/occurrences/` e são servidos **somente leitura** em `GET /uploads/...`.
- Ao excluir uma ocorrência, as linhas de mídia caem em CASCADE e os arquivos são removidos do disco.
- Erros comuns: **413** (acima do limite), **415** (mimetype não suportado), **400** (campo
  inesperado / sem arquivos).

## 8.6 OpenAPI / Swagger

A especificação completa está em `openapi.json` (OpenAPI 3.0.3, 37 operações) no repositório do
backend. O servidor **não** monta um Swagger UI em runtime (`app.js` não registra `swagger-ui`, não
há `/api-docs` ativo): para visualizar a documentação interativa, importe o `openapi.json` em uma
ferramenta como [Swagger Editor](https://editor.swagger.io/), Redoc ou Postman/Insomnia.
