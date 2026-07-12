# CHANGELOG

## [1.1.0] — 2026-06-26

### Segurança
- **F0.1** Eliminado XSS em `renderJournal()` e `N()` — `escapeHTML()` aplicada em todos os pontos de injeção de dados do usuário via `innerHTML`
- **F0.2** Senhas dos tutores migradas de texto puro para SHA-256 com salt (`amj_salt_2026_`); migração automática de senhas legadas no primeiro login
- **F1.5** Cookie `amj3` agora usa `SameSite=Strict` e flag `Secure` (condicional em HTTPS)

### Correções Críticas de Conteúdo
- **F0.3** Removida afirmação científica incorreta sobre digestão suína do Estudo 8; substituída por argumento exclusivamente teológico (sabedoria do Criador em Levítico 11)
- **F2.1** Corrigida numeração dos recaps dos Estudos 10–16 (estavam deslocados em -1): agora exibem corretamente Estudo 10, 11, 12, 13, 14, 15, 16
- **F2.2** Título do Estudo 14 corrigido: "Jesus e o cansaço" → "Jesus e o Descanso Sagrado" (título agora corresponde ao conteúdo sobre o 2º e 4º mandamentos)
- **F2.3** Parágrafo de abertura do Estudo 14 reescrito com tom pastoral — removida linguagem confrontadora; substituída por convite ao exame das Escrituras

### Integridade de Dados
- **F1.1** Adicionado versionamento de schema (`ST._v = 1`) e função `migrateSchema()` — garante que todos os campos obrigatórios existam, extensível para versões futuras
- **F1.2** Campo de nome usa `debouncedSave()` (400ms) em vez de `save()` a cada keystroke — elimina JSON.stringify + 3 writes síncronos por tecla
- **F1.3** `confirmRestart()` agora zera `ST.badges`, `ST.obstacleActions` e `ST.missionsDone` — estado visual após restart é completamente consistente com progresso zero
- **F1.4** `save()` detecta `QuotaExceededError` e exibe toast não-bloqueante orientando o usuário a exportar o diário

### PWA & Performance
- **F3.1** Google Fonts (`fonts.gstatic.com`, `fonts.googleapis.com`) agora cacheadas pelo Service Worker com estratégia Cache-First + revalidação em background — tipografia disponível offline
- **F3.2** Adicionados `<link rel="preconnect">` para fonts.googleapis.com e fonts.gstatic.com; `dns-prefetch` para o AI Proxy — reduz FCP em 200–500ms
- **F3.3** Ícones no `<head>` migrados de URLs absolutas (`capelaniahospitalar.github.io/...`) para paths relativos (`./icon-192.png`)
- **F3.5** Criado pipeline CI/CD (`.github/workflows/build.yml`) com `html-minifier-terser` e deploy automático para GitHub Pages; adicionado `package.json`

### UX & Acessibilidade
- **F4.1** Botão flutuante do WhatsApp movido de `left: .85rem` para `right: .85rem` — zona natural do polegar direito
- **F4.2** Tela de IA usa `height: 100dvh` com fallback `100vh`; listener `visualViewport.resize` ajusta altura quando teclado iOS abre
- **F4.3** Todos os `alert()` (3) e `confirm()` (1) nativos substituídos por `showModal()` estilizado — consistente com identidade visual do app

### Mentor IA
- **F5.1** Respostas da IA renderizam Markdown básico (`**negrito**`, `*itálico*`, `- lista`, parágrafos) via `renderMarkdown()` — pipeline seguro: `escapeHTML` antes de qualquer formatação
- **F5.2** `aiHistory.push({ role:'user' })` movido para após resposta bem-sucedida — histórico nunca contém mensagens do usuário sem resposta correspondente

### Diário Espiritual
- **F6.1** Datas das entradas do diário armazenadas em ISO 8601 (`toISOString()`) — ordenável e locale-independente; exibição formatada em português via `formatJournalDate()`
- **F6.3** Adicionada exportação do diário em `.txt` (legível) via botão "⬇ Exportar" na tela do diário; função `exportJournalJSON()` disponível para backup técnico

### Funções Utilitárias Adicionadas
- `escapeHTML(str)` — sanitização XSS para todos os pontos de injeção HTML
- `debouncedSave()` — debounce de 400ms para `save()`
- `showModal({ title, message, actions })` / `closeModal()` — sistema de modal estilizado
- `showStorageWarning()` — toast de aviso de armazenamento cheio
- `renderMarkdown(text)` — parser Markdown mínimo seguro (escape-first)
- `hashPass(pass)` / `isPassHash(s)` — SHA-256 com salt para senhas de tutores
- `formatJournalDate(val)` — formatação de datas ISO para exibição em pt-BR
- `downloadFile(filename, content, type)` — helper de download de arquivos
- `exportJournalText()` / `exportJournalJSON()` — exportação do diário
- `migrateSchema(st)` — migração de dados e normalização do ST
- `doRestart()` — lógica de reset separada de `confirmRestart()`

---

## [1.0.0] — 2026-06-25 (baseline)

Versão inicial auditada: `index.html` 4069 linhas, 347KB.  
Funcionalidades: 16 estudos, Mentor IA, Diário Espiritual, Painel do Tutor, PWA básico.
