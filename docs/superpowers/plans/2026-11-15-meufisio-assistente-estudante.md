# MeuFisio — Assistente do Estudante (v1) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Construir a v1 do meufisio.com.br: um PWA em que estudantes de fisioterapia investigam casos com uma IA guiada (mapa do corpo + perguntas → hipóteses, testes, conduta com PubMed), salvam casos, recebem aviso de caso parecido e consultam um glossário.

**Architecture:** Next.js (App Router) único para UI e API, hospedado na Vercel com funções em `gru1`. Supabase (Postgres + Auth + pgvector, região São Paulo) guarda usuários, casos, mensagens da IA (append-only), catálogo de testes, cache de glossário/PubMed e cotas. A investigação é um loop manual de *tool use* com a API do Claude (ferramentas `buscar_pubmed` e `listar_testes`), resposta final em JSON estruturado validada por Zod e pós-validada no servidor (citações, testes do catálogo).

**Tech Stack:** Next.js 15+ / React / TypeScript estrito, Tailwind CSS, `@supabase/ssr` + `@supabase/supabase-js`, Supabase CLI (local via Docker) + pgTAP, `@anthropic-ai/sdk` + `zod`, Voyage AI (embeddings), NCBI E-utilities (PubMed), Vitest + Testing Library, Playwright (smoke e2e).

**Spec:** `docs/superpowers/specs/2026-11-15-meufisio-assistente-estudante-design.md` (mockups em `docs/superpowers/specs/mockups/`). Leia o spec antes de qualquer task.

## Global Constraints

- Toda a interface e todas as respostas da IA em **português do Brasil**.
- **Nenhum campo de nome do paciente** em nenhum lugar (banco, formulário, prompt). Só `sexo` e `idade`.
- Aviso fixo junto de todo campo de texto livre do caso: **"Não inclua dados que identifiquem o paciente."**
- Hipóteses usam apenas os níveis `alta` / `media` / `baixa` (rótulos "Alta" / "Média" / "Baixa"); nunca porcentagens.
- Toda tela de resultado mostra: **"Hipóteses para você confirmar com testes e com seu preceptor — não é um diagnóstico."**
- Citação só com **PMID retornado pela ferramenta `buscar_pubmed` no mesmo caso**; conduta sem citação válida nunca é exibida como conduta (vai para "Não encontrei evidência no PubMed para …").
- v1 mostra **apenas fontes PubMed**.
- Modelo: `claude-opus-5-5`, `thinking: {type: "adaptive"}`, `output_config.effort` por rota (investigação `medium`, glossário `low`), com `fallbacks: "default"` + beta `server-side-fallback-2026-07-01`. Sempre checar `stop_reason` (`refusal`, `max_tokens`) antes de ler conteúdo.
- Histórico enviado à API é **append-only** (as mensagens da API são gravadas como vieram e reenviadas sem edição).
- Supabase região São Paulo; Vercel `regions: ["gru1"]`.
- Cadastro público: confirmação de e-mail obrigatória; aceite de termos + consentimento (processamento por provedores no exterior) antes de usar qualquer função de IA.
- Campo `plano` no perfil com valor padrão `'gratuito'`.
- Cotas por usuário/dia configuráveis por env: `COTA_INVESTIGACAO_TURNOS_DIA` (padrão `60`), `COTA_GLOSSARIO_DIA` (padrão `40`). Rate limit: no máximo `1` turno de IA simultâneo por usuário.
- Tema claro/escuro seguindo o sistema + alternância manual; acentos verde-menta `#3ee6a8` e azul `#1fb6d4`; ícones de menu/navegação em círculo com gradiente `linear-gradient(135deg,#3ee6a8,#1fb6d4)` (ver `mockups/icones.html`, estilo C).
- Segredos só em env (`ANTHROPIC_API_KEY`, `VOYAGE_API_KEY`, `NCBI_API_KEY`, `SUPABASE_SERVICE_ROLE_KEY`); nunca no cliente.

## Review Focus

1. **Sinal cai no meio de um turno** — a resposta do estudante já gravada não se perde; ao reabrir o caso, a última pergunta pendente é refeita sem duplicar o turno (Task 8).
2. **PubMed fora do ar / timeout / zero resultados** — a investigação continua; a conduta afetada vai para "sem evidência", nunca erro 500 (Tasks 5 e 7).
3. **A IA devolve PMID que não buscou, ou teste fora do catálogo** — PMID é descartado; teste vira texto sem vídeo e entra em `testes_pendentes` (Task 7).
4. **Cota diária atingida no meio de um caso** — mensagem amigável, caso continua salvo e retomável no dia seguinte (Tasks 4 e 8).
5. **Recusa (`stop_reason: "refusal"`) ou corte por `max_tokens`** — mensagem amigável em português, nada parcial gravado como resposta da IA (Task 7).

---

## File Structure

```
src/
  app/
    layout.tsx, globals.css, manifest.ts            # shell, tokens de tema, PWA
    (auth)/entrar/page.tsx, (auth)/cadastrar/page.tsx, (auth)/recuperar/page.tsx
    auth/confirmar/route.ts                         # callback de confirmação de e-mail
    termos/page.tsx, privacidade/page.tsx, consentimento/page.tsx
    (app)/layout.tsx                                # nav inferior, guarda de sessão+consentimento
    (app)/page.tsx                                  # página inicial
    (app)/investigar/[casoId]/page.tsx              # tela de investigação
    (app)/casos/page.tsx, (app)/casos/[casoId]/page.tsx, (app)/casos/comparar/page.tsx
    (app)/glossario/page.tsx
    (app)/admin/testes/page.tsx                     # vídeos + fila de testes pendentes
    api/casos/route.ts                              # POST cria caso
    api/casos/[casoId]/turno/route.ts               # POST turno (SSE)
    api/casos/parecidos/route.ts                    # GET ?q=
    api/glossario/route.ts                          # GET ?termo=
  lib/
    supabase/servidor.ts, supabase/navegador.ts, supabase/admin.ts
    regioes.ts                                      # enums de corpo + rótulos
    cota.ts
    pubmed/cliente.ts, pubmed/parse.ts
    testes/catalogo.ts
    ia/cliente.ts, ia/schema.ts, ia/prompt.ts, ia/ferramentas.ts, ia/investigar.ts, ia/citacoes.ts, ia/glossario.ts
    embeddings.ts
    casos/repositorio.ts
  components/
    icones/Icone.tsx, icones/IconeGradiente.tsx
    TemaToggle.tsx, NavInferior.tsx, Logo.tsx, AvisoPrivacidade.tsx
    MapaCorpo.tsx, GavetaHipoteses.tsx, CartaoPergunta.tsx, CartaoTeste.tsx, AlertaRedFlag.tsx, Conduta.tsx
    CartaoCasoParecido.tsx, ListaCasos.tsx
supabase/
  migrations/0001_base.sql, 0002_rls.sql, 0003_funcoes.sql
  seed.sql                                          # catálogo de testes
  tests/rls.test.sql, tests/cota.test.sql
tests/unit/**.test.ts(x)                            # Vitest
e2e/smoke.spec.ts                                   # Playwright
vercel.json
```

---

### Task 1: Scaffold do app, tema e PWA

**Files:**
- Create: projeto Next.js na raiz do repositório (manter `docs/` e `.gitignore` existentes), `vercel.json`, `src/app/layout.tsx`, `src/app/globals.css`, `src/app/manifest.ts`, `src/components/TemaToggle.tsx`, `src/components/Logo.tsx`, `vitest.config.ts`, `tests/setup.ts`
- Test: `tests/unit/tema.test.tsx`

**Interfaces:**
- Produces: tokens CSS `--bg`, `--fg`, `--muted`, `--card`, `--borda`, `--acento` (`#3ee6a8`), `--acento-2` (`#1fb6d4`), `--gradiente`; atributo `data-tema="claro"|"escuro"` em `<html>` (ausente = segue o sistema); `<TemaToggle />`; `<Logo tamanho?: number />`; scripts `npm test` (Vitest) e `npm run test:e2e` (Playwright).

- [ ] **Step 1:** Gerar com `npx create-next-app@latest meufisio-tmp --ts --tailwind --eslint --app --src-dir --import-alias "@/*" --use-npm` numa pasta temporária fora do repo e mover o conteúdo para a raiz, mesclando o `.gitignore` gerado com o existente (manter `.superpowers/`). Instalar `vitest @vitejs/plugin-react jsdom @testing-library/react @testing-library/jest-dom @playwright/test`.
- [ ] **Step 2: Write the failing test** `tests/unit/tema.test.tsx`:

```tsx
it("alterna tema e persiste a escolha", async () => {
  render(<TemaToggle />);
  await userEvent.click(screen.getByRole("button", { name: /tema/i }));
  expect(document.documentElement.dataset.tema).toBe("escuro");
  expect(localStorage.getItem("meufisio-tema")).toBe("escuro");
  await userEvent.click(screen.getByRole("button", { name: /tema/i }));
  expect(document.documentElement.dataset.tema).toBe("claro");
});
it("sem escolha salva, não define data-tema (segue o sistema)", () => {
  localStorage.clear(); render(<TemaToggle />);
  expect(document.documentElement.dataset.tema).toBeUndefined();
});
```

- [ ] **Step 3:** Run `npm test -- tema` → FAIL (componente não existe).
- [ ] **Step 4:** Implementar `TemaToggle` (lê/escreve `localStorage` em try/catch), tokens em `globals.css` para claro, `@media (prefers-color-scheme: dark)` com `:root:not([data-tema="claro"])`, e `[data-tema="escuro"]`. Um script inline no `layout.tsx` aplica o tema salvo antes da pintura. `Logo` reproduz o símbolo e o texto "Meu**Fisio**" de `mockups/pagina-inicial.html`. `manifest.ts`: `name: "MeuFisio"`, `lang: "pt-BR"`, `display: "standalone"`, `theme_color: "#0d1714"`, ícones 192/512. `vercel.json`: `{ "regions": ["gru1"] }`. `<html lang="pt-BR">`.
- [ ] **Step 5:** Run `npm test -- tema` → PASS; `npm run build` → sucesso.
- [ ] **Step 6: Commit** `feat: scaffold Next.js com tema claro/escuro e PWA`

---

### Task 2: Banco de dados, RLS e catálogo de testes

**Files:**
- Create: `supabase/config.toml` (via `npx supabase init`), `supabase/migrations/0001_base.sql`, `0002_rls.sql`, `supabase/seed.sql`, `supabase/tests/rls.test.sql`, `src/lib/regioes.ts`, `src/lib/supabase/{servidor,navegador,admin}.ts`
- Test: `supabase/tests/rls.test.sql`, `tests/unit/regioes.test.ts`

**Interfaces:**
- Produces (tabelas, todas com `id uuid default gen_random_uuid()` e `criado_em timestamptz default now()`):
  - `perfis(id uuid pk = auth.users.id, nome_exibicao text, plano text not null default 'gratuito', termos_aceitos_em timestamptz, papel text not null default 'estudante' check (papel in ('estudante','admin')))` — criado por trigger em `auth.users`.
  - `casos(usuario_id uuid, sexo text check in ('M','F','outro') null, idade int check 0..120 null, regiao text null, lado text null, face text null, duracao text null, queixa text not null, status text not null default 'em_investigacao' check in ('em_investigacao','confirmado'), hipotese_final text null, anotacoes text not null default '', ultimo_turno jsonb null, embedding vector(1024) null, atualizado_em timestamptz)`
  - `caso_mensagens(caso_id uuid, ordem int, papel text check in ('user','assistant'), conteudo jsonb not null, idempotencia uuid null, unique(caso_id, ordem), unique(caso_id, idempotencia))` — mensagens da API, append-only (sem UPDATE/DELETE por RLS).
  - `caso_testes(caso_id uuid, teste_id uuid null, nome text not null, resultado text check in ('positivo','negativo','nao_realizado'))`
  - `caso_pmids(caso_id uuid, pmid text, unique(caso_id,pmid))` — PMIDs retornados pela ferramenta neste caso.
  - `testes_clinicos(slug text unique, nome text, sinonimos text[], regiao text, como_fazer text, interpretacao text, video_youtube_id text null, revisado bool default false)`
  - `testes_pendentes(nome_sugerido text, regiao text null, caso_id uuid null, contagem int default 1)`
  - `glossario(termo_normalizado text unique, termo text, explicacao jsonb, embedding vector(1024) null)`
  - `pubmed_cache(chave text unique, resposta jsonb, expira_em timestamptz)`
  - `uso_ia(usuario_id uuid, dia date, tipo text check in ('investigacao','glossario'), quantidade int default 0, unique(usuario_id,dia,tipo))`
- Produces (`src/lib/regioes.ts`): `REGIOES = ['cabeca','cervical','ombro','cotovelo','punho_mao','toracica','lombar','quadril','joelho','tornozelo_pe'] as const`, `LADOS = ['direito','esquerdo','central']`, `FACES = ['anterior','posterior','lateral','medial']`, tipos `Regiao`, `Lado`, `Face`, `rotuloRegiao(r, lado?, face?): string` (ex.: `rotuloRegiao('ombro','direito','anterior') === "Ombro D · anterior"`).
- Produces: `criarClienteServidor()` (cookies, sessão do usuário), `criarClienteNavegador()`, `criarClienteAdmin()` (service role, só no servidor).

- [ ] **Step 1: Write the failing tests.** `tests/unit/regioes.test.ts`: `rotuloRegiao('ombro','direito','anterior')` → `"Ombro D · anterior"`; `rotuloRegiao('lombar','central')` → `"Lombar"`; `rotuloRegiao('joelho','esquerdo')` → `"Joelho E"`. `supabase/tests/rls.test.sql` (pgTAP), com dois usuários A e B: B não lê nem altera `casos`, `caso_mensagens`, `caso_testes`, `caso_pmids` de A; A não consegue `UPDATE`/`DELETE` em `caso_mensagens` próprias; qualquer autenticado lê `testes_clinicos` e `glossario`, mas só `papel='admin'` altera `testes_clinicos`; `uso_ia` e `pubmed_cache` não são legíveis/graváveis pelo cliente (só service role); novo usuário em `auth.users` gera `perfis` com `plano='gratuito'`.
- [ ] **Step 2:** Run `npx supabase start && npx supabase test db` e `npm test -- regioes` → FAIL.
- [ ] **Step 3:** Escrever `0001_base.sql` (inclui `create extension vector`, trigger de perfil, índice `ivfflat`/`hnsw` em `casos.embedding` e `glossario.embedding`), `0002_rls.sql`, `regioes.ts`, clientes Supabase. `seed.sql`: catálogo inicial com `revisado=false` cobrindo pelo menos — ombro: Neer, Hawkins-Kennedy, Jobe (lata vazia), Speed, Yergason, Apreensão, O'Brien, Drop arm, Rotação externa resistida; joelho: Lachman, Gaveta anterior, Gaveta posterior, McMurray, Thessaly, Apley, Estresse em valgo, Estresse em varo, Clarke; lombar/quadril: Elevação da perna estendida (Lasègue), Slump, FABER (Patrick), FADIR, Thomas, Trendelenburg; cervical: Spurling, Distração cervical, ULTT1; tornozelo: Gaveta anterior do tornozelo, Thompson, Inclinação talar; cotovelo/punho: Cozen, Mill, Phalen, Tinel, Finkelstein. `como_fazer`/`interpretacao` em português, curtos, para revisão posterior pela estudante.
- [ ] **Step 4:** Run `npx supabase db reset && npx supabase test db` e `npm test -- regioes` → PASS.
- [ ] **Step 5: Commit** `feat: schema Supabase, RLS e catálogo inicial de testes`

---

### Task 3: Cadastro, login, confirmação de e-mail e consentimento

**Files:**
- Create: `src/app/(auth)/entrar/page.tsx`, `cadastrar/page.tsx`, `recuperar/page.tsx`, `src/app/auth/confirmar/route.ts`, `src/app/termos/page.tsx`, `src/app/privacidade/page.tsx`, `src/app/consentimento/page.tsx`, `src/middleware.ts`, `src/lib/acesso.ts`
- Test: `tests/unit/acesso.test.ts`, `e2e/smoke.spec.ts`

**Interfaces:**
- Produces: `destinoDeAcesso(estado: { logado: boolean; emailConfirmado: boolean; termosAceitos: boolean }, caminho: string): string | null` — `null` = pode seguir; caso contrário, rota de redirecionamento. Server action `aceitarTermos(): Promise<void>` (grava `perfis.termos_aceitos_em = now()`).

- [ ] **Step 1: Write the failing test** `tests/unit/acesso.test.ts`:

```ts
expect(destinoDeAcesso({logado:false,emailConfirmado:false,termosAceitos:false}, "/")).toBe("/entrar");
expect(destinoDeAcesso({logado:false,emailConfirmado:false,termosAceitos:false}, "/termos")).toBeNull();
expect(destinoDeAcesso({logado:true,emailConfirmado:false,termosAceitos:false}, "/")).toBe("/entrar?confirme=1");
expect(destinoDeAcesso({logado:true,emailConfirmado:true,termosAceitos:false}, "/casos")).toBe("/consentimento");
expect(destinoDeAcesso({logado:true,emailConfirmado:true,termosAceitos:true}, "/casos")).toBeNull();
expect(destinoDeAcesso({logado:true,emailConfirmado:true,termosAceitos:true}, "/entrar")).toBe("/");
```
Rotas públicas: `/entrar`, `/cadastrar`, `/recuperar`, `/auth/confirmar`, `/termos`, `/privacidade`.

- [ ] **Step 2:** Run `npm test -- acesso` → FAIL.
- [ ] **Step 3:** Implementar `destinoDeAcesso` e usá-lo no `middleware.ts` (também protege `/api/*` com 401 em JSON em vez de redirecionar). Telas com Supabase Auth (e-mail + senha, `emailRedirectTo` = `/auth/confirmar`). `/consentimento` mostra resumo e links para termos/privacidade, com o texto: "Para gerar as hipóteses, o texto do caso é processado por provedores de IA fora do Brasil. Não inclua dados que identifiquem o paciente." e botão "Aceito e quero continuar". Termos e privacidade: texto-base em português marcado no topo com `<!-- revisar com advogado de LGPD antes de vender -->`.
- [ ] **Step 4:** Run `npm test -- acesso` → PASS. `e2e/smoke.spec.ts`: visitante em `/` é levado a `/entrar`; `/termos` abre sem login. Run `npm run test:e2e` → PASS.
- [ ] **Step 5: Commit** `feat: autenticação, confirmação de e-mail e consentimento`

---

### Task 4: Cotas de uso e limite de concorrência

**Files:**
- Create: `supabase/migrations/0003_funcoes.sql`, `supabase/tests/cota.test.sql`, `src/lib/cota.ts`
- Test: `supabase/tests/cota.test.sql`, `tests/unit/cota.test.ts`

**Interfaces:**
- Produces (SQL, `security definer`, executável só por service role): `consumir_cota(p_usuario uuid, p_tipo text, p_limite int) returns boolean` — incrementa atomicamente `uso_ia` do dia corrente em `America/Sao_Paulo`; retorna `false` sem incrementar se já está no limite.
- Produces (TS): `class CotaEsgotadaError extends Error`; `consumirCota(usuarioId: string, tipo: 'investigacao'|'glossario'): Promise<void>` (lança `CotaEsgotadaError`); `comTravaDeTurno<T>(usuarioId: string, fn: () => Promise<T>): Promise<T>` — lança `TurnoEmAndamentoError` se já houver turno ativo do mesmo usuário (usar `pg_try_advisory_lock(hashtext(usuarioId))` via RPC `tentar_trava_turno`/`liberar_trava_turno`, liberando em `finally`).
- Mensagens: cota → "Você chegou ao limite de hoje. Seu caso está salvo — continue amanhã."; trava → "Aguarde a resposta anterior terminar."

- [ ] **Step 1: Write the failing tests.** pgTAP: com limite 2, três chamadas retornam `true,true,false` e `quantidade` fica `2`; tipos diferentes contam separado; virada do dia (inserir linha de ontem) não conta. Vitest (`cota.test.ts`, Supabase mockado): `consumirCota` lança `CotaEsgotadaError` quando a RPC retorna `false`; usa `COTA_INVESTIGACAO_TURNOS_DIA` com padrão `60` e `COTA_GLOSSARIO_DIA` com padrão `40`; `comTravaDeTurno` libera a trava mesmo quando `fn` lança.
- [ ] **Step 2:** Run `npx supabase test db` e `npm test -- cota` → FAIL.
- [ ] **Step 3:** Implementar a migration e `cota.ts`.
- [ ] **Step 4:** Run ambos → PASS.
- [ ] **Step 5: Commit** `feat: cotas diárias de IA e trava de turno por usuário`

---

### Task 5: Cliente PubMed

**Files:**
- Create: `src/lib/pubmed/cliente.ts`, `src/lib/pubmed/parse.ts`, `tests/fixtures/pubmed/esearch.json`, `tests/fixtures/pubmed/efetch.xml`
- Test: `tests/unit/pubmed.test.ts`

**Interfaces:**
- Produces: `type ArtigoPubmed = { pmid: string; titulo: string; ano: number | null; revista: string | null; resumo: string; url: string }` (`url = https://pubmed.ncbi.nlm.nih.gov/<pmid>/`); `parseEfetch(xml: string): ArtigoPubmed[]`; `buscarPubmed(consulta: string, opcoes?: { max?: number }): Promise<ArtigoPubmed[]>` — `max` padrão `5`; usa `esearch.fcgi` (`db=pubmed`, `sort=relevance`, filtro de idioma inglês/português e preferência por revisões/ensaios: acrescentar `AND (systematic review[pt] OR randomized controlled trial[pt] OR review[pt])`, e se vier vazio repetir sem o filtro) e `efetch.fcgi` (`rettype=abstract`, `retmode=xml`); envia `tool=meufisio` e `api_key` se `NCBI_API_KEY` existir; cache em `pubmed_cache` por 7 dias (chave = consulta normalizada + max); timeout de 8 s por requisição; **em erro ou timeout retorna `[]`** e registra no log (nunca lança). Artigos sem resumo são descartados.

- [ ] **Step 1: Write the failing tests:** `parseEfetch` do fixture devolve PMIDs, título, ano e resumo (resumos com várias seções `AbstractText` concatenados com rótulo, ex. `"BACKGROUND: …"`); `buscarPubmed` com `fetch` mockado devolve artigos na ordem do esearch; com esearch filtrado vazio, faz segunda busca sem filtro; com `fetch` rejeitando/timeout devolve `[]`; segunda chamada idêntica usa o cache (fetch chamado uma vez).
- [ ] **Step 2:** Run `npm test -- pubmed` → FAIL.
- [ ] **Step 3:** Implementar (parse com `fast-xml-parser`).
- [ ] **Step 4:** Run `npm test -- pubmed` → PASS.
- [ ] **Step 5: Commit** `feat: cliente PubMed com cache e degradação graciosa`

---

### Task 6: Catálogo de testes no servidor

**Files:**
- Create: `src/lib/testes/catalogo.ts`
- Test: `tests/unit/catalogo.test.ts`

**Interfaces:**
- Produces: `type TesteClinico = { id: string; slug: string; nome: string; sinonimos: string[]; regiao: Regiao; como_fazer: string; interpretacao: string; video_youtube_id: string | null }`; `listarTestes(regiao?: Regiao): Promise<TesteClinico[]>`; `resolverTeste(nomeOuSlug: string, catalogo: TesteClinico[]): TesteClinico | null` — compara sem acento, sem caixa, ignorando prefixo "teste de"/"test"/"sinal de" e sufixo "'s test", contra `slug`, `nome` e `sinonimos`; `registrarTestePendente(nome: string, regiao: Regiao | null, casoId: string): Promise<void>` (upsert incrementando `contagem`).

- [ ] **Step 1: Write the failing test:** com catálogo fixture contendo `{slug:'speed', nome:'Teste de Speed', sinonimos:["Speed's test",'Speed']}`: `resolverTeste('Speed')`, `resolverTeste("speed's test")`, `resolverTeste('Teste de Speed')`, `resolverTeste('speed')` retornam o mesmo item; `resolverTeste('Teste de Lasegue')` resolve `Lasègue` (acentos); `resolverTeste('Teste Inventado')` → `null`.
- [ ] **Step 2:** Run `npm test -- catalogo` → FAIL.
- [ ] **Step 3:** Implementar.
- [ ] **Step 4:** Run → PASS.
- [ ] **Step 5: Commit** `feat: catálogo de testes clínicos com resolução de sinônimos`

---

### Task 7: Motor de investigação (IA)

**Files:**
- Create: `src/lib/ia/cliente.ts`, `src/lib/ia/schema.ts`, `src/lib/ia/prompt.ts`, `src/lib/ia/ferramentas.ts`, `src/lib/ia/citacoes.ts`, `src/lib/ia/investigar.ts`
- Test: `tests/unit/citacoes.test.ts`, `tests/unit/investigar.test.ts`

**Interfaces:**
- Consumes: `buscarPubmed`, `ArtigoPubmed` (Task 5); `listarTestes`, `resolverTeste`, `registrarTestePendente`, `TesteClinico` (Task 6); `Regiao`, `Lado`, `Face` (Task 2).
- Produces (`schema.ts`, Zod — também usado na UI):

```ts
export const TurnoIA = z.object({
  ficha: z.object({ sexo: z.enum(['M','F','outro']).nullable(), idade: z.number().int().nullable(),
    regiao: z.enum(REGIOES).nullable(), lado: z.enum(LADOS).nullable(), face: z.enum(FACES).nullable(),
    duracao: z.string().nullable() }),
  pergunta: z.object({ texto: z.string(), opcoes: z.array(z.string()).max(4), porque: z.string() }).nullable(),
  hipoteses: z.array(z.object({ nome: z.string(), nivel: z.enum(['alta','media','baixa']), justificativa: z.string() })).max(5),
  proximo_teste: z.object({ nome: z.string(), motivo: z.string() }).nullable(),
  alerta: z.object({ motivo: z.string(), orientacao: z.string() }).nullable(),
  conduta: z.array(z.object({ descricao: z.string(), pmids: z.array(z.string()) })),
  finalizado: z.boolean(),
});
export type TurnoIA = z.infer<typeof TurnoIA>;
// Depois da pós-validação:
export type TurnoValidado = Omit<TurnoIA,'proximo_teste'|'conduta'> & {
  proximo_teste: { nome: string; motivo: string; teste: TesteClinico | null } | null;
  conduta: { descricao: string; citacoes: ArtigoPubmed[] }[];
  sem_evidencia: string[];
};
```
- Produces (`citacoes.ts`): `validarConduta(conduta: TurnoIA['conduta'], artigosDoCaso: Map<string, ArtigoPubmed>): { conduta: TurnoValidado['conduta']; sem_evidencia: string[] }` — remove PMIDs fora do mapa; item sem nenhum PMID válido vai para `sem_evidencia` (só a `descricao`).
- Produces (`ferramentas.ts`): ferramentas `buscar_pubmed({ consulta: string })` (consulta em inglês; retorna JSON com `pmid,titulo,ano,resumo` e grava PMIDs em `caso_pmids`) e `listar_testes({ regiao: Regiao })` (retorna `slug,nome,sinonimos,regiao`), ambas `strict: true` com `additionalProperties: false`.
- Produces (`investigar.ts`): `executarTurno(params: { casoId: string; usuarioId: string; mensagemEstudante: string; idempotencia: string; aoProgresso?: (evento: { tipo: 'buscando_pubmed' | 'consultando_testes' }) => void }): Promise<{ ok: true; turno: TurnoValidado } | { ok: false; motivo: 'recusa' | 'limite_tokens' | 'erro' }>`.
  - Lê `caso_mensagens` do caso em ordem, anexa a mensagem do estudante, roda loop manual com `client.beta.messages.stream(...)` + `finalMessage()`: `model: 'claude-opus-5-5'`, `max_tokens: 16000`, `thinking: {type:'adaptive'}`, `output_config: { effort: 'medium', format: zodOutputFormat(TurnoIA) }`, `tools`, `betas: ['server-side-fallback-2026-07-01']`, `fallbacks: 'default'`, `cache_control` no system prompt. Executa ferramentas em paralelo e devolve todos os `tool_result` numa só mensagem `user`; ferramenta que falha → `is_error: true`. Para em `end_turn`; `refusal` → `{ok:false,motivo:'recusa'}`; `max_tokens` → `'limite_tokens'`; no máximo 6 iterações de ferramenta → `'erro'`.
  - Em sucesso: grava **todas** as novas mensagens (user, assistant com blocos `thinking`/`tool_use` intactos, tool_results) em `caso_mensagens` numa transação, depois a pós-validação: `proximo_teste.nome` resolvido por `resolverTeste`; se `null`, chama `registrarTestePendente`; `validarConduta` com os artigos de `caso_pmids` do caso. Atualiza `casos` (`ficha` → colunas, `ultimo_turno` = `TurnoValidado`, `atualizado_em`). Em falha: **nada** é gravado.
- `prompt.ts` (`SYSTEM_PROMPT`, texto fixo, sem data/hora) deve instruir: papel de preceptor de fisioterapia ensinando um estagiário; responder em português do Brasil; **uma pergunta por turno**, com até 4 opções curtas e o campo `porque` explicando o raciocínio; preencher a `ficha` conforme aprender (pedir sexo e idade cedo); hipóteses só como hipóteses, níveis alta/média/baixa; escolher testes chamando `listar_testes` e usando o nome do catálogo; antes de preencher `conduta`, chamar `buscar_pubmed` com consulta em inglês e citar apenas PMIDs cujo **resumo sustenta** a recomendação, senão deixar `pmids` vazio; `alerta` preenchido e `pergunta` nula quando houver sinais de alerta (ex.: suspeita de fratura, déficit neurológico progressivo, sinais de cauda equina, dor torácica, febre com dor na coluna, perda de peso inexplicada) com orientação de encaminhar/avisar o preceptor; `finalizado=true` quando houver hipótese principal com testes indicados e conduta (tipicamente 5–10 perguntas); nunca pedir nem repetir nome do paciente.

- [ ] **Step 1: Write the failing tests.** `citacoes.test.ts`: item com PMIDs `['1','999']` e mapa com `'1'` → mantém só o artigo `1`; item com `['999']` → vai para `sem_evidencia`; lista vazia → `{conduta:[], sem_evidencia:[]}`. `investigar.test.ts` com cliente Anthropic e Supabase falsos (injeção via parâmetro opcional `deps` em `executarTurno`):
  - turno sem ferramentas → `ok:true`, grava 2 mensagens (`user`,`assistant`) com `ordem` sequencial a partir do histórico;
  - turno com `tool_use` `buscar_pubmed` → executa, envia `tool_result`, grava 4 mensagens, PMIDs gravados em `caso_pmids`, conduta citada mantém artigo;
  - `stop_reason: 'refusal'` → `{ok:false,motivo:'recusa'}` e **nenhuma** escrita;
  - `stop_reason: 'max_tokens'` → `'limite_tokens'`, nenhuma escrita;
  - `proximo_teste.nome = 'Teste Inventado'` → `turno.proximo_teste.teste === null` e `registrarTestePendente` chamado;
  - ferramenta `buscar_pubmed` retornando `[]` (PubMed fora) → turno conclui; conduta sem PMIDs vai para `sem_evidencia`;
  - histórico existente é reenviado sem alteração (os blocos `thinking` das mensagens antigas aparecem idênticos na requisição).
- [ ] **Step 2:** Run `npm test -- citacoes investigar` → FAIL.
- [ ] **Step 3:** Implementar. Antes, ler o `README.md` e `tool-use.md` TypeScript da skill `claude-api` (loop manual com streaming, structured outputs, refusal fallbacks) — não adivinhar a API do SDK.
- [ ] **Step 4:** Run → PASS.
- [ ] **Step 5: Commit** `feat: motor de investigação com tool use, citações validadas e red flags`

---

### Task 8: API de casos e turnos (SSE) com retomada

**Files:**
- Create: `src/lib/casos/repositorio.ts`, `src/app/api/casos/route.ts`, `src/app/api/casos/[casoId]/turno/route.ts`
- Test: `tests/unit/api-turno.test.ts`

**Interfaces:**
- Consumes: `executarTurno` (Task 7), `consumirCota`, `comTravaDeTurno`, `CotaEsgotadaError`, `TurnoEmAndamentoError` (Task 4).
- Produces:
  - `POST /api/casos` body `{ queixa: string; regiao?: Regiao; lado?: Lado; face?: Face }` → `201 { casoId }`. `queixa` 3–1000 caracteres.
  - `POST /api/casos/:casoId/turno` body `{ mensagem: string; idempotencia: string }` (`idempotencia` = UUID gerado pelo cliente por envio) → `text/event-stream` com eventos `progresso` (`{tipo}`), `turno` (`TurnoValidado`), `erro` (`{codigo: 'cota'|'ocupado'|'recusa'|'limite_tokens'|'erro', mensagem}`).
  - Repositório: `criarCaso`, `obterCaso(casoId)`, `listarCasos(filtros: { regiao?: Regiao; busca?: string })`, `atualizarCaso(casoId, campos: Partial<{status; hipotese_final; anotacoes}>)`, `registrarResultadoTeste(casoId, { teste_id?: string; nome: string; resultado })`.
- Retomada: o cliente guarda a `idempotencia` do último envio; se a conexão cair e o usuário reenviar a mesma `idempotencia`, e o turno já tiver sido gravado (coluna `caso_mensagens.idempotencia` da mensagem `user` do estudante; `executarTurno` recebe e grava esse valor — o `conteudo` enviado à API nunca o contém), a rota devolve o `ultimo_turno` sem chamar a IA nem consumir cota.
- Mensagens de erro em português: `cota` e `ocupado` conforme Task 4; `recusa` → "Não consegui analisar essa resposta. Reformule ou fale com seu preceptor."; `limite_tokens`/`erro` → "Algo deu errado. Sua resposta está salva no rascunho — tente de novo."

- [ ] **Step 1: Write the failing tests** (handlers chamados direto com `Request`, dependências mockadas): sem sessão → 401; caso de outro usuário → 404; cota esgotada → evento `erro` `codigo:'cota'` e `executarTurno` não chamado; reenvio com mesma `idempotencia` já gravada → evento `turno` com `ultimo_turno`, sem chamar IA nem `consumirCota`; sucesso → eventos `progresso*` e depois `turno`; `queixa` com 2 caracteres → 400.
- [ ] **Step 2:** Run `npm test -- api-turno` → FAIL.
- [ ] **Step 3:** Implementar (`export const runtime = 'nodejs'`, `maxDuration = 120`).
- [ ] **Step 4:** Run → PASS.
- [ ] **Step 5: Commit** `feat: API de casos e turnos com SSE, cotas e retomada idempotente`

---

### Task 9: Embeddings e "caso parecido"

**Files:**
- Create: `src/lib/embeddings.ts`, `src/app/api/casos/parecidos/route.ts`; Modify: `supabase/migrations/0003_funcoes.sql` (função `casos_parecidos`), `src/lib/casos/repositorio.ts`, `src/lib/ia/investigar.ts` (atualizar embedding ao gravar turno)
- Test: `tests/unit/parecidos.test.ts`, `supabase/tests/parecidos.test.sql`

**Interfaces:**
- Produces: `gerarEmbedding(texto: string, tipo: 'documento'|'consulta'): Promise<number[]>` — Voyage AI `POST https://api.voyageai.com/v1/embeddings`, modelo em `VOYAGE_MODELO` (padrão `voyage-3.5`), `output_dimension: 1024`, `input_type` `document`/`query`. Texto do caso para embedding = `queixa` + rótulo da região + hipótese principal.
- SQL: `casos_parecidos(p_usuario uuid, p_embedding vector(1024), p_excluir uuid default null, p_limite int default 3, p_similaridade_min float default 0.78) returns table(id uuid, queixa text, hipotese_final text, status text, regiao text, lado text, criado_em timestamptz, similaridade float)` — só casos do próprio usuário com embedding, ordenado por similaridade (cosseno).
- `GET /api/casos/parecidos?q=<texto>&excluir=<casoId?>` → `200 { casos: [...] }`; `q` com menos de 12 caracteres → `{ casos: [] }` sem chamar Voyage. Falha do Voyage → `{ casos: [] }` (nunca 500).
- `executarTurno` atualiza `casos.embedding` após cada turno bem-sucedido (falha no embedding não falha o turno).

- [ ] **Step 1: Write the failing tests:** pgTAP: com 3 casos do usuário A e 1 do B, consulta com vetor igual ao caso A1 retorna A1 primeiro e nunca o de B; abaixo do limiar não retorna. Vitest: `q` curto não chama `gerarEmbedding`; Voyage falhando → `{casos:[]}`; `excluir` repassado à função SQL.
- [ ] **Step 2:** Run `npx supabase test db` e `npm test -- parecidos` → FAIL.
- [ ] **Step 3:** Implementar.
- [ ] **Step 4:** Run → PASS.
- [ ] **Step 5: Commit** `feat: busca de casos parecidos por similaridade semântica`

---

### Task 10: Shell do app e página inicial

**Files:**
- Create: `src/components/icones/Icone.tsx`, `src/components/icones/IconeGradiente.tsx`, `src/components/NavInferior.tsx`, `src/components/AvisoPrivacidade.tsx`, `src/components/CartaoCasoParecido.tsx`, `src/app/(app)/layout.tsx`, `src/app/(app)/page.tsx`
- Test: `tests/unit/inicio.test.tsx`

**Interfaces:**
- Consumes: `GET /api/casos/parecidos` (Task 9), `POST /api/casos` (Task 8), `Logo`, `TemaToggle` (Task 1).
- Produces: `<Icone nome: NomeIcone; tamanho?: number />` e `<IconeGradiente nome: NomeIcone; tamanho?: number />` com `NomeIcone = 'investigar'|'glossario'|'testes'|'exercicios'|'casos'|'inicio'|'buscar'|'perfil'|'voz'|'corpo'|'ombro'|'joelho'|'coluna'` — paths SVG copiados de `mockups/icones.html` e `mockups/meus-casos.html`; `<AvisoPrivacidade />` com o texto da Global Constraint.
- Página inicial (layout de `mockups/pagina-inicial.html` + ícones estilo C): `Logo`, "Olá, {nome_exibicao}" + "Qual é o caso de hoje?", campo de busca (texto) com botão do mapa do corpo (sem botão de voz na v1), `AvisoPrivacidade`, cartão de destaque **"Investigar caso"**, atalhos **Glossário** e **Meus casos** (sem Testes clínicos e Exercícios na v1), `NavInferior` (Início · Buscar · Casos · Perfil; "Buscar" leva ao Glossário com foco na busca; "Perfil" mostra e-mail, plano, tema e sair).
- Enquanto digita (debounce 400 ms), consulta `parecidos`; se houver resultado, mostra `CartaoCasoParecido` com "Você teve um caso parecido há N dias", subtítulo `Caso · rótulo da região · hipótese (status)`, e botões **Ver caso**, **Comparar**, **Novo caso**. Enviar a busca (ou "Novo caso"/"Investigar caso") cria o caso com a queixa e navega para `/investigar/:casoId`; botão do mapa do corpo navega para `/investigar/novo?mapa=1`.

- [ ] **Step 1: Write the failing test** `inicio.test.tsx` (fetch mockado, timers falsos): digitar "dor no ombro direito ao levantar o braço" e avançar 400 ms faz **uma** chamada a `/api/casos/parecidos?q=…` e exibe "Você teve um caso parecido há 12 dias" (fixture com `criado_em` = hoje − 12 dias); resposta vazia não exibe cartão; "Comparar" navega para `/casos/comparar?a=<id>&q=<texto>`; não existe botão de voz; o aviso de privacidade está visível.
- [ ] **Step 2:** Run `npm test -- inicio` → FAIL.
- [ ] **Step 3:** Implementar.
- [ ] **Step 4:** Run → PASS.
- [ ] **Step 5: Commit** `feat: página inicial com busca e aviso de caso parecido`

---

### Task 11: Tela de investigação

**Files:**
- Create: `src/app/(app)/investigar/[casoId]/page.tsx`, `src/components/MapaCorpo.tsx`, `src/components/GavetaHipoteses.tsx`, `src/components/CartaoPergunta.tsx`, `src/components/CartaoTeste.tsx`, `src/components/AlertaRedFlag.tsx`, `src/components/Conduta.tsx`, `src/lib/sse.ts`
- Test: `tests/unit/mapa-corpo.test.tsx`, `tests/unit/investigar-tela.test.tsx`, `tests/unit/sse.test.ts`

**Interfaces:**
- Consumes: `POST /api/casos`, `POST /api/casos/:id/turno` (Task 8), `TurnoValidado` (Task 7), `rotuloRegiao` (Task 2).
- Produces: `<MapaCorpo vista: 'frente'|'costas'; selecionado?: {regiao, lado, face}; aoSelecionar(sel: { regiao: Regiao; lado: Lado; face: Face }): void />` — silhueta de `mockups/mapa-corpo-v2.html`, alternância Frente/Costas, áreas tocáveis (mín. 44×44 px) por região/lado; frente → `face:'anterior'`, costas → `face:'posterior'`; ponto pulsante + rótulo `rotuloRegiao`. Botão "↺ 3D" **não** existe na v1. `lerEventosSSE(resposta: Response, aoEvento: (nome: string, dados: unknown) => void): Promise<void>`.
- `/investigar/novo?mapa=1`: mostra o mapa; ao tocar, cria o caso com `queixa = "Dor em " + rótulo` e entra no fluxo. `/investigar/:casoId`: se o caso não tem turno, dispara o primeiro turno com a queixa.
- Comportamento da tela: `CartaoPergunta` mostra `pergunta.texto`, `porque` (recolhível) e `opcoes` como botões; campo de texto livre com `AvisoPrivacidade`; o envio gera `idempotencia` (UUID), guarda no `sessionStorage` com o texto até receber `turno` (reenviar com a mesma chave se a conexão cair; rascunho restaurado ao reabrir). Durante o turno, mostra progresso ("Buscando evidências no PubMed…", "Consultando testes…"). `GavetaHipoteses` (gaveta inferior) com a frase de Global Constraint e níveis Alta/Média/Baixa. `CartaoTeste`: nome, motivo, `como_fazer`/`interpretacao`, vídeo `youtube-nocookie.com/embed/<id>` quando houver; botões Positivo / Negativo / Não fiz (grava via `registrarResultadoTeste` e envia "Resultado do <teste>: <resultado>" como próxima mensagem). `AlertaRedFlag` cobre a tela, com `motivo` e `orientacao`, e botão "Entendi" (o caso fica salvo). Com `finalizado`: `Conduta` lista itens com citações (título, ano, link PubMed) e bloco "Não encontrei evidência no PubMed para: …"; botões "Salvar em Meus casos" (hipótese final inicial = a primeira hipótese de nível `alta`, ou a primeira da lista se não houver `alta`; editável) e "Continuar investigando".
- Erros SSE mostram a `mensagem` recebida e mantêm o rascunho.

- [ ] **Step 1: Write the failing tests.** `sse.test.ts`: corpo com `event: progresso\ndata: {...}\n\nevent: turno\ndata: {...}\n\n` em pedaços quebrados no meio → `aoEvento` chamado 2 vezes com JSON correto. `mapa-corpo.test.tsx`: tocar no ombro direito na frente chama `aoSelecionar({regiao:'ombro',lado:'direito',face:'anterior'})` e exibe "Ombro D · anterior"; alternar para Costas e tocar na lombar → `{regiao:'lombar',lado:'central',face:'posterior'}`. `investigar-tela.test.tsx` (fetch mockado): turno com `pergunta` mostra as opções e clicar numa envia `{mensagem: opcao, idempotencia}`; turno com `alerta` mostra o alerta e esconde a pergunta; turno `finalizado` mostra citações e o bloco "Não encontrei evidência no PubMed para"; evento `erro` `cota` mostra a mensagem e mantém o texto digitado; hipóteses aparecem com "Alta"/"Média"/"Baixa" e nunca "%".
- [ ] **Step 2:** Run `npm test -- sse mapa-corpo investigar-tela` → FAIL.
- [ ] **Step 3:** Implementar seguindo visualmente `mockups/mapa-corpo-v2.html` (tema claro e escuro).
- [ ] **Step 4:** Run → PASS.
- [ ] **Step 5: Commit** `feat: tela de investigação com mapa do corpo, hipóteses e conduta`

---

### Task 12: Meus casos (lista, detalhe, comparar)

**Files:**
- Create: `src/app/(app)/casos/page.tsx`, `src/app/(app)/casos/[casoId]/page.tsx`, `src/app/(app)/casos/comparar/page.tsx`, `src/components/ListaCasos.tsx`, `src/app/(app)/casos/acoes.ts` (server actions)
- Test: `tests/unit/casos.test.tsx`

**Interfaces:**
- Consumes: repositório (Task 8), `IconeGradiente` (Task 10), `rotuloRegiao` (Task 2).
- Produces: server actions `salvarAnotacoes(casoId, texto)`, `alterarStatus(casoId, status)`, `definirHipoteseFinal(casoId, texto)`.
- Lista (layout `mockups/meus-casos.html`, tela 2): busca por queixa/hipótese/nome de teste (`ilike` sem acento), chips de região (só regiões presentes + "Todos · N"), cartão com ícone da região, título = `hipotese_final` ou `queixa`, meta `Caso #<n> · <sexo>, <idade>a · <rótulo região> · há <tempo>`, etiqueta **Confirmado**/**Em investigação**, etiquetas dos testes positivos. `#<n>` = ordem de criação por usuário.
- Detalhe (tela 3): ficha, hipótese final (editável), status (alternável), testes com resultado (Positivo verde / Negativo vermelho), conduta com citações e "sem evidência", anotações (autosave com debounce 800 ms, `AvisoPrivacidade` junto), botão "Continuar investigação" → `/investigar/:id`.
- Comparar: `?a=<casoId>&q=<texto>` mostra o caso salvo ao lado da nova queixa, com botão "Investigar este novo caso"; `?a=&b=` compara dois casos salvos lado a lado.

- [ ] **Step 1: Write the failing test** `casos.test.tsx`: lista com fixtures filtra por chip "Joelho"; busca "speed" encontra caso com teste Speed positivo; detalhe mostra "Positivo" e "Negativo" corretos; editar anotações e avançar 800 ms chama `salvarAnotacoes` uma vez; meta sem nenhum campo de nome.
- [ ] **Step 2:** Run `npm test -- casos` → FAIL.
- [ ] **Step 3:** Implementar.
- [ ] **Step 4:** Run → PASS.
- [ ] **Step 5: Commit** `feat: Meus casos com lista, detalhe, anotações e comparação`

---

### Task 13: Glossário

**Files:**
- Create: `src/lib/ia/glossario.ts`, `src/app/api/glossario/route.ts`, `src/app/(app)/glossario/page.tsx`
- Test: `tests/unit/glossario.test.ts`

**Interfaces:**
- Consumes: `consumirCota` (Task 4), `gerarEmbedding` (Task 9), cliente IA (Task 7).
- Produces: `normalizarTermo(t: string): string` (minúsculas, sem acento, espaços colapsados, trim); `type Explicacao = { termo: string; definicao: string; contexto_clinico: string; sinonimos: string[]; sigla_de: string | null }`; `explicarTermo(termo: string, usuarioId: string): Promise<{ explicacao: Explicacao; doCache: boolean }>` — procura `glossario.termo_normalizado`; se não há, procura por embedding (similaridade ≥ 0.92) e reaproveita; senão consome cota `glossario`, chama `claude-opus-5-5` com `effort: 'low'` e `output_config.format` = `Explicacao`, grava no cache compartilhado. Termo 2–80 caracteres.
- `GET /api/glossario?termo=` → `200 { explicacao, doCache }`; cota → `429 { mensagem }`; recusa → `422 { mensagem: "Não consegui explicar esse termo." }`.
- Página: busca + resultado em cartão + lista dos 20 termos mais recentes do cache.

- [ ] **Step 1: Write the failing test:** `normalizarTermo('  Síndrome   do Impacto ')` → `'sindrome do impacto'`; termo em cache não chama IA nem cota (`doCache:true`); termo novo chama cota e IA uma vez e grava; termo de 1 caractere → 400; cota esgotada → 429.
- [ ] **Step 2:** Run `npm test -- glossario` → FAIL.
- [ ] **Step 3:** Implementar.
- [ ] **Step 4:** Run → PASS.
- [ ] **Step 5: Commit** `feat: glossário com cache compartilhado`

---

### Task 14: Administração do catálogo de testes

**Files:**
- Create: `src/app/(app)/admin/testes/page.tsx`, `src/app/(app)/admin/testes/acoes.ts`
- Test: `tests/unit/admin-testes.test.ts`

**Interfaces:**
- Consumes: `testes_clinicos`, `testes_pendentes` (Task 2).
- Produces: `extrairIdYoutube(url: string): string | null` (aceita `youtube.com/watch?v=`, `youtu.be/`, `youtube.com/shorts/`, `youtube.com/embed/`); server actions (só `papel='admin'`, senão 403): `definirVideo(testeId, url)`, `marcarRevisado(testeId, revisado)`, `editarTeste(testeId, {como_fazer, interpretacao, sinonimos})`, `promoverPendente(pendenteId, dados: {slug,nome,regiao,como_fazer,interpretacao})` (cria em `testes_clinicos`, remove da fila).
- Página: tabela filtrável por região com status de vídeo/revisão; fila de pendentes ordenada por `contagem`. Link só aparece no Perfil para admins. O primeiro admin é definido manualmente via SQL (documentar no README).

- [ ] **Step 1: Write the failing test:** `extrairIdYoutube` para os 4 formatos retorna `dQw4w9WgXcQ`; URL de outro domínio → `null`; ação chamada por estudante → erro de permissão; `promoverPendente` cria o teste e remove o pendente.
- [ ] **Step 2:** Run `npm test -- admin-testes` → FAIL.
- [ ] **Step 3:** Implementar.
- [ ] **Step 4:** Run → PASS.
- [ ] **Step 5: Commit** `feat: administração de vídeos e fila de testes pendentes`

---

### Task 15: Deploy e verificação ponta a ponta

**Files:**
- Create: `README.md` (setup local, envs, criar admin, deploy), `.env.example`, `tests/fixtures/turnos/*.json`; Modify: `e2e/smoke.spec.ts`, `src/lib/ia/investigar.ts` e `src/lib/pubmed/cliente.ts` (modo `MEUFISIO_IA_FAKE=1`, ignorado quando `NODE_ENV=production`)

**Interfaces:**
- `.env.example` lista: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `ANTHROPIC_API_KEY`, `VOYAGE_API_KEY`, `VOYAGE_MODELO`, `NCBI_API_KEY`, `COTA_INVESTIGACAO_TURNOS_DIA`, `COTA_GLOSSARIO_DIA`, `NEXT_PUBLIC_SITE_URL`.

- [ ] **Step 1:** Ampliar `e2e/smoke.spec.ts` (contra Supabase local, IA mockada por `MEUFISIO_IA_FAKE=1` que faz `executarTurno` devolver turnos fixos de `tests/fixtures/turnos/*.json` e `buscarPubmed` devolver o fixture): cadastrar → confirmar e-mail (Inbucket local do Supabase) → consentimento → página inicial → digitar queixa → responder 2 perguntas → ver conduta com citação → salvar → caso aparece em Meus casos → nova queixa parecida mostra "Você teve um caso parecido".
- [ ] **Step 2:** Run `npm run test:e2e` → PASS; `npm test` → todos PASS; `npx supabase test db` → PASS; `npm run build` → sucesso.
- [ ] **Step 3:** Documentar no README o deploy: projeto Supabase em `sa-east-1` (`supabase link` + `supabase db push` + seed), variáveis na Vercel, domínio `meufisio.com.br`, URL de redirecionamento de auth. O deploy em si é feito pelo autor (contas e domínio são dele).
- [ ] **Step 4: Commit** `docs: README de setup e deploy; e2e do fluxo principal`
