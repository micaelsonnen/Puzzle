# CLAUDE.md — guia pra trabalhar neste repositório

Este arquivo existe pra qualquer conversa nova (comigo ou com outro assistente de IA) conseguir mexer no projeto **sem quebrar o que já funciona**. Leia isto inteiro antes de editar qualquer coisa.

---

## O que é este projeto

**Puzzle** (antes chamado "Mind" — nome já trocado em tudo, mas pode aparecer em nomes técnicos internos como e-mails de demo ou nomes de função/tabela). Plataforma de psicoterapia com financiamento tripartite: empresa patrocina parte da sessão, paciente paga coparticipação, psicólogo recebe por sessão e paga mensalidade pela plataforma.

Leia o `README.md` primeiro pra entender o modelo de negócio e o mapa de páginas — este arquivo aqui é sobre **como mexer no código sem causar estrago**, não sobre o que o produto faz.

---

## Stack e restrições importantes

- **Site estático puro**: HTML + CSS + JS vanilla em cada arquivo, sem framework, sem build step, sem bundler. Cada página é autocontida (CSS e JS inline no próprio `<head>`/`<script>`).
- **Backend é 100% Supabase**: Postgres + Auth + Storage + Row Level Security + `pg_cron` + `pg_net`. Não existe servidor próprio, não existe Edge Function em uso (decisão deliberada — tudo é feito via SQL puro pra não exigir deploy via CLI).
- **Hospedagem**: GitHub Pages, repositório `micaelsonnen/Puzzle`, branch `main`. Publica automaticamente alguns minutos depois do push.
- **Sem CI/CD, sem testes automatizados**. Toda validação é manual, testando no navegador depois de publicar.

## Credenciais no código (não são segredo, mas precisam bater)

Toda página que fala com o Supabase tem isto hardcoded:
```js
const SUPABASE_URL = 'https://xzkmokjyaqutkjxqlkck.supabase.co';
const SUPABASE_KEY = 'sb_publishable_5OMjosvKS6_Mt3gGYUxQ0w_JrMtH1jZ';
```
Isso é a chave **pública** (`anon`/`publishable`), segura de expor — a segurança de verdade é toda via RLS no banco, não pela chave. **Nunca** coloque a chave `service_role` em nenhum arquivo `.html`.

⚠️ **Já aconteceu desses dois valores ficarem desatualizados** (projeto Supabase foi recriado em algum momento e o código continuou apontando pro projeto antigo, causando login "misteriosamente" não funcionar). Se qualquer coisa relacionada a login/dados parar de funcionar sem motivo aparente, **primeiro confira se esses dois valores batem com o projeto atual** (Project Settings → API no painel do Supabase) antes de investigar qualquer outra coisa.

---

## Como as migrações SQL funcionam

Todo arquivo `N_migration_*.sql` (numerado) é executado **manualmente** pelo humano, colando no SQL Editor do Supabase — eu (IA) nunca tenho acesso direto ao banco, nem consigo rodar nada sozinho. Isso significa:

1. **Eu não sei o estado real do banco** a menos que o humano rode uma consulta e me cole o resultado. Nunca assuma que uma migração já rodou — pergunte ou peça pra conferir.
2. **Sempre escreva migrações idempotentes**: `create table if not exists`, `add column if not exists`, `drop policy if exists` antes de `create policy`, `create or replace function`. A única exceção são os `seed_*.sql` (dados fictícios), que **não** são seguros rodar duas vezes.
3. **Numere sequencialmente** o próximo arquivo de migração (olhe o `README.md`, seção 3, pra saber o próximo número livre) e **atualize o README** depois de criar uma nova.
4. Se uma migração corrige/substitui uma anterior (já aconteceu 2x — vídeo e métricas de admin), **não apague a antiga do histórico**, só deixe claro no README qual delas usar.

## ⚠️ A armadilha do `pg_net` — leia antes de fazer qualquer integração com API externa

Se você for integrar qualquer serviço externo (tipo fizemos com Resend e Daily.co), **nunca** escreva uma função SQL que faça `net.http_post()` e depois espere a resposta num loop **dentro da mesma função/transação**. Isso trava pra sempre (timeout garantido), porque o worker do `pg_net` só processa pedidos de transações já commitadas, e a transação da sua função só commita quando ela termina.

**Padrão correto**: duas funções separadas — uma que **dispara** a chamada e retorna imediatamente (`net.http_post()` sem esperar nada), outra que **checa** se a resposta já chegou (consulta `net._http_response`, retorna `null` se ainda não). O **navegador do usuário** (JS no front-end) chama a função de checagem repetidamente, a cada ~500ms, até ter resposta ou desistir. Veja `migration_video_daily_fix.sql` como referência de implementação.

Detalhe operacional: depois de `create extension pg_net`, às vezes o worker só sobe de verdade depois de um **restart do projeto** no painel do Supabase.

Pra chamadas onde você **não precisa** da resposta de volta (ex: mandar um e-mail e seguir em frente), pode usar `perform net.http_post(...)` direto, sem essa complicação — só não tente ler `net._http_response` na mesma chamada.

---

## Convenções de código deste projeto

### Segurança
- **Toda** string vinda do usuário e inserida via `innerHTML` precisa passar por `escapeHtml()` primeiro. Cada arquivo tem sua própria cópia dessa função (não tem módulo compartilhado, é site estático). Se criar um arquivo novo que renderiza dado dinâmico, copie a função de qualquer outro arquivo:
  ```js
  function escapeHtml(str) {
    if (str === null || str === undefined) return '';
    return String(str).replace(/[&<>"']/g, c => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]));
  }
  ```
- Chaves de API de serviços externos (Resend, Daily.co) ficam no **Vault** do Supabase (`vault.decrypted_secrets`), nunca em texto puro em tabela ou função — funções SQL são legíveis por qualquer usuário autenticado por padrão.
- Buckets de Storage: documentos de verificação (CRP, diploma) são **privados** com signed URL; fotos de perfil/logo são **públicas** de propósito. Não inverta isso.
- RLS é a linha de defesa real, não a UI. Toda checagem de "quem pode ver o quê" deve existir como policy no banco — uma tela que só *parece* bloqueada mas sem RLS por trás não protege nada.

### Papéis e acesso
- **Psicólogo**: identificado pela tabela `psicologos`, e-mail bate com `auth.jwt() ->> 'email'`.
- **Paciente**: tabela `pacientes`, mesma lógica.
- **Empresa**: tabela `empresas`, só enxerga **views agregadas** (`pacientes_agregado_empresa`, `sessoes_agregado_empresa`, `pagamentos_agregado_empresa`) — nunca a tabela `sintomas`, `resumo_sessao` ou `diagnostico_cid` diretamente. Isso é core do modelo de confiança do produto, não mexa nisso sem entender por que existe.
- **Admin**: tabela `admins` própria (não é mais e-mail fixo em policy). Promover/remover admin é só `insert`/`delete` nessa tabela.
- **Admin vendo dado clínico**: só com autorização explícita do psicólogo responsável, por paciente, com prazo de 7 dias (`autorizacoes_suporte`). Isso é deliberado — não dê ao admin acesso geral a `sessoes.resumo_sessao`/`sintomas`/`anamneses` sem essa autorização, mesmo que pareça conveniente pra alguma feature nova.
- **`login.html`** e o modal de login em `index.html` têm a **mesma lógica de detecção de papel duplicada** em dois lugares (checa admin → psicólogo → empresa → senão paciente). Se mudar a lógica de redirecionamento, mude nos dois arquivos.

### Roteamento entre páginas
Não tem router nenhum — é tudo link `<a href="pagina.html?param=valor">` e `URLSearchParams` lendo query string. Perfil de psicólogo aceita tanto `?slug=` (preferido, amigável) quanto `?id=` (fallback, links antigos).

### Estilo visual
Cada arquivo tem seu próprio `<style>` com variáveis CSS parecidas mas não centralizadas (`--navy`, `--sage`, `--bg`, `--surface`, etc.) — os valores exatos variam ligeiramente entre arquivos porque foram escritos em momentos diferentes. Ao criar uma tela nova, copie o `:root` de um arquivo existente parecido (dashboard → dashboard, página pública → página pública) em vez de inventar paleta nova.

---

## Coisas que já foram corrigidas — não reintroduza

Esse projeto teve um padrão recorrente de "front-end escrito assumindo uma coluna/tabela que nunca foi criada no banco". Se você adicionar um campo novo em algum formulário ou tela, **sempre crie a migração correspondente na mesma tarefa** — não deixe pra depois. Já aconteceu isso quebrar silenciosamente várias vezes (anamnese, hipótese diagnóstica, avaliações, endereço, idiomas...).

Outros bugs específicos já corrigidos, não reintroduzir:
- XSS via `innerHTML` sem escape (corrigido em ~70 pontos — ver acima).
- Nomes de coluna inconsistentes entre telas (ex: `data_registro` vs `data`, `status` vs `status_pagamento`) — sempre confira o nome real da coluna na migração mais recente que a criou, não assuma pelo nome "óbvio".
- E-mail de confirmação/lembrete "fire and forget" funciona com `perform net.http_post(...)` simples; **qualquer coisa que precise ler o corpo da resposta exige o padrão de duas etapas** (ver seção pg_net acima).

---

## Como eu (IA) trabalho neste repo, na prática

1. O repositório é **público** em `github.com/micaelsonnen/Puzzle` — eu consigo ler arquivos direto de lá via busca/fetch antes de editar, pra garantir que estou partindo da versão mais atual.
2. Eu **não tenho** permissão de commit direta (decisão consciente do dono do projeto, dado que é dado de saúde sensível). O fluxo é: eu leio o arquivo atual → você me diz o que precisa mudar → eu devolvo o arquivo completo editado → você salva localmente e comita/publica pelo GitHub Desktop.
3. Eu não tenho acesso ao banco Supabase. Qualquer teste de SQL, migração, ou consulta precisa ser rodado por você e o resultado colado de volta pra mim.
4. Sempre que eu terminar de editar um arquivo, rodo uma checagem automática (todo `onclick` tem função correspondente, todo `getElementById` aponta pra um `id` que existe) antes de te devolver — mas isso não substitui testar de verdade no navegador.

---

## Setup externo pendente (não é código, mas afeta o que funciona)

- **Daily.co** (videochamada): precisa de cartão cadastrado na conta pra funcionar de verdade, mesmo no plano gratuito.
- **Resend** (e-mail): usando domínio de teste (`onboarding@resend.dev`) até verificar um domínio próprio — só entrega e-mail pro e-mail cadastrado na conta Resend até isso ser resolvido.
- Ver `README.md`, seções 7 e 8, pra lista completa de pendências (técnicas e de negócio).
