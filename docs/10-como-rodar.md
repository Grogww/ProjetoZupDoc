# 10. Como Rodar o Projeto

> Instruções de execução das **duas aplicações**. O backend (`ProjetoZup`) sobe a API + banco; o
> frontend (`ZUP X` / `ProjetoZupFront`) consome essa API. Para o sistema funcionar de ponta a ponta,
> o backend precisa estar no ar antes do front.

- **Backend:** <https://github.com/Grogww/ProjetoZup>
- **Frontend:** <https://github.com/Grogww/ProjetoZupFront>

## 10.1 Backend (API ProjetoZup)

### Pré-requisitos

- **Node.js 20+** e **npm**.
- **PostgreSQL 17 + PostGIS 3.5** acessível (ou usar o Docker Compose, que já provê o banco).

### Com Docker (recomendado)

Sobe o banco (PostGIS) e a API já conectados. O banco restaura automaticamente o dump de `db/init/`
no primeiro boot.

1. Copie o modelo de variáveis e ajuste os valores:
   ```bash
   cp .env.example .env
   ```
2. Suba os serviços:
   ```bash
   # Produção
   npm run docker:prod          # docker compose up --build

   # Desenvolvimento (hot reload via nodemon + volume montado)
   npm run docker:dev           # docker compose -f docker-compose.yml -f docker-compose.dev.yml up --build
   ```
3. A API ficará disponível em `http://localhost:<PORT>` (padrão `3000`). Teste:
   ```
   GET http://localhost:3000/api/health  →  { "status": "ok" }
   ```

No Compose, o serviço `api` recebe automaticamente `DB_HOST=db` e `DB_PORT=5432`; os uploads são
persistidos no volume `zup_uploads` e os dados do Postgres no volume `zup_pgdata`.

### Localmente (sem Docker)

1. `npm install`.
2. Crie o `.env` (ver §10.3) apontando `DB_*` para o seu Postgres.
3. Garanta a extensão PostGIS e restaure o schema/seed (ver §10.4).
4. Rode:
   ```bash
   npm run dev      # com nodemon (auto-restart)
   npm start        # produção
   ```

### Scripts npm (backend)

| Script | Ação |
| --- | --- |
| `npm start` | Sobe o servidor (produção) |
| `npm run dev` | Sobe com `nodemon` (auto-restart) |
| `npm run docker:dev` | Docker Compose em modo desenvolvimento (hot reload) |
| `npm run docker:prod` | Docker Compose em modo produção |

## 10.2 Frontend (ZUP X)

### Pré-requisitos

- **Node.js 20+** e **npm**.
- O **backend ProjetoZup** rodando (por padrão `http://localhost:3000`, rotas sob `/api`).

### Configuração de ambiente

```bash
cp .env.example .env
```

```env
# URL base da API Node/Express (inclua o /api)
VITE_API_BASE_URL=http://localhost:3000/api
```

> Toda chamada HTTP usa `VITE_API_BASE_URL` — nunca uma URL fixa. Sem a variável, o app avisa no
> console e as requisições falham.

### Instalação e execução (dev)

```bash
npm install
npm run dev
```

Aplicação em **http://localhost:8080**.

### Scripts (frontend)

| Script | Descrição |
|--------|-----------|
| `npm run dev` | Servidor de desenvolvimento (Vite), porta `8080` |
| `npm run build` | Build de produção |
| `npm run build:dev` | Build em modo desenvolvimento |
| `npm run preview` | Pré-visualiza o build |
| `npm run lint` | Análise estática (ESLint) |

### Docker (deploy desacoplado do front)

A imagem é genérica: o bundle chama `/api` (relativo) e o **Nginx** do container faz **proxy
reverso** para o backend definido em runtime por `BACKEND_URL` (sem CORS; com fallback de SPA). A
mesma imagem roda em qualquer ambiente, sem rebuildar.

```bash
# recomendado
docker compose up --build           # front em http://localhost:8080

# ou só docker
docker build -t zup-frontend .
docker run -p 8080:80 -e BACKEND_URL=http://host.docker.internal:3000 zup-frontend
```

| Variável (container) | Padrão | Descrição |
|----------------------|--------|-----------|
| `BACKEND_URL` | `http://host.docker.internal:3000` | Origem do backend **sem** `/api`. O Nginx repassa `/api/...` para lá. Use `host.docker.internal` quando o back roda na máquina host; use o nome do serviço (ex.: `http://back:3000`) se ambos estiverem no mesmo compose |

Arquivos relacionados (repo do front): `Dockerfile` (multi-stage Node 20 → Nginx), `nginx.conf`
(proxy `/api` + fallback SPA), `docker-compose.yml`, `.dockerignore`.

> Os valores casam com o backend: `FRONTEND_URL` (padrão `http://localhost:5173`) e
> `FRONTEND_RESET_PATH` (`/reset-password`) são usados pelo backend para montar o link do e-mail de
> recuperação de senha — no front, a rota correspondente é `/recuperar-senha`. Ajuste `FRONTEND_URL`
> no backend para a porta real do front (`8080`) quando necessário.

## 10.3 Variáveis de ambiente do backend

Configuradas via `.env` (carregado pelo `dotenv`). Ver `.env.example` para o modelo completo.

| Variável | Descrição | Padrão |
| --- | --- | --- |
| `PORT` | Porta HTTP da API | `3000` |
| `DB_HOST` / `DB_PORT` / `DB_USER` / `DB_PASSWORD` / `DB_NAME` | Conexão com o PostgreSQL | — |
| `JWT_SECRET` | Segredo para assinar os JWT (string longa e aleatória) | — |
| `JWT_EXPIRES_IN` | Expiração do access token (`1h`, `24h`, `7d`…) | `3h` (`24h` se ausente) |
| `JWT_REFRESH_EXPIRES_IN` | Expiração do refresh token | `7d` |
| `BCRYPT_SALT_ROUNDS` | Custo do bcrypt | `10` |
| `SMTP_HOST` / `SMTP_PORT` / `SMTP_SECURE` / `SMTP_USER` / `SMTP_PASS` | Servidor SMTP | `smtp.gmail.com` / `465` / `true` |
| `MAIL_FROM` | Remetente dos e-mails | `SMTP_USER` |
| `APP_NAME` | Nome usado nos e-mails | `ProjetoZup` |
| `FRONTEND_URL` / `FRONTEND_RESET_PATH` | Base e caminho do front para o link de reset | `http://localhost:5173` / `/reset-password` |
| `RESET_TOKEN_BYTES` | Tamanho (bytes) do token de reset | `32` |
| `RESET_TOKEN_EXPIRES_MINUTES` | Validade do token de reset (min) | `30` |
| `RATE_LIMITERS_ENABLED` | Liga/desliga os limitadores | `true` |
| `RATE_LIMIT_FORGOT_PASSWORD_WINDOW_MS` / `_MAX` | Janela/limite para `forgot-password` | `900000` / `5` |
| `RATE_LIMIT_RESET_PASSWORD_WINDOW_MS` / `_MAX` | Janela/limite para `reset-password` | `900000` / `10` |
| `UPLOAD_DIR` | Pasta raiz dos uploads (em volume no Docker) | `uploads` |
| `PUBLIC_BASE_URL` | Base pública para a URL das mídias (vazio = relativa) | _vazio_ |
| `MAX_UPLOAD_MB` | Tamanho máximo por arquivo (MB) | `10` |
| `MAX_UPLOAD_FILES` | Máximo de arquivos por requisição | `5` |
| `ALLOWED_MIME_TYPES` | Mimetypes permitidos (CSV) | `image/jpeg,image/png,image/webp,image/gif` |
| `OCCURRENCE_EDIT_WINDOW_HOURS` | Janela de edição da ocorrência (horas) | `24` |
| `USE_MOCK_AUTH` | Se `true`, injeta um usuário admin fake (apenas dev) | _não definido_ |
| `SA_PASSWORD` | Senha de SA do banco (uso opcional de infra) | — |

> **Segurança:** SVG fica de fora da allowlist de mídias de propósito (risco de XSS). `USE_MOCK_AUTH=true`
> **nunca** deve ser usado em produção.

### Variáveis do frontend

| Variável | Onde | Obrigatória | Função |
|----------|------|:-----------:|--------|
| `VITE_API_BASE_URL` | build/dev (`.env`) | ✅ | URL base da API **incluindo `/api`** (ex.: `http://localhost:3000/api`) |
| `BACKEND_URL` | runtime do container | ✅ (Docker) | Origem do backend **sem** `/api`. O Nginx faz proxy de `/api/...` para lá |

## 10.4 Banco de dados / seed

O **frontend não possui banco**. A criação do schema (PostGIS), as migrations e o **seed** (bairros
de Videira com geometria, categorias, órgãos) são executados no **backend**. Sem bairros com
geometria carregados, o geofencing (`/neighborhoods/locate`) e o contorno do mapa ficam vazios.

O backend usa **PostgreSQL 17 + PostGIS 3.5**:

- **Via Docker:** nada manual é necessário. No primeiro boot, o container do banco executa
  `db/_CreateExtensionPostGIS.sql` (habilita PostGIS) e depois `db/restore.sh`, que faz `pg_restore`
  do dump em `db/init/zup_backup.backup`.
- **Sem Docker:** habilite a extensão e restaure o backup:
  ```sql
  CREATE EXTENSION IF NOT EXISTS postgis;
  ```
  ```bash
  pg_restore --username "$DB_USER" --dbname "$DB_NAME" --no-owner --no-privileges --verbose db/init/zup_backup.backup
  ```

O fuso horário esperado pela sessão do banco é **`America/Sao_Paulo`** (Brasília), relevante para os
cálculos de tempo do módulo de analytics. A conexão usa um **pool** (`pg.Pool`) e, ao iniciar, a
aplicação faz um `SELECT NOW()` de teste.

### Acesso de demonstração (dados sanitizados)

O backup público (`db/init/zup_backup.backup`) é **sanitizado**: os dados pessoais reais dos usuários
(nome, e-mail, CPF e senha) são substituídos por valores fictícios — porém **válidos** para passar
nas validações do sistema (o login exige um CPF com dígitos verificadores corretos).

O login é por **CPF + senha**. Todos os usuários do backup de demonstração têm **role `admin`**
(acesso total) e a **mesma senha**:

| Usuário     | CPF (login)      | Senha       | Role    |
| ----------- | ---------------- | ----------- | ------- |
| `Usuário 1` | `001.234.567-97` | `Demo@1234` | `admin` |
| `Usuário 2` | `002.469.134-87` | `Demo@1234` | `admin` |
| `Usuário 3` | `003.703.701-39` | `Demo@1234` | `admin` |

> O CPF pode ser enviado com ou sem máscara (ex.: `001.234.567-97` ou `00123456797`).

Exemplo de login:

```bash
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{ "cpf": "00123456797", "password": "Demo@1234" }'
```

### Regenerar o backup público (mantenedores)

Os backups **oficiais com dados reais** ficam em `db/init/history/` e **não vão para o repositório**
(ignorados no `.gitignore`). Sempre que esses dados mudarem, regenere o backup público sanitizado
(Docker em execução, a partir da raiz do projeto):

```powershell
./db/regenerate-public-backup.ps1
```

O script sobe um container PostGIS temporário e isolado, restaura o backup oficial mais recente de
`history/`, executa `db/sanitize.sql` e gera o novo `db/init/zup_backup.backup` — removendo o
container ao final. Use `-OfficialBackup <arquivo>` para escolher outra origem. **Só** o
`db/init/zup_backup.backup` (sanitizado) deve ser commitado.

## 10.5 Documentação Swagger / OpenAPI

O backend **não** serve um Swagger UI em runtime (`app.js` não monta `swagger-ui`; não há `/api-docs`
ativo). O contrato existe como arquivo estático `openapi.json` (OpenAPI 3.0.3, 37 operações) na raiz
do repositório do backend. Para visualizar a documentação interativa, importe esse `openapi.json` em
[Swagger Editor](https://editor.swagger.io/), Redoc ou Postman/Insomnia. Ver
[08. API e Contrato](./08-api-e-contrato.md).
