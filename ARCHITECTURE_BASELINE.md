# ARCHITECTURE_BASELINE.md
### App: "Aos Pés do Mestre Jesus" — Jornada Discipular (Hospital Adventista Silvestre)
### Repositório: `capelaniahospitalar/jornada-discipular`
### Data da auditoria: 2026-07-07

Este documento é um **diagnóstico**, não uma proposta de mudança. Nenhum arquivo de produção foi alterado para gerá-lo. Ele existe para servir de linha de base (RC0): qualquer refatoração futura (ex.: PersistenceManager) deve ser comparada contra o comportamento aqui descrito para identificar regressões.

---

## 1. Visão geral do projeto

- **Tipo:** PWA (Progressive Web App) client-side, sem build step — um único arquivo `index.html` (4.054 linhas) contendo HTML + CSS + JavaScript, mais `manifest.json` e `service-worker.js`.
- **Hospedagem:** GitHub Pages (`capelaniahospitalar.github.io/jornada-discipular/`).
- **Backend:** nenhum backend próprio. Duas integrações externas via `fetch`:
  - Google Apps Script (`SYNC_URL` / `TUTOR_PANEL_URL` — **mesma URL** usada para dois propósitos: enviar progresso do discípulo e consultar lista de discípulos do painel do tutor).
  - Cloudflare Worker (`AI_PROXY_URL`) — proxy para o assistente de IA.
- **Persistência:** 100% no dispositivo do usuário (localStorage + sessionStorage + cookie como fallback), sem login/conta real.
- **Distribuição/CI:** nenhuma. Publicação é manual (commit + push, ou upload direto pela interface web do GitHub).

---

## 2. Inventário de módulos existentes

O arquivo não tem módulos formais (não há `import`/`export`); a organização é por blocos de funções dentro de um único `<script>`. Agrupando por responsabilidade:

| Módulo (informal) | Linhas aprox. | Responsabilidade |
|---|---|---|
| **Helpers de texto bíblico** | 834–876 | `V()` (formata versículo), `N()` (nome do usuário), `toggleJoao1`/`toggleV` (expandir/colher blocos de texto) |
| **Modelo de conteúdo dos estudos** | 878–2078 | `STUDIES[]` — array com os 17 estudos, cada um com `phases[]` (tipos: `opening`, `question`, `content`, `deeper`, `insight`, `apply`, `decision`, `close`) |
| **Assistente de IA** | 1926–2078 | `openAI`, `addAIMsg`, `askAI` — chat via `AI_PROXY_URL` |
| **Compartilhar/Convidar** | 2079–2122 | `openShare`, `copyLink` — gera link genérico + mensagem de WhatsApp |
| **Painel do Tutor** | 2124–2557 | Autenticação por senha local (`amjTutorPass`), menu do tutor, consulta de discípulos via Google Apps Script, detalhe do discípulo |
| **Obstáculos espirituais** | 2559–2667 | `OBSTACLES[]`, progresso e desbloqueio associados a estudos específicos |
| **Missões da Jornada** | 2669–2862 | `JOURNEY_MISSIONS[]` — sistema de missões com `check()` (função de verificação) e XP |
| **Diário Espiritual** | 2863–2943 | `openJournal`, `renderJournal`, `saveJournalEntry`, `showJournalPrompt` |
| **RPG/Gamificação** | 2944–3184 | `RPG_LEVELS[]`, `ATTRIBUTES[]`, `STUDY_ATTRS[]`, `getAttrs`, `getRpgLevel`, `renderLevelCard`, `renderAttrs` |
| **Seleção de Tutor** | 3185–3330 | `openTutor`, `selectTutor`, `selectGuestTutor`, botão flutuante do WhatsApp |
| **Sincronização com backend** | 3316–3368 | `syncToTutorBackend` — debounce de 1.5s, envia payload consolidado ao Apps Script |
| **Persistência (estado principal)** | 3370–3400 | `save(s)` / `load()` — grava/lê em localStorage + sessionStorage + cookie |
| **Estado global e normalização** | 3400–3420 | `let ST = load()` + defaults (`ST.xp`, `ST.done`, `ST.badges`, `ST.streak`, `ST.lastDate`, `ST.obstacleActions`) |
| **Home / Dashboard** | 3422–3626 | `renderHome` (orquestra ~10 sub-renders), streak, bloqueio de reflexão |
| **Fluxo do estudo** | 3640–3808 | `openStudy`, `renderPhase`, `selectOpt`, `nextPhase`, `completeStudy` |
| **Módulo Bônus** | 3808–3856 | `openBonus` — conteúdo extra pós-conclusão |
| **Onboarding / Boas-vindas** | 3856–4002 | `confirmRestart`, `selectWelcomeTutor`, `confirmGuestWelcome`, `startJourney`, `initApp` |
| **Navegação genérica** | 3882–3919 | `showScreen`, `resetScroll` |
| **Service Worker + Instalação (PWA)** | 3996–4054 | Registro do service worker, `beforeinstallprompt`, `handleInstallClick`, `isAppInstalled` |

---

## 3. Fluxo de navegação entre telas

O app usa **uma única página** com 12 `<div class="screen" id="screen-*">`, alternadas via `showScreen(nome)` (mostra/esconde por CSS, sem router/URL):

```
screen-welcome  (onboarding: nome, perfil, tutor)
      │  startJourney() / initApp() se ST.welcomeDone
      ▼
screen-home  ◄──────────────────────────────────────┐
   │  openStudy(idx)          │ openJournal()        │
   ▼                          ▼                      │
screen-study               screen-journal            │
   │  completeStudy()          │ (volta)              │
   ▼                                                  │
screen-complete ──(showJournalPrompt bloqueia)────────┤
                                                       │
screen-reflection (bloqueio temporário entre estudos)─┤
screen-bonus (conteúdo extra)─────────────────────────┤
screen-tutor (escolher tutor)──────────────────────────┤
screen-ai (chat com IA)────────────────────────────────┤
screen-share (convidar amigo)──────────────────────────┤
screen-panel (meu progresso)───────────────────────────┤
screen-tutorpanel (login tutor → menu → lista → detalhe)┘
```

Observações:
- Não há histórico de navegação (sem "voltar" do navegador); todo "voltar" é um botão explícito que chama `goHome()` ou `showScreen(...)`.
- `screen-tutorpanel` tem sub-estados internos (login → criar/errar senha → menu → lista → detalhe) controlados por `innerHTML` trocado dinamicamente dentro do mesmo `<div>`, não por `showScreen`.
- `initApp()` decide a tela inicial (`welcome` vs `home`) com base em `ST.welcomeDone` — é o único ponto de "roteamento" na carga da página.

---

## 4. Estrutura de dados persistidos

### 4.1 Estado principal — chave `amj3`
Gravado via `save(ST)`, lido via `load()`. Guardado em **três lugares simultaneamente** (redundância proposital contra perda de dados em Safari/iOS): `localStorage`, `sessionStorage` e `document.cookie` (validade de 1 ano). `load()` tenta os três nesta ordem de prioridade.

Campos conhecidos do objeto `ST` (nem todos normalizados na inicialização — ver seção 6):

| Campo | Tipo | Default explícito? | Onde é definido |
|---|---|---|---|
| `userName` | string | não | onboarding |
| `userProfile` | `'paciente'\|'colaborador'\|'amigo'` | não | `selectProfile` |
| `tutor` | index numérico em `TUTORS[]` ou `'guest'` | não | `selectTutor`/`selectWelcomeTutor` |
| `guestTutor` | `{name, wa}` | não | `confirmGuestWelcome`/`updateGuestTutor` |
| `welcomeDone` | boolean | não | `startJourney` |
| `xp` | number | **sim** (`|| 0`) | `completeStudy`, missões |
| `done` | number[] (índices de estudos concluídos) | **sim** (`|| []`) | `completeStudy` |
| `badges` | string[] | **sim** (`|| []`) | `completeStudy`, `updStreak` |
| `streak` | number | **sim** (`|| 0`) | `updStreak` |
| `lastDate` | string (toDateString) | **sim** (`|| null`) | `updStreak` |
| `obstacleActions` | object (map por obstáculo) | **sim** (`|| {}`) | `doObstacleAction` |
| `journal` | array de `{studyIdx, ...}` | não (usa `(ST.journal||[])` pontualmente) | `saveJournalEntry` |
| `missionsDone` | string[] (ids de missões manuais) | não (usa `(ST.missionsDone||[])` pontualmente) | `completeMission` |
| `completedAt` | object `{[idx]: timestamp}` | não | `completeStudy` |
| `sponsor` | string | não — **nunca é escrito em lugar nenhum** (recurso incompleto, ver seção 6) | — |
| `tutorPanelAuth` | string (nome do tutor logado) | não | `tutorLogin`/`tutorLogout` |
| `syncDataInicio` | string ISO | não | `syncToTutorBackend` |

### 4.2 Senhas dos tutores — chave `amjTutorPass`
Objeto `{ [nomeDoTutor]: senhaEmTextoPuro }`, gravado via `localStorage.setItem` direto (não passa por `save`/`load` do estado principal). **Senha fica em texto puro no dispositivo** — ver riscos.

### 4.3 Escopo compartilhado de origem (risco de colisão)
O GitHub Pages serve todos os apps da organização sob o mesmo domínio (`capelaniahospitalar.github.io`). `localStorage`/`cookie` são isolados por **origem** (protocolo+domínio), não por caminho — ou seja, em teoria, se outro app do mesmo domínio usasse uma chave chamada `amj3` ou `amjTutorPass`, haveria colisão de dados entre apps diferentes. Hoje isso é mitigado apenas pelo prefixo específico dessas duas chaves (`amj*`), não por um mecanismo formal de namespace.

### 4.4 Dados enviados ao backend (não persistidos localmente, apenas transmitidos)
`syncToTutorBackend()` envia ao Apps Script: nome, tutor, estudos concluídos, XP, nível, streak, perfil, data de início, atributos, obstáculos vencidos, total de entradas no diário. Isso é uma cópia "espelho" do estado local — não há mecanismo de reconciliação caso o backend e o localStorage divirjam (ex.: usuário troca de celular).

---

## 5. Dependências entre funções (pontos centrais do grafo de chamadas)

- **`ST` (variável global mutável)** é lida/gravada diretamente por **mais de 60 funções** em todo o arquivo — não existe um único ponto de acesso.
- **`save(ST)`** é chamado a partir de ~15 pontos diferentes (onboarding, seleção de tutor, conclusão de estudo, missões, diário, streak, reset). Qualquer mudança na função `save` afeta todos esses fluxos simultaneamente.
- **`completeStudy()`** é a função com maior acoplamento interno: lê/escreve 8 campos de `ST`, contém regras de desbloqueio de badges com **números de estudo hardcoded** (ex. `ST.done.includes(14)`), manipula ~10 elementos do DOM por `getElementById`, e dispara `showJournalPrompt` + `updateCompleteHomeBtn` + `save`.
- **`renderHome()`** orquestra ~10 sub-funções de renderização (`renderLevelCard`, `renderAttrs`, `renderBosses`, `renderMissions`, `renderShareArea`, `renderAIArea`, `renderTutorArea`, etc.), cada uma escrevendo diretamente em um `id` específico do DOM.
- **`STUDIES[]`** e **`STUDY_ATTRS[]`** são acoplados por **posição no array** (índice `i` de `STUDIES[i]` deve corresponder a `STUDY_ATTRS[i]`) — não há chave nomeada ligando os dois.
- **`TUTORS[]`** é acoplado ao estado salvo por **índice numérico**: `ST.tutor` grava a posição do tutor no array, não um id estável. Reordenar ou remover um tutor da lista corrompe silenciosamente os dados já salvos de usuários que escolheram tutores por índices maiores.
- **`syncToTutorBackend`** depende de `getAttrs()`, `getRpgLevel()`, `OBSTACLES`, `STUDIES` e do `ST` global — qualquer mudança de forma nesses dados muda o payload enviado ao Apps Script sem que o Apps Script seja avisado.
- Praticamente toda a interatividade da UI é feita via **atributos `onclick="funcaoGlobal(...)"` inline no HTML gerado por template strings** — isso acopla fortemente nomes de funções JS aos textos de template espalhados pelo arquivo; renomear uma função exige busca textual em todo o arquivo.

---

## 6. Riscos técnicos identificados

1. **Acesso a `localStorage` disperso em 2 pontos distintos** (`save`/`load` do estado principal, e `getTutorPassStore`/`setTutorPass`), sem tratamento de erro consistente, sem validação de schema, sem versionamento dos dados salvos. *(Este é o risco que motivou a proposta de um `PersistenceManager` centralizado.)*
2. **Senhas de tutor em texto puro** no `localStorage` (`amjTutorPass`), sem hash. Risco baixo dado o contexto (ferramenta interna, sem dados sensíveis de pacientes), mas vale nota.
3. **Normalização de estado incompleta**: alguns campos (`done`, `xp`, `badges`, `streak`, `lastDate`, `obstacleActions`) recebem default logo após `load()`; outros (`journal`, `missionsDone`) são tratados com `|| []` só localmente, em alguns pontos de uso — inconsistência que pode gerar bugs sutis se um novo trecho de código esquecer o fallback.
4. **Acoplamento por índice numérico** entre `ST.tutor` e `TUTORS[]`, e entre `STUDIES[]`/`STUDY_ATTRS[]`/`FASE_UNLOCK`/ícones de conclusão (array `icons` em `completeStudy`). Qualquer reordenação quebra dados de usuários existentes ou desalinha atributos/badges.
5. **Regras de negócio com números mágicos**: desbloqueio de badges e fases usa índices de estudo hardcoded em condicionais (`ST.done.length >= 9 && [5,6,7,8].every(...)`), duplicando a mesma informação que já existe em `FASE_UNLOCK`.
6. **Mesma URL do Apps Script usada para dois propósitos** (`SYNC_URL` === `TUTOR_PANEL_URL`), o que mistura responsabilidades de escrita (sync de progresso) e leitura (consulta do painel) num único endpoint externo, sem controle de versão de API.
7. **Sem tratamento de falha de sincronização**: `fetch(SYNC_URL, ...).catch(() => {})` descarta silenciosamente qualquer erro de rede — não há fila de reenvio nem indicação ao usuário/tutor de que os dados não sincronizaram.
8. **Arquivo único de 4.054 linhas** (HTML+CSS+JS), sem build step, sem módulos ES, sem testes automatizados — qualquer alteração depende de revisão manual e teste manual completo.
9. **Sem mecanismo de migração de dados**: se um campo do `ST` for renomeado ou tiver o formato alterado no futuro, usuários com dados antigos salvos localmente não têm caminho de conversão automática.
10. **Risco de colisão de `localStorage` entre apps** hospedados na mesma origem GitHub Pages (ver seção 4.3) — mitigado apenas por convenção de nomes de chave, não por isolamento técnico.
11. **Credenciais/endpoints expostos no código-fonte cliente** (URLs do Apps Script e do Worker de IA são públicas, visíveis a qualquer um que inspecione o app) — comum em apps client-only, mas vale registrar como superfície de abuso potencial (qualquer pessoa pode enviar payloads arbitrários ao `SYNC_URL`).

---

## 7. Pontos de alto acoplamento (resumo)

| Ponto de acoplamento | Por quê é frágil |
|---|---|
| `ST` global mutável acessado em 60+ locais | Sem encapsulamento; qualquer função pode corromper o estado de qualquer outra |
| `completeStudy()` | Concentra regras de XP, badges, fases e navegação num único ponto extenso |
| `ST.tutor` = índice de `TUTORS[]` | Dado salvo do usuário depende da ordem de um array de conteúdo |
| `STUDIES[i]` ↔ `STUDY_ATTRS[i]` ↔ ícones de `completeStudy` | Três listas paralelas que devem permanecer sincronizadas por posição |
| `onclick="..."` inline em templates | Acopla nomes de função globais ao texto de template espalhado pelo arquivo |
| `SYNC_URL` = `TUTOR_PANEL_URL` | Um único endpoint externo serve dois propósitos distintos (escrita e leitura) |
| Acesso direto a `localStorage` em 2 módulos separados | Sem camada única de validação/erro (motivo da RC1 proposta) |

---

## 8. Recomendações de refatoração, por prioridade

**P1 — Baixo risco, alto valor (bom próximo passo / RC1):**
- Centralizar acesso a `localStorage` num módulo único (`PersistenceManager` ou similar), preservando exatamente as chaves (`amj3`, `amjTutorPass`) e o comportamento de fallback (`sessionStorage`/cookie) já existente, para manter compatibilidade total com dados já salvos nos dispositivos dos usuários.
- Normalizar **todos** os campos do `ST` logo após `load()` (não só os 6 já tratados), eliminando os `|| []`/`|| {}` espalhados.

**P2 — Risco moderado, recomendado a médio prazo:**
- Extrair as regras de desbloqueio de badges/fases de dentro de `completeStudy()` para uma configuração declarativa única (reaproveitando `FASE_UNLOCK` em vez de duplicar índices em condicionais).
- Trocar `ST.tutor` (índice) por um **id estável** (ex. slug do nome do tutor), com migração automática na primeira carga para usuários que já têm um índice salvo.

**P3 — Risco maior / esforço maior, avaliar caso a caso:**
- Separar `SYNC_URL` de `TUTOR_PANEL_URL` em endpoints distintos (ou ao menos em rotas/parâmetros claramente diferenciados) no Apps Script.
- Adicionar hashing simples para senhas de tutor (mesmo que client-side, reduz exposição em texto puro).
- Introduzir um esquema de versionamento no objeto salvo (`ST.__version`) para permitir migrações futuras sem quebrar usuários existentes.

**P4 — Longo prazo / opcional:**
- Modularizar o arquivo único (ES modules ou build step) caso o projeto continue crescendo — hoje isso traria risco desproporcional ao benefício, dado que não há pipeline de build nem testes automatizados no fluxo atual de publicação (upload manual/GitHub Desktop).

---

## 9. O que este documento **não** fez

- Não alterou `index.html`, `manifest.json`, `service-worker.js` ou qualquer outro arquivo de produção.
- Não alterou comportamento, layout, CSS ou dados salvos.
- Não validou o backend Google Apps Script (fora do escopo — código não está neste repositório).
