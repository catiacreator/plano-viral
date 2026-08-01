# Plano Viral — Guia de Construção Completo

Tudo o que precisas para construir o **Plano Viral**: base de dados, funções nos bastidores, ligação ao Google e à Social Blade, e o que dizer ao Lovable. Segue por ordem, de cima para baixo.

---

## 0. O que vais construir

Uma app onde criadoras seguem uma **trilha diária de 60 roteiros**, acompanham o **crescimento no Instagram** e recebem uma **análise do perfil**. Tu (professora) geres tudo num **painel admin**.

**O que vive onde:**

| Peça | Para quê | Custo |
|---|---|---|
| **Lovable** | Construir e alojar a app | O que já pagas |
| **Supabase** | Login + base de dados + funções | **Grátis** (plano free) |
| **Claude API** | Ler PDFs e preencher a análise automaticamente | Cêntimos por documento |
| **Social Blade** | Buscar seguidores por @ | 1 crédito (pago) por consulta |
| **Google Sheets** | Espelhar leads e seguidores na tua Drive | Grátis |

> Regra de ouro: se o Lovable e o SQL "discutirem", **manda o SQL**. Corre o SQL primeiro e diz ao Lovable "usa as tabelas que já existem, não cries novas".

---

## 1. Supabase — Base de dados

No Supabase: **SQL Editor → New query → cola → Run**. Corre os blocos **por esta ordem**.

### 1.1 — Tabelas base + segurança + perfil automático

```sql
-- PERFIS (ligado ao login)
create table if not exists public.profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  nome text,
  avatar_url text,
  instagram_handle text,
  papel text default 'aluno',
  criado_em timestamptz default now()
);

-- ROTEIROS (conteúdo do curso)
create table if not exists public.roteiros (
  id uuid primary key default gen_random_uuid(),
  dia int not null,
  titulo text not null,
  categoria text,
  duracao_min int,
  dificuldade text,
  tema_central text,
  gancho_inicial text,
  roteiro_completo text,
  videos_referencia text,
  criado_em timestamptz default now()
);

-- PROGRESSO (uma linha por aluna+roteiro)
create table if not exists public.progresso (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  roteiro_id uuid not null references public.roteiros(id) on delete cascade,
  feito boolean default false,
  concluido_em timestamptz,
  unique (user_id, roteiro_id)
);

-- MÉTRICAS (seguidores por dia)
create table if not exists public.metricas (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  data date not null,
  seguidores int,
  alcance int,
  criado_em timestamptz default now()
);

-- ANÁLISE (diagnóstico do perfil, por secção)
create table if not exists public.analise (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  seccao text not null,
  como_esta text,
  como_ficaria text,
  observacao text,
  criado_em timestamptz default now()
);

-- Índices (velocidade)
create index if not exists idx_progresso_user on public.progresso (user_id);
create index if not exists idx_progresso_roteiro on public.progresso (roteiro_id);
create index if not exists idx_metricas_user_data on public.metricas (user_id, data);
create index if not exists idx_analise_user on public.analise (user_id);
create index if not exists idx_roteiros_dia on public.roteiros (dia);

-- Segurança (RLS)
alter table public.profiles  enable row level security;
alter table public.roteiros  enable row level security;
alter table public.progresso enable row level security;
alter table public.metricas  enable row level security;
alter table public.analise   enable row level security;

create policy "perfil_proprio_update" on public.profiles
  for update to authenticated using ((select auth.uid()) = id);

-- Criar perfil automático no registo
create or replace function public.handle_new_user()
returns trigger language plpgsql security definer set search_path = public as $$
begin
  insert into public.profiles (id, nome)
  values (new.id, new.raw_user_meta_data->>'nome');
  return new;
end;
$$;
drop trigger if exists on_auth_user_created on auth.users;
create trigger on_auth_user_created
  after insert on auth.users
  for each row execute procedure public.handle_new_user();
```

### 1.2 — Admin (professora) + trilha personalizada

```sql
-- Dono do roteiro (NULL = base, para todas)
alter table public.roteiros
  add column if not exists user_id uuid references auth.users(id) on delete cascade;
create index if not exists idx_roteiros_user on public.roteiros (user_id);

-- "O utilizador atual é admin?"
create or replace function public.is_admin()
returns boolean language sql security definer set search_path = public stable as $$
  select exists (select 1 from public.profiles
                 where id = auth.uid() and papel = 'admin');
$$;

-- ROTEIROS: aluna vê base + os seus; professora vê e edita tudo
drop policy if exists "roteiros_leitura" on public.roteiros;
create policy "roteiros_leitura" on public.roteiros
  for select to authenticated
  using (user_id is null or user_id = (select auth.uid()) or public.is_admin());
create policy "roteiros_admin_escrita" on public.roteiros
  for all to authenticated
  using (public.is_admin()) with check (public.is_admin());

-- ANÁLISE: aluna lê a sua; professora preenche a de qualquer aluna
drop policy if exists "analise_propria" on public.analise;
create policy "analise_leitura" on public.analise
  for select to authenticated
  using ((select auth.uid()) = user_id or public.is_admin());
create policy "analise_admin_escrita" on public.analise
  for all to authenticated
  using (public.is_admin()) with check (public.is_admin());

-- PROGRESSO: aluna gere o seu; professora vê o de todas
drop policy if exists "progresso_proprio" on public.progresso;
create policy "progresso_aluna" on public.progresso
  for all to authenticated
  using ((select auth.uid()) = user_id) with check ((select auth.uid()) = user_id);
create policy "progresso_admin_leitura" on public.progresso
  for select to authenticated using (public.is_admin());

-- MÉTRICAS: aluna gere as suas; professora vê as de todas
drop policy if exists "metricas_proprias" on public.metricas;
create policy "metricas_aluna" on public.metricas
  for all to authenticated
  using ((select auth.uid()) = user_id) with check ((select auth.uid()) = user_id);
create policy "metricas_admin_leitura" on public.metricas
  for select to authenticated using (public.is_admin());

-- PROFILES: professora vê a lista de alunas
drop policy if exists "perfil_proprio_select" on public.profiles;
create policy "perfil_select" on public.profiles
  for select to authenticated
  using ((select auth.uid()) = id or public.is_admin());

-- Métricas: um número por dia (para o Social Blade não duplicar)
alter table public.metricas
  add constraint metricas_user_data_unique unique (user_id, data);
```

### 1.3 — Documentos por aluna (os 2 PDFs)

```sql
-- Balde privado para os PDFs
insert into storage.buckets (id, name, public)
values ('documentos', 'documentos', false) on conflict (id) do nothing;

create table if not exists public.documentos (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  tipo text not null check (tipo in ('analise_perfil','plano_conteudo')),
  caminho text not null,
  nome_ficheiro text,
  criado_em timestamptz default now(),
  unique (user_id, tipo)
);
create index if not exists idx_documentos_user on public.documentos (user_id);
alter table public.documentos enable row level security;

create policy "documentos_leitura" on public.documentos
  for select to authenticated
  using ((select auth.uid()) = user_id or public.is_admin());
create policy "documentos_admin_escrita" on public.documentos
  for all to authenticated
  using (public.is_admin()) with check (public.is_admin());

-- Ficheiros no Storage: {user_id}/analise-perfil.pdf, {user_id}/plano-conteudo.pdf
create policy "docs_storage_leitura" on storage.objects
  for select to authenticated
  using (bucket_id = 'documentos'
    and ((storage.foldername(name))[1] = (select auth.uid())::text or public.is_admin()));
create policy "docs_storage_admin_escrita" on storage.objects
  for all to authenticated
  using (bucket_id = 'documentos' and public.is_admin())
  with check (bucket_id = 'documentos' and public.is_admin());
```

### 1.4 — Leads (captura)

```sql
create table if not exists public.leads (
  id uuid primary key default gen_random_uuid(),
  instagram_handle text,
  email text,
  whatsapp text,
  criado_em timestamptz default now()
);
alter table public.leads enable row level security;
create policy "leads_admin_leitura" on public.leads
  for select to authenticated using (public.is_admin());
-- (as escritas são feitas pelas funções com service role)
```

### 1.5 — Tornares-te professora (admin)

Depois de te registares na app com o teu email, corre uma vez (troca o email):

```sql
update public.profiles set papel = 'admin'
where id = (select id from auth.users where email = 'catiacreator@gmail.com');
```

---

## 2. As três funções (Edge Functions)

No Supabase: **Edge Functions → Create function**. Uma por cada. O Lovable também consegue criá-las.

### 2.1 — `processar-documento` (Claude lê o PDF → preenche a análise)

Precisa do secret: `ANTHROPIC_API_KEY`.

```typescript
import Anthropic from "npm:@anthropic-ai/sdk";
import { createClient } from "npm:@supabase/supabase-js@2";

const cors = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Headers": "authorization, x-client-info, apikey, content-type",
};
const MODEL = "claude-opus-5"; // "claude-sonnet-5" é mais barato

Deno.serve(async (req) => {
  if (req.method === "OPTIONS") return new Response("ok", { headers: cors });
  try {
    const { aluna_id, tipo, file_base64 } = await req.json();

    const userClient = createClient(
      Deno.env.get("SUPABASE_URL")!, Deno.env.get("SUPABASE_ANON_KEY")!,
      { global: { headers: { Authorization: req.headers.get("Authorization")! } } },
    );
    const { data: { user } } = await userClient.auth.getUser();
    if (!user) return json({ error: "nao_autenticado" }, 401);
    const { data: perfil } = await userClient.from("profiles").select("papel").eq("id", user.id).single();
    if (perfil?.papel !== "admin") return json({ error: "sem_permissao" }, 403);

    const anthropic = new Anthropic({ apiKey: Deno.env.get("ANTHROPIC_API_KEY")! });
    const isAnalise = tipo === "analise_perfil";
    const resposta = await anthropic.messages.create({
      model: MODEL, max_tokens: 8000,
      output_config: { format: { type: "json_schema", schema: isAnalise ? ANALISE_SCHEMA : PLANO_SCHEMA } },
      messages: [{ role: "user", content: [
        { type: "document", source: { type: "base64", media_type: "application/pdf", data: file_base64 } },
        { type: "text", text: isAnalise ? PROMPT_ANALISE : PROMPT_PLANO },
      ] }],
    });
    const dados = JSON.parse(resposta.content.find((b) => b.type === "text").text);

    const admin = createClient(Deno.env.get("SUPABASE_URL")!, Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!);
    if (isAnalise) {
      await admin.from("analise").delete().eq("user_id", aluna_id);
      await admin.from("analise").insert(dados.seccoes.map((s) => ({ user_id: aluna_id, ...s })));
      return json({ ok: true, inseridos: dados.seccoes.length });
    } else {
      await admin.from("roteiros").insert(dados.roteiros.map((r) => ({ user_id: aluna_id, ...r })));
      return json({ ok: true, inseridos: dados.roteiros.length });
    }
  } catch (e) { return json({ error: String(e) }, 500); }
});

function json(body: unknown, status = 200) {
  return new Response(JSON.stringify(body), { status, headers: { ...cors, "Content-Type": "application/json" } });
}

const PROMPT_ANALISE = `Este PDF é a análise do perfil de Instagram de uma aluna. Extrai as secções
(foto de perfil, @, bio, feed/destaques). Para cada uma preenche "como_esta", "como_ficaria"
e "observacao", em português de Portugal.`;
const PROMPT_PLANO = `Este PDF é um plano de conteúdo com roteiros. Extrai cada roteiro numerado por dia,
com título, categoria (Reels/Stories/Carrossel), duração em minutos, dificuldade, tema central,
gancho inicial e o roteiro completo, em português de Portugal.`;

const ANALISE_SCHEMA = {
  type: "object", additionalProperties: false,
  properties: { seccoes: { type: "array", items: {
    type: "object", additionalProperties: false,
    properties: { seccao: { type: "string" }, como_esta: { type: "string" },
      como_ficaria: { type: "string" }, observacao: { type: "string" } },
    required: ["seccao", "como_esta", "como_ficaria", "observacao"] } } },
  required: ["seccoes"],
};
const PLANO_SCHEMA = {
  type: "object", additionalProperties: false,
  properties: { roteiros: { type: "array", items: {
    type: "object", additionalProperties: false,
    properties: { dia: { type: "integer" }, titulo: { type: "string" }, categoria: { type: "string" },
      duracao_min: { type: "integer" }, dificuldade: { type: "string" }, tema_central: { type: "string" },
      gancho_inicial: { type: "string" }, roteiro_completo: { type: "string" } },
    required: ["dia", "titulo", "categoria", "duracao_min", "dificuldade", "tema_central", "gancho_inicial", "roteiro_completo"] } } },
  required: ["roteiros"],
};
```

### 2.2 — `buscar-seguidores` (Social Blade → gráfico + folha "Seguidores")

Precisa dos secrets: `SOCIALBLADE_CLIENT_ID`, `SOCIALBLADE_TOKEN`, `GOOGLE_SA_EMAIL`, `GOOGLE_SA_KEY`, `GOOGLE_SHEET_ID`.

```typescript
import { JWT } from "npm:google-auth-library@9";
import { createClient } from "npm:@supabase/supabase-js@2";

const cors = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Headers": "authorization, x-client-info, apikey, content-type",
};

Deno.serve(async (req) => {
  if (req.method === "OPTIONS") return new Response("ok", { headers: cors });
  try {
    const { aluna_id, instagram } = await req.json();
    const handle = String(instagram || "").replace(/^@/, "").trim();
    if (!handle) return json({ error: "sem_@" }, 400);

    const userClient = createClient(
      Deno.env.get("SUPABASE_URL")!, Deno.env.get("SUPABASE_ANON_KEY")!,
      { global: { headers: { Authorization: req.headers.get("Authorization")! } } },
    );
    const { data: { user } } = await userClient.auth.getUser();
    if (!user) return json({ error: "nao_autenticado" }, 401);
    let alvo = user.id;
    if (aluna_id && aluna_id !== user.id) {
      const { data: p } = await userClient.from("profiles").select("papel").eq("id", user.id).single();
      if (p?.papel !== "admin") return json({ error: "sem_permissao" }, 403);
      alvo = aluna_id;
    }

    const r = await fetch(
      `https://matrix.sbapis.com/b/instagram/statistics?query=${encodeURIComponent(handle)}&history=default&allow-stale=false`,
      { headers: { clientid: Deno.env.get("SOCIALBLADE_CLIENT_ID")!, token: Deno.env.get("SOCIALBLADE_TOKEN")! } },
    );
    const sb = await r.json();
    if (!sb?.status?.success) return json({ error: "social_blade_falhou", detalhe: sb?.status?.error }, 502);

    const total = sb.data.statistics.total;
    const g = sb.data.statistics.growth?.followers ?? {};
    const grade = sb.data.misc?.grade?.grade ?? "";
    const daily = Array.isArray(sb.data.daily) ? sb.data.daily : [];
    const agora = new Date().toISOString();

    const admin = createClient(Deno.env.get("SUPABASE_URL")!, Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!);
    await admin.from("profiles").update({ instagram_handle: handle }).eq("id", alvo);
    let linhas = daily.map((d: any) => ({ user_id: alvo, data: d.date, seguidores: d.followers }));
    if (linhas.length === 0) linhas = [{ user_id: alvo, data: agora.slice(0, 10), seguidores: total.followers }];
    await admin.from("metricas").upsert(linhas, { onConflict: "user_id,data" });

    try {
      const jwt = new JWT({
        email: Deno.env.get("GOOGLE_SA_EMAIL")!,
        key: (Deno.env.get("GOOGLE_SA_KEY")!).replace(/\\n/g, "\n"),
        scopes: ["https://www.googleapis.com/auth/spreadsheets"],
      });
      const { access_token } = await jwt.authorize();
      const sheetId = Deno.env.get("GOOGLE_SHEET_ID")!;
      await fetch(
        `https://sheets.googleapis.com/v4/spreadsheets/${sheetId}/values/Seguidores!A:G:append?valueInputOption=USER_ENTERED&insertDataOption=INSERT_ROWS`,
        { method: "POST",
          headers: { Authorization: `Bearer ${access_token}`, "Content-Type": "application/json" },
          body: JSON.stringify({ values: [[agora, "@" + handle, total.followers, g["7"] ?? "", g["30"] ?? "", total.engagement_rate ?? "", grade]] }) },
      );
    } catch (_) { /* se a folha falhar, os dados já ficaram no Supabase */ }

    return json({ ok: true, seguidores: total.followers, crescimento_7d: g["7"] ?? null,
      crescimento_30d: g["30"] ?? null, creditos_restantes: sb.info?.credits?.available ?? null });
  } catch (e) { return json({ error: String(e) }, 500); }
});

function json(body: unknown, status = 200) {
  return new Response(JSON.stringify(body), { status, headers: { ...cors, "Content-Type": "application/json" } });
}
```

### 2.3 — `guardar-lead` (captura → Supabase + folha "Leads")

Precisa dos secrets: `GOOGLE_SA_EMAIL`, `GOOGLE_SA_KEY`, `GOOGLE_SHEET_ID`. **É pública** (a captura é antes do login).

```typescript
import { JWT } from "npm:google-auth-library@9";
import { createClient } from "npm:@supabase/supabase-js@2";

const cors = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Headers": "authorization, x-client-info, apikey, content-type",
};

Deno.serve(async (req) => {
  if (req.method === "OPTIONS") return new Response("ok", { headers: cors });
  try {
    const { instagram, email, whatsapp } = await req.json();
    const handle = String(instagram || "").replace(/^@/, "").trim();
    if (!handle && !email) return json({ error: "dados_em_falta" }, 400);
    const agora = new Date().toISOString();

    const admin = createClient(Deno.env.get("SUPABASE_URL")!, Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!);
    await admin.from("leads").insert({ instagram_handle: handle, email, whatsapp });

    const jwt = new JWT({
      email: Deno.env.get("GOOGLE_SA_EMAIL")!,
      key: (Deno.env.get("GOOGLE_SA_KEY")!).replace(/\\n/g, "\n"),
      scopes: ["https://www.googleapis.com/auth/spreadsheets"],
    });
    const { access_token } = await jwt.authorize();
    const sheetId = Deno.env.get("GOOGLE_SHEET_ID")!;
    const r = await fetch(
      `https://sheets.googleapis.com/v4/spreadsheets/${sheetId}/values/Leads!A:D:append?valueInputOption=USER_ENTERED&insertDataOption=INSERT_ROWS`,
      { method: "POST",
        headers: { Authorization: `Bearer ${access_token}`, "Content-Type": "application/json" },
        body: JSON.stringify({ values: [[agora, "@" + handle, email ?? "", whatsapp ?? ""]] }) },
    );
    if (!r.ok) return json({ ok: true, sheet: "falhou", detalhe: await r.text() });
    return json({ ok: true });
  } catch (e) { return json({ error: String(e) }, 500); }
});

function json(body: unknown, status = 200) {
  return new Response(JSON.stringify(body), { status, headers: { ...cors, "Content-Type": "application/json" } });
}
```

---

## 3. Configuração do Google (uma vez)

Faz tudo **logada na conta `catiacreator.oficial@gmail.com`**.

1. **Cria a folha** na Drive. Cria duas abas:
   - **`Leads`** → 1ª linha: `Data | @ Instagram | Email | WhatsApp`
   - **`Seguidores`** → 1ª linha: `Data | @ Instagram | Seguidores | Crescimento 7d | Crescimento 30d | Engajamento (%) | Nota`
   - Copia o **ID** do link (a parte entre `/d/` e `/edit`).
2. **Cria o robô:** [console.cloud.google.com](https://console.cloud.google.com) → novo projeto → ativa **"Google Sheets API"** → cria uma **Service Account** → gera uma **chave JSON** e descarrega.
3. **Abre o JSON** e copia `client_email` e `private_key`.
4. **Partilha a folha** com o `client_email` (o endereço `...gserviceaccount.com`) como **Editor**.

---

## 4. Secrets do Supabase (checklist)

Supabase → **Edge Functions → Secrets**. (`SUPABASE_URL`, `SUPABASE_ANON_KEY` e `SUPABASE_SERVICE_ROLE_KEY` já existem automaticamente.)

| Secret | De onde vem | Usado por |
|---|---|---|
| `ANTHROPIC_API_KEY` | O teu plano Claude | processar-documento |
| `SOCIALBLADE_CLIENT_ID` | Painel developer da Social Blade | buscar-seguidores |
| `SOCIALBLADE_TOKEN` | Painel developer da Social Blade | buscar-seguidores |
| `GOOGLE_SA_EMAIL` | `client_email` do JSON | buscar-seguidores, guardar-lead |
| `GOOGLE_SA_KEY` | `private_key` do JSON (inteira, com `\n`) | buscar-seguidores, guardar-lead |
| `GOOGLE_SHEET_ID` | ID da tua Google Sheet | buscar-seguidores, guardar-lead |

---

## 5. O prompt do Lovable

Cola isto no Lovable, liga o Supabase, e depois pede-lhe para criar as 3 Edge Functions acima.

> **Constrói uma app web (SPA) chamada "Plano Viral" — criadoras seguem uma trilha diária de roteiros e acompanham o crescimento no Instagram. Usa Supabase (as tabelas JÁ EXISTEM — não cries novas).**
>
> **Marca:** cores sólidas (design liso, SEM gradientes) exceto no gráfico de crescimento e na página de captura. Rosa #D1356F, Roxo #7C3BC0 (ação), Âmbar #F2A24E; neutros Tinta #1A1524, Grafite #5B5560, Cartão #F7F5F8. Tipografia: **Inter** (títulos 800, tracking apertado), **Fraunces** itálico só em citações. Cantos 12–20px, muito branco.
>
> **Página de entrada (captura):** título "Quantos seguidores já **deverias ter**?" (destaque em gradiente), campos **@ do Instagram · email · WhatsApp**, botão "Analisar o meu perfil". Ao submeter, chama a Edge Function `guardar-lead` com `{ instagram, email, whatsapp }` (pública). Link "Já tenho conta, quero entrar".
>
> **Autenticação:** registo/login Supabase (email+password); no registo pede o nome (guarda em `options.data.nome`).
>
> **Início:** saudação com o nome; card "Crescimento" com gráfico semanal (de `metricas`) e stats esta semana/mês. Botão para sincronizar com o Instagram → chama `buscar-seguidores` com `{ instagram }`.
>
> **Minha trilha:** lista de `roteiros` (base + os da aluna) por `dia`; abas "A fazer"/"Concluídos" (de `progresso`); "X de 60 feitos" + barra; pesquisa; cards com tag de categoria. Clicar um card (em qualquer aba) abre o roteiro.
>
> **Roteiro:** tema central, gancho inicial, roteiro completo, cada um com "Copiar"; painel lateral com "Copiar tudo" e "Marcar como feito" (upsert em `progresso`).
>
> **Análise:** documentos da aluna (de `documentos`) para descarregar + diagnóstico por secções (de `analise`).
>
> **Admin (rota `/admin`, só `papel = 'admin'`):** KPIs; gestão de roteiros (campo "Aluna": vazio = base, escolhida = individual); lista de alunas com progresso; perfil da aluna com **dois uploads de PDF** (Análise de perfil, Plano de conteúdo) → guarda no Storage `documentos` e chama `processar-documento` com `{ aluna_id, tipo, file_base64 }` para preencher os campos automaticamente.
>
> **Respeita o RLS:** escritas em `roteiros`/`analise` só admin; cada aluna só vê os seus dados.

---

## 6. Ordem de construção (do princípio ao fim)

1. [ ] Correr o **SQL** (1.1 → 1.5) no Supabase.
2. [ ] Registares-te na app e correr o **update de admin** (1.5).
3. [ ] Configurar o **Google** (secção 3) e partilhar a folha com o robô.
4. [ ] Meter os **secrets** no Supabase (secção 4).
5. [ ] Criar as **3 Edge Functions** (secção 2).
6. [ ] Colar o **prompt do Lovable** (secção 5) e construir a app.
7. [ ] **Inserir os 60 roteiros base** (Table Editor ou importar CSV).
8. [ ] **Ligar a Hotmart** (secção 8): SQL do acesso pago + função `hotmart-webhook` + webhook na Hotmart.
9. [ ] Testar: captura → lead na folha; sincronizar @ → gráfico + folha; upload PDF → campos preenchidos; compra Hotmart → trilha desbloqueia.

---

## 7. Custos e cuidados

- **Social Blade:** 1 crédito por consulta. Chama `buscar-seguidores` na entrada e num **refresh semanal**, nunca a cada abertura de página.
- **Claude:** cêntimos por PDF. Só corre quando carregas um documento.
- **Supabase e Google Sheets:** grátis para este volume.
- **Chaves:** ficam sempre nos **secrets do Supabase** (servidor), nunca na app nem no browser.

---

## 8. Acesso pago (Hotmart) — desbloquear a trilha só a quem comprou

Os 60 roteiros existem na app, mas **só aparecem a quem pagou na Hotmart**. A Hotmart avisa a app por webhook; a app marca o email como pago e a trilha desbloqueia.

### 8.1 — SQL (corre depois da secção 1)

```sql
-- Fonte de verdade dos pagamentos (só as funções escrevem aqui)
create table if not exists public.acessos (
  email text primary key,
  ativo boolean default true,
  atualizado_em timestamptz default now()
);
alter table public.acessos enable row level security;

-- Flag de acesso no perfil
alter table public.profiles add column if not exists pago boolean default false;

-- Ao registar, herda o acesso se o email já comprou
create or replace function public.handle_new_user()
returns trigger language plpgsql security definer set search_path = public as $$
begin
  insert into public.profiles (id, nome, pago)
  values (
    new.id,
    new.raw_user_meta_data->>'nome',
    exists (select 1 from public.acessos a where a.email = new.email and a.ativo)
  );
  return new;
end;
$$;

-- Aplicar o pagamento a uma conta já existente (usado pelo webhook)
create or replace function public.sincronizar_pago(p_email text, p_ativo boolean)
returns void language sql security definer set search_path = public as $$
  update public.profiles set pago = p_ativo
  where id in (select id from auth.users where email = p_email);
$$;

-- ROTEIROS: só quem pagou (ou admin) vê. SUBSTITUI a policy da secção 1.2.
drop policy if exists "roteiros_leitura" on public.roteiros;
create policy "roteiros_leitura" on public.roteiros
  for select to authenticated
  using (
    public.is_admin()
    or (
      (user_id is null or user_id = (select auth.uid()))
      and exists (select 1 from public.profiles p
                  where p.id = (select auth.uid()) and p.pago = true)
    )
  );
```

### 8.2 — Edge Function `hotmart-webhook` (pública; recebe os avisos da Hotmart)

Precisa do secret `HOTMART_HOTTOK` (o teu token da Hotmart, para confirmar que o aviso é mesmo deles).

```typescript
import { createClient } from "npm:@supabase/supabase-js@2";

const cors = { "Access-Control-Allow-Origin": "*", "Access-Control-Allow-Headers": "*" };

Deno.serve(async (req) => {
  if (req.method === "OPTIONS") return new Response("ok", { headers: cors });
  try {
    const body = await req.json();
    const hottok = req.headers.get("x-hotmart-hottok") ?? body?.hottok;
    if (hottok !== Deno.env.get("HOTMART_HOTTOK")) return json({ error: "nao_autorizado" }, 401);

    const evento = body?.event ?? body?.data?.event ?? "";
    const email = String(body?.data?.buyer?.email ?? body?.buyer?.email ?? "").toLowerCase().trim();
    if (!email) return json({ ok: true, ignorado: "sem_email" });

    const da = ["PURCHASE_APPROVED", "PURCHASE_COMPLETE"].includes(evento);
    const tira = ["PURCHASE_REFUNDED", "PURCHASE_CHARGEBACK", "PURCHASE_CANCELED", "SUBSCRIPTION_CANCELLATION"].includes(evento);
    if (!da && !tira) return json({ ok: true, ignorado: evento });

    const admin = createClient(Deno.env.get("SUPABASE_URL")!, Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!);
    await admin.from("acessos").upsert({ email, ativo: da, atualizado_em: new Date().toISOString() });
    await admin.rpc("sincronizar_pago", { p_email: email, p_ativo: da });
    return json({ ok: true, email, acesso: da });
  } catch (e) { return json({ error: String(e) }, 500); }
});

function json(body: unknown, status = 200) {
  return new Response(JSON.stringify(body), { status, headers: { ...cors, "Content-Type": "application/json" } });
}
```

### 8.3 — Ligar na Hotmart

Na Hotmart: **Ferramentas → Webhook (Postback)** → aponta para o URL da função `hotmart-webhook` → escolhe os eventos **Compra aprovada, Reembolso, Chargeback, Cancelamento** → copia o **hottok** e mete-o no secret `HOTMART_HOTTOK` do Supabase.

### 8.4 — Na app (Lovable)

> **A trilha (roteiros) só aparece a quem tem `profiles.pago = true`. Se `pago = false`, mostra a trilha bloqueada com um aviso "Desbloqueia o teu Plano Viral" e um botão para a página de compra na Hotmart. O RLS já garante que sem pagamento não chegam roteiros — o bloqueio visual é a cereja no topo.**

**Fluxo completo:** pessoa compra na Hotmart → webhook marca o email como pago → a pessoa regista-se (ou já estava registada) com esse email → a trilha desbloqueia automaticamente.

---

*Guia Plano Viral — Cátia Creator. Constrói por ordem, testa cada peça antes de avançar.*
