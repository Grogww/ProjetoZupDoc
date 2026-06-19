# 2. Requisitos Funcionais e Não Funcionais

Identificadores: `RF-xx` (funcionais), `RNF-xx` (não funcionais). A coluna **Impl.** indica a
aderência ao código atual: ✅ implementado · 🟡 parcial · ⛔ roadmap (não implementado).

A tabela de requisitos funcionais é **unificada**: cada requisito liga o critério de aceitação ao
**endpoint/módulo do backend** (que o realiza e valida) e ao **módulo do frontend** (que o consome e
expõe na interface). Quando um requisito existe só de um lado, a outra coluna traz "—".

## 2.1 Requisitos Funcionais (RF)

| ID | Requisito | Ator | Critério de aceitação | Backend | Frontend | Impl. |
|----|-----------|------|-----------------------|---------|----------|:-----:|
| RF-01 | Cadastro de usuário | Visitante | CPF com DV, e-mail único; cria `citizen`; senha em bcrypt | `POST /auth/register` · `authService.register` | `Register.tsx`, `registerUser` | ✅ |
| RF-02 | Login por CPF + senha | Cidadão | Credenciais válidas → access + refresh token; inválidas → 401 genérico | `POST /auth/login` · `authService.login` | `Login.tsx`, `loginWithCpf`, `useAuth` | ✅ |
| RF-03 | Renovação de sessão | Cidadão | `refresh_token` válido e não revogado emite nova sessão | `POST /auth/refresh` | `api.ts` (refresh automático em 401) | ✅ |
| RF-04 | Recuperação de senha | Cidadão | Token de uso único por e-mail; resposta genérica; rate-limited | `POST /auth/forgot-password`, `/reset-password` | `ForgotPassword.tsx` | ✅ |
| RF-05 | Ver/editar o próprio perfil | Cidadão | `GET/PATCH /users/me` sem expor dados sensíveis | `usersController.me/updateMe` | `useAuth`, painel do cidadão | ✅ |
| RF-06 | Registrar ocorrência georreferenciada | Cidadão | Título/descrição/lat/lng/categoria válidos criam ocorrência `pending` | `POST /occurrences` | `CreateReportModal.tsx`, `createOccurrence` | ✅ |
| RF-07 | Geofencing (bairro pelo ponto) | Sistema | Sem `neighborhood_id`, deriva o bairro por point-in-polygon (`null` se fora) | `occurrencesService.createOccurrence`, `GET /neighborhoods/locate` | `locateNeighborhood` | 🟡 |
| RF-08 | Visualizar ocorrências próximas no registro | Cidadão | `GET /occurrences/nearby?lat&lng&radius` retorna por distância | `occurrencesController.nearby` | `listNearbyOccurrences`, `CreateReportModal` | ✅ |
| RF-09 | Prevenção de duplicidade | Sistema | Mesma categoria + 500 m + não-finalizada → 409 com `duplicate_id` | `occurrencesService.createOccurrence` | tratamento do 409 em `CreateReportModal` | ✅ |
| RF-10 | Anexar mídias à ocorrência | Cidadão (autor) | Upload `multipart` campo `media`, allowlist/limite; nomes gerados pelo servidor | `POST /occurrences/:id/media` | `uploadOccurrenceMedia` | ✅ |
| RF-11 | Editar ocorrência dentro da janela | Autor/Admin | Edição só por autor/admin e dentro de 24 h; senão 403 | `PATCH /occurrences/:id` | `updateOccurrence` | ✅ |
| RF-12 | Transição de status | Autenticado (alvo: órgão/admin) | Transições validadas pela máquina de estados; inválida → 409 | `PATCH /occurrences/:id/status` | `StatusControl.tsx`, `updateOccurrenceStatus` | 🟡 |
| RF-13 | Acompanhar histórico de status | Qualquer | `GET /occurrences/:id/status-history` lista a trilha | `occurrencesController.listStatusHistory` | `ReportDetailModal`, `listStatusHistory` | ✅ |
| RF-14 | Votar em ocorrência | Cidadão | up/down único por usuário; recálculo transacional; voto em `closed` → 409 | `POST …/upvote`/`downvote`, `DELETE …/vote` | `evaluations-api.ts` | ✅ |
| RF-15 | Listar/filtrar ocorrências | Qualquer | Filtros: status, categoria, subcategoria, bairro, autor, órgão, `limit`/`offset` | `GET /occurrences` | `useOccurrences`, `MapPage`/`MapView` | ✅ |
| RF-16 | Detalhar ocorrência | Qualquer (auth opcional) | Inclui `media`; se autenticado, `voted_user` | `GET /occurrences/:id` | `ReportDetailModal` | ✅ |
| RF-17 | Reabrir/registrar recorrência | Cidadão (alvo: órgão/admin) | Reabre `resolved`/`closed` criando nova ocorrência encadeada; `reason` obrigatório | `POST /occurrences/:id/reopen` | `reopenOccurrence`, `StatusControl` | ✅ |
| RF-18 | Consultar recorrência de um problema | Qualquer | `GET /occurrences/:id/reopens` lista a cadeia pela raiz | `occurrencesController.listReopens` | `mapOccurrenceToReport` | ✅ |
| RF-19 | Visualização por bairro ("Minha Cidade") | Qualquer | `GET /neighborhoods/:id/occurrences` e fronteira/centro em GeoJSON | `neighborhoodsController` | `Dashboard`, `useCityOverview`, `useNeighborhoodStats` | ✅ |
| RF-20 | Mapa interativo com filtros | Visitante | Ocorrências, legenda por status, contorno de bairros; filtros bairro/categoria/status | `GET /occurrences`, `/neighborhoods*` | `MapPage.tsx`, `MapView.tsx`, `useNeighborhoodBoundaries` | ✅ |
| RF-21 | Mapa de calor | Qualquer | Pontos `[{lat,lng,weight}]` por bbox/status/categoria com `limit` | `GET /analytics/heatmap` | `Dashboard` (`leaflet.heat`), `getAnalyticsHeatmap` | ✅ |
| RF-22 | Dashboard — KPIs globais | Qualquer | Totais, grupos de status, taxa de resolução, tempos médios, recorrência, top categorias | `GET /analytics/overview` | `useAnalyticsOverview` | ✅ |
| RF-23 | Dashboard — por bairro | Qualquer/Gestão | Pendências/indicadores por bairro (com per capita) | `GET /analytics/by-neighborhood` | `getAnalyticsByNeighborhood` | ✅ |
| RF-24 | Tempo de resposta/resolução | Qualquer/Gestão | Média e mediana, com `group_by` (categoria/bairro/mês) e `sample_size` | `GET /analytics/response-time` | `getAnalyticsResponseTime` | ✅ |
| RF-25 | Eficiência por órgão | Admin | Backlog, taxa de resolução, reincidência por órgão | `GET /analytics/by-organization` | `getAnalyticsByOrganization` | ✅ |
| RF-26 | Painel do cidadão | Cidadão | Lista minhas ocorrências e atividade | `GET /occurrences?author_id=` | `CitizenPanel.tsx`, `useMyOccurrences` | ✅ |
| RF-27 | Painel de gestão pública | Órgão/Admin | Triagem, mudança de status, órgão atribuído | `GET /occurrences`, `PATCH …/status` | `GestaoPanel.tsx`, `InstitutionalPanel.tsx` | 🟡 |
| RF-28 | Gestão de usuários e papéis | Admin | Listar/detalhar usuários e alterar `role` | `GET /users`, `/users/:id`, `PATCH /users/:id/role` | `AdminPanel.tsx`, `listUsers` | ✅ |
| RF-29 | Gerir categorias/subcategorias | Admin | CRUD com slug, ícone, cor, ativação; remoção em uso → 409 | `/categories`, `/subcategories` | taxonomia (`useTaxonomy`, leitura) | ✅ |
| RF-30 | Listar órgãos | Qualquer | `GET /organizations` lista órgãos responsáveis | `organizationsController.list` | `useTaxonomy`, `organizations-api` | ✅ |
| RF-31 | Onboarding | Usuário | Primeiro acesso exibe orientação de uso | — | `OnboardingModal.tsx` | ✅ |
| RF-32 | Suporte / FAQ | Visitante | FAQ navegável e formulário de contato | — *(sem endpoint de envio)* | `Support.tsx`, `support/*`, `useSupportContact` | 🟡 |
| RF-33 | Taxonomia viva | Sistema | Categorias, subcategorias, bairros e órgãos vêm da API (não hardcoded) | `/categories`, `/subcategories`, `/neighborhoods`, `/organizations` | `useTaxonomy.ts` | ✅ |
| RF-34 | Healthcheck | Qualquer | `GET /health` → `{ status: "ok" }` | `healthRoutes` | — | ✅ |
| RF-35 | Validação por relevância (votação) | Sistema | Upvotes/downvotes apuram a relevância; ao ultrapassar uma taxa aceitável, promove a `validated` | `PATCH /occurrences/:id/status` (futuro) | — | ⛔ |
| RF-36 | Priorização por votação | Sistema | `score` alimenta a fila/ordenação de prioridade das ocorrências validadas | — | enum `Priority` (placeholder de UI) | ⛔ |
| RF-37 | Geofencing como bloqueio municipal | Sistema | Rejeitar ocorrência fora dos limites de Videira | — | — | ⛔ |
| RF-38 | Notificações | Cidadão | Notificar autor/seguidores em mudanças de status | — | placeholder na UI | ⛔ |

> **Sobre RF-12/RF-14/RF-17 (autorização):** no servidor, nenhuma dessas rotas usa `requireRole` —
> apenas `auth`. `PATCH /occurrences/:id/status`, `POST /occurrences/:id/reopen` e os votos ficam
> abertos a qualquer usuário autenticado. A restrição por perfil no front (esconder o `StatusControl`
> para não-institucionais) é **apenas visual**; a restrição efetiva por papel depende do backend (ver
> [01-regras-de-negocio.md](./01-regras-de-negocio.md), RN-05; e [04-perfis-e-permissoes.md](./04-perfis-e-permissoes.md)).
> Na edição/exclusão, a regra de autor/admin + janela de 24 h já é aplicada (RN-10).

> **Sobre RF-38 (notificações):** o backend não expõe módulo de notificações (nenhum endpoint ou
> serviço). O requisito depende de implementação no servidor antes de o front poder consumi-lo.

### Histórias de usuário (amostra dos RF prioritários)

**RF-06 — Registrar ocorrência georreferenciada**
> *Como* cidadão, *quero* registrar um problema urbano com localização no mapa e foto, *para que*
> a administração tome conhecimento.
> **Critérios de aceitação:**
> - Dado título, descrição, latitude/longitude válidas e uma categoria existente, a ocorrência é
>   criada com status `pending` e devolve **201** com o recurso.
> - Se já houver ocorrência da **mesma categoria** aberta a ≤ 500 m, recebo **409** com o
>   `duplicate_id` e a distância.
> - Se eu não informar o bairro, o sistema o deriva da localização (ou deixa `null` se fora dos
>   bairros cadastrados).

**RF-12 — Transição de status**
> *Como* responsável pelo acompanhamento, *quero* avançar o status da ocorrência, *para que* o
> cidadão acompanhe a evolução.
> **Critérios de aceitação:**
> - Uma transição prevista pela máquina de estados atualiza o status, carimba `resolved_at`/
>   `closed_at` quando aplicável e registra o histórico.
> - Uma transição não prevista retorna **409** com `{ from, to, allowed }`.
> - *Roadmap:* nesta etapa qualquer autenticado pode transicionar; a restrição dos estados
>   operacionais ao papel `agent`/`admin` será aplicada junto à evolução do módulo do agente.

**RF-17 — Reabrir / registrar recorrência**
> *Como* cidadão, *quero* reabrir um problema que voltou a ocorrer, *para que* fique registrado
> que é reincidência e não um caso novo isolado.
> **Critérios de aceitação:**
> - Só ocorrências `resolved`/`closed` podem ser reabertas; senão **409**.
> - A reabertura cria uma **nova** ocorrência `pending` encadeada (`parent`/`root`/`reopen_count`),
>   exige `reason` e grava auditoria em `occurrence_reopens`.
> - Reabrir uma ocorrência que já foi reaberta retorna **409** apontando a última da cadeia.

---

## 2.2 Requisitos Não Funcionais (RNF)

Os RNF estão divididos por camada porque cada lado atende preocupações distintas: o **backend**
responde por desempenho geoespacial, segurança, integridade e portabilidade do servidor; o
**frontend** responde por desempenho de cliente, usabilidade, acessibilidade e resiliência da UI.

### 2.2.1 Backend

| ID | Categoria | Requisito | Como é atendido |
|----|-----------|-----------|-----------------|
| RNF-B01 | Desempenho | Consultas geoespaciais eficientes | `ST_DWithin`/`ST_Contains` sobre `geometry` SRID 4326; cast para `::geography` só para distância real. Índices **GiST** no DDL: `idx_occurrences_location`, `idx_neighborhoods_boundary`, `idx_neighborhoods_center` (+ btree em status/category/neighborhood/created_at) |
| RNF-B02 | Desempenho | Pool de conexões ao banco | `pg.Pool` em `config/database.js`; `SELECT NOW()` de verificação no boot |
| RNF-B03 | Desempenho | Limites de paginação e de volume | `GET /occurrences` `limit ≤ 200`; heatmap `limit ≤ 50000`; `nearby` raio ≤ 50 km |
| RNF-B04 | Segurança | Autenticação stateless | JWT (`access`/`refresh`) assinado com `JWT_SECRET`; access carrega `sub,name,email,role,type` |
| RNF-B05 | Segurança | Senhas protegidas | bcrypt (`BCRYPT_SALT_ROUNDS`, padrão 10); senha nunca retornada |
| RNF-B06 | Segurança | Proteção de endpoints sensíveis | `express-rate-limit` em `forgot/reset-password`; token de reset hasheado (SHA-256) e de uso único |
| RNF-B07 | Segurança | Anti-fraude / anti-duplicidade | CPF com DV validado (RN-01); duplicidade por raio (RN-04) |
| RNF-B08 | Segurança | Upload seguro | Allowlist de mimetypes (sem SVG), limite de tamanho/quantidade, nomes gerados pelo servidor, servir somente leitura |
| RNF-B09 | Privacidade | Minimização de dados (LGPD) | `sanitize()` remove `password_hash/cpf/tokens`; analytics público só agregados |
| RNF-B10 | Integridade | Consistência geográfica | SRID 4326 (WGS84) uniforme e **tipado**; geometria exposta como GeoJSON (`ST_AsGeoJSON`). Bairros vieram do **IBGE** em **SIRGAS 2000** e foram **reprojetados para 4326** com PostGIS na importação |
| RNF-B11 | Integridade | Transações atômicas | `BEGIN/COMMIT/ROLLBACK` em criação, transição, voto e reabertura; travas `FOR UPDATE` |
| RNF-B12 | Integridade | Fuso horário consistente | Sessão do banco em `America/Sao_Paulo`; cálculos de analytics ajustam o fuso |
| RNF-B13 | Usabilidade | Mensagens de erro padronizadas | JSON `{ error, details? }` com códigos HTTP consistentes (ver [08-api-e-contrato.md](./08-api-e-contrato.md)) |
| RNF-B14 | Portabilidade/custo | Ferramentas abertas e gratuitas | Node.js, PostgreSQL/PostGIS; sem dependência paga |
| RNF-B15 | Portabilidade | Containerização | Docker + Docker Compose (perfis dev/prod); restore automático do dump no primeiro boot |
| RNF-B16 | Manutenibilidade | Arquitetura em camadas | routes → controller → service → model; erros de domínio com `err.code` simbólico |
| RNF-B17 | Manutenibilidade | Documentação de API | OpenAPI 3.0 versionado (`openapi.json`) |
| RNF-B18 | Configurabilidade | Parâmetros por ambiente | `.env`/`dotenv` para portas, JWT, SMTP, limites de upload, janela de edição, rate limit |

> **Próximos passos de RNF de backend (planejados):** testes automatizados, *migrations* versionadas
> (o schema hoje é distribuído como dump binário), pipeline de CI/CD e cache/TTL nos endpoints
> públicos de analytics — ver [Plano de Projeto → Roadmap](./03-plano-de-projeto.md).

### 2.2.2 Frontend

| ID | Categoria | Requisito | Como é atendido |
|----|-----------|-----------|-----------------|
| RNF-F01 | Desempenho | Cache de estado de servidor | **TanStack Query** com `staleTime` por domínio (30 s–1 h) evita refetch desnecessário; geofencing/nearby/heatmap delegados ao PostGIS no back |
| RNF-F02 | Desempenho | Carga única da taxonomia | Categorias/subcategorias/bairros/órgãos com `staleTime` de 1 h em `useTaxonomy` |
| RNF-F03 | Segurança | Autenticação por token | **JWT Bearer** injetado em toda chamada autenticada; **refresh automático** em 401 (`api.ts`) |
| RNF-F04 | Segurança | Proteção de rotas internas | `ProtectedRoute` redireciona não autenticados e não institucionais |
| RNF-F05 | Integridade | Consistência geográfica | Localização como **GeoJSON Point `[lng, lat]`**; mapeamento `coordinates[1]=lat`, `coordinates[0]=lng` (`occurrences-api.ts`) |
| RNF-F06 | Integridade | Robustez do mapeamento de status | Status desconhecido normaliza para `pending` (`normalizeStatus`) — a UI nunca quebra |
| RNF-F07 | Usabilidade | Registro guiado | Formulário em 4 passos (Local → Fotos → Detalhes → Revisão) com validação por passo |
| RNF-F08 | Usabilidade | Responsividade e acessibilidade | **shadcn/ui + Radix**, `aria-*`, tema claro/escuro (`useTheme`) |
| RNF-F09 | Usabilidade | SEO básico | `react-helmet-async` (`Seo.tsx`) |
| RNF-F10 | Portabilidade/custo | Ferramentas abertas | **OpenStreetMap** (tiles), Leaflet, React/Vite — sem custo de licença |
| RNF-F11 | Portabilidade | Configuração por ambiente | `VITE_API_BASE_URL` (nunca URL fixa); container desacoplado via Nginx + `BACKEND_URL` |
| RNF-F12 | Manutenibilidade | Camada de integração isolada | `src/lib/*-api.ts` por domínio; hooks (`src/hooks/`) estabilizam o shape para a UI |
| RNF-F13 | Confiabilidade | Tolerância a falhas de UI | `ErrorBoundary` global; falha de upload não perde a ocorrência |

> **Sobre RNF-F10/RNF-B14 (tiles abertos):** os tiles usam o servidor público do OpenStreetMap
> (`OSM_URL` em `MapView.tsx`; URL equivalente em `Dashboard.tsx`), adequado a desenvolvimento e
> demonstração. Para tráfego de **produção**, recomenda-se um provedor de tiles com cota própria,
> conforme a política de uso do OSM (ver [Roadmap](./03-plano-de-projeto.md)).
