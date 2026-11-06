# OLIENT 2027

Aplicação web para a Olimpíada Interna do Colégio Novos Tempos — Unidade Betim. O frontend usa React, TypeScript e Vite; autenticação, persistência e autorização ficam no Supabase Auth/PostgreSQL com RLS. O app não contém resultados nem pontuações fictícias.

## Requisitos e execução

- Node.js 20.19+ ou 22.12+ (requisito do Vite 6).
- npm 10+ ou pnpm 9+.
- Projeto Supabase para habilitar dados e autenticação.

```sh
pnpm install
cp .env.example .env
# preencha URL e chave publicável/anon do projeto Supabase
pnpm dev
```

Sem `.env`, o shell visual funciona em estado vazio, sem inventar dados. Para verificar compilação e regras puras: `pnpm build` e `pnpm test`.

## Configuração Supabase

1. Crie um projeto Supabase sob gestão da escola e configure URL/site redirect em Auth.
2. Copie URL e chave publicável (anon) para `.env`. Nunca use `service_role` no cliente.
3. Instale Supabase CLI e execute `supabase link --project-ref <ref>` e `supabase db push`.
4. Crie o usuário inicial em Auth (convite/e-mail confirmado conforme política escolar) e entre pela tela administrativa.
5. Execute uma vez, autenticado como primeiro usuário, a RPC `setup_first_admin` com seu nome. A função só funciona enquanto não houver administrador ativo. A criação de contas de representantes é feita por uma Edge Function protegida, descrita a seguir.
6. Crie uma versão de regulamento e regras aprovadas antes de confirmar resultados. Não existem regras/valores 2027 pré-aprovados no repositório.

### Contas diretas de representantes das equipes

Para habilitar **Equipes → Acesso dos representantes → Criar conta**, aplique `202610280017_student_team_representatives.sql` e `202610280018_admin_create_representative_accounts.sql` em ordem. Depois publique a função com `supabase functions deploy admin-create-team-representative`. A função usa `SUPABASE_SERVICE_ROLE_KEY` somente no ambiente Edge do Supabase; nunca adicione essa chave ao `.env` do Vite nem ao cliente web. Cada conta é criada com e-mail confirmado, papel `viewer` e vínculo a um único aluno/equipe; a senha informada não é gravada no banco.

Migrations: `202610260001_initial_schema.sql` (entidades e índices), `202610260002_security_and_operations.sql` (funções, visão agregada, lançamentos, confirmação e reversão), `202610260003_rls_policies.sql` (RLS e grants), `202610260004_tournament_scheduler.sql` (agenda e permissões operacionais) e `202610260005_classes_students_admin.sql` (grants de escrita sujeitos às políticas RLS de administrador para turmas e alunos). Execute em ordem pela CLI; não aplique via SQL Editor em produção sem revisão institucional.

## Segurança e modelo

- Papéis: `administrator`, `organizer`, `viewer`. Promoção de perfil não pode ser feita pelo usuário comum via UPDATE; primeira promoção é protegida pela função de bootstrap, posteriormente só admin.
- RLS habilitada em tabelas. Público lê somente agregados de equipes/modalidades/publicações/resultados publicados; dados de estudantes são reservados a staff autenticado. Não use `service_role` no browser.
- `score_ledger` é a fonte para totais. Reversões criam lançamento compensatório e preservam histórico. `record_manual_score`, `confirm_match_result` e `reverse_score_entry` validam papel/regras no Postgres e usam chaves idempotentes ou relação de reversão.
- Regras aprovadas pelo regulamento são necessárias para lançamento de pontos. Critérios de desempate devem ser documentados por modalidade; o painel não apresenta uma equipe empatada como desempate regulamentar.
- Bootstrap precisa ser executado somente uma vez. Proteja credenciais de Auth, restrinja criação de contas e configure política institucional de retenção/privacidade.

## Estrutura

```text
src/components/     (componentes adicionais)
src/lib/             cliente, tipos e regra pura de classificação
src/pages/           aplicação e telas
src/services/        consultas/mutações Supabase
src/types/           tipos da aplicação
supabase/migrations/ schema, operações e RLS
tests/               testes adicionais
```

## Implantação

`pnpm build` gera `dist/`, publicável em hospedagem estática (Netlify, Vercel, Cloudflare Pages etc.). Configure as mesmas duas variáveis Vite no build. Cadastre URL pública e redirects de autenticação no Supabase. O deploy funcional depende de projeto/credenciais e configuração institucional.

## Pendências do primeiro ciclo

- URL/chave Supabase e provisionamento real de usuários não foram fornecidos.
- Escola precisa aprovar regulamento 2027, valores, categorias, regras de empate, composição das equipes e política para dados de estudantes.
- O frontend inicial dá suporte público e cadastro de equipe/modalidade; telas de CRUD de estudantes/inscrições, partidas e gincana, relatórios CSV, gestão de usuários e controles de publicação precisam ser conectadas às funções/tabelas antes do uso operacional integral.
- O bootstrap de administrador é RPC única; para produção real, recomende desativar/revogar essa RPC após uso por migration específica e implementar gestão de convites via Edge Function segura.

O projeto não declara testes de RLS, implantação, conexão Lovable nem Supabase executados sem ambiente configurado.

## Regras operacionais e calendário

Consulte [docs/PLANEJAMENTO_OPERACIONAL.md](docs/PLANEJAMENTO_OPERACIONAL.md) para o estado do motor de agendamento, prévia, requisitos de identidade dos alunos e pendências para habilitar salvamento/publicação de tabelas oficiais. O gerador atual é apenas uma prévia administrativa; não use como calendário oficial.
