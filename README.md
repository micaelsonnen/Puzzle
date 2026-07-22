# Puzzle — Plataforma de Psicoterapia Acessível

[#puzzle--plataforma-de-psicoterapia-acessível](#puzzle--plataforma-de-psicoterapia-acessível)
> HealthTech com financiamento tripartite (empresa + paciente + psicólogo) e prontuário estruturado sem IA. Stack: HTML/CSS/JS vanilla + Supabase (Postgres + Auth + Storage) + Chart.js + jsPDF.

Este README documenta o estado do projeto após as rodadas de implementação registradas nesta conversa. Serve como ponto de partida para quem continuar o desenvolvimento.

---

## 1. Modelo de negócio (resumo)

[#1-modelo-de-negócio-resumo](#1-modelo-de-negócio-resumo)

- **Empresa** patrocina parte da sessão (R$40) via programa ESG, sem acesso a nenhum dado clínico individual — só indicadores agregados.
- **Paciente** paga coparticipação (R$10/sessão) quando tem direito ao valor social; caso contrário, paga o valor integral.
- **Psicólogo** recebe R$50/sessão e paga mensalidade (R$120) pela plataforma.
- **Elegibilidade ao valor social não é automática por vínculo com empresa** — desde a rodada mais recente, existe um questionário de vulnerabilidade socioeconômica com pontuação automática (ver seção 9) que decide quem tem direito, e a empresa contrata **pacotes de vagas** que sustentam esse valor social para um número limitado de pacientes por vez.
- **Prontuário inteligente** = estruturação dos registros feitos pelo próprio psicólogo (campos, categorias, linha do tempo). **Sem IA, sem NLP, sem processamento automático** — isso é uma decisão de produto deliberada, reforçada na política de privacidade.

---

## 2. Estrutura de páginas

[#2-estrutura-de-páginas](#2-estrutura-de-páginas)

| Página                                                | Público             | Função                                                                                            |
| ----------------------------------------------------- | ------------------- | ------------------------------------------------------------------------------------------------- |
| `index.html`                                          | Todos               | Landing page, modal de login                                                                      |
| `login.html`                                          | Todos               | Login único (detecta o papel pelo e-mail)                                                         |
| `cadastro-paciente.html`                              | Paciente            | Autocadastro / ativação de conta                                                                  |
| `cadastro-psicologo.html`                             | Psicólogo           | Cadastro completo (4 abas: pessoal, profissional, atendimento, financeiro) + upload de documentos |
| `cadastro-status.html`                                | Psicólogo           | Status da análise do cadastro (pendente/aprovado/rejeitado)                                       |
| `contratos.html` / `contrato-prestacao-servicos.html` | Psicólogo           | Contrato de prestação de serviço                                                                  |
| `psicologos.html`                                     | Público             | Vitrine/listagem de psicólogos                                                                    |
| `psicologo.html`                                      | Público             | Perfil público + agendamento real                                                                 |
| `novo-paciente.html`                                  | Psicólogo           | Cadastro manual de paciente (com anamnese)                                                        |
| `paciente.html`                                       | Psicólogo           | Ficha clínica completa do paciente                                                                |
| `dashboard-paciente.html`                             | Paciente            | Progresso, linha do tempo, sessões, pagamento                                                     |
| `dashboard-psicologo.html`                            | Psicólogo           | Agenda, solicitações, pacientes, financeiro, alertas                                              |
| `plano.html`                                          | Psicólogo           | Assinatura da plataforma                                                                          |
| `empresa.html`                                        | Empresa             | Dashboard agregado + relatórios em PDF                                                            |
| `avaliacao-vulnerabilidade.html`                      | Paciente            | Questionário socioeconômico + upload de comprovantes para valor social                            |
| `admin.html`                                          | Admin               | Aprovação de psicólogo + fila de exclusão LGPD                                                    |
| `admin-revisao-vulnerabilidade.html`                  | Admin               | Fila de revisão manual das avaliações de vulnerabilidade + aprovação/rejeição                     |
| `admin-pacotes-empresa.html`                          | Admin               | Cadastro e gestão de pacotes de vagas sociais por empresa                                         |
| `meus-dados.html`                                     | Todos               | Autoatendimento LGPD (baixar dados / pedir exclusão)                                              |
| `politica-de-privacidade.html`, `termos-de-uso.html`  | Público             | Documentos legais                                                                                 |

O acesso de admin é verificado contra uma tabela `admins` de verdade (`email` cadastrado), não um e-mail fixo hardcoded — para dar acesso a alguém novo, basta adicionar uma linha nessa tabela.

---

## 3. Migrações SQL — rodar NESTA ORDEM no SQL Editor do Supabase

[#3-migrações-sql--rodar-nesta-ordem-no-sql-editor-do-supabase](#3-migrações-sql--rodar-nesta-ordem-no-sql-editor-do-supabase)

| #  | Arquivo                                     | O que faz                                                                                                     |
| --- | ------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| 1  | `migration_seguranca_critica.sql`           | RLS em todas as tabelas + views agregadas anônimas para empresa                                               |
| 2  | `migration_paciente_selfservice.sql`        | Paciente pode criar a própria ficha                                                                           |
| 3  | `migration_prontuario_inteligente.sql`      | Cria `anamneses`, `hipoteses_diagnosticas`, `intercorrencias` + colunas de funcionamento em `sintomas`        |
| 4  | `migration_visibilidade_intercorrencia.sql` | Campo `visivel_paciente` — intercorrência nasce privada por padrão                                            |
| 5  | `migration_varredura_geral.sql`             | Cria `avaliacoes`, `mensalidades_psicologos` + ~25 colunas que faltavam em `psicologos`                       |
| 6  | `seed_dados_fantasia.sql`                   | *(opcional)* popula dados fictícios ligados a uma conta real existente                                        |
| 7  | `seed_login_demo.sql`                       | *(opcional)* trio de demo com login: empresa + psicóloga + paciente                                           |
| 8  | `migration_admin_aprovacao.sql`             | Bucket privado de documentos + policies de admin                                                              |
| 9  | `migration_exclusao_lgpd.sql`               | Fila de solicitações de exclusão de conta                                                                     |
| 10 | `migration_lembretes_email.sql`             | Lembrete de sessão (24h antes) + confirmação de agendamento via Resend (**exige setup manual — ver seção 6**) |
| 11 | `migration_vulnerabilidade_social.sql`      | Questionário socioeconômico: tabelas, trigger de pontuação automática, RLS, bucket de comprovantes            |
| 12 | `migration_precificacao_social.sql`         | Tabela `config_precos_sessao` (valores editáveis) + função `paciente_tem_vulnerabilidade_valida`               |
| 13 | `migration_vaga_social_pacientes.sql`       | Pacotes de vaga por empresa + lógica que decide `pacientes.valor_paciente` automaticamente                    |

⚠️ Todas são idempotentes na parte de schema (`if not exists` / `drop policy if exists`), exceto os **seeds (6 e 7)** — rodar duas vezes duplica dados fictícios.

⚠️ **`migration_pacotes_empresa.sql` está obsoleta** — foi uma primeira tentativa da funcionalidade de pacotes que partia de uma premissa errada (recalcular o subsídio a cada sessão, em vez de reservar a vaga uma vez na aprovação). Se esse arquivo existir no repositório, não rode — `migration_vaga_social_pacientes.sql` a substitui inteira e já contém os `DROP` necessários pra desfazer, caso a antiga já tenha sido aplicada.

Scripts auxiliares (não fazem parte da sequência de migrations, rodar sob demanda):

| Arquivo | Quando rodar |
|---|---|
| `reprocessamento_retroativo.sql` | Só se já existiam avaliações aprovadas no banco antes de `migration_vaga_social_pacientes.sql` existir |
| `agendar_liberacao_vagas.sql` | Uma vez, para agendar via `pg_cron` a liberação diária de vagas de pacotes expirados |

---

## 4. O que foi implementado nesta rodada

[#4-o-que-foi-implementado-nesta-rodada](#4-o-que-foi-implementado-nesta-rodada)

### Correções de bugs críticos (recuperação de funcionalidade)

[#correções-de-bugs-críticos-recuperação-de-funcionalidade](#correções-de-bugs-críticos-recuperação-de-funcionalidade)

- **Paciente não conseguia logar de jeito nenhum** — nenhum paciente cadastrado tinha conta de autenticação. Resolvido com `cadastro-paciente.html` (cobre tanto ativação de conta pré-cadastrada quanto autocadastro).
- **Agendamento não verificava conflito de horário nem disponibilidade real** — reescrito para checar `agendamentos` + `sessoes` antes de mostrar/confirmar horário.
- **Solicitação de agendamento nunca virava sessão de verdade** — `dashboard-psicologo.html` agora tem painel de aceitar/recusar.
- **`anamneses`, `hipoteses_diagnosticas`, `intercorrencias`, `avaliacoes`, `mensalidades_psicologos`** — tabelas inteiras que o front-end já usava mas nunca existiam no schema.
- **`disponibilidade` do psicólogo nunca era preenchida** — não havia campo no formulário de cadastro; sem isso, o agendamento sempre caía no fallback genérico.
- **Bug de nomenclatura**: `dashboard-psicologo.html` consultava colunas erradas (`data_registro`, gravidade `media`/`alta`) que nunca existiram — corrigido para os nomes reais (`data`, `moderada`/`grave`).
- **Bug visual**: `psicologos.html` tinha uma expressão JS (`${Array(6).fill(...)}`) colada direto no HTML estático, fora de `<script>` — aparecia como texto literal na tela antes dos cards carregarem.

### Funcionalidades novas

[#funcionalidades-novas](#funcionalidades-novas)

- **Prontuário inteligente completo**: anamnese editável, hipótese diagnóstica versionada (histórico preservado, nunca sobrescrito), linha do tempo consolidada (sessões + hipóteses + intercorrências) — visível tanto para o psicólogo (completa) quanto para o paciente (somente leitura, com nota interna vs. compartilhada).
- **Relatórios da empresa em PDF de verdade** (jsPDF): impacto semestral, extrato financeiro, adesão por departamento, carta de privacidade — todos usando só as views agregadas, nunca dado clínico.
- **Painel de admin** (`admin.html`): aprovação de cadastro de psicólogo com visualização segura de documentos (signed URL, 5 min de validade) + fila de solicitações de exclusão LGPD.
- **Autoatendimento LGPD** (`meus-dados.html`): exportar dados em `.json`, solicitar exclusão (fila revisada por humano, por causa da retenção obrigatória de prontuário).
- **Lembretes por e-mail** via Resend: confirmação imediata quando o psicólogo aceita um agendamento + lembrete 24h antes da sessão (via `pg_cron` + `pg_net`, chave guardada no Vault do Supabase).
- **Vulnerabilidade social e vagas por empresa** (ver seção 9 para detalhes): questionário com pontuação automática, painel de revisão manual, pacotes de vaga por empresa com alocação atômica e fila de espera.

### Segurança

[#segurança](#segurança)

- **XSS corrigido em 8 arquivos** (~70 pontos): função `escapeHtml()` aplicada em todo campo de texto vindo do usuário antes de qualquer `innerHTML`. Página pública do psicólogo (`psicologo.html`) era o maior risco, por ser vista por visitantes não-autenticados.
- **Bucket de documentos (`psicologos-documentos`) agora é privado**, com policy por pasta (`auth.uid()`) + acesso de admin via signed URL. O mesmo padrão foi replicado no bucket `comprovantes-vulnerabilidade`.
- **Chave `service_role` nunca exposta** no front-end — verificado, só a `anon`/`publishable` key aparece nos arquivos (isso é esperado e seguro).
- **Chave da API do Resend armazenada no Vault** do Supabase, nunca em texto puro em nenhuma tabela ou função (funções SQL são legíveis por qualquer usuário autenticado por padrão).
- **Acesso de admin verificado contra tabela real** (`admins`), não e-mail fixo hardcoded nas policies — todas as migrations de vulnerabilidade social usam `exists (select 1 from admins where email = auth.jwt() ->> 'email')`.
- **Contrato de prestação de serviço revisado** contra o Art. 7º da Resolução CFP nº 09/2024 — cláusula de foro e descrição do meio tecnológico da sessão estavam ausentes; adicionados como placeholder explícito para revisão jurídica.

---

## 5. Contas de demonstração (se rodou `seed_login_demo.sql`)

[#5-contas-de-demonstração-se-rodou-seed_login_demosql](#5-contas-de-demonstração-se-rodou-seed_login_demosql)

| Papel     | E-mail                            | Senha          |
| --------- | --------------------------------- | -------------- |
| Empresa   | `empresa.demo-mind@example.com`   | `MindDemo123!` |
| Psicóloga | `psicologa.demo-mind@example.com` | `MindDemo123!` |
| Paciente  | `paciente.demo-mind@example.com`  | `MindDemo123!` |

Admin (acesso a `admin.html` e às telas de vulnerabilidade social): qualquer e-mail cadastrado na tabela `admins`.

*Nota: as credenciais de demo acima usam "mind" no e-mail/senha porque foram semeadas (`seed_login_demo.sql`) antes do nome do produto ser definido como Puzzle — são só identificadores, não afetam a marca.*

---

## 6. Setup pendente fora do código (você precisa fazer manualmente)

[#6-setup-pendente-fora-do-código-você-precisa-fazer-manualmente](#6-setup-pendente-fora-do-código-você-precisa-fazer-manualmente)

- [ ] **Resend**: criar conta, verificar domínio de envio, gerar API key, salvar no Vault (`vault.create_secret(...)` ou pela tela de Vault do painel), trocar o remetente `onboarding@resend.dev` pelo domínio verificado em `migration_lembretes_email.sql` antes de rodar em produção.
- [ ] **Rate limiting**: conferir em Authentication → Rate Limits no painel do Supabase (tentativas de login e de cadastro por hora).
- [ ] **Backup**: conferir plano/retenção em Database → Backups; considerar upgrade para Pro (Point-in-Time Recovery) antes de ter dado clínico real em produção.
- [ ] **Confirmação de e-mail no Supabase Auth**: confirmar se "Confirm email" está ligado ou desligado (Authentication → Sign In / Providers) — os fluxos de cadastro assumem que `signUp()` retorna sessão utilizável na hora.
- [x] **pg_cron para liberação de vagas expiradas**: job `liberar-vagas-pacotes-expirados` agendado para rodar todo dia às 3h (`agendar_liberacao_vagas.sql`).

---

## 7. Pendências de negócio (fora do escopo de código)

[#7-pendências-de-negócio-fora-do-escopo-de-código](#7-pendências-de-negócio-fora-do-escopo-de-código)

- **Processamento de pagamento real com gateway** — os valores de cada sessão (paciente/empresa/psicólogo) já são calculados e registrados automaticamente no banco (ver seção 9), mas ainda não há integração com um gateway de pagamento de verdade (Stripe, Pagar.me, Asaas etc.) que efetivamente cobre e repasse o dinheiro. Combinado ficar para depois da abertura do CNPJ/conta PJ.
- **CNPJ, estrutura societária, contador** — fora do escopo técnico.
- **Revisão jurídica final** do contrato de prestação de serviço, política de privacidade e termos de uso (ambos já têm citações corretas da Resolução CFP nº 001/2009 e nº 006/2019 sobre retenção de prontuário — verificado, mas advogado deve validar o texto completo).
- **RIPD (Relatório de Impacto à Proteção de Dados)** e nomeação formal de Encarregado (DPO) — decisões de negócio, não código. Ficou ainda mais relevante com a adição do questionário de vulnerabilidade, que coleta dados sensíveis (renda, benefícios sociais) além dos dados clínicos.
- **Auditoria amostral das aprovações automáticas de vulnerabilidade** — hoje qualquer pontuação acima do limite libera automaticamente; vale ter revisão manual de uma amostra aleatória para evitar que alguém preencha o questionário só para bater o limite de pontos.
- **Revalidação periódica da aprovação de vulnerabilidade** — existe uma janela de validade configurável (`validade_avaliacao_meses`, padrão 6 meses) mas nada notifica o paciente para refazer o questionário quando expira.

---

## 8. Modelo de dados (tabelas principais)

[#8-modelo-de-dados-tabelas-principais](#8-modelo-de-dados-tabelas-principais)

`psicologos` · `pacientes` · `empresas` · `sessoes` · `sintomas` · `pagamentos` · `agendamentos` · `anamneses` · `hipoteses_diagnosticas` · `intercorrencias` · `avaliacoes` · `mensalidades_psicologos` · `solicitacoes_exclusao` · `admins` · `avaliacoes_socioeconomicas` · `comprovantes_vulnerabilidade` · `config_pontuacao_vulnerabilidade` · `config_precos_sessao` · `pacotes_empresa`

Todas com RLS habilitado. Relação de acesso: psicólogo vê/edita seus próprios pacientes; paciente vê (e em alguns casos edita) só a própria ficha; empresa só vê views agregadas e anônimas (`pacientes_agregado_empresa`, `sessoes_agregado_empresa`, `pagamentos_agregado_empresa`, `resumo_pacotes_empresa`) — nunca a tabela `sintomas`, `resumo_sessao` ou `diagnostico_cid` diretamente. Admin acessa tudo relacionado a aprovação e revisão via policy contra a tabela `admins`.

---

## 9. Vulnerabilidade social e vagas por empresa

[#9-vulnerabilidade-social-e-vagas-por-empresa](#9-vulnerabilidade-social-e-vagas-por-empresa)

Sistema que decide, de forma automática e auditável, quem tem direito ao valor social e garante que esse valor só é concedido enquanto houver lastro financeiro (vaga contratada por uma empresa parceira).

### Fluxo

1. Paciente responde o questionário em `avaliacao-vulnerabilidade.html` (renda per capita, moradores, situação de emprego, CadÚnico, Bolsa Família, deficiência na família) e pode opcionalmente enviar comprovantes.
2. Uma trigger no banco calcula a pontuação (pesos editáveis em `config_pontuacao_vulnerabilidade`) e decide o status: `aprovado_automatico` (pontuação acima do limite), `em_revisao` (abaixo do limite, mas com comprovante enviado) ou `pendente`.
3. Casos em `em_revisao` ou `pendente` aparecem na fila de `admin-revisao-vulnerabilidade.html`, onde o admin aprova ou rejeita manualmente (rejeição exige justificativa).
4. **No momento em que o status vira aprovado** (automático ou manual), uma trigger reserva uma vaga no pacote de empresa com saldo que vence primeiro, e seta `pacientes.valor_paciente` para o valor de coparticipação social. Se não houver vaga disponível, o paciente fica com `status_financeiro_social = 'aguardando_empresa'` e continua pagando o valor vigente até uma vaga abrir.
5. Empresas contratam pacotes de vaga via `admin-pacotes-empresa.html` (cadastrado pelo admin hoje). **Importante**: a vaga é por paciente, não por sessão — um pacote de 20 vagas sustenta até 20 pacientes com valor social simultaneamente, independente de quantas sessões cada um faça.
6. Criar um pacote novo ou estender a vigência de um existente reprocessa a fila de espera automaticamente, em ordem de chegada.
7. Quando um pacote expira, o job agendado via `pg_cron` (`liberar_vagas_expiradas()`, todo dia às 3h) libera as vagas dos pacientes que ocupavam vaga nele.
8. `pagamentos` não recalcula nada — só copia o `valor_paciente` (e o subsídio da empresa, se aplicável) no momento em que a sessão é criada, como um retrato histórico para fins de auditoria financeira.

### Concorrência

A alocação de vaga usa `for update skip locked` — dois pacientes sendo aprovados ao mesmo tempo não conseguem reservar a mesma vaga; um deles cai em `aguardando_empresa` em vez de gerar uma condição de corrida.

### O que ainda não existe

- Tela para a própria empresa (não só o admin) cadastrar seus próprios pacotes.
- Edição de `quantidade_vagas` de um pacote já em uso (só criação de pacote novo e extensão de vigência estão implementadas — reduzir abaixo do que já está ocupado quebraria uma constraint de banco).
- Notificação ativa ao paciente quando sua vaga é confirmada ou quando a avaliação expira (hoje é preciso checar `pacientes.status_financeiro_social` manualmente ou via `snippet-aviso-vaga-social.html`).
