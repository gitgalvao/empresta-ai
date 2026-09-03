# ADR 001: Definição da Stack Tecnológica e Arquitetura Base

- **Status:** Aceito
- **Data:** 2026-09-03
- **Decisores:** Time Técnico / Operações (empresta.ai)
- **Origem / Insumo:** [docs/STACKS.md](../STACKS.md)

---

## 1. Contexto e Problema

O projeto **empresta.ai** tem como objetivo resolver o controle interno de empréstimos de equipamentos (notebooks, monitores, periféricos, câmeras etc.), hoje gerido por planilhas manuais com alto índice de desorganização, atrasos e perda de rastreabilidade.

Para viabilizar a entrega de forma rápida, previsível, com baixo custo operacional e alinhada ao escopo didático/acadêmico da disciplina, faz-se necessária a padronização das decisões técnicas de infraestrutura, arquitetura de software, frontend, backend, persistência, autenticação e processo de integração contínua.

---

## 2. Decisão

Adota-se a seguinte composição de stack tecnológica e arquitetura:

### 2.1 Repositórios e Estrutura
- **Dois repositórios separados:** Um para o frontend (SPA) e um para a API (backend). Front e API possuem ciclos de release, dependências e responsáveis distintos. A separação mantém histórico e permissões segregados.
- **Projetos na Vercel:** Dois projetos independentes, mapeando cada repositório.
- **Compartilhamento de tipos via OpenAPI:** Não haverá monorepo nem publicação de pacote npm privado de tipos. O contrato atravessa a fronteira via arquivo de especificação OpenAPI gerado automaticamente pelo backend e versionado no front.

### 2.2 Frontend
- **Linguagem:** TypeScript em modo `strict` (garante tratamento rigoroso de nulos alinhado aos tipos da API).
- **Biblioteca de UI:** React.
- **Build / Dev Server:** Vite (SPA leve, sem overhead de SSR/framework full-stack).
- **Roteamento:** React Router (data APIs) com rotas aninhadas e proteção por sessão.
- **Estado de Servidor:** TanStack Query (cache, revalidação, retries e estados assíncronos prontos).
- **Estado de Cliente:** Zustand (gestão pontual de filtros, modais e wizards de UI).
- **Formulários:** React Hook Form integrado a schemas Zod via `zodResolver`.
- **Estilização e Componentes:** Tailwind CSS com componentes copiados via **shadcn/ui** (customização direta e fidelidade total ao design Figma).
- **Cliente da API:** Gerado automaticamente a partir de `openapi.json` usando **orval**.

### 2.3 Backend
- **Runtime:** Node.js LTS com TypeScript.
- **Framework:** NestJS (modularização clara e injeção de dependência para facilidade de testes unitários).
- **Adapter HTTP Serverless:** Express via `@vendia/serverless-express` com instância da aplicação cacheada (evita o custo de re-bootstrap do container DI em cada request na Vercel).
- **Organização:** Por domínio (`src/modules/<dominio>/`).
- **Camadas:** `Controller` → `Service` → `Repository` (Repository isola chamadas do Prisma).
- **Validação de Entrada:** Zod via `nestjs-zod` (fonte única para validação runtime e documentação OpenAPI).
- **Contexto de Request:** `AsyncLocalStorage` para propagação segura de `tenant_id` e `user_id` sem passar manualmente por parâmetro.

### 2.4 Contrato de API
- **Estilo:** REST sobre HTTP, com prefixo de rota versionado `/v1`.
- **Fonte do Contrato:** OpenAPI gerado a partir dos schemas Zod do NestJS.
- **Distribuição:** CI da API gera e publica o artefato `openapi.json`.
- **Consumo:** Frontend executa script de geração de cliente versionado no repositório.
- **Formato de Erro:** RFC 9457 (Problem Details).
- **Paginação:** Baseada em cursor (performance previsível e consistência contra escritas concorrentes).
- **Comunicação em Tempo Real:** Fora de escopo (atualizações no front ocorrem via polling do TanStack Query).

### 2.5 Dados e Persistência
- **Banco de Dados:** PostgreSQL hospedado e gerenciado no Supabase (região `sa-east-1` - São Paulo).
- **ORM:** Prisma.
- **Dono do Schema:** Prisma Migrate como **dono único** de todas as migrações estruturais.
- **Objetos SQL Nativos:** RLS, policies, functions e triggers versionados como SQL bruto dentro das próprias migrações do Prisma Migrate.
- **Conexão de Runtime:** Supavisor em porta 6543 (`?pgbouncer=true&connection_limit=1`) para evitar esgotamento de pool em ambiente serverless.
- **Conexão de Migração:** `directUrl` em porta 5432 (porta direta necessária para comandos DDL).
- **Seed:** Script `prisma/seed.ts` idempotente, provisionando o tenant inicial `suporte_ti`.

### 2.6 Multi-Tenancy
- **Modelo:** Tenant discriminado por coluna (`tenant_id`) em banco de dados único.
- **Mapeamento:** Coluna `tenant_id` em todas as tabelas de domínio do sistema.
- **Vínculo Usuário–Tenant:** Tabela `memberships (user_id, tenant_id, role)`.
- **Primeiro Tenant:** `suporte_ti` gerado no seed inicial.
- **Origem do Tenant no Request:** Extraído diretamente da claim validada do JWT, processado em Guards do Nest, nunca aceito via corpo da requisição ou query parameters.

### 2.7 Autenticação e Autorização
- **Provedor de Identidade:** Supabase Auth (e-mail e senha).
- **Cadastro:** Fechado (somente por convite emitido por administradores).
- **Sessão:** Bearer Token no cabeçalho HTTP gerenciado no cliente via SDK `supabase-js`.
- **Domínio:** `*.vercel.app` (um por projeto).
- **Validação de Token na API:** NestJS valida assinatura JWT diretamente contra os JWKS públicos do Supabase.
- **CORS:** Allowlist restrita com a origem do front e `credentials: true`.
- **Autorização Primária:** Guards do NestJS com biblioteca **CASL** (autorização por papel e tenant).
- **RLS (Row Level Security):** Habilitado em todas as tabelas com política *deny by default*, atuando como camada de defesa em profundidade.

### 2.8 Testes e Qualidade
- **Backend:** Testes unitários com Jest; testes de integração contra banco de dados real via **Testcontainers** (PostgreSQL real); testes de API com **supertest**.
- **Frontend:** Testes unitários com Vitest; testes E2E com Playwright cobrindo fluxos críticos de ponta a ponta.
- **Teste Mandatório de Multi-tenant:** Teste automatizado por endpoint validando que o Tenant A não enxerga dados do Tenant B.
- **Verificação de Contrato:** Job no CI do front que falha se o cliente gerado divergir do `openapi.json` emitido pela API.
- **Padronização:** ESLint + Prettier com regras padronizadas; hook pré-commit via `husky` + `lint-staged`.

### 2.9 Hospedagem e Operação
- **Hospedagem:** Vercel no plano Hobby (functions nos EUA, teto de 60s por requisição).
- **Configurações:** Variáveis de ambiente validadas via Zod no startup da aplicação.
- **Logs:** Pino estruturado em JSON contendo `request_id` e `tenant_id`.
- **Observabilidade de Erros:** Sentry integrado no front e na API.
- **CI/CD:** GitHub Actions (workflows dedicados por repositório); deploy de migrações (`prisma migrate deploy`) executado via job de CI antes do deploy da aplicação.

### 2.10 Segurança
- **Cabeçalhos:** `helmet` ativo na API.
- **CSP:** Content Security Policy restritiva sem `unsafe-inline`.
- **Rate Limit:** Rate limiting por IP e por usuário persistido no Postgres (evitando contadores isolados em memória).
- **Chaves Privilegiadas:** `service_role` restrita ao backend, jamais exposta no bundle client-side.
- **Auditoria:** Tabela append-only registrando autor, tenant, ação e timestamp.

---

## 3. Alternativas Descartadas

| Alternativa | Motivo do Descarte |
|---|---|
| Monorepo com pacote de tipos compartilhado | Contraria a decisão de segregação de repositórios, pipelines e permissões. |
| ts-rest | Exige importação direta do mesmo pacote em ambos os lados, inviável em repositórios separados. |
| Pacote npm privado com tipos | Complexidade de release e atrito de versionamento desproporcionais ao ganho. |
| RLS como autorização primária | Prisma conecta como dono das tabelas e bypassa RLS por padrão. Impor RLS exigiria transações manuais com `SET LOCAL` em cada request, penalizando o pooler. |
| Front comunicando direto com Supabase sem API | Elimina camada central para regras de negócio, auditoria e isolamento de tenant. |
| Cadastro público aberto | Sem domínio corporativo para filtrar, qualquer usuário externo poderia criar conta e acessar inventário de TI. |
| SSO corporativo (OIDC/SAML) | Inexistência de diretório/domínio corporativo próprio no contexto atual. |
| Cookie `httpOnly` para sessão | Subdomínios `vercel.app` constam na Public Suffix List, impedindo compartilhamento de cookies entre front e API sem domínio personalizado. |
| Domínio próprio customizado | Custo e overhead de configuração desnecessários para a fase acadêmica. |
| Supabase CLI gerindo migrações concorrentemente | Ter dois donos de migração (Prisma Migrate + Supabase CLI) causa conflitos de estado no schema. |
| Filas assíncronas (BullMQ, pg-boss) / Workers | Ausência de volume que justifique e impossibilidade de manter workers de longa duração na Vercel. |
| Vercel Cron | Ausência de rotinas agendadas no escopo da primeira versão. |
| Banco ou schema separado por tenant | Complexidade desnecessária de manutenção de DDL para um sistema interno de escopo delimitado. |
| `@nestjs/throttler` em memória | Em funções serverless escaladas horizontalmente, o estado em memória não limita requisições reais. |
| tRPC / GraphQL | Adoção de REST já consolidada para simplificar integração e suporte a ferramentas padronizadas. |
| Cliente HTTP escrito à mão | Risco elevado de inconsistência com os tipos da API em produção. |
| `class-validator` | Não unifica validação em runtime e geração de schema OpenAPI a partir da mesma fonte (Zod). |
| Jest no frontend | Incompatibilidade nativa com a cadeia de build do Vite, exigindo transpilação paralela. |
| Redux Toolkit | Complexidade e boilerplate desnecessários para o volume reduzido de estado de cliente. |
| Componentes com design opinionated (MUI, Mantine) | Conflito com a identidade visual e protótipo definido no Figma. |
| Next.js | SPA atende integralmente os requisitos sem o acoplamento de backend/SSR. |
| VPS / Containers dedicados (Fly.io, AWS ECS) | Custo adicional e complexidade de orquestração fora do escopo da hospedagem serverless escolhida. |

---

## 4. Consequências

### Positivas (Ganhos)
- **Independência Operacional:** Repositórios, deploys e ciclos de entrega de frontend e backend desacoplados.
- **Alta Produtividade e Tipagem Ponta a Ponta:** Tipagem estrita desde o banco (Prisma) até o formulário do front (Orval + Zod + React Hook Form).
- **Testabilidade:** Módulos e Services do NestJS testáveis isoladamente com mocks, enquanto repositórios são validados contra Postgres real com Testcontainers.
- **Autorização Centralizada:** Regras com CASL concentram permissões em um único local auditável.
- **Custo Zero:** Execução viável dentro das camadas gratuitas (Vercel Hobby e Supabase Free Tier).

### Negativas (Trade-offs e Desafios)
- **Coordenação de Releases:** Features que alteram contratos demandam PRs e sincronização manual entre repositórios.
- **Latência Transatlântica:** Serverless functions da Vercel no plano Hobby rodam nos EUA, enquanto o banco Supabase está em São Paulo (`sa-east-1`).
- **Cold Starts:** Requests após períodos ociosos sofrem com o tempo de boot do runtime do NestJS.
- **Restrição de Tempo (60s):** Todas as operações síncronas precisam completar rigorosamente abaixo de 60 segundos.
- **Segurança do Token:** O token JWT mantido no cliente requer Content Security Policy rigorosa para mitigar riscos de XSS.

### Limitações Futuras (Custos de Reversão)
- **WebSockets / Tempo Real:** A arquitetura serverless na Vercel impede conexões persistentes; futuras demandas exigirão polling ou serviços externos de pub/sub.
- **Processamento Assíncrono Pesado:** Operações demoradas demandarão adoção de filas externas e workers desacoplados da Vercel.
- **Troca de ORM / Banco:** Regras de RLS e migrações atreladas ao Prisma e PostgreSQL exigem esforço substancial caso haja migração de banco.

---

## 5. Itens Não Cobertos por este Registro

Este documento não delibera sobre:
- Modelagem detalhada das entidades de domínio e regras de negócio do empréstimo.
- Matriz granular de papéis e permissões por tenant.
- Fluxos de negócio para convite de usuários e gestão de acessos.
- Provedores e templates de e-mail transacional.
- Estratégia de backup, retenção e disaster recovery.
- Políticas de branches, promoção entre ambientes e feature flags.
- Aspectos regulatórios avançados (conformidade LGPD e política de retenção).

