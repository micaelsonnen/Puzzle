# Mind — Plataforma de Psicoterapia Acessível

> HealthTech com financiamento tripartite (empresa + paciente + psicólogo) e prontuário estruturado sem IA. Stack: HTML/CSS/JS vanilla + Supabase (Postgres + Auth + Storage) + Chart.js + jsPDF.

Este README documenta o estado do projeto após a rodada de implementação registrada nesta conversa. Serve como ponto de partida para quem continuar o desenvolvimento.

---

## 1. Modelo de negócio (resumo)

- **Empresa** patrocina parte da sessão (R$40) via programa ESG, sem acesso a nenhum dado clínico individual — só indicadores agregados.
- **Paciente** paga coparticipação (R$10/sessão).
- **Psicólogo** recebe R$50/sessão e paga mensalidade (R$120) pela plataforma.
- **Prontuário inteligente** = estruturação dos registros feitos pelo próprio psicólogo (campos, categorias, linha do tempo). **Sem IA, sem NLP, sem processamento automático** — isso é uma decisão de produto deliberada, reforçada na política de privacidade.

---

## 2. Estrutura de páginas

| Página | Público | Função |
|---|---|---|
| `index.html` | Todos | Landing page, modal de login |
| `login.html` | Todos | Login único (detecta o papel pelo e-mail) |
| `cadastro-paciente.html` | Paciente | Autocadastro / ativação de conta |
| `cadastro-psicologo.html` | Psicólogo | Cadastro completo (4 abas: pessoal, profissional, atendimento, financeiro) + upload de documentos |
| `cadastro-status.html` | Psicólogo | Status da análise do cadastro (pendente/aprovado/rejeitado) |
| `contratos.html` / `contrato-prestacao-servicos.html` | Psicólogo | Contrato de prestação de serviço |
| `psicologos.html` | Público | Vitrine/listagem de psicólogos |
| `psicologo.html` | Público | Perfil público + agendamento real |
| `novo-paciente.html` | Psicólogo | Cadastro manual de paciente (com anamnese) |
| `paciente.html` | Psicólogo | Ficha clínica completa do paciente |
| `dashboard-paciente.html` | Paciente | Progresso, linha do tempo, sessões, pagamento |
| `dashboard-psicologo.html` | Psicólogo | Agenda, solicitações, pacientes, financeiro, alertas |
| `plano.html` | Psicólogo | Assinatura da plataforma |
| `empresa.html` | Empresa | Dashboard agregado + relatórios em PDF |
| `admin.html` | Admin (e-mail fixo) | Aprovação de psicólogo + fila de exclusão LGPD |
| `meus-dados.html` | Todos | Autoatendimento LGPD (baixar dados / pedir exclusão) |
| `politica-de-privacidade.html`, `termos-de-uso.html` | Público | Documentos legais |

---

## 3. Migrações SQL — rodar NESTA ORDEM no SQL Editor do Supabase

| # | Arquivo | O que faz |
|---|---|---|
| 1 | `migration_seguranca_critica.sql` | RLS em todas as tabelas + views agregadas anônimas para empresa |
| 2 | `migration_paciente_selfservice.sql` | Paciente pode criar a própria ficha |
| 3 | `migration_prontuario_inteligente.sql` | Cria `anamneses`, `hipoteses_diagnosticas`, `intercorrencias` + colunas de funcionamento em `sintomas` |
| 4 | `migration_visibilidade_intercorrencia.sql` | Campo `visivel_paciente` — intercorrência nasce privada por padrão |
| 5 | `migration_varredura_geral.sql` | Cria `avaliacoes`, `mensalidades_psicologos` + ~25 colunas que faltavam em `psicologos` |
| 6 | `seed_dados_fantasia.sql` | *(opcional)* popula dados fictícios ligados a uma conta real existente |
| 7 | `seed_login_demo.sql` | *(opcional)* trio de demo com login: empresa + psicóloga + paciente |
| 8 | `migration_admin_aprovacao.sql` | Bucket privado de documentos + policies de admin (**troque o e-mail admin antes de rodar**) |
| 9 | `migration_exclusao_lgpd.sql` | Fila de solicitações de exclusão de conta |
| 10 | `migration_lembretes_email.sql` | Lembrete de sessão (24h antes) + confirmação de agendamento via Resend (**exige setup manual — ver seção 6**) |

⚠️ Todas são idempotentes na parte de schema (`if not exists` / `drop policy if exists`), exceto os **seeds (6 e 7)** — rodar duas vezes duplica dados fictícios.

---

## 4. O que foi implementado nesta rodada

### Correções de bugs críticos (recuperação de funcionalidade)
- **Paciente não conseguia logar de jeito nenhum** — nenhum paciente cadastrado tinha conta de autenticação. Resolvido com `cadastro-paciente.html` (cobre tanto ativação de conta pré-cadastrada quanto autocadastro).
- **Agendamento não verificava conflito de horário nem disponibilidade real** — reescrito para checar `agendamentos` + `sessoes` antes de mostrar/confirmar horário.
- **Solicitação de agendamento nunca virava sessão de verdade** — `dashboard-psicologo.html` agora tem painel de aceitar/recusar.
- **`anamneses`, `hipoteses_diagnosticas`, `intercorrencias`, `avaliacoes`, `mensalidades_psicologos`** — tabelas inteiras que o front-end já usava mas nunca existiam no schema.
- **`disponibilidade` do psicólogo nunca era preenchida** — não havia campo no formulário de cadastro; sem isso, o agendamento sempre caía no fallback genérico.
- **Bug de nomenclatura**: `dashboard-psicologo.html` consultava colunas erradas (`data_registro`, gravidade `media`/`alta`) que nunca existiram — corrigido para os nomes reais (`data`, `moderada`/`grave`).
- **Bug visual**: `psicologos.html` tinha uma expressão JS (`${Array(6).fill(...)}`) colada direto no HTML estático, fora de `<script>` — aparecia como texto literal na tela antes dos cards carregarem.

### Funcionalidades novas
- **Prontuário inteligente completo**: anamnese editável, hipótese diagnóstica versionada (histórico preservado, nunca sobrescrito), linha do tempo consolidada (sessões + hipóteses + intercorrências) — visível tanto para o psicólogo (completa) quanto para o paciente (somente leitura, com nota interna vs. compartilhada).
- **Relatórios da empresa em PDF de verdade** (jsPDF): impacto semestral, extrato financeiro, adesão por departamento, carta de privacidade — todos usando só as views agregadas, nunca dado clínico.
- **Painel de admin** (`admin.html`): aprovação de cadastro de psicólogo com visualização segura de documentos (signed URL, 5 min de validade) + fila de solicitações de exclusão LGPD.
- **Autoatendimento LGPD** (`meus-dados.html`): exportar dados em `.json`, solicitar exclusão (fila revisada por humano, por causa da retenção obrigatória de prontuário).
- **Lembretes por e-mail** via Resend: confirmação imediata quando o psicólogo aceita um agendamento + lembrete 24h antes da sessão (via `pg_cron` + `pg_net`, chave guardada no Vault do Supabase).

### Segurança
- **XSS corrigido em 8 arquivos** (~70 pontos): função `escapeHtml()` aplicada em todo campo de texto vindo do usuário antes de qualquer `innerHTML`. Página pública do psicólogo (`psicologo.html`) era o maior risco, por ser vista por visitantes não-autenticados.
- **Bucket de documentos (`psicologos-documentos`) agora é privado**, com policy por pasta (`auth.uid()`) + acesso de admin via signed URL.
- **Chave `service_role` nunca exposta** no front-end — verificado, só a `anon`/`publishable` key aparece nos arquivos (isso é esperado e seguro).
- **Chave da API do Resend armazenada no Vault** do Supabase, nunca em texto puro em nenhuma tabela ou função (funções SQL são legíveis por qualquer usuário autenticado por padrão).
- **Contrato de prestação de serviço revisado** contra o Art. 7º da Resolução CFP nº 09/2024 — cláusula de foro e descrição do meio tecnológico da sessão estavam ausentes; adicionados como placeholder explícito para revisão jurídica.

---

## 5. Contas de demonstração (se rodou `seed_login_demo.sql`)

| Papel | E-mail | Senha |
|---|---|---|
| Empresa | `empresa.demo-mind@example.com` | `MindDemo123!` |
| Psicóloga | `psicologa.demo-mind@example.com` | `MindDemo123!` |
| Paciente | `paciente.demo-mind@example.com` | `MindDemo123!` |

Admin (acesso a `admin.html`): `micaelsonnen@gmail.com` (definido nas policies da migração 8 — trocar lá se mudar de responsável).

---

## 6. Setup pendente fora do código (você precisa fazer manualmente)

- [ ] **Resend**: criar conta, verificar domínio de envio, gerar API key, salvar no Vault (`vault.create_secret(...)` ou pela tela de Vault do painel), trocar o remetente `onboarding@resend.dev` pelo domínio verificado em `migration_lembretes_email.sql` antes de rodar em produção.
- [ ] **Rate limiting**: conferir em Authentication → Rate Limits no painel do Supabase (tentativas de login e de cadastro por hora).
- [ ] **Backup**: conferir plano/retenção em Database → Backups; considerar upgrade para Pro (Point-in-Time Recovery) antes de ter dado clínico real em produção.
- [ ] **Confirmação de e-mail no Supabase Auth**: confirmar se "Confirm email" está ligado ou desligado (Authentication → Sign In / Providers) — os fluxos de cadastro assumem que `signUp()` retorna sessão utilizável na hora.

---

## 7. Pendências de negócio (fora do escopo de código)

- **Processamento de pagamento real com split** (empresa/paciente/psicólogo) — hoje os valores são só registrados no banco, não há integração com gateway. Combinado ficar para depois da abertura do CNPJ/conta PJ.
- **CNPJ, estrutura societária, contador** — fora do escopo técnico.
- **Revisão jurídica final** do contrato de prestação de serviço, política de privacidade e termos de uso (ambos já têm citações corretas da Resolução CFP nº 001/2009 e nº 006/2019 sobre retenção de prontuário — verificado, mas advogado deve validar o texto completo).
- **RIPD (Relatório de Impacto à Proteção de Dados)** e nomeação formal de Encarregado (DPO) — documentos/decisões de negócio, não código.

---

## 8. Modelo de dados (tabelas principais)

`psicologos` · `pacientes` · `empresas` · `sessoes` · `sintomas` · `pagamentos` · `agendamentos` · `anamneses` · `hipoteses_diagnosticas` · `intercorrencias` · `avaliacoes` · `mensalidades_psicologos` · `solicitacoes_exclusao`

Todas com RLS habilitado. Relação de acesso: psicólogo vê/edita seus próprios pacientes; paciente vê (e em alguns casos edita) só a própria ficha; empresa só vê views agregadas e anônimas (`pacientes_agregado_empresa`, `sessoes_agregado_empresa`, `pagamentos_agregado_empresa`) — nunca a tabela `sintomas`, `resumo_sessao` ou `diagnostico_cid` diretamente.
