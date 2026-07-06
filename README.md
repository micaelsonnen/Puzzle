# Mind — Psicoterapia acessível para quem realmente precisa

Plataforma HealthTech que democratiza o acesso à psicoterapia por meio de um
modelo de **financiamento compartilhado** entre empresas patrocinadoras,
psicólogos e pacientes, aliado a um **prontuário estruturado** que organiza
a evolução clínica ao longo do tratamento — sem uso de inteligência artificial
sobre o conteúdo das sessões.

> Projeto candidato ao Centelha. Ver `Projeto.pdf` (briefing original) para o
> racional completo de negócio, impacto social e modelo financeiro.

## Stack

- HTML + CSS + JavaScript puro (sem build step, sem framework)
- [Supabase](https://supabase.com) (Postgres + Auth + Storage) como backend
- [Chart.js](https://www.chartjs.org/) para os gráficos de evolução

Não há processo de build: qualquer arquivo `.html` pode ser aberto
diretamente ou servido como estático (ex.: Vercel, Netlify, GitHub Pages).

## Modelo de negócio (resumo)

Por sessão realizada com paciente patrocinado:

| Quem paga | Valor |
|---|---|
| Empresa patrocinadora | R$ 40,00 |
| Paciente (coparticipação) | R$ 10,00 |
| **Psicólogo recebe** | **R$ 50,00** |

O psicólogo paga à plataforma uma mensalidade fixa de **R$ 120,00**,
independente do número de sessões.

## Estrutura de páginas

### Público (sem login)
| Arquivo | Descrição |
|---|---|
| `index.html` | Landing page, carrossel de psicólogos, modal de login |
| `psicologos.html` | Listagem/busca de psicólogos, com filtros e nota média |
| `psicologo.html` | Perfil público do psicólogo, avaliações, agendamento |
| `login.html` | Login (redireciona por papel: psicólogo / empresa / paciente) |
| `cadastro-psicologo.html` | Onboarding do psicólogo em 4 abas (Pessoal, Profissional, Atendimento, Financeiro) |
| `termos-de-uso.html`, `politica-de-privacidade.html`, `contrato-prestacao-servicos.html` | Documentos legais (ver seção própria abaixo) |

### Psicólogo (autenticado)
| Arquivo | Descrição |
|---|---|
| `dashboard-psicologo.html` | Visão geral, agenda, pacientes, financeiro, alertas clínicos |
| `paciente.html` | Ficha do paciente: anamnese, hipótese diagnóstica (com histórico), linha do tempo, sessões, evolução |
| `novo-paciente.html` | Cadastro de paciente novo + anamnese inicial |
| `plano.html` | Gestão de assinatura (mensal/anual) e histórico de pagamento da mensalidade |
| `contratos.html` | Status de aceite dos documentos legais |
| `cadastro-status.html` | Tela de espera enquanto o CRP está em validação |

### Paciente (autenticado)
| Arquivo | Descrição |
|---|---|
| `dashboard-paciente.html` | Progresso (sintomas + funcionamento), sessões, financeiro, check-in emocional |

### Empresa (autenticada)
| Arquivo | Descrição |
|---|---|
| `empresa.html` | Indicadores **agregados e anonimizados** (nunca dado clínico individual) |

## Banco de dados — ordem de execução das migrações

As migrações estão na raiz do repo (`migration_*.sql`) e devem ser rodadas
**nesta ordem** no SQL Editor do Supabase, pois migrações posteriores
dependem de colunas/tabelas criadas pelas anteriores:

0. **`migration_0_schema_base.sql`** — roda primeiro, sempre. Garante que
   colunas que o front-end sempre assumiu existir (ex.: `psicologos.abordagem`,
   `psicologos.bio`, `psicologos.disponibilidade`) realmente existem no banco.
   Sem isso, as migrações seguintes (que criam views referenciando essas
   colunas) falham com erro `column "..." does not exist`.
1. `migration_prontuario_inteligente.sql` — anamnese, hipótese diagnóstica
   (com histórico), intercorrências, extensão de `sintomas`
2. `migration_cadastro_psicologo.sql` — onboarding completo do psicólogo
   (dados pessoais, CRP, documentos, financeiro) + bucket de Storage
3. `migration_plano_psicologo.sql` — assinatura/mensalidade da plataforma
4. `migration_avaliacoes.sql` — avaliações públicas de pacientes sobre psicólogos
5. **`migration_seguranca_critica.sql`** — **rodar sempre por último**, e
   novamente sempre que uma tabela ou coluna nova for criada. Habilita Row
   Level Security em todas as tabelas e cria as views anonimizadas usadas
   pelas telas públicas e pela empresa.

Todas as migrações são seguras para rodar mais de uma vez (`if not exists` /
`drop policy if exists` antes de recriar).

## Segurança — leia antes de mexer em qualquer tabela

A chave `SUPABASE_KEY` usada no front-end é a chave **anon**, pública por
design — ela fica visível no HTML e isso é esperado. **Quem protege os dados
de verdade é o Row Level Security (RLS) de cada tabela.** Toda vez que uma
tabela nova for criada:

1. Habilite RLS: `alter table <tabela> enable row level security;`
2. Escreva policies explícitas (nada de tabela sem RLS "pra testar depois" —
   sem RLS, a tabela fica 100% legível/gravável por qualquer pessoa com a
   anon key, sem precisar de login).
3. Se a tabela tiver dado que empresas podem precisar em agregado, crie uma
   **view** anonimizada (ver `pacientes_agregado_empresa`,
   `sessoes_agregado_empresa`, `pagamentos_agregado_empresa` e
   `psicologos_publico` como exemplos) em vez de dar policy direta pra
   empresa na tabela base.

Regra de ouro do produto: **empresa patrocinadora nunca vê nome, diagnóstico,
resumo de sessão ou qualquer dado clínico individual** — só números agregados
(nº de atendimentos, taxa de permanência, etc.). Isso é compromisso contratual
(ver `contrato-prestacao-servicos.html`) e requisito de LGPD, não só boa prática.

## Documentos legais

`termos-de-uso.html`, `politica-de-privacidade.html` e
`contrato-prestacao-servicos.html` (+ versões `.docx` geradas junto) são
**minutas estruturadas**, não pareceres jurídicos. Antes de publicar de
verdade:

- Preencher os campos `[ENTRE COLCHETES]` (razão social, CNPJ, endereço,
  e-mail do DPO, foro, prazos de repasse).
- Revisão por advogado(a) — especialmente as cláusulas de responsabilidade
  civil e a definição de quem é controlador/operador dos dados (hoje: o
  psicólogo é controlador do prontuário, a Mind é operadora da infraestrutura).

## Mobile

Todas as páginas têm `@media` para telas ≤ 900px (sidebars viram barra
horizontal, grids colapsam para 1 coluna, tabelas ganham scroll horizontal).
`index.html` tem menu hambúrguer para o nav principal.

## Pendências conhecidas (não bloqueantes)

- **Painel administrativo** para a equipe Mind aprovar/rejeitar cadastro de
  psicólogo (CRP). Hoje isso só pode ser feito manualmente no Supabase
  (`update psicologos set status_cadastro = 'aprovado'`).
- **Relatório em PDF** para empresa (hoje o botão "Gerar PDF" é um `alert()`).
- **Agendamento real**: horários hoje são fixos (9 slots), sem checar conflito
  real de agenda do psicólogo.
- Onboarding de **paciente** e de **empresa** ainda não existe como fluxo de
  autocadastro (paciente é criado pelo psicólogo via `novo-paciente.html`).
