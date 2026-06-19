# Documentação de Engenharia — Projeto ZUP

**ZUP (Zeladoria Urbana Participativa)** é uma plataforma cívica para centralização de reclamações e
solicitações urbanas com geolocalização, contextualizada no município de **Videira/SC**. Esta pasta
reúne a documentação de engenharia **dos dois lados do projeto** — backend (API REST) e frontend
(aplicação web) — de forma consolidada e sem duplicidade.

## Como ler

Cada documento descreve o **comportamento real** do código:

- **Backend** (`ProjetoZup`) — `src/controllers`, `src/services`, `src/models`, `src/routes`,
  `src/middlewares`, `src/utils`, `openapi.json` e os scripts de `db/`.
- **Frontend** (`ZUP X` / `ProjetoZupFront`) — `src/lib/`, `src/hooks/`, `src/components/`,
  `src/data/`, `src/pages/`.

**O backend é a fonte da verdade** das regras de negócio; o frontend as consome e reflete na
interface. Onde a UI sugere algo que o servidor ainda não impõe (ou diverge dele), o texto aponta a
**limitação/divergência conhecida** de forma objetiva. Recursos ainda **planejados** (não
implementados nesta etapa) aparecem marcados como *(Roadmap)* e são detalhados no
[Plano de Projeto](./03-plano-de-projeto.md).

## Índice

| # | Documento | Prioridade | Conteúdo |
|---|-----------|------------|----------|
| 1 | [Regras de Negócio](./01-regras-de-negocio.md) | **Prioritário** | `RN-xx`, máquina de estados, geofencing, anti-duplicidade, reabertura, votação, janela de edição |
| 2 | [Requisitos Funcionais e Não Funcionais](./02-requisitos.md) | **Prioritário** | `RF-xx`, `RNF-xx`, rastreabilidade requisito → endpoint (back) → módulo (front) |
| 3 | [Plano de Projeto](./03-plano-de-projeto.md) | **Prioritário** | Escopo, equipe, cronograma (back + front), marcos, estado atual, roadmap |
| 4 | [Perfis e Permissões](./04-perfis-e-permissoes.md) | **Prioritário** | Papéis, mapeamento back↔front, matriz de permissões (real e pretendida) |
| 5 | [Documentação Técnica do Backend](./05-backend.md) | Importante | Camadas, autenticação, variáveis de ambiente, ADRs |
| 6 | [Documentação Técnica do Frontend](./06-frontend.md) | Importante | Stack, rotas/telas, estado, camada de mapa e identidade visual |
| 7 | [Modelo de Dados](./07-modelo-de-dados.md) | Importante | Diagrama ER, enums, views de analytics, decisões geoespaciais, integridade referencial |
| 8 | [API e Contrato](./08-api-e-contrato.md) | Importante | Referência de endpoints, autenticação, formato de erros e contrato consumido pelo front |
| 9 | [Diagramas](./09-diagramas.md) | Apoio | Casos de uso, sequência, arquitetura full-stack |
| 10 | [Como Rodar o Projeto](./10-como-rodar.md) | Apoio | Pré-requisitos, `.env`, seed do banco, Docker, Swagger/OpenAPI (back + front) |

## Convenções

- **Idioma:** português. **Diagramas:** Mermaid.
- **Identificadores:** `RN-` (regra de negócio), `RF-` (requisito funcional), `RNF-` (requisito não
  funcional), `ADR-` (decisão de arquitetura), `R-` (item de roadmap), para rastreabilidade.
- **Referências de código** aparecem como `caminho/arquivo` e, quando útil, `arquivo:linha`,
  relativas ao repositório correspondente.

## Funcionalidades planejadas (roadmap)

Alguns recursos do produto **ainda serão implementados**. Estão documentados como **roadmap**
(ver [Plano de Projeto](./03-plano-de-projeto.md)) e marcados como *(Roadmap)* ao longo dos
documentos. Resumo do que ainda não está no código:

| Recurso planejado | Situação atual |
|---|---|
| **Validação por relevância (votação)** | A votação (upvote/downvote → `score`) já funciona; falta a regra que promove a ocorrência ao ultrapassar uma taxa aceitável de apoio. Substitui o antigo papel "Validador" e a validação por adjacência de bairros. |
| **Priorização por votação** | O `score` é calculado e exibido, mas ainda não ordena uma fila de prioridade — as listagens ordenam por data. O front fixa `priority: "media"` até a regra existir. |
| **Geofencing como bloqueio** ao território de Videira | Hoje o geofencing só **deriva o bairro** do ponto (intencional, para testes/visualização); não **bloqueia** ocorrências fora do município. |
| Papel `agent` com permissões operacionais | O papel existe no enum, mas **nenhuma rota** exige `agent`; será desenvolvido junto ao módulo principal (comunidade/registro público), levando consigo a segregação das transições de status. O gating por perfil no front é, até lá, apenas visual. |
| **Notificações** | Não há endpoint nem serviço de notificações; a UI tem o placeholder, mas depende de implementação no backend. |

> A tabela `neighborhood_adjacency` permanece no schema (modelagem da adjacência de bairros), mas
> ficou **reservada** — a validação passou a usar relevância por votação, não adjacência.
