# DATA_REVIEW.md — Revisão de Armazenamento de Dados
**Aplicativo:** Aos Pés do Mestre Jesus  
**Versão analisada:** index.html (bed2252) | 4069 linhas | 347 KB  
**Analistas:** Arquiteto de Software Sênior · Especialista em PWA · Especialista em QA · Revisor de Código  
**Data:** 26/06/2026  
**Status:** Somente análise — nenhuma alteração implementada  

---

## Índice

1. [Visão Geral da Arquitetura de Dados](#1-visão-geral-da-arquitetura-de-dados)
2. [Chaves de Armazenamento](#2-chaves-de-armazenamento)
3. [Mecanismo de Escrita — `save()`](#3-mecanismo-de-escrita--save)
4. [Mecanismo de Leitura — `load()`](#4-mecanismo-de-leitura--load)
5. [Esquema do Objeto ST](#5-esquema-do-objeto-st)
6. [localStorage](#6-localstorage)
7. [sessionStorage](#7-sessionstorage)
8. [Cookies](#8-cookies)
9. [IndexedDB](#9-indexeddb)
10. [Cache API e Service Worker](#10-cache-api-e-service-worker)
11. [Backup e Recuperação](#11-backup-e-recuperação)
12. [Migração e Versionamento de Esquema](#12-migração-e-versionamento-de-esquema)
13. [Detecção de Corrupção](#13-detecção-de-corrupção)
14. [Apagamento de Dados — `confirmRestart()`](#14-apagamento-de-dados--confirmrestart)
15. [Sincronização com Google Sheets](#15-sincronização-com-google-sheets)
16. [Senhas do Painel do Tutor](#16-senhas-do-painel-do-tutor)
17. [Persistência Offline](#17-persistência-offline)
18. [Privacidade e Segurança dos Dados](#18-privacidade-e-segurança-dos-dados)
19. [Limites de Armazenamento](#19-limites-de-armazenamento)
20. [Tabela de Riscos Consolidada](#20-tabela-de-riscos-consolidada)
21. [Recomendações Prioritárias](#21-recomendações-prioritárias)

---

## 1. Visão Geral da Arquitetura de Dados

O aplicativo utiliza uma estratégia de armazenamento **triplo redundante no lado do cliente**: localStorage como armazenamento primário, com fallback para sessionStorage e cookie de sessão longa. Não há backend próprio para armazenamento de dados do usuário — todo o progresso espiritual existe exclusivamente no dispositivo e no navegador do usuário.

```
┌─────────────────────────────────────────────────────────┐
│                   DISPOSITIVO DO USUÁRIO                │
│                                                         │
│  ┌──────────────┐  ┌───────────────┐  ┌─────────────┐ │
│  │ localStorage │  │sessionStorage │  │   Cookie    │ │
│  │   (primary)  │  │  (fallback 1) │  │ (fallback 2)│ │
│  │    amj3      │  │     amj3      │  │    amj3     │ │
│  │ amjTutorPass │  │               │  │             │ │
│  └──────────────┘  └───────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────┘
           │
           │  syncToTutorBackend() (one-way, agregado)
           ▼
┌──────────────────────────────┐
│  Google Sheets (Apps Script) │  ← Painel do tutor apenas
│  Dados agregados, sem diário │
└──────────────────────────────┘
```

**Ausências críticas:** IndexedDB, Cache API, Service Worker ativo, exportação de dados, backup remoto.

---

## 2. Chaves de Armazenamento

| Chave | Mecanismo | Conteúdo | Tamanho estimado |
|-------|-----------|----------|-----------------|
| `amj3` | localStorage + sessionStorage + cookie | Objeto ST completo (progresso, diário, perfil) | 2–50 KB dependendo do diário |
| `amjTutorPass` | localStorage apenas | Senhas dos tutores em texto puro | < 1 KB |

> ⚠️ **Nota de rastreabilidade:** Documentações anteriores desta sessão mencionavam a chave `jornada_st`. O código atual (linha 3421) usa **`amj3`**. Se houver dados de versões anteriores sob `jornada_st`, eles são silenciosamente ignorados — nenhuma migração ocorre.

---

## 3. Mecanismo de Escrita — `save()`

**Localização:** `index.html` linha 3419

```javascript
function save(s) {
  const data = JSON.stringify(s);
  try { localStorage.setItem('amj3', data); } catch {}
  try { sessionStorage.setItem('amj3', data); } catch {}
  try {
    const exp = new Date();
    exp.setFullYear(exp.getFullYear() + 1);
    document.cookie = 'amj3=' + encodeURIComponent(data) + ';expires=' + exp.toUTCString() + ';path=/;SameSite=Lax';
  } catch {}
  syncToTutorBackend();
}
```

**Características:**
- **Escrita atômica total:** o objeto ST inteiro é serializado e sobrescrito a cada chamada. Não há escrita incremental ou por campo.
- **Sem confirmação de sucesso:** os `try/catch` silenciam erros. Se `localStorage.setItem` falhar (quota excedida, modo privado bloqueado), a aplicação continua sem aviso.
- **Sem verificação de escrita:** não há leitura de retorno para confirmar que o dado foi gravado corretamente.
- **Trigger de sync:** cada `save()` dispara `syncToTutorBackend()` com debounce de 1.500ms.
- **Frequência:** `save()` é chamado após cada fase concluída, ao salvar o diário, ao atualizar streak, e em múltiplos outros pontos. Em um estudo ativo, pode ser chamado dezenas de vezes.

**Riscos:**
- Falha silenciosa de gravação → perda de progresso sem notificação ao usuário
- Sobrescrição com objeto corrompido (ex.: campo `null` indevido) apaga dado anterior sem possibilidade de rollback

---

## 4. Mecanismo de Leitura — `load()`

**Localização:** `index.html` linha 3431

```javascript
function load() {
  try {
    const ls = localStorage.getItem('amj3');
    if (ls) return JSON.parse(ls);
  } catch {}
  try {
    const ss = sessionStorage.getItem('amj3');
    if (ss) return JSON.parse(ss);
  } catch {}
  try {
    const match = document.cookie.match(/amj3=([^;]+)/);
    if (match) return JSON.parse(decodeURIComponent(match[1]));
  } catch {}
  return {};
}
```

**Características:**
- **Fallback em cascata:** localStorage → sessionStorage → cookie → objeto vazio `{}`
- **Sem logging:** se localStorage falhar e o dado for recuperado do cookie, o usuário não é informado
- **`JSON.parse` protegido por try/catch:** ao contrário do carregamento de senhas (que também tem try/catch), aqui há proteção adequada
- **Retorno de objeto vazio:** em caso de falha total ou primeiro acesso, retorna `{}` — os campos são normalizados logo após (linhas 3450–3455)

**Risco:**
- Dados existentes em sessionStorage ou cookie podem estar desatualizados em relação a localStorage. A lógica prioriza localStorage sem verificar se os outros mecanismos têm versão mais recente.

---

## 5. Esquema do Objeto ST

O objeto ST é o único modelo de dados da aplicação. Não há tipagem, validação de esquema ou versionamento.

```javascript
ST = {
  // Identidade
  userName: string,
  userProfile: 'paciente' | 'colaborador' | 'amigo',
  tutor: number | 'guest',
  guestTutor: { name: string, phone: string } | null,
  welcomeDone: boolean,
  sponsor: string | undefined,

  // Progresso
  done: number[],           // índices dos estudos concluídos
  xp: number,
  streak: number,
  lastStudy: number | null, // índice do último estudo
  lastDate: string | null,  // ex: "Thu Jun 26 2026" (toDateString())
  completedAt: { [studyIndex]: ISODateString },
  badges: string[],         // ex: ['nivel1', 'streak7']

  // Missões e obstáculos
  missions: { [missionId]: boolean },
  missionsDone: any[] | undefined,
  obstacleActions: { [obstacleId]: boolean[] },

  // Diário espiritual
  journal: [{
    title: string,          // ex: "Reflexão — Estudo 3"
    date: string,           // ex: "26/06/2026" (toLocaleDateString pt-BR)
    verse: string,
    learned: string,
    apply: string,
    prayer: string,
    chosen: string
  }],

  // Painel do tutor
  tutorPanelAuth: boolean,
  syncDataInicio: ISOString | undefined,
}
```

**Problemas do esquema:**
- **Sem campo de versão:** impossível detectar se ST foi criado por versão antiga ou nova do app
- **Campos opcionais não declarados:** `sponsor`, `missionsDone`, `syncDataInicio` aparecem apenas em runtime, tornando o esquema implícito
- **`lastDate` usa `toDateString()`:** formato como `"Thu Jun 26 2026"` — dependente de locale do navegador, não é ISO 8601
- **`journal[].date` usa `toLocaleDateString('pt-BR')`:** formato `"26/06/2026"` — string não ordenável como data

---

## 6. localStorage

**Primário para:** chave `amj3` (progresso completo) + `amjTutorPass` (senhas)

**Comportamento por contexto:**

| Contexto | Disponibilidade |
|----------|----------------|
| Uso normal (Chrome, Firefox, Safari) | ✅ Persistente entre sessões |
| Navegação anônima / privada | ⚠️ Persiste durante a sessão, apagado ao fechar |
| `Clear browsing data` pelo usuário | ❌ Apagado sem aviso |
| Desinstalação do PWA (Android) | ❌ Apagado |
| Expiração automática | ✅ Não expira (diferente de cookie) |
| Múltiplos dispositivos | ❌ Não compartilhado |
| Múltiplos usuários no mesmo dispositivo | ❌ Sem isolamento |

**Ausência de:**
- Criptografia (dados em texto puro)
- Controle de acesso
- Compressão (JSON cru)

---

## 7. sessionStorage

**Uso:** fallback secundário para `amj3`

**Características:**
- Persiste apenas durante a aba aberta
- Apagado ao fechar a aba ou o navegador
- Cada aba tem sua própria sessionStorage independente

**Problema crítico de sincronização:** se o usuário abrir o app em duas abas simultâneas, cada aba lê e escreve seu próprio sessionStorage. A aba mais recente pode sobrescrever dados da aba anterior ao fechar. localStorage mitiga isso parcialmente, mas as abas não se comunicam em tempo real.

**Valor como fallback:** sessionStorage é útil quando localStorage está bloqueado (modo privado no Safari antigo) ou quando a cota foi excedida, mas oferece proteção zero contra perda entre sessões.

---

## 8. Cookies

**Uso:** fallback terciário para `amj3`  
**Expiração:** 1 ano a partir do momento do primeiro save  
**Atributos:** `path=/;SameSite=Lax`

**Riscos de segurança:**
- **Sem flag `Secure`:** o cookie é transmitido em conexões HTTP além de HTTPS. Se o app for acessado por HTTP (ex.: rede local sem TLS), o cookie trafega em texto claro.
- **Sem flag `HttpOnly`:** JavaScript pode ler o cookie via `document.cookie`. Qualquer script injetado (XSS via diário, conforme JOURNAL_REVIEW.md) pode exfiltrar todos os dados do usuário.
- **Tamanho máximo:** cookies têm limite de ~4 KB por domínio. Com um diário extenso, `amj3` pode exceder esse limite. O `try/catch` silencia o erro, resultando em cookie desatualizado ou ausente.

**Risco de transmissão:** cada requisição ao servidor (Google Fonts, AI Proxy, Google Sheets sync) transmite o cookie `amj3` no cabeçalho `Cookie` se as requisições forem para o mesmo domínio. Dado que todas as URLs externas são de domínios diferentes, isso não ocorre aqui — mas é um vetor de risco se o domínio do app for expandido.

---

## 9. IndexedDB

**Status:** ❌ **Não utilizado em nenhuma parte do código**

Grep confirmou ausência total de: `indexedDB`, `IDBDatabase`, `openCursor`, `createObjectStore`.

**O que se perde:**
- Armazenamento de centenas de MB (vs ~5–10 MB do localStorage)
- Operações assíncronas não bloqueantes
- Transações com rollback em caso de erro
- Índices para busca eficiente no diário
- Armazenamento binário (para futuros áudios/imagens devocionais)
- API estruturada para migração de esquema

**Impacto real no uso atual:** para o volume esperado de dados (< 100 KB), a ausência de IndexedDB não causa problemas imediatos. Torna-se crítico se o app escalar para suportar áudio, exportação ou backup local estruturado.

---

## 10. Cache API e Service Worker

**Status:** ❌ **Service Worker referenciado mas arquivo ausente**

**Localização da referência:** `index.html` linha 4059–4067

```javascript
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('service-worker.js')
    .then(...)
    .catch(...);
}
```

O arquivo `service-worker.js` **não existe** no repositório. Isso significa:

| Capacidade PWA | Status |
|----------------|--------|
| Cache de assets (HTML, CSS, JS) | ❌ Indisponível |
| Funcionamento offline | ❌ Indisponível |
| Instalação como app (Add to Home Screen) | ⚠️ Parcial (depende de manifest.json) |
| Background sync | ❌ Indisponível |
| Push notifications | ❌ Indisponível |
| Cache da IA (respostas repetidas) | ❌ Indisponível |

**Impacto para o usuário:**  
Ao abrir o app sem conexão, o navegador não encontra `index.html` em cache e exibe página de erro. Todo o progresso espiritual salvo no localStorage permanece intacto, mas o app é completamente inacessível. O usuário não tem como saber que seu progresso está seguro.

**Dependências que requerem rede:**
- Google Fonts (Lora + Source Sans 3) — sem `font-display: swap`
- Cloudflare Worker (AI Proxy)
- Google Apps Script (painel do tutor)

Sem Service Worker, nenhum desses recursos pode ser cacheado preventivamente.

---

## 11. Backup e Recuperação

**Status:** ❌ **Nenhum mecanismo de backup ou recuperação existe**

O aplicativo não oferece:
- Exportação de dados (JSON, PDF, texto)
- Importação de dados de dispositivo anterior
- Backup em nuvem
- QR Code de recuperação
- Link de transferência entre dispositivos

**Cenários de perda total de dados:**

| Cenário | Probabilidade | Recuperação possível |
|---------|---------------|---------------------|
| Usuário clica "Limpar dados do site" | Alta (usuário tentando "resolver problemas") | ❌ Zero |
| Troca de celular | Muito alta (ciclo de vida normal) | ❌ Zero |
| Reinstalação do navegador | Média | ❌ Zero |
| Corrupção de localStorage pelo navegador | Baixa | ❌ Zero |
| Atualização do SO que limpa dados do app | Baixa | ❌ Zero |
| Acidente com cookie expirado + LS apagado | Baixa | ❌ Zero |

**Impacto espiritual e pedagógico:**  
Um usuário que perdeu 8 estudos concluídos, 15 entradas de diário e 3 semanas de streak provavelmente abandona o app. A perda de dados em um aplicativo devocional é devastadora para o vínculo emocional construído ao longo da jornada.

**Nota:** o Google Sheets recebe dados agregados (`syncToTutorBackend`), mas apenas nome, tutor, contagem de estudos, XP e nível. **Nenhum dado do diário é sincronizado.** Em caso de perda de localStorage, o tutor pode saber que o usuário chegou ao Estudo 6, mas não pode restaurar o progresso.

---

## 12. Migração e Versionamento de Esquema

**Status:** ❌ **Nenhum sistema de versionamento ou migração existe**

O objeto ST não possui campo de versão (`schemaVersion`, `v`, `_v`). Ausência confirmada por grep.

**Cenário de falha de migração:**

```
Versão 1.0 do app → ST = { done: [], xp: 0, journal: [] }
Versão 2.0 do app → ST = { done: [], xp: 0, journal: [], badges: [], missions: {} }

Usuário com V1.0 abre V2.0:
- ST.badges → undefined
- ST.badges.push('nivel1') → TypeError: Cannot read property 'push' of undefined
```

O código atual mitiga parcialmente isso com defaults nas linhas 3450–3455:
```javascript
ST.xp      = ST.xp || 0;
ST.done    = ST.done || [];
ST.badges  = ST.badges || [];
ST.streak  = ST.streak || 0;
ST.lastDate= ST.lastDate || null;
ST.obstacleActions = ST.obstacleActions || {};
```

Mas **não há defaults para:** `journal`, `missions`, `completedAt`, `missionsDone`, `tutorPanelAuth`, `sponsor`, `welcomeDone`, `userName`, `userProfile`, `tutor`, `guestTutor`.

**Cenário crítico:** se uma futura versão renomear um campo (ex.: `done` → `completedStudies`), todos os usuários existentes perderão seu histórico de estudos silenciosamente.

---

## 13. Detecção de Corrupção

**Status:** ⚠️ **Parcial — inconsistente entre as chaves**

**Para `amj3` (dados principais):**
```javascript
// load() — linha 3433
try {
  const ls = localStorage.getItem('amj3');
  if (ls) return JSON.parse(ls);  // ← parse protegido
} catch {}
```
Se `JSON.parse` lançar exceção (JSON malformado), o `catch {}` captura silenciosamente e tenta o próximo mecanismo. **Não há log, não há alerta ao usuário, não há tentativa de reparo.**

**Para `amjTutorPass` (senhas):**
```javascript
// getTutorPassStore() — linha 2131
function getTutorPassStore() {
  try {
    return JSON.parse(localStorage.getItem('amjTutorPass') || '{}');
  } catch { return {}; }
}
```
Similar — corrupção silenciosa, retorna objeto vazio (todas as senhas são perdidas).

**Cenários que causam corrupção:**
- Escrita interrompida (tab fechada durante `setItem`)
- Bug no código que grava string não-JSON em `amj3`
- Extensão de navegador maliciosa que modifica localStorage
- Quota excedida parcialmente (trunca o JSON)

**Ausência de:**
- Checksum / hash de integridade
- Cópia de segurança "generation" (amj3_backup com versão anterior)
- Detecção de campos obrigatórios ausentes
- Recovery wizard para o usuário

---

## 14. Apagamento de Dados — `confirmRestart()`

**Localização:** `index.html` linha 3906

```javascript
function confirmRestart() {
  const ok = confirm('⚠️ Recomeçar a jornada?...');
  if (!ok) return;

  ST.done        = [];
  ST.xp          = 0;
  ST.streak      = 0;
  ST.lastStudy   = null;
  ST.lastDate    = null;
  ST.completedAt = {};
  ST.missions    = {};
  pendingJournalIdx   = null;
  pendingJournalTitle = null;

  save(ST);
  goHome();
}
```

**O que é zerado:** done, xp, streak, lastStudy, lastDate, completedAt, missions  
**O que é preservado:** userName, userProfile, tutor, guestTutor, journal, sponsor, syncDataInicio

**Inconsistências detectadas:**

| Campo | Zerado no restart? | Deveria ser zerado? | Problema |
|-------|-------------------|---------------------|---------|
| `ST.badges` | ❌ Não | ✅ Sim | Badges aparecem como conquistadas mas sem progresso correspondente |
| `ST.obstacleActions` | ❌ Não | ✅ Sim | Obstáculos/batalhas aparecem como vencidos sem estudos concluídos |
| `ST.missionsDone` | ❌ Não | Depende | Possível inconsistência visual nas missões |
| `pendingJournalIdx` | ✅ Sim (memória) | — | Correto, mas volátil — já se perde ao recarregar |

**Bug confirmado:** após `confirmRestart()`, a tela de badges pode mostrar conquistas de níveis que o usuário "ainda não alcançou", criando incoerência visual e espiritual.

**UX crítico:** o único aviso é um `confirm()` nativo do navegador — conforme identificado em UX_REVIEW.md, isso quebra o contexto visual do app. O diálogo não menciona explicitamente que badges e histórico de obstáculos são preservados.

---

## 15. Sincronização com Google Sheets

**Localização:** `index.html` linhas 3365, 3381–3417

**Dados enviados:**
```javascript
{
  nome: ST.userName,
  tutor: tutorName,
  estudos: ST.done.length,
  estudosConcluidos: ['Jesus e Sua Vida', ...],
  xp: ST.xp,
  nivel: 'Discípulo',
  streak: ST.streak,
  perfil: 'Paciente',
  dataInicio: ST.syncDataInicio,
  attrs: { fe, oração, serviço, ... },
  obstaculosVencidos: [...],
  totalDiario: ST.journal.length
}
```

**Dados NÃO enviados:** conteúdo do diário, missões individuais, senhas, tutor auth

**Características técnicas:**
- **Debounce 1.500ms:** evita disparos em sequência rápida
- **One-way:** apenas escrita, sem leitura de retorno
- **`Content-Type: text/plain`:** técnica para evitar preflight CORS
- **Falha silenciosa:** `.catch(() => {})` ignora erros de rede
- **Sem retentativa:** se a sync falhar (offline), o dado não é reenviado

**Risco de privacidade:** o Google Sheets é controlado pelo administrador do hospital. Nome completo do usuário, perfil (paciente/colaborador), tutor, progresso e data de início são transmitidos. O usuário não é explicitamente informado sobre essa sincronização durante o onboarding.

---

## 16. Senhas do Painel do Tutor

**Localização:** `index.html` linhas 2131–2143

```javascript
function getTutorPassStore() {
  try {
    return JSON.parse(localStorage.getItem('amjTutorPass') || '{}');
  } catch { return {}; }
}
function setTutorPass(tutorName, pass) {
  const store = getTutorPassStore();
  store[tutorName] = pass;
  try { localStorage.setItem('amjTutorPass', JSON.stringify(store)); } catch {}
}
function getTutorPass(tutorName) {
  const store = getTutorPassStore();
  return store[tutorName] || null;
}
```

**Formato armazenado:**
```json
{
  "Capelão Wladimir": "senha123",
  "Enfermeira Ana": "outrasenha"
}
```

**Riscos críticos:**
1. **Texto puro:** qualquer pessoa com acesso ao DevTools → Application → localStorage vê todas as senhas de todos os tutores
2. **Compartilhamento de dispositivo:** pacientes que emprestam o celular expõem senhas dos tutores
3. **XSS → exfiltração:** conforme JOURNAL_REVIEW.md, há vulnerabilidade XSS no diário. Um payload malicioso no campo "aprendizado" poderia executar `localStorage.getItem('amjTutorPass')` e exfiltrar senhas via fetch
4. **Sem rotação:** senhas não têm expiração
5. **Sem limite de tentativas:** bruteforce não tem proteção

---

## 17. Persistência Offline

**Mapa de funcionalidades por conectividade:**

| Funcionalidade | Online | Offline |
|----------------|--------|---------|
| Acessar o app (carregar HTML) | ✅ | ❌ Sem SW |
| Ler estudos (conteúdo em STUDIES[]) | ✅ | ❌ Sem SW |
| Salvar progresso no localStorage | ✅ | ✅ |
| Diário espiritual (ler/escrever) | ✅ | ❌ Sem SW |
| Mentor IA | ✅ | ❌ Requer rede |
| Fontes tipográficas (Google Fonts) | ✅ | ❌ Sem cache |
| Sincronização com Google Sheets | ✅ | ❌ Silenciosamente ignorado |
| Painel do tutor | ✅ | ❌ Silenciosamente ignorado |

**Resultado prático:** o aplicativo é **completamente inacessível sem internet**, apesar de ser descrito e distribuído como PWA. A tela de boas-vindas promete experiência offline que não existe.

**Dados sobrevivem offline?** Sim — localStorage não requer rede. Mas o usuário não pode acessar a interface para ver ou usar esses dados.

---

## 18. Privacidade e Segurança dos Dados

### 18.1 Dados em dispositivo compartilhado
Contexto hospitalar implica alto risco de compartilhamento de dispositivo. Qualquer pessoa que acesse o app no mesmo navegador vê nome, perfil, progresso e entradas do diário do usuário anterior. Não há:
- PIN/senha de acesso ao app
- Timeout de sessão
- Opção "sair" (logout)
- Ocultação de conteúdo sensível do diário

### 18.2 Acessibilidade via DevTools
Todos os dados são visíveis em texto puro via `F12 → Application → Local Storage`. Isso inclui:
- Nome do paciente
- Perfil (paciente/colaborador)
- Todas as entradas do diário espiritual (conteúdo íntimo)
- Senhas dos tutores

### 18.3 XSS e dados
Conforme documentado em JOURNAL_REVIEW.md, há vulnerabilidade XSS via `innerHTML` no diário. Um payload injetado no campo `learned` pode:
- Ler todo o localStorage (`amj3` + `amjTutorPass`)
- Exfiltrar via `fetch()` para servidor externo
- Modificar dados do usuário silenciosamente

### 18.4 Cookie sem flags de segurança
O cookie `amj3` é criado sem `Secure` e sem `HttpOnly` (ver seção 8). Em contexto hospitalar, onde redes internas podem usar HTTP, isso é um risco real.

---

## 19. Limites de Armazenamento

| Mecanismo | Limite típico | Uso estimado | Risco de overflow |
|-----------|--------------|--------------|------------------|
| localStorage | 5–10 MB por origem | 2–50 KB | Baixo para uso normal |
| sessionStorage | 5–10 MB por aba | 2–50 KB | Muito baixo |
| Cookie | ~4 KB total | 2–50 KB | **Alto para diários extensos** |

**Cálculo do diário:**  
Cada entrada tem 5 campos de texto livre. Se cada campo tiver 500 caracteres: 5 × 500 = 2.500 chars × encoding ≈ 3.000 bytes por entrada.  
Com 30 entradas: ~90 KB → seguro no localStorage.  
Com 100 entradas: ~300 KB → muito acima do limite do cookie (4 KB). O cookie fallback falha silenciosamente.

**Comportamento quando localStorage está cheio:**  
`localStorage.setItem()` lança `DOMException: QuotaExceededError`. O `try/catch` captura sem aviso. O dado não é salvo. O usuário continua usando o app sem saber que o progresso foi perdido.

---

## 20. Tabela de Riscos Consolidada

| # | Risco | Severidade | Probabilidade | Impacto |
|---|-------|-----------|--------------|---------|
| R01 | Perda total ao limpar dados do navegador | 🔴 Crítica | Alta | Perda irreversível de todo progresso e diário |
| R02 | Perda de dados ao trocar de dispositivo | 🔴 Crítica | Muito alta | 100% dos usuários que trocam de celular |
| R03 | Service Worker ausente — app inativo offline | 🔴 Crítica | Certa | App inacessível sem rede |
| R04 | XSS no diário permite exfiltrar todos os dados | 🔴 Crítica | Baixa | Exposição de dados íntimos e senhas |
| R05 | Senhas de tutores em texto puro | 🔴 Crítica | Média | Acesso não autorizado ao painel |
| R06 | Cookie sem `Secure` em redes HTTP | 🟠 Alta | Média | Dados em trânsito sem criptografia |
| R07 | Sem versionamento de esquema | 🟠 Alta | Certa ao atualizar | Dados silenciosamente corrompidos ou perdidos em updates |
| R08 | Falha de save silenciosa (quota excedida) | 🟠 Alta | Baixa | Progresso perdido sem aviso |
| R09 | Badges não zerados no restart | 🟡 Média | Alta | Incoerência visual e espiritual |
| R10 | Obstáculos não zerados no restart | 🟡 Média | Alta | Incoerência no mapa de batalhas |
| R11 | Sincronização com Google Sheets sem consentimento explícito | 🟡 Média | Certa | Conformidade LGPD questionável |
| R12 | Cookie de 1 ano sem aviso (LGPD) | 🟡 Média | Certa | Não há banner de cookies |
| R13 | Múltiplas abas causam conflito de dados | 🟡 Média | Baixa | Progresso de uma aba sobrescreve outra |
| R14 | Fontes não cacheadas — tipografia quebrada offline | 🟡 Média | Alta | Sem rede: Lora/Source Sans 3 ausentes |
| R15 | `lastDate` em formato locale-dependente | 🟢 Baixa | Baixa | Streak incorreto em locales não pt-BR |
| R16 | `journal[].date` não ordenável | 🟢 Baixa | Baixa | Futuras funcionalidades de busca quebradas |

---

## 21. Recomendações Prioritárias

> **Atenção:** este documento é exclusivamente analítico. Nenhuma alteração deve ser implementada sem revisão e aprovação explícita.

### P0 — Crítico (implementar antes de qualquer nova funcionalidade)

**P0.1 — Service Worker e cache offline**  
Criar `service-worker.js` com estratégia Cache-First para o `index.html`. Sem isso, o app não é um PWA e falha completamente offline.

**P0.2 — Exportação de dados do usuário**  
Adicionar botão "Exportar minha jornada" que gera JSON ou texto legível para download. É a única forma de backup acessível sem backend.

**P0.3 — Corrigir XSS no diário**  
Sanitizar entradas do diário antes de renderizar via `innerHTML` (usar `textContent` ou biblioteca DOMPurify).

### P1 — Alta prioridade

**P1.1 — Versionamento de esquema**  
Adicionar campo `_v: 1` ao ST e função `migrateSchema(st)` que atualiza campos ao carregar.

**P1.2 — Hash de senhas dos tutores**  
Substituir texto puro por hash bcrypt ou, minimamente, hash SHA-256 com salt fixo. Preferencialmente mover autenticação para o servidor.

**P1.3 — Aviso de falha de save**  
Envolver `localStorage.setItem` em função que detecta `QuotaExceededError` e avisa o usuário com opção de exportar os dados.

**P1.4 — Corrigir confirmRestart()**  
Zerar `ST.badges` e `ST.obstacleActions` no restart. Substituir `confirm()` nativo por modal estilizado.

### P2 — Média prioridade

**P2.1 — Cookie com flags `Secure;HttpOnly`**  
Adicionar ambas as flags ao cookie `amj3`.

**P2.2 — Consentimento de sincronização**  
Informar o usuário durante o onboarding que progresso é sincronizado com o tutor via Google Sheets, com opção de opt-out.

**P2.3 — Datas em ISO 8601**  
Substituir `toDateString()` e `toLocaleDateString()` por `new Date().toISOString()` para portabilidade e ordenação correta.

**P2.4 — Defaults completos no carregamento**  
Normalizar todos os campos do ST (`journal`, `missions`, `completedAt`, `welcomeDone`, etc.) após `load()`.

### P3 — Baixa prioridade / planejamento futuro

**P3.1 — IndexedDB para diário**  
Migrar o diário espiritual para IndexedDB para suportar busca indexada e volumes maiores.

**P3.2 — Sincronização cross-device**  
Implementar exportação/importação via QR Code ou link criptografado para transferência entre dispositivos sem backend.

**P3.3 — Timeout de sessão em dispositivos compartilhados**  
Detectar inatividade e apresentar tela de bloqueio em contextos de dispositivo compartilhado.

---

## Resumo Executivo

O aplicativo implementa uma estratégia de persistência robusta para o ambiente de navegador (triple-write em localStorage + sessionStorage + cookie), que é uma solução criativa e bem executada para o contexto monolítico. A função `load()` com fallback em cascata demonstra pensamento defensivo.

Contudo, há três riscos sistêmicos que comprometem a promessa espiritual do app:

1. **Fragilidade absoluta dos dados:** todo o progresso existe em um único local por dispositivo, sem backup, sem recuperação, sem sincronização. A troca de celular — evento absolutamente previsível no ciclo de vida de qualquer usuário — resulta em perda total e irreversível.

2. **PWA sem Service Worker:** o arquivo `service-worker.js` referenciado não existe. O app não funciona offline e não é tecnicamente um PWA completo.

3. **Segurança insuficiente para dados sensíveis:** o diário espiritual (conteúdo íntimo, reflexões pessoais) e as senhas dos tutores são armazenados sem proteção adequada, vulneráveis a XSS e acessíveis a qualquer pessoa com acesso ao dispositivo.

---

*Relatório gerado como análise estática do código-fonte. Nenhuma alteração foi implementada.*  
*Próximo passo sugerido: CODE_REVIEW.md (revisão geral de qualidade e bugs do código JS) ou SECURITY_REVIEW.md (análise focada nas vulnerabilidades identificadas).*
