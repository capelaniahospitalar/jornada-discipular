# JOURNAL_REVIEW.md
## Análise Profunda do Diário Espiritual — Aos Pés do Mestre Jesus
**Data:** 26 de junho de 2026  
**Versão analisada:** index (26).html  
**Escopo:** criação, edição, salvamento, persistência, pesquisa, organização, exportação, recuperação, segurança, privacidade  
**Análise estática — nenhuma alteração implementada**

---

## 1. ARQUITETURA DO DIÁRIO

### 1.1 Estrutura de dados

Cada entrada do diário é um objeto JavaScript com os seguintes campos:

```javascript
{
  study:    string,   // título do estudo associado (ex: "Jesus e a oração")
  date:     string,   // data formatada em pt-BR (ex: "26/06/2026")
  studyIdx: number,   // índice do estudo (0–15) ou null se não vinculado
  learned:  string,   // "O que aprendi"
  godSaid:  string,   // "O que Deus falou comigo"
  change:   string,   // "O que preciso mudar"
  prayer:   string,   // "Minha oração"
  decision: string    // "Minha decisão"
}
```

O diário completo é armazenado em `ST.journal` — um array de objetos de entrada. O estado `ST` é salvo em `localStorage` sob a chave `jornada_st`.

### 1.2 Integração com o estado do app

| Uso | Localização |
|---|---|
| Array de entradas | `ST.journal[]` |
| Pendência de escrita | `pendingJournalIdx`, `pendingJournalTitle` (variáveis globais em memória) |
| Contagem de registros | `(ST.journal||[]).length` (referenciado em 6 locais) |
| Verificação de entrada por estudo | `hasJournalForStudy(idx)` |
| Desbloqueio de missões | `check: () => (ST.journal||[]).length >= N` |

### 1.3 Fluxo completo

```
Completa estudo → showComplete()
  → pendingJournalIdx = curIdx
  → showJournalPrompt()  [prompt dourado na tela de conclusão]
  → updateCompleteHomeBtn()  [bloqueia botão "Voltar ao início"]

Usuário abre diário (obrigatório ou voluntário)
  → openJournal() → renderJournal()
  → aba "Novo" exibida automaticamente se pendingJournalIdx ativo

Usuário preenche campos e salva
  → saveJournalEntry(studyTitle)
  → ST.journal.push(entry)
  → save(ST)  [localStorage]
  → checkAutoMissions()
  → updateCompleteHomeBtn()  [desbloqueia se pendingJournalIdx satisfeito]
  → redireciona para tela de conclusão (se pendente) ou aba "Registros"

Back button
  → journalBack():
    → se pendente e não preenchido → volta à tela de conclusão (não à home)
    → se não pendente → vai à home
```

---

## 2. CRIAÇÃO DE ENTRADAS

### 2.1 Formulário de nova entrada

```html
<div class="journal-field"><label>O que aprendi</label>
  <textarea id="j-learned" rows="2" placeholder="O que este estudo me ensinou..."></textarea></div>
<div class="journal-field"><label>O que Deus falou comigo</label>
  <textarea id="j-godsaid" rows="2" placeholder="Uma palavra, versículo ou percepção..."></textarea></div>
<div class="journal-field"><label>O que preciso mudar</label>
  <textarea id="j-change" rows="2" placeholder="Uma área da minha vida..."></textarea></div>
<div class="journal-field"><label>Minha oração</label>
  <textarea id="j-prayer" rows="2" placeholder="Fale com Deus aqui..."></textarea></div>
<div class="journal-field"><label>Minha decisão</label>
  <textarea id="j-decision" rows="2" placeholder="O que vou fazer a partir de hoje..."></textarea></div>
```

**Avaliação do formulário:**

Os 5 campos são teologicamente bem escolhidos — cobrem os 3 movimentos clássicos da espiritualidade devocional (receber → processar → responder):
- Receber: "O que aprendi" + "O que Deus falou comigo"  
- Processar: "O que preciso mudar"  
- Responder: "Minha oração" + "Minha decisão"

Este mapeamento é conciso e não sobrecarrega — 5 campos curtos são mais preenchíveis do que 10 campos longos.

**Problemas identificados:**

**C1 — `rows="2"` é insuficiente para escrita espiritual**  
2 linhas de textarea com `font-size: 17.5px` equivalem a ~64px de altura. Para um usuário que deseja escrever uma oração genuína ou registrar uma experiência espiritual significativa, 2 linhas forçam um scroll interno imediato. A ausência de `auto-resize` (crescimento automático conforme digitação) torna a experiência frustrante.

**C2 — Sem autosave durante digitação**  
O usuário digita em 5 textareas separadas e só salva ao final via botão. Se ele sair da tela acidentalmente (back gesture no Android, chamada telefônica, reinício do browser), tudo o que escreveu é perdido. Não há debounce/autosave para o localStorage durante a digitação.

**C3 — Título do estudo hardcoded como último estudo**  
```javascript
const lastStudy = ST.done.length > 0 ? STUDIES[ST.done[ST.done.length-1]].title : 'Geral';
```
O título associado à entrada é sempre o último estudo concluído. Se o usuário abrir o diário voluntariamente (sem estar no fluxo obrigatório), a entrada será associada ao último estudo — mesmo que ele queira registrar algo não relacionado a nenhum estudo específico ou a um estudo anterior.

**C4 — Ausência de campo para data personalizada**  
A data é sempre a data atual de salvamento (`new Date().toLocaleDateString('pt-BR')`). O usuário não pode registrar um pensamento com uma data anterior (ex: "lembrei de algo que aconteceu ontem durante o estudo").

**C5 — Sem seleção manual do estudo associado**  
O campo `study` é sempre o último estudo concluído — o usuário não pode escolher qual estudo quer associar ao registro, nem criar um registro "avulso" sem associação de estudo. Isso limita o diário a ser estritamente um registro pós-estudo, não um diário espiritual geral.

---

## 3. EDIÇÃO DE ENTRADAS

### 3.1 Capacidade de edição atual

**Não existe edição.** As entradas são exibidas no modo de visualização apenas:

```javascript
body.innerHTML = entries.map(e => `
  <div class="journal-entry-wrap">
    <div class="journal-entry-hdr"><span>${e.study||'Registro'}</span><small>${e.date||''}</small></div>
    <div class="journal-entry-body">
      ${e.learned  ? `<div class="journal-field"><label>O que aprendi</label><p>${e.learned}</p></div>` : ''}
      ${e.godSaid  ? `... <p>${e.godSaid}</p>...</div>` : ''}
      ${e.change   ? `... <p>${e.change}</p>...</div>` : ''}
      ${e.prayer   ? `... <p>${e.prayer}</p>...</div>` : ''}
      ${e.decision ? `... <p>${e.decision}</p>...</div>` : ''}
    </div>
  </div>`).join('');
```

**Problemas:**

**E1 — Impossível corrigir erros tipográficos**  
Uma vez salva, uma entrada não pode ser corrigida. Um erro de digitação, uma frase incompleta ou uma reflexão que o usuário quer aprofundar ficam permanentes.

**E2 — Impossível excluir entradas**  
Não há botão de exclusão. Se o usuário criar uma entrada acidental (pressionou "Salvar" sem querer), ela fica permanentemente no diário.

**E3 — Impossível adicionar a uma entrada existente**  
Uma reflexão que o usuário queira complementar 2 dias depois exige criar uma nova entrada (sem vínculo com a anterior). Entradas anteriores são arquivos mortos.

---

## 4. SALVAMENTO

### 4.1 Mecanismo de salvamento

```javascript
function saveJournalEntry(studyTitle) {
  const get = id => document.getElementById(id)?.value.trim() || '';
  const entry = {
    study: studyTitle,
    date: new Date().toLocaleDateString('pt-BR'),
    studyIdx: pendingJournalIdx,
    learned:  get('j-learned'),
    godSaid:  get('j-godsaid'),
    change:   get('j-change'),
    prayer:   get('j-prayer'),
    decision: get('j-decision')
  };
  // Só salva se ao menos um campo de conteúdo não estiver vazio
  if (!Object.entries(entry)
    .filter(([k]) => !['study', 'date', 'studyIdx'].includes(k))
    .some(([, v]) => v)) return;

  if (!ST.journal) ST.journal = [];
  ST.journal.push(entry);
  save(ST);
  // ...
}
```

**O que funciona bem:**
- Validação implícita: não salva entradas completamente vazias (pelo menos um campo com conteúdo)
- `save(ST)` persiste imediatamente no localStorage
- `studyIdx` preserva o vínculo com o estudo correspondente

**Problemas:**

**S1 — `date` como string localizada sem timestamp**  
A data é salva como `new Date().toLocaleDateString('pt-BR')` → `"26/06/2026"`. Isso é uma string legível, mas:
- Não pode ser ordenada cronologicamente de forma confiável (comparação de strings "26/06/2026" vs "05/07/2026" falha com ordenação lexicográfica)
- Não inclui hora — duas entradas no mesmo dia são indistinguíveis cronologicamente
- Não inclui timezone — se o usuário trocar de fuso horário ou o dispositivo estiver com hora errada, as datas ficam inconsistentes
- Não pode ser reformatada para outros idiomas ou formatos no futuro (a string é o dado, não um timestamp)

**S2 — Nenhum ID único por entrada**  
As entradas não têm `id` — são referenciadas apenas por posição no array (`ST.journal[i]`). Se uma entrada for removida (ex: ao implementar exclusão no futuro), os índices de todas as entradas seguintes mudam — quebrando qualquer referência por índice.

**S3 — `pendingJournalIdx` é variável global em memória**  
```javascript
let pendingJournalIdx = null, pendingJournalTitle = null;
```
Se a página for recarregada enquanto há um estudo pendente de diário, `pendingJournalIdx` é zerado. O usuário pode então voltar à home sem ter registrado no diário — o bloqueio é contornado por recarga de página.

**S4 — Sem confirmação visual após salvamento**  
Após `saveJournalEntry()`, a interface redireciona silenciosamente (para a tela de conclusão ou aba de registros). Não há toast, animação ou mensagem de confirmação explícita de que o registro foi salvo com sucesso.

---

## 5. PERSISTÊNCIA

### 5.1 Mecanismo de persistência

```javascript
// Função save() — salva o objeto ST completo no localStorage
function save(ST) {
  localStorage.setItem('jornada_st', JSON.stringify(ST));
}
```

Todo o estado do aplicativo (incluindo `ST.journal`) é serializado como JSON e salvo em uma única chave `jornada_st`.

**Avaliação:**

| Aspecto | Status |
|---|---|
| Persistência básica | ✅ localStorage — sobrevive ao fechar o browser |
| Persistência entre recargas | ✅ Funcional |
| Persistência offline | ✅ localStorage não requer internet |
| Persistência multi-dispositivo | ❌ Não existe — vinculado ao dispositivo/browser |
| Persistência após limpar dados do browser | ❌ Perdido permanentemente |
| Persistência após desinstalar PWA | ❌ Depende do browser (geralmente perdido) |
| Backup automático | ❌ Não existe |
| Sincronização em nuvem | ❌ Não existe |

**P1 — Tamanho do localStorage**  
O localStorage tem limite de ~5MB por origem (varia por browser). Um diário com 16 entradas densas (cada uma com ~500 caracteres por campo × 5 campos = ~2.500 chars/entrada × 16 = ~40KB de texto) mais o conteúdo estático do app em `jornada_st` ficaria bem abaixo do limite. O risco de overflow é baixo para o caso de uso esperado, mas inexistente monitoramento do espaço usado.

**P2 — Dado único sem redundância**  
O diário existe em exatamente um lugar: `localStorage['jornada_st']` no dispositivo do usuário. Uma falha do dispositivo, uma formatação acidental, uma limpeza de cache ou uma troca de smartphone apaga irreversivelmente anos de reflexões espirituais. Para um diário que o app descreve como "registro espiritual" de uma jornada transformadora, esta fragilidade é o problema mais grave de todo o sistema.

**P3 — Serialização do objeto ST completo a cada save**  
Cada chamada a `save(ST)` serializa e salva o objeto ST **completo** — incluindo os 16 estudos completados, missões, atributos, badges, histórico, e o diário inteiro. À medida que o diário cresce, esta operação se torna progressivamente mais cara (mas ainda dentro de limites aceitáveis para o tamanho esperado).

---

## 6. VISUALIZAÇÃO E ORGANIZAÇÃO

### 6.1 Lista de entradas

```javascript
const entries = (ST.journal || []).slice().reverse();
```

As entradas são exibidas em **ordem cronológica inversa** (mais recente primeiro) — correto para um diário. `.slice()` cria uma cópia antes de reverter, evitando mutação do array original.

**Avaliação visual:**

Cada entrada exibe:
- Header navy (Lora) com título do estudo + data em texto opaco
- Corpo com os 5 campos preenchidos (campos vazios são omitidos)
- Labels em 12.5px uppercase — hierarquia clara
- Texto das entradas em 19.5px / line-height 1.6 — boa legibilidade

**Problemas de organização:**

**O1 — Sem agrupamento por fase ou estudo**  
Todos os registros aparecem em lista plana cronológica. Não há separação visual por fase da jornada (Iniciante / Aprendiz / Discípulo / Comprometido) ou por estudo. Com 16+ entradas, a lista se torna uma rolagem longa sem estrutura.

**O2 — Sem pesquisa ou filtro**  
Não existe campo de busca, filtro por estudo, filtro por data, ou qualquer mecanismo de recuperação seletiva de entradas. Para um usuário que quer reler o que escreveu sobre "perdão" (estudo 4), precisa scrollar manualmente toda a lista.

**O3 — Sem indicação de qual campo foi preenchido**  
No cabeçalho da entrada, não há indicação dos campos preenchidos. O usuário só descobre o conteúdo abrindo e scrollando a entrada.

**O4 — Entradas não são clicáveis/expansíveis**  
Todas as entradas são sempre expandidas (conteúdo completo visível). Com muitas entradas e texto longo em cada campo, a página fica extremamente longa e difícil de navegar.

**O5 — Sem contagem de palavras ou indicador de profundidade**  
O usuário não tem feedback sobre a riqueza de suas reflexões ao longo do tempo. Não há comparação entre entradas curtas e entradas profundas.

---

## 7. PESQUISA

**Ausência total de pesquisa.** Não existe nenhuma funcionalidade de busca no diário — nem por texto, nem por estudo, nem por data, nem por campo.

**PR1 — Impacto prático:**
- Com 16 entradas (um por estudo), o usuário pode scrollar manualmente — é viável.
- Com entradas extras (criadas voluntariamente), o volume cresce rapidamente.
- Um usuário que quer revisitar sua "oração do estudo 7" precisa abrir todas as entradas visualmente até encontrá-la.

**PR2 — Busca dentro de entradas:**  
Cada entrada exibe todo o texto sem destaque ou navegação. Para um campo "O que Deus falou comigo" com 5 parágrafos, o usuário lê tudo sem auxílio.

---

## 8. EXPORTAÇÃO

**Não existe exportação.** O diário espiritual não pode ser:
- Exportado como PDF
- Exportado como texto (.txt)
- Exportado como JSON
- Compartilhado como arquivo
- Enviado por e-mail
- Impresso diretamente

**EX1 — Impacto:**  
Para um registro que pode representar semanas ou meses de reflexões espirituais profundas, a impossibilidade de exportar é uma limitação séria. Um paciente hospitalizado que se recupera e quer levar consigo suas reflexões da jornada não tem como fazer isso.

**EX2 — Compartilhamento parcial via WhatsApp:**  
O painel do discípulo (`openPanel()`) permite enviar ao tutor via WhatsApp uma mensagem com estatísticas — mas **não inclui o conteúdo do diário**, apenas a contagem de registros:
```javascript
`📔 Registros no diário: ${(ST.journal||[]).length}`
```

Ou seja, o tutor sabe que o discípulo tem 7 registros, mas não tem acesso ao conteúdo de nenhum deles.

---

## 9. RECUPERAÇÃO

### 9.1 Recuperação de dados

**Não existe recuperação.** Se o localStorage for apagado, o diário é perdido permanentemente. Não há:
- Backup automático
- Backup manual
- Restauração a partir de arquivo
- Sincronização com servidor

**RC1 — O app não avisa o usuário sobre este risco.**  
Não há nenhuma mensagem no app explicando que os dados estão armazenados localmente e podem ser perdidos. Um usuário que limpa o cache do browser sem saber as implicações perde tudo.

### 9.2 Recuperação de pendência

**RC2 — `pendingJournalIdx` é volátil:**  
Se o usuário completar um estudo e a página for recarregada antes de registrar no diário, `pendingJournalIdx` (variável em memória) é zerada. O bloqueio do botão "Voltar ao início" é contornado — o usuário avança sem registrar.

```javascript
let pendingJournalIdx = null; // perdido em qualquer reload
```

**RC3 — Entrada parcialmente preenchida não é recuperada:**  
Se o usuário preenche 3 dos 5 campos e sai da tela (acidentalmente), ao voltar encontra o formulário vazio — sem nenhuma recuperação de rascunho (draft).

---

## 10. SEGURANÇA

### 10.1 Injeção de HTML

```javascript
body.innerHTML = entries.map(e => `
  ...
  ${e.learned ? `<div class="journal-field"><label>O que aprendi</label><p>${e.learned}</p></div>` : ''}
  ...
`).join('');
```

**⚠️ VULNERABILIDADE CRÍTICA — XSS via conteúdo do diário (SEG1):**  
O conteúdo de cada campo do diário (`e.learned`, `e.godSaid`, `e.change`, `e.prayer`, `e.decision`) é inserido diretamente no DOM via `innerHTML` **sem sanitização**.

Se um usuário (ou um ataque de injeção via qualquer outro vetor) inserir HTML ou JavaScript no campo do diário, esse código será executado quando a aba "Registros" for renderizada.

**Exemplo de ataque:**  
Um usuário digita no campo "Minha oração":  
```html
<img src="x" onerror="alert('XSS')">
```
Ao salvar e reabrir a aba de registros, `onerror` é executado.

**Vetor real:** embora o vetor principal seja o próprio usuário digitando em seus campos, vale notar:
1. Se o campo `study` for manipulado (ex: via `lastStudy` que vem de `STUDIES[i].title` — conteúdo estático do código — este vetor é seguro)
2. Os campos digitados pelo usuário (`j-learned`, etc.) são diretamente inseridos sem escape — este vetor é vulnerável

**Correção:** substituir `innerHTML` por `textContent` na renderização dos campos, ou usar uma função de escape de HTML antes da interpolação.

### 10.2 Injeção no `studyTitle`

```javascript
body.innerHTML = `
  ...
  <button class="nav-btn" onclick="saveJournalEntry('${lastStudy}')">Salvar registro</button>
  ...`;
```

O `lastStudy` é inserido dentro de um atributo `onclick` como string JavaScript. Se o título de um estudo contivesse aspas simples (`'`), quebraria a sintaxe do handler. Os títulos atuais são hardcoded no array `STUDIES[]` (sem aspas simples), então o risco atual é baixo — mas é uma prática insegura que poderia ser explorada se os títulos fossem dinâmicos no futuro.

### 10.3 Proteção de dados

**SEG2 — localStorage não é criptografado:**  
O conteúdo do diário é armazenado em texto plano no localStorage. Em um dispositivo compartilhado (ex: tablet de uso compartilhado em enfermaria hospitalar), qualquer pessoa com acesso ao browser pode abrir o DevTools e ler `localStorage.getItem('jornada_st')` — acessando todo o diário espiritual do usuário.

Este é um risco real no contexto hospitalar onde o app é utilizado — tablets institucionais compartilhados são comuns.

---

## 11. PRIVACIDADE

### 11.1 Armazenamento local vs. remoto

Todo o diário permanece no dispositivo do usuário — não é enviado a nenhum servidor automaticamente. Isso é positivamente diferente de muitos apps.

**Exceção:** a contagem de registros (`totalDiario`) é enviada ao Google Sheets via TUTOR_PANEL_URL quando o discípulo sincroniza seu progresso:

```javascript
totalDiario: (ST.journal || []).length
```

O **conteúdo** do diário nunca sai do dispositivo — apenas a contagem. Isso é adequado e respeita a privacidade do conteúdo espiritual.

### 11.2 Visibilidade pelo tutor

O tutor, via painel, vê apenas:
- Quantidade de registros no diário (`totalDiario`)
- Não tem acesso ao conteúdo de nenhuma entrada

Isso é correto — o diário é pessoal. O painel do tutor intencionalmente não expõe o conteúdo.

### 11.3 Ausência de PIN / bloqueio

**PRIV1 — Sem proteção de acesso ao diário:**  
O diário está acessível a qualquer pessoa que abra o app no dispositivo. Não há:
- PIN ou senha para acessar o diário
- Opção de bloquear a aba "Diário"
- Aviso de que o conteúdo é privado

No contexto de um tablet compartilhado em hospital, um familiar ou enfermeiro que pega o dispositivo pode acessar as reflexões íntimas do paciente.

### 11.4 Ausência de aviso de privacidade

**PRIV2 — O app não informa o usuário:**
- Onde os dados são armazenados (localStorage no dispositivo)
- Que os dados podem ser perdidos ao limpar o cache
- Que o conteúdo não é criptografado
- Que o tutor não tem acesso ao conteúdo (positivo, mas não comunicado)

---

## 12. INTEGRAÇÃO COM A JORNADA

### 12.1 Diário obrigatório pós-estudo

```javascript
pendingJournalIdx = curIdx;
pendingJournalTitle = s.title;
showJournalPrompt(s.title);
updateCompleteHomeBtn(); // bloqueia "Voltar ao início"
```

A obrigatoriedade do diário após cada estudo é a decisão pedagógica mais impactante do módulo. É teologicamente correta — a reflexão escrita consolida o aprendizado e cria responsabilidade pessoal.

**Avaliação:** o mecanismo de bloqueio funciona bem para o fluxo principal. A mensagem "Registre seu diário para continuar" no botão é clara.

**Problema:** `pendingJournalIdx` em memória (não salvo no `ST`) significa que o bloqueio é contornável por qualquer reload. Para uma funcionalidade pedagógica central, essa fragilidade reduz sua efetividade.

### 12.2 Missões vinculadas ao diário

```javascript
{ id:'mj_s3_pray', check: () => (ST.journal||[]).length >= 1 },  // Primeiro registro
{ id:'mj_s4_journal', check: () => (ST.journal||[]).length >= 2 }, // Segundo registro
```

As missões que verificam o diário usam apenas contagem total — não verificam se o conteúdo é genuíno ou se está vinculado ao estudo correto. Um usuário pode criar uma entrada vazia (o filtro bloqueia entradas totalmente vazias) com uma única letra em qualquer campo para destravar missões.

### 12.3 Botão de acesso na home

```javascript
document.getElementById('h-journal-btn').innerHTML = `
  <button onclick="openJournal()" ...>
    📔 Meu Diário Espiritual
    <span ...>${(ST.journal||[]).length} registro(s)</span>
  </button>`;
```

O botão de acesso ao diário na home mostra o contador de registros — boa visibilidade do progresso. O botão é `style="background:var(--card)"` — visualmente secundário em relação ao botão de estudo (navy) e ao AI (navy), o que é hierarquicamente correto.

---

## 13. ANÁLISE DE CAMPOS

### 13.1 Avaliação teológica dos 5 campos

| Campo | Pergunta | Movimento espiritual | Avaliação |
|---|---|---|---|
| `learned` | "O que aprendi" | Cognição / recepção | ✅ Fundamental — ancora o conhecimento |
| `godSaid` | "O que Deus falou comigo" | Encontro / percepção | ✅ O mais espiritualmente profundo |
| `change` | "O que preciso mudar" | Convicção / arrependimento | ✅ Crucial para transformação real |
| `prayer` | "Minha oração" | Resposta / diálogo | ✅ Fecha o ciclo devocional |
| `decision` | "Minha decisão" | Compromisso / ação | ✅ Transforma reflexão em prática |

Os 5 campos mapeiam perfeitamente o ciclo H.E.A.R. (Highlight + Explain + Apply + Respond) de discipleship journaling — uma das metodologias mais respeitadas de diário bíblico. A escolha é excelente e teologicamente madura.

### 13.2 Campos ausentes

| Campo potencial | Justificativa | Prioridade |
|---|---|---|
| Versículo marcante | Capturar o versículo que mais impactou | Média |
| Data personalizada | Permitir registros retrospectivos | Baixa |
| Humor espiritual | Tracking de estado emocional ao longo da jornada | Baixa |
| Sticker/emoji espiritual | Expressão rápida do estado de ânimo | Muito baixa |
| Vínculo com estudo específico (manual) | Permitir associar ao estudo desejado | Média |

Nenhum campo ausente é crítico — os 5 existentes são suficientes. A adição de mais campos riscaria sobrecarregar o formulário para usuários em contexto hospitalar.

---

## 14. TABELA-RESUMO DE PROBLEMAS

| ID | Categoria | Descrição | Gravidade |
|---|---|---|---|
| SEG1 | **Segurança** | XSS — conteúdo do diário inserido via `innerHTML` sem sanitização | 🔴 Crítico |
| P2 | **Persistência** | Dado único sem backup — perda total em caso de limpeza de cache | 🔴 Alto |
| C2 | **Criação** | Sem autosave — perda total em saída acidental durante digitação | 🔴 Alto |
| E1 | **Edição** | Impossível corrigir ou editar entradas após salvamento | 🔴 Alto |
| E2 | **Edição** | Impossível excluir entradas | 🟡 Médio |
| S1 | **Salvamento** | Data como string localizada — impossível ordenar ou reformatar | 🟡 Médio |
| S2 | **Salvamento** | Sem ID único por entrada — referência por índice frágil | 🟡 Médio |
| S3 | **Salvamento** | `pendingJournalIdx` em memória — bloqueio pedagógico contornável por reload | 🟡 Médio |
| S4 | **Salvamento** | Sem confirmação visual de salvamento | 🟡 Médio |
| RC1 | **Recuperação** | App não avisa sobre risco de perda de dados | 🟡 Médio |
| RC3 | **Recuperação** | Sem recuperação de rascunho (draft) após saída acidental | 🟡 Médio |
| EX1 | **Exportação** | Sem nenhuma forma de exportação do diário | 🟡 Médio |
| PRIV1 | **Privacidade** | Sem PIN/bloqueio — diário acessível a qualquer pessoa com o dispositivo | 🟡 Médio |
| PRIV2 | **Privacidade** | Sem aviso ao usuário sobre armazenamento local e riscos | 🟡 Médio |
| C1 | **Criação** | `rows="2"` — textarea muito pequena para reflexão espiritual profunda | 🟡 Médio |
| C3 | **Criação** | Título sempre vinculado ao último estudo — sem escolha manual | 🔵 Baixo |
| C4 | **Criação** | Sem campo de data personalizada | 🔵 Baixo |
| O1 | **Organização** | Sem agrupamento por fase ou estudo | 🔵 Baixo |
| O2 | **Pesquisa** | Sem campo de busca ou filtro | 🔵 Baixo |
| O4 | **Organização** | Entradas sempre expandidas — sem accordion/collapse | 🔵 Baixo |
| SEG2 | **Segurança** | localStorage em texto plano — risco em dispositivos compartilhados | 🔵 Baixo |

---

## 15. O QUE ESTÁ MUITO BEM

1. **5 campos teologicamente precisos** — mapeiam o ciclo completo de H.E.A.R. (Highlight, Explain, Apply, Respond). Escolha excepcional para discipulado.

2. **Ordem inversa na listagem** — entradas mais recentes primeiro, correto para diário.

3. **Omissão de campos vazios na visualização** — entradas com campos parcialmente preenchidos não exibem seções em branco, mantendo a visualização limpa.

4. **Integração com o fluxo obrigatório** — a obrigatoriedade do diário após cada estudo é pedagogicamente corajosa e espiritualmente sábia. Poucos apps de discipulado têm esse nível de comprometimento com a prática reflexiva.

5. **`pendingJournalIdx` no objeto entry** — o vínculo entre entrada e índice do estudo (`studyIdx`) está salvo no dado, permitindo consultas futuras como `hasJournalForStudy(idx)`.

6. **`slice().reverse()`** — cria cópia antes de reverter, sem mutação do array original. Código defensivo correto.

7. **Validação de entrada vazia** — `saveJournalEntry()` não salva entradas completamente vazias. Proteção básica mas essencial.

8. **Privacidade do conteúdo** — o tutor nunca vê o conteúdo do diário, apenas a contagem. Limite de privacidade correto e respeitoso.

9. **`journalBack()` com lógica pedagógica** — o botão de voltar retorna à tela de conclusão (não à home) se o diário pendente não foi preenchido. Mantém o fluxo obrigatório mesmo ao tentar sair.

10. **Contagem de registros na home** — o botão do diário mostra `N registro(s)` — feedback motivador do volume acumulado de reflexões.

---

## 16. RECOMENDAÇÕES POR PRIORIDADE

### Prioridade 1 — Críticas (implementar antes de uso em produção)

**R1 — Corrigir XSS na renderização das entradas (SEG1)**

Substituir a interpolação direta por uma função de escape:

```javascript
function escapeHtml(str) {
  return (str || '')
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#039;');
}
// Uso:
${e.learned ? `<p>${escapeHtml(e.learned)}</p>` : ''}
```

**R2 — Adicionar autosave de rascunho (C2)**

Salvar o rascunho no `localStorage` durante a digitação com debounce:

```javascript
const DRAFT_KEY = 'jornada_journal_draft';
// Ao digitar:
textarea.addEventListener('input', debounce(() => {
  localStorage.setItem(DRAFT_KEY, JSON.stringify({ learned, godSaid, change, prayer, decision }));
}, 1500));
// Ao abrir o formulário:
const draft = JSON.parse(localStorage.getItem(DRAFT_KEY) || 'null');
if (draft) { /* prefill textareas */ }
// Ao salvar com sucesso:
localStorage.removeItem(DRAFT_KEY);
```

**R3 — Implementar edição de entradas (E1)**

Adicionar botão "Editar" no header de cada entrada que substitui os `<p>` por `<textarea>` pré-preenchidos e exibe um botão "Salvar alterações".

```javascript
function editJournalEntry(idx) {
  // Converter visualização em modo de edição para a entrada [idx]
}
function updateJournalEntry(idx) {
  ST.journal[idx] = { ...ST.journal[idx], /* campos atualizados */ };
  save(ST);
}
```

### Prioridade 2 — Importantes

**R4 — Salvar data como timestamp ISO (S1)**

```javascript
const entry = {
  // ...
  date: new Date().toLocaleDateString('pt-BR'), // manter para exibição
  ts:   new Date().toISOString(),               // timestamp para ordenação e filtros
};
```

**R5 — Adicionar ID único por entrada (S2)**

```javascript
const entry = {
  id: Date.now() + '-' + Math.random().toString(36).slice(2, 7),
  // ...
};
```

**R6 — Persistir `pendingJournalIdx` no ST (S3)**

```javascript
// Ao completar estudo:
ST.pendingJournalIdx = curIdx;
save(ST);

// Ao salvar diário:
ST.pendingJournalIdx = null;
save(ST);

// Na função hasJournalForStudy, verificar também ST.pendingJournalIdx
```

**R7 — Implementar exportação básica (EX1)**

Botão "Exportar diário" na aba "Registros" que gera um arquivo de texto:

```javascript
function exportJournal() {
  const text = (ST.journal || []).map(e => [
    `=== ${e.study} — ${e.date} ===`,
    e.learned  ? `O que aprendi:\n${e.learned}` : '',
    e.godSaid  ? `O que Deus falou:\n${e.godSaid}` : '',
    e.change   ? `O que preciso mudar:\n${e.change}` : '',
    e.prayer   ? `Minha oração:\n${e.prayer}` : '',
    e.decision ? `Minha decisão:\n${e.decision}` : '',
  ].filter(Boolean).join('\n\n')).join('\n\n' + '─'.repeat(40) + '\n\n');

  const blob = new Blob([text], { type: 'text/plain;charset=utf-8' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = `diario-espiritual-${ST.userName || 'discipulo'}.txt`;
  a.click();
  URL.revokeObjectURL(url);
}
```

**R8 — Aumentar `rows` e adicionar auto-resize (C1)**

```html
<textarea id="j-learned" rows="4" ...></textarea>
```

```javascript
// Auto-resize ao digitar:
textarea.addEventListener('input', function() {
  this.style.height = 'auto';
  this.style.height = this.scrollHeight + 'px';
});
```

**R9 — Adicionar confirmação visual após salvamento (S4)**

```javascript
// Após save(ST):
const toast = document.createElement('div');
toast.textContent = '✓ Registro salvo';
toast.style.cssText = 'position:fixed;bottom:80px;left:50%;transform:translateX(-50%);background:var(--teal);color:#fff;padding:.5rem 1.25rem;border-radius:20px;font-size:16px;z-index:200;animation:fadeout 2s forwards';
document.body.appendChild(toast);
setTimeout(() => toast.remove(), 2000);
```

**R10 — Excluir entradas (E2)**

Botão "Excluir" com confirmação inline (não `confirm()` nativo):

```javascript
function deleteJournalEntry(idx) {
  ST.journal.splice(idx, 1);
  save(ST);
  renderJournal();
}
```

### Prioridade 3 — Melhorias

**R11 — Colapsar entradas com accordion (O4)**

Exibir apenas o cabeçalho (estudo + data) por padrão, expandindo ao toque.

**R12 — Filtro por estudo (O2)**

Dropdown com os estudos concluídos para filtrar entradas por estudo.

**R13 — Aviso sobre armazenamento local (PRIV2, RC1)**

Mensagem discreta na aba "Registros" (na primeira abertura):
> "📱 Seus registros são salvos apenas neste dispositivo. Para não perdê-los, exporte-os periodicamente."

**R14 — Adicionar opção de PIN (PRIV1)**

Campo de PIN de 4 dígitos opcional na tela de configurações, verificado ao abrir a tela do diário.

---

## 17. SÍNTESE ARQUITETURAL — PONTOS CRÍTICOS

```
ESTADO ATUAL DO DIÁRIO:

FORÇA:   Formulário de 5 campos pedagogicamente excelente
FORÇA:   Integração obrigatória com o fluxo de estudos
FORÇA:   Privacidade respeitada (tutor não vê conteúdo)
FORÇA:   Dado permanece no dispositivo (offline-first)

RISCO 1: XSS — conteúdo do usuário inserido sem escape [CRÍTICO]
RISCO 2: Dado único sem backup — perda irreversível [CRÍTICO]
RISCO 3: Sem autosave — perda por saída acidental [ALTO]
RISCO 4: Sem edição — entrada errada é permanente [ALTO]

LACUNA 1: Sem exportação — reflexões espirituais presas no dispositivo
LACUNA 2: Sem pesquisa — impossível recuperar reflexão específica
LACUNA 3: Sem PIN — diário íntimo acessível a qualquer um com o device
```

---

## 18. CONCLUSÃO

O Diário Espiritual tem uma **fundamentação teológica e pedagógica superior** à maioria dos aplicativos de discipulado. Os 5 campos cobrem o ciclo completo de reflexão espiritual. A integração obrigatória com o fluxo dos estudos é uma decisão corajosa que garante a prática reflexiva.

Os problemas mais urgentes são técnicos, não conceituais:

1. **A vulnerabilidade XSS** precisa ser corrigida antes de qualquer uso em produção — é o único problema de segurança ativo no código do diário.

2. **A ausência de backup/exportação** é o risco mais impactante para o usuário final — reflexões espirituais de semanas ou meses podem ser perdidas por uma limpeza de cache acidental.

3. **A imutabilidade das entradas** (sem edição, sem exclusão) limita a maturidade do diário como ferramenta espiritual de longo prazo.

Com as correções de prioridade 1 e 2 implementadas, o Diário Espiritual se tornaria um dos recursos mais diferenciados e confiáveis de todo o aplicativo — à altura da qualidade espiritual do conteúdo dos estudos que o alimentam.

---

*Relatório gerado por análise estática do código-fonte (index.html, linhas 295–308, 640, 750–766, 2850–2927 e referências ao objeto `ST.journal` em toda a base de código).*  
*Nenhuma modificação foi feita ao código do aplicativo.*  
*Data: 26 de junho de 2026*
