# ControlDesk_Auto

# DDM Control Desk

Painel de operação (control desk) para call center de cobrança: discagem,
pausas, performance de agentes, tabulações, compliance de horário e gestão
de time — tudo alimentado **automaticamente** por um banco espelho da OLOS,
sem upload manual de relatório.

Projeto autônomo — sem dependência de plataforma externa. Login próprio
(usuário/senha), banco MySQL, sem serviços de terceiros.

## Índice

- [O que o sistema faz](#o-que-o-sistema-faz)
- [Sincronização automática com a OLOS](#sincronização-automática-com-a-olos)
- [Papéis de usuário](#papéis-de-usuário)
- [Como rodar](#como-rodar)
- [Variáveis de ambiente](#variáveis-de-ambiente)
- [Stack técnica](#stack-técnica)
- [Estrutura do projeto](#estrutura-do-projeto)
- [Limitações conhecidas](#limitações-conhecidas)
- [Roadmap](#roadmap)

## O que o sistema faz

### Performance por Campanha (Discagem)
Funil completo — Discagens → Atendidas → CPC → CPCA → Promessa — por
organização e campanha, com:
- Quantidade de **Mailing** automática por campanha (registros distintos
  discados, via `tentativas_contato.mailing_registro_id`).
- **Qualidade Técnica**: % de ruído (caixa postal, queda, mensagem de
  operadora) por rota/campanha.
- **Conformidade de Horário**: ligações fora do horário permitido
  (antes das 8h, depois das 20h, ou domingo), por organização/campanha.
- Visão "Por Organização" e aba de referência "Assistência" (cheat-sheet
  de discador/OLOS).

### TEMPOS (Controle de Pausas / NR17)
- Limites de pausa configuráveis em runtime (sem precisar mexer em código).
- Abas por motivo (Banheiro, Feedback, Outros) e NR17 com ranking de
  ofensores — **por agente e por supervisor**, mostrando os nomes dos
  responsáveis em cada time.

### Performance em Tempo Real (AgentDay)
- Tabela completa + análise por quartil (Q1–Q4).
- **Login & Logout** e **Faltas por Carteira** (cruza com Dimensionamento).
- **Baixa Produção** — pensada pra não repetir o Quartil: mostra *onde* o
  problema se concentra (por célula/campanha e por supervisor), não só uma
  lista de nomes. Inclui:
  - **Zerados** (logados 30min+, zero chamadas), com coluna de
    **possível causa** (rota com ruído técnico alto, ou agente não
    alocado / divergente no MOP).
  - **Baixa Conversão** — agente com CPC alto mas pouco acordo/promessa.
  - Filtros de exclusão por **célula** e por **supervisor** (separados —
    útil quando um supervisor específico gerencia um time com função
    diferente, ex: BKO, que naturalmente fica zerado).
  - Limiares ajustáveis direto na tela.
  - Gráfico de **tendência** (14 dias) de % zerados e NR17 excedido.

### Tabulações
Ranking unificado + por agente + por supervisor, com filtro de tabulação
que afeta as três visões de forma consistente.

### Wallboard (TV)
Tela full-screen pra TV do operacional: KPIs do funil de discagem +
**ranking de agentes em tempo real**, com busca/filtro de campanha,
atualização automática configurável (10/15 min) e tudo salvo no navegador
(sobrevive se a TV reiniciar).

### Book Operacional
Importação do `.xlsb` mensal (BCampHC/B1-B4/B-abs), com:
- Ranking de agentes (TMA, %Ocupação, %Idle, Esforço de Discagem,
  CPC/DU, Acordo/DU, Quartil).
- Tendência mensal com cards de variação (Esforço de Discagem, TMA,
  % Ocupação) vs mês anterior.
- **Comparativo por Organização × Mês** (cruza com `discagem_records` pra
  saber a organização de cada campanha).

### MOP Operacional
Alocação de operadores em campanhas (supervisores decidem quem trabalha
onde), cruzado automaticamente com a Baixa Produção pra apontar
inconsistência (agente zerado mas não alocado, ou alocado em campanha
diferente da célula do Dimensionamento).

### Sugestões / Bugs
Abas separadas, com nome de quem reportou visível pra todo mundo.

### Dimensionamento / Canais & Rotas / Discador (referência)
Cadastro de headcount (import de Excel por cabeçalho, não por posição) e
referência técnica sobre discador/OLOS — só acesso Control/Planejamento.

### Alertas (sininho)
No topo da tela, sempre visível: agentes zerados hoje, pausas excedidas e
ligações fora do horário — sem precisar abrir cada tela pra descobrir.
Atualiza sozinho a cada 5 min, sem precisar de e-mail/Slack configurado.

### Relatório Executivo / Boletim Diário
Export em PNG/PDF com seletor de intervalo de datas. O **Boletim Diário**
junta num só lugar: funil de Discagem, Baixa Produção por Supervisor e
NR17 excedido por Supervisor — pronto pra encaminhar.

## Sincronização automática com a OLOS

O coração do sistema: uma conexão de **leitura** com o banco espelho da
OLOS substitui todo o fluxo de "baixar relatório e importar na mão".

| Automação | Fonte | Frequência |
|---|---|---|
| Discagem (funil) | `bilhetagem` + `tentativas_contato` | A cada N min (configurável) |
| Pausas (TEMPOS) | `estados_agente` | A cada N min |
| Performance em Tempo Real | `estados_agente` + `bilhetagem` | A cada N min |
| Tabulações | `tentativas_contato` | A cada N min |
| Mailing | `tentativas_contato` (COUNT DISTINCT) | Manual (mais pesado) |

Cada automação também tem um botão **"Sincronizar agora"** direto na tela
correspondente, além do painel central em **Sincronização** (status da
conexão, testar conexão, histórico do último run).

**Decisões técnicas que valem registrar:**
- Toda consulta usa **intervalo de data** (`>= / <`), nunca `DATE(coluna) =`
  — isso é o que mantém as consultas rápidas (usa o índice) mesmo com o
  histórico da OLOS crescendo pra sempre.
- Timeout de 2 min por consulta (configurável por automação) — antes
  disso, uma consulta travada ficava girando pra sempre sem erro nenhum.
- **Multi-campanha**: quando um agente está logado em mais de uma
  campanha ao mesmo tempo, o mesmo intervalo de pausa aparece
  multiplicado nas tabelas da OLOS — todas as consultas de `estados_agente`
  deduplicam isso via `ROW_NUMBER() OVER (PARTITION BY agente, início, fim)`.
- **CPC nunca pode ser maior que Atendidas**: o status técnico
  (`tabulacoes_bilhetagem`, em inglês) e o resultado de negócio
  (`tabulacoes`, em português) são sinais separados, combinados com E —
  nunca um substituindo o outro.
- Cadastro automático no Dimensionamento quando um agente aparece no
  AgentDay mas ainda não estava cadastrado.

## Papéis de usuário

| Papel | Quem | Acesso |
|---|---|---|
| **admin** | Control / Analista de Planejamento | Todas as telas, sem restrição |
| **user** | Supervisor / Coordenador de Operação | Só: Performance em Tempo Real, Login & Logout, Faltas por Carteira, TEMPOS, Performance por Campanha, Tabulações, Book Operacional, MOP Operacional, Sugestões |

Supervisores podem ter uma **célula/campanha responsável** associada
(campo opcional), definida pelo admin na tela **Usuários**.

## Como rodar

### Com Docker (recomendado)

```bash
cp .env.example .env
# edite o .env: troque as senhas, o JWT_SECRET, e preencha OLOS_DB_* se
# já tiver o banco espelho pronto
docker compose up -d --build
```

Sobe dois containers (`mysql` + `app`). Na primeira execução, as migrations
rodam sozinhas e um admin é criado com as credenciais do `.env`
(`DEFAULT_ADMIN_USER` / `DEFAULT_ADMIN_PASSWORD`).

Acesse em `http://localhost:3000`.

Pra parar: `docker compose down` (dados persistem no volume `mysql_data`;
use `down -v` pra descartar tudo).

### Sem Docker (banco externo)

```bash
pnpm install
cp .env.example .env
# defina DATABASE_URL e JWT_SECRET no .env
pnpm db:push
pnpm dev        # desenvolvimento (http://localhost:3000)
# ou pra produção:
pnpm build && pnpm start
```

## Variáveis de ambiente

| Variável | Obrigatória | Descrição |
|---|---|---|
| `DATABASE_URL` | Sim (fora do Docker) | Connection string do MySQL da própria aplicação |
| `JWT_SECRET` | Sim | Segredo pra assinar o cookie de sessão (`openssl rand -hex 32`) |
| `DEFAULT_ADMIN_USER` | Não (padrão `christyan`) | Login do admin criado na primeira execução |
| `DEFAULT_ADMIN_PASSWORD` | Não (padrão `ddm123`) | Senha do admin (salva como hash) |
| `PORT` | Não (padrão `3000`) | Porta do servidor |
| `OLOS_DB_HOST` / `PORT` / `USER` / `PASSWORD` / `NAME` | Não | Conexão **de leitura** com o banco espelho da OLOS. Sem isso, a sincronização automática fica desligada |
| `OLOS_SYNC_INTERVAL_MIN` | Não (padrão `10`) | De quanto em quanto tempo as automações automáticas rodam |

**Importante sobre a conexão com a OLOS**: use um usuário de banco com
permissão **só de SELECT** — nenhuma automação aqui faz
INSERT/UPDATE/DELETE no banco da OLOS, só leitura.

O admin padrão só é criado se a tabela `users` estiver vazia. Depois da
primeira execução, gerencie contas pela tela **Usuários** (admin-only).

## Stack técnica

- **Frontend**: React + Vite + Tailwind + tRPC + Recharts
- **Backend**: Express + tRPC + Drizzle ORM
- **Banco**: MySQL 8 (aplicação) + conexão de leitura a um banco MySQL
  espelho da OLOS
- **Autenticação**: cookie de sessão (JWT) assinado localmente, senha com
  hash bcrypt — sem OAuth externo
- **Export**: html-to-image (PNG) + jsPDF (PDF)

## Estrutura do projeto
