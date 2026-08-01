# Prompt para o Lovable — Plano Viral

> Cola **tudo isto** num projeto novo do Lovable. Depois liga o Supabase e pede-lhe para criar as Edge Functions. Junta também os screenshots do protótipo como referência visual.

---

Constrói uma aplicação web completa (React) chamada **Plano Viral** — uma plataforma onde criadoras de conteúdo seguem uma trilha diária de roteiros e acompanham o crescimento do Instagram. Usa **Supabase** para autenticação (email + password) e base de dados. **As tabelas JÁ EXISTEM no Supabase — NÃO cries tabelas novas nem alteres o schema. Usa exatamente: `profiles`, `roteiros`, `progresso`, `metricas`, `analise`, `documentos`, `leads`, `acessos`.** Respeita sempre o Row Level Security (RLS).

## Identidade visual (segue à risca)

- **Design LISO (flat), cores sólidas, SEM gradientes** — exceto em DOIS sítios apenas: (a) a **linha do gráfico** de crescimento, e (b) o **destaque do título e o botão** da página de captura.
- **Cores:** Roxo **#7C3BC0** (cor de ação — botões, estados ativos, dia selecionado, links), Rosa **#D1356F**, Âmbar **#F2A24E**. Neutros: Tinta **#1A1524** (texto), Grafite **#5B5560** (corpo), Névoa **#9A9499** (secundário), Cartão **#F7F5F8**, Branco **#FFFFFF**.
- **Gradiente** (só nos 2 sítios acima), sempre nesta ordem: `linear-gradient(120deg, #7C3BC0, #AF3890, #D1356F, #E13D3F, #E97C57, #F2A24E)`.
- **Tipografia:** **Inter** (Google Fonts) — títulos peso 800, tracking -0.03em; corpo 400/500 em Grafite. **Fraunces itálico** (Google Fonts) SÓ para citações.
- Cantos arredondados 12–20px, cards com sombra suave, MUITO espaço branco. Uma barra fina roxa sólida no topo de tudo. Suporta **tema claro/escuro** (botão de alternância).
- **Categorias por cor:** Reels = roxo, Stories = rosa, Carrossel = âmbar.

## Ecrãs (constrói todos)

### 1. Funil de entrada (público) — rota `/`
Um só ecrã com **três passos** (mostra um de cada vez, centrado, uma coluna):

**Passo A — Captura.** Logo "Plano Viral". Título "Quantos seguidores já **deverias ter**?" (as palavras "deverias ter" no gradiente). Campos **@ do Instagram · email · WhatsApp**. Botão gradiente "Analisar o meu perfil" → chama `guardar-lead` com `{ instagram, email, whatsapp }` (pública) e **avança para o Passo B**. Link "Já tenho o meu Plano Viral, entrar" → Passo C. Rodapé: "🔒 Sem palavra-passe, sem aceder à tua conta."

**Passo B — Projeção (a barreira).** Título "Olha o que podes **alcançar**, @{handle}". (1) **Gráfico de projeção** — curva com o gradiente, de "Hoje" para "Em 3 meses" (usa `buscar-seguidores` para o número real de hoje via Social Blade e mostra uma estimativa de crescimento). (2) **Calendário de posts BLOQUEADO** — o mês inteiro com **uma marca por dia**, mas **só a indicação Reels / Carrossel** (sem títulos); é clicável mas **não abre nada**; selo "🔒 Bloqueado". (3) Botão gradiente **"Desbloquear o meu Plano Viral"** → abre a página de compra na **Hotmart**. Link "Já tenho o Plano Viral, entrar" → Passo C.

**Passo C — Login (clientes).** Campos **@ do Instagram** e **telemóvel**. Botão "Entrar". Usa **autenticação por telemóvel (OTP por SMS) do Supabase**: a pessoa mete o telemóvel, recebe um código por SMS, entra; guarda o @ no perfil. Só quem tem `pago = true` acede à trilha.

> **Nota técnica:** o login por telemóvel (OTP SMS) precisa de um fornecedor de SMS ligado ao Supabase (ex.: Twilio) — tem custo por SMS. Alternativa mais barata: email + password, mostrando o @ apenas como campo de perfil.

### 3. Estrutura da app (depois do login)
Barra lateral fixa à esquerda (desktop): logo "Plano Viral", menu **Início · Minha trilha · Calendário · Análise**, e — só se `profiles.papel = 'admin'` — uma secção "Professora" com **Painel admin**. Em telemóvel, esconde a barra lateral e mostra uma **barra de navegação inferior** com os mesmos itens. Barra superior: pesquisa, botão "Minha trilha", botão de tema, e avatar (nome + "Aluno").

### 4. Início — rota `/inicio`
Saudação "Olá, {nome} 👋" + data por cima. Card **"Crescimento no Instagram"**: número grande de seguidores (de `metricas`), "+X esta semana", **gráfico de linha da evolução** (ESTE é o sítio do gradiente — a linha usa o gradiente Cat.IA), e stats "esta semana / média / melhor semana". Botão **"Sincronizar com o Instagram"** que chama `buscar-seguidores` com `{ instagram }` e recarrega o gráfico. Secção "Trilha do mês": card do próximo roteiro a fazer + grelha de miniaturas dos próximos.

### 5. Minha trilha — rota `/trilha`
**BLOQUEIO (paywall):** se `profiles.pago = false`, mostra a trilha **bloqueada** com "Desbloqueia o teu Plano Viral" e um botão para a página de compra na Hotmart. Se `pago = true`, mostra normal:
Título "Minha trilha". Card de progresso: "X de 60 feitos", percentagem, barra, botão "Baixar PDF". Abas **"A fazer" / "Concluídos"** (de `progresso`). Pesquisa por título/categoria. Lista de `roteiros` (base com `user_id` nulo + os da própria aluna) ordenados por `dia`: cada card tem badge do dia, tag de categoria colorida, duração, dificuldade. **Clicar em qualquer card (nas duas abas) abre o roteiro.** Os concluídos mostram "✓ Feito" a verde.

### 5b. Calendário — rota `/calendario` (só quem tem `pago = true`)
Vista de **mês** com **um post por dia**: cada dia mostra o roteiro desse dia (título + cor da categoria: Reels roxo, Stories rosa, Carrossel âmbar), com o **dia de hoje destacado**. Constrói a partir da tabela `roteiros` — mapeia o `dia` de cada roteiro a uma data real a partir de `profiles.inicio_trilha` (data em que a aluna começou) — e marca como **publicados** os que estão em `progresso` (feito, por `concluido_em`). Tem navegação de mês (‹ Agosto 2026 ›), legenda de categorias, e um resumo "X de N publicados". O calendário enche-se sozinho à medida que houver roteiros.

### 6. Roteiro — rota `/trilha/:id`
Link "Voltar para Minha trilha". Cabeçalho: tag de categoria, dia, duração, dificuldade, link "Ver vídeos de referência". Blocos **Tema central**, **Gancho inicial** (em itálico serif Fraunces), **Roteiro completo** — cada um com botão "Copiar". Painel lateral fixo: botão "Copiar roteiro inteiro" + botão verde **"Marcar como feito"** (upsert em `progresso`: feito=true, concluido_em=agora) + resumo (formato, duração, dificuldade, progresso X/60).

### 7. Análise — rota `/analise`
"Os teus documentos": dois cards — **Análise de perfil** e **Plano de conteúdo** — para descarregar (de `documentos`, URL assinado do Storage). "Diagnóstico rápido": um card por secção (de `analise`: foto de perfil, @, bio, feed), cada um a comparar "como está agora" vs "como ficaria" + observação.

### 8. Painel admin — rota `/admin` (só `papel = 'admin'`; esconde o menu e bloqueia a rota para os outros)
Badge "Vista de professora". 4 indicadores no topo (alunas ativas, roteiros publicados X/60, taxa de conclusão, conclusões hoje) com mini-gráficos. Coluna esquerda: **"Roteiros da trilha"** — tabela (dia, título, categoria, estado, editar) + botão "Novo roteiro" com campo opcional **"Aluna"** (vazio = roteiro base para todas com `user_id` nulo; escolhida = roteiro individual com o `user_id` dela). Coluna direita: **"Alunas"** (lista de `profiles` com avatar, nome, @, barra de progresso X/60) — clicar numa aluna abre o perfil dela; e **"Análise de perfil"** (formulário: escolher aluna + secção + gravar em `analise`).

### 9. Admin → Perfil da aluna
Cabeçalho com avatar, nome, @, X/60. Secção **"Documentos da aluna"** com dois campos de upload de PDF: **Análise de perfil** e **Plano de conteúdo**. Ao carregar, envia o ficheiro para o Storage bucket `documentos` no caminho `{user_id}/{tipo}.pdf`, faz upsert em `documentos`, e chama a Edge Function `processar-documento` com `{ aluna_id, tipo, file_base64 }` (que lê o PDF e preenche automaticamente a `analise` e/ou os `roteiros` da aluna). Mostra "A processar…" enquanto corre.

## Ligações às Edge Functions (do Supabase)
- `guardar-lead` → na página de captura (pública).
- `buscar-seguidores` → no Início (botão sincronizar) e no admin.
- `processar-documento` → no admin, ao carregar um PDF.
- `hotmart-webhook` → NÃO é chamada pela app (é a Hotmart que a chama); só desbloqueia o acesso pago.

## Regras finais
- Respeita o RLS: escritas em `roteiros`/`analise` só admin; cada aluna só vê os seus dados; a trilha só aparece a quem tem `pago = true`.
- Cores **sólidas em tudo** — gradiente só no gráfico e na página de captura.
- **Responsivo** (funciona bem no telemóvel), com tema claro/escuro.
