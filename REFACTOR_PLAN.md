# REFACTOR_PLAN.md — Plano Completo de Refatoração
**Aplicativo:** Aos Pés do Mestre Jesus  
**Versão base:** index.html (bed2252) · 4069 linhas · 347 KB  
**Origem:** Síntese das 7 auditorias: SPIRITUAL_JOURNEY · MENTOR_AI · UX · JOURNAL · DATA · PERFORMANCE · THEOLOGICAL  
**Data:** 26/06/2026  
**Status:** Plano aprovado para execução futura — **nenhuma alteração implementada**

---

## Convenções deste documento

- **Fase:** conjunto coeso de mudanças que pode ser entregue e testado de forma independente
- **Cada fase tem:** escopo → arquivos → critérios de teste → definição de "pronto"
- **Nenhuma fase depende de outra** para estar funcional — o app sempre opera em estado válido entre fases
- **Origem:** cada item referencia a auditoria que o identificou

```
P0 = Crítico — segurança, corrupção de dados, erros factuais
P1 = Alta — impacto direto no usuário, regressões visíveis
P2 = Média — qualidade, consistência, UX
P3 = Baixa — melhorias incrementais, conteúdo novo
```

---

## Visão Geral das Fases

| Fase | Nome | Prioridade | Natureza | Esforço estimado |
|------|------|-----------|---------|-----------------|
| [F0](#fase-0--segurança-crítica) | Segurança Crítica | 🔴 P0 | Correção de bugs | Pequeno |
| [F1](#fase-1--integridade-de-dados) | Integridade de Dados | 🔴 P0 | Correção de bugs | Médio |
| [F2](#fase-2--correções-de-conteúdo) | Correções de Conteúdo | 🔴 P0 | Conteúdo | Pequeno |
| [F3](#fase-3--pwa--performance) | PWA & Performance | 🟠 P1 | Otimização | Médio |
| [F4](#fase-4--ux--acessibilidade) | UX & Acessibilidade | 🟠 P1 | UX | Médio |
| [F5](#fase-5--mentor-ia) | Mentor IA | 🟠 P1 | Feature | Médio |
| [F6](#fase-6--diário-espiritual) | Diário Espiritual | 🟡 P2 | Feature | Grande |
| [F7](#fase-7--aprofundamento-teológico) | Aprofundamento Teológico | 🟡 P2 | Conteúdo | Grande |
| [F8](#fase-8--arquitetura) | Arquitetura | 🟢 P3 | Refatoração | Grande |

---

## FASE 0 — Segurança Crítica

**Objetivo:** eliminar as três vulnerabilidades que expõem dados do usuário ou comprometem a integridade da informação apresentada.  
**Pode ser implementada em:** 1 sessão de trabalho focada.  
**Não depende de nenhuma outra fase.**

---

### F0.1 — Corrigir XSS no Diário Espiritual

**Origem:** JOURNAL_REVIEW.md §10, DATA_REVIEW.md §18.3  
**Severidade:** 🔴 Crítica — execução arbitrária de JavaScript via entrada do usuário

**Problema:**  
`renderJournal()` injeta entradas do diário via `innerHTML` sem sanitização:
```javascript
// Linha ~2890 — vulnerável
body.innerHTML = entries.map(e => `...<p>${e.learned}</p>...`).join('');
```
Um campo como `learned = "<img src=x onerror='fetch(atob(...))'>"` executa código arbitrário, podendo exfiltrar todo o localStorage (incluindo senhas dos tutores).

**Solução:**  
Substituir a renderização por criação de elementos DOM com `textContent`, ou aplicar uma função de escape HTML antes de toda injeção via `innerHTML`. Criar função utilitária `escapeHTML(str)`:

```javascript
// Implementar função — não injetar diretamente
function escapeHTML(str) {
  return String(str)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#039;');
}
```

Aplicar em todos os pontos onde dados de entrada do usuário são inseridos em HTML: `learned`, `apply`, `prayer`, `chosen`, `verse`, `userName`.

**Arquivos:** `index.html` (função `renderJournal`, e qualquer ponto de `innerHTML` com dados do ST)

**Critérios de teste:**
- [ ] Salvar entrada de diário com payload `<img src=x onerror=alert(1)>` em qualquer campo
- [ ] Abrir o diário — o payload deve aparecer como texto literal, não executar
- [ ] Verificar que nomes de usuário com caracteres especiais (`O'Brien`, `<José>`) renderizam corretamente
- [ ] Testar com payload `<script>document.title='XSS'</script>` — título da página não deve mudar
- [ ] Confirmar que entradas normais continuam exibindo quebras de linha e formatação básica

**Definição de pronto:** nenhum conteúdo de entrada do usuário pode ser interpretado como HTML pelo navegador.

---

### F0.2 — Proteger Senhas dos Tutores

**Origem:** DATA_REVIEW.md §16, JOURNAL_REVIEW.md §10 (XSS como vetor)  
**Severidade:** 🔴 Crítica — senhas em texto puro acessíveis via DevTools e por XSS

**Problema:**  
```javascript
// Chave amjTutorPass — conteúdo atual
{ "Capelão Wladimir": "senha123", "Capelão Renan": "outrasenha" }
```
Qualquer pessoa com acesso ao dispositivo, ao console do navegador, ou que explore o XSS do item F0.1, obtém as senhas dos tutores.

**Solução em dois níveis:**

**Nível mínimo (implementável sem backend):**  
Substituir armazenamento de senha em texto puro por hash SHA-256 com salt fixo. Na autenticação, hashear a entrada e comparar com o hash armazenado.

```javascript
// Antes de armazenar
async function hashPass(pass) {
  const encoded = new TextEncoder().encode('amj_salt_2026_' + pass);
  const buf = await crypto.subtle.digest('SHA-256', encoded);
  return Array.from(new Uint8Array(buf)).map(b => b.toString(16).padStart(2,'0')).join('');
}
```

**Nível ideal (requer backend):**  
Mover a autenticação do tutor inteiramente para o servidor (Google Apps Script pode validar credenciais via POST sem expor hashes ao cliente). O cliente envia a senha e recebe um token temporário.

**Arquivos:** `index.html` (funções `getTutorPassStore`, `setTutorPass`, `getTutorPass`, e toda lógica de autenticação do painel)

**Critérios de teste:**
- [ ] Abrir DevTools → Application → Local Storage: a chave `amjTutorPass` não deve conter senhas legíveis
- [ ] Autenticar com senha correta — deve funcionar normalmente
- [ ] Autenticar com senha incorreta — deve ser rejeitada
- [ ] Testar XSS: com o F0.1 corrigido, confirmar que mesmo se houvesse XSS, não haveria senha legível para exfiltrar
- [ ] Tutor que esqueceu a senha — documentar o fluxo de reset

**Definição de pronto:** `localStorage.getItem('amjTutorPass')` não retorna nenhuma senha legível por humanos.

---

### F0.3 — Remover Afirmação Científica Incorreta (Estudo 8)

**Origem:** THEOLOGICAL_REVIEW.md §11 (Estudo 8), §A3  
**Severidade:** 🔴 Crítica — informação factualmente errada compromete a credibilidade do material

**Problema:**  
O Estudo 8 afirma:
> *"os porcos digerem alimentos em apenas 4 horas, acumulando toxinas que permanecem na carne"*

Essa afirmação é cientificamente incorreta. O tempo de digestão suíno é similar ao de outros mamíferos (4–6 horas), e não há mecanismo de acumulação de toxinas superior a outros animais. A afirmação também é feita para frutos do mar de forma igualmente imprecisa.

**Solução:**  
Remover os parágrafos com afirmações científicas não verificadas. O argumento teológico de Levítico 11 como sabedoria do Criador é suficiente por si mesmo. A substituição pode focar no princípio bíblico sem necessidade de validação científica incorreta:

> *"Deus, como Criador, conhece o funcionamento do corpo humano melhor do que qualquer nutricionista. Os princípios dietéticos de Levítico 11 não precisam de validação científica para serem seguidos — mas a ciência moderna tem cada vez mais reconhecido os benefícios de dietas baseadas em alimentos íntegros e minimamente processados."*

**Arquivos:** `index.html` (Estudo 8, seção `type:'content'`, parágrafos sobre animais limpos/impróprios)

**Critérios de teste:**
- [ ] Ler o Estudo 8 completo — nenhuma afirmação sobre tempo de digestão suíno deve estar presente
- [ ] Verificar que a recomendação dietética bíblica (Levítico 11) permanece intacta
- [ ] Verificar que a citação de Gênesis 1.29 (dieta original) permanece
- [ ] Verificar que Daniel 1.12-15 permanece como exemplo bíblico
- [ ] Pedir ao Mentor IA uma pergunta sobre a dieta bíblica — a resposta não deve reproduzir o argumento removido

**Definição de pronto:** o Estudo 8 não contém nenhuma afirmação científica sobre mecanismos biológicos que não possa ser verificada e referenciada.

---

**Relatório de conclusão da Fase 0:**
Ao final desta fase: XSS eliminado, senhas protegidas, erro factual removido. O app está seguro para uso em produção do ponto de vista das vulnerabilidades críticas.

---

## FASE 1 — Integridade de Dados

**Objetivo:** garantir que os dados do usuário nunca sejam perdidos silenciosamente, que o estado do app seja sempre consistente, e que futuras atualizações não corrompam dados existentes.  
**Pode ser implementada em:** 1–2 sessões de trabalho.  
**Não depende de nenhuma outra fase.**

---

### F1.1 — Versionamento de Esquema e Migração

**Origem:** DATA_REVIEW.md §12  
**Severidade:** 🟠 Alta — qualquer mudança futura no esquema do ST corrompe dados de usuários existentes

**Problema:**  
O objeto ST não possui campo de versão. Se uma futura atualização renomear, adicionar ou remover campos, usuários com dados antigos podem perder progresso silenciosamente ou encontrar erros de runtime.

**Solução:**  
Adicionar campo `_v` ao ST e função `migrateSchema(st)` que é chamada após `load()`:

```javascript
const SCHEMA_VERSION = 1;

function migrateSchema(st) {
  // Sem versão = dados legados (pré-versionamento)
  if (!st._v) {
    // Garantir todos os campos obrigatórios
    st.done        = st.done || [];
    st.xp          = st.xp  || 0;
    st.badges      = st.badges || [];
    st.streak      = st.streak || 0;
    st.lastDate    = st.lastDate || null;
    st.journal     = st.journal || [];
    st.missions    = st.missions || {};
    st.completedAt = st.completedAt || {};
    st.obstacleActions = st.obstacleActions || {};
    st.missionsDone = st.missionsDone || [];
    st._v = 1;
  }
  // Futuras migrações: if (st._v === 1) { ... st._v = 2; }
  return st;
}

// Após load():
let ST = migrateSchema(load());
```

**Arquivos:** `index.html` (após a função `load()`, linha ~3449)

**Critérios de teste:**
- [ ] Simular dados antigos (sem `_v`) no localStorage e recarregar — ST deve ser migrado corretamente
- [ ] Verificar que `ST._v === 1` após a migração
- [ ] Adicionar manualmente um campo ausente ao localStorage e confirmar que a migração o preenche com o valor padrão
- [ ] Confirmar que usuários com dados completos não têm nenhuma alteração ao migrar
- [ ] Confirmar que `save(ST)` persiste `_v` corretamente

**Definição de pronto:** o ST sempre tem `_v` definido após `load()`, e a função de migração é extensível para versões futuras.

---

### F1.2 — Debounce no Campo de Nome

**Origem:** DATA_REVIEW.md §5.1, PERFORMANCE_REVIEW.md §5.1  
**Severidade:** 🟠 Alta — `save()` disparado a cada keystroke executa JSON.stringify + 3 writes síncronos

**Problema:**  
```html
<!-- Linha 483 — atual -->
<input oninput="ST.userName=this.value.trim();save(ST)">
```
Com ST de 100KB+ (diário extenso), cada tecla pode bloquear o thread principal por 5–15ms, causando jank perceptível.

**Solução:**  
```html
<input oninput="ST.userName=this.value.trim();debouncedSave()">
```
```javascript
// Adicionar função utilitária
let _saveTimer = null;
function debouncedSave() {
  if (_saveTimer) clearTimeout(_saveTimer);
  _saveTimer = setTimeout(() => { save(ST); _saveTimer = null; }, 400);
}
```

**Arquivos:** `index.html` (input `user-name-input`, linha ~483, e nova função `debouncedSave`)

**Critérios de teste:**
- [ ] Digitar um nome rapidamente (10+ teclas em 1 segundo) — o localStorage deve ser atualizado apenas uma vez após parar de digitar
- [ ] Confirmar que o nome é salvo corretamente após 400ms de inatividade
- [ ] Recarregar a página — o nome digitado deve ser recuperado corretamente
- [ ] Testar com ST de tamanho grande (simular diário extenso) — sem jank perceptível ao digitar
- [ ] Confirmar que outros campos não-nome não foram afetados

**Definição de pronto:** `localStorage.setItem` é chamado no máximo uma vez por burst de digitação, com delay de 400ms após o último keystroke.

---

### F1.3 — Corrigir `confirmRestart()` — Campos Não Zerados

**Origem:** DATA_REVIEW.md §14, THEOLOGICAL_REVIEW.md (mencionado indiretamente)  
**Severidade:** 🟠 Alta — badges e obstáculos aparecem como conquistados sem o progresso correspondente

**Problema:**  
```javascript
// confirmRestart() — linha 3906
// NÃO zerados atualmente:
ST.badges          // badges de níveis aparecem sem estudos concluídos
ST.obstacleActions // obstáculos aparecem como vencidos sem estudos
```

**Solução:**
```javascript
function confirmRestart() {
  // Substituir confirm() nativo por modal estilizado (ver F4.3)
  // ...
  ST.done            = [];
  ST.xp              = 0;
  ST.streak          = 0;
  ST.lastStudy       = null;
  ST.lastDate        = null;
  ST.completedAt     = {};
  ST.missions        = {};
  ST.missionsDone    = [];
  ST.badges          = [];          // ← ADICIONAR
  ST.obstacleActions = {};          // ← ADICIONAR
  pendingJournalIdx   = null;
  pendingJournalTitle = null;
  save(ST);
  goHome();
}
```

**Arquivos:** `index.html` (função `confirmRestart`, linha ~3906)

**Critérios de teste:**
- [ ] Completar 3 estudos e conquistar badges, depois fazer restart
- [ ] Verificar que a tela de badges não mostra nenhuma conquista após o restart
- [ ] Verificar que o mapa de obstáculos aparece limpo (nenhum obstáculo vencido)
- [ ] Confirmar que nome, perfil, tutor e diário são preservados
- [ ] Verificar que `ST.badges` é `[]` no localStorage após o restart
- [ ] Verificar que XP e streak estão zerados na home

**Definição de pronto:** após `confirmRestart()`, o estado visual do app é completamente consistente com `ST.done = []` — nenhuma conquista visual persiste sem o progresso correspondente.

---

### F1.4 — Aviso de Falha de Gravação

**Origem:** DATA_REVIEW.md §3, §19  
**Severidade:** 🟠 Alta — localStorage cheio perde dados silenciosamente

**Problema:**  
```javascript
try { localStorage.setItem('amj3', data); } catch {}
// QuotaExceededError é silenciado — usuário perde progresso sem saber
```

**Solução:**  
Detectar `QuotaExceededError` e exibir aviso não-bloqueante:
```javascript
function save(s) {
  const data = JSON.stringify(s);
  let saved = false;
  try {
    localStorage.setItem('amj3', data);
    saved = true;
  } catch (e) {
    if (e.name === 'QuotaExceededError' || e.name === 'NS_ERROR_DOM_QUOTA_REACHED') {
      showStorageWarning();
    }
  }
  // ... sessionStorage e cookie como antes
}

function showStorageWarning() {
  // Toast não-intrusivo (ver F4 para sistema de toast)
  // "Seu armazenamento está cheio. Exporte seu diário para liberar espaço."
}
```

**Arquivos:** `index.html` (função `save`, linha ~3419, e nova função `showStorageWarning`)

**Critérios de teste:**
- [ ] Simular localStorage cheio (preencher com dados até quota) e tentar salvar — o aviso deve aparecer
- [ ] Verificar que o aviso não bloqueia a UI (não é um `alert()`)
- [ ] Verificar que dados em sessionStorage e cookie ainda são tentados mesmo com LS cheio
- [ ] Com localStorage normal, confirmar que o aviso não aparece
- [ ] O botão de exportação (Fase 6) deve ser acessível a partir do aviso

**Definição de pronto:** o usuário é informado quando o armazenamento está cheio, com orientação clara sobre como proceder.

---

### F1.5 — Cookie com Flags de Segurança

**Origem:** DATA_REVIEW.md §8  
**Severidade:** 🟠 Alta — cookie sem `Secure` e sem `HttpOnly` é vulnerável

**Problema:**  
```javascript
// Linha ~3424
document.cookie = 'amj3=' + encodeURIComponent(data) + ';expires=' + exp.toUTCString() + ';path=/;SameSite=Lax';
// Faltam: Secure e HttpOnly
```

**Solução:**  
```javascript
// Adicionar Secure (só ativo em HTTPS — GitHub Pages sempre usa HTTPS)
const secureFlag = location.protocol === 'https:' ? ';Secure' : '';
document.cookie = 'amj3=' + encodeURIComponent(data) +
  ';expires=' + exp.toUTCString() +
  ';path=/;SameSite=Strict' + secureFlag;
// Nota: HttpOnly não é definível via document.cookie — requer Set-Cookie do servidor
// SameSite=Strict em vez de Lax oferece proteção CSRF adicional
```

**Arquivos:** `index.html` (função `save`, linha ~3424)

**Critérios de teste:**
- [ ] Em HTTPS (GitHub Pages): DevTools → Application → Cookies — cookie `amj3` deve ter flag `Secure`
- [ ] Em HTTP local (desenvolvimento): cookie deve ser criado sem `Secure` (não quebrar desenvolvimento)
- [ ] `SameSite=Strict` verificado no inspetor
- [ ] Recarregar a página — dados recuperados corretamente do cookie quando localStorage está vazio

**Definição de pronto:** o cookie `amj3` inclui `Secure` em HTTPS e `SameSite=Strict`.

---

**Relatório de conclusão da Fase 1:**
Ao final desta fase: esquema versionado, salvamento sem jank, restart consistente, falhas de gravação avisadas, cookie seguro.

---

## FASE 2 — Correções de Conteúdo

**Objetivo:** corrigir todos os erros de conteúdo identificados nas auditorias — numeração, títulos, contexto bíblico e tom pastoral.  
**Pode ser implementada em:** 1 sessão de trabalho focada.  
**Não depende de nenhuma outra fase.**

---

### F2.1 — Corrigir Numeração dos Recaps (Estudos 10–16)

**Origem:** SPIRITUAL_JOURNEY_REVIEW.md §recapitulação, THEOLOGICAL_REVIEW.md §10.3  
**Severidade:** 🔴 P0 — erro factual no texto do app

**Problema:** Os cartões de recapitulação dos estudos 10–16 exibem numeração incorreta (deslocada em -1):

| Estudo real | Rótulo atual | Rótulo correto |
|-------------|--------------|---------------|
| Estudo 10 ("Jesus irá voltar?") | "Recapitulação — Estudo 9" | "Recapitulação — Estudo 10" |
| Estudo 11 ("Permanecendo com Jesus") | "Recapitulação — Estudo 10" | "Recapitulação — Estudo 11" |
| Estudo 12 ("Jesus quer morar com você") | "Recapitulação — Estudo 11" | "Recapitulação — Estudo 12" |
| Estudo 13 ("Ressurreição dos Mortos") | "Recapitulação — Estudo 12" | "Recapitulação — Estudo 13" |
| Estudo 14 ("Jesus e o cansaço") | "Recapitulação — Estudo 13" ou "14" | "Recapitulação — Estudo 14" |
| Estudo 15 ("Juízo Final") | "Recapitulação — Estudo 14" ou "15" | "Recapitulação — Estudo 15" |

Além disso, o resumo no Estudo 16 lista:  
*"O que voltará em glória (est. 9)"* → deve ser *"(est. 10)"*  
*"a videira (est. 10)"* → deve ser *"(est. 11)"*  
… e assim por diante até o est. 15.

**Arquivos:** `index.html` (todos os cartões de recapitulação dos STUDIES[9] ao STUDIES[15])

**Critérios de teste:**
- [ ] Percorrer os 16 estudos e confirmar que cada cartão de recapitulação exibe o número correto
- [ ] Verificar que o resumo no Estudo 16 lista os números corretos de (est. 1) a (est. 15)
- [ ] Confirmar que a numeração dos estudos 1–9 permanece correta (não foi alterada)

**Definição de pronto:** todos os 16 recaps exibem o número correto do estudo correspondente.

---

### F2.2 — Corrigir Título do Estudo 14

**Origem:** SPIRITUAL_JOURNEY_REVIEW.md §Estudo 14, THEOLOGICAL_REVIEW.md §10.2, §11 (Estudo 15)  
**Severidade:** 🔴 P0 — título não corresponde ao conteúdo

**Problema:**  
O Estudo 14 é intitulado *"Jesus e o cansaço"* mas aborda exclusivamente o 2º mandamento (imagens) e o 4º mandamento (sábado). Não há relação com cansaço.

**Solução:**  
Alterar o campo `title` do STUDIES[13]:

**Opções de título** (decisão do capelão):
- `'Jesus e o Descanso Sagrado'` — conecta o sábado ao descanso sem mencionar o 2º mandamento
- `'Jesus e o Sábado'` — direto e preciso
- `'Jesus e a Adoração a Deus'` — abrange ambos os mandamentos (2º e 4º)

**Recomendado:** `'Jesus e o Descanso Sagrado'` — preserva a conexão com o sábado como descanso, é acolhedor e não-apologético no título.

**Arquivos:** `index.html` (STUDIES[13], campo `title`, linha ~3714)

**Critérios de teste:**
- [ ] Verificar que o título na tela de lista de estudos exibe o novo nome
- [ ] Verificar que o título na tela de estudo exibe o novo nome
- [ ] Verificar que o resumo no Estudo 16 referencia o tema correto no slot do est. 14
- [ ] Confirmar que badges e missões associadas ao estudo 14 não foram afetadas

**Definição de pronto:** o título do STUDIES[13] descreve com precisão o conteúdo do estudo.

---

### F2.3 — Revisar Tom do Parágrafo de Abertura do Estudo 14

**Origem:** THEOLOGICAL_REVIEW.md §6.2, §11 (Estudo 15), §15 (R1)  
**Severidade:** 🟠 Alta — risco pastoral com pacientes católicos em contexto hospitalar

**Problema:**  
```
"O mais surpreendente é quem se sente na autoridade para mudar a Lei de Deus
que define pecado — em outras palavras, tais pessoas redefiniram o que é e o 
que não é pecado. Chega a ser uma blasfêmia."
```
A palavra "blasfêmia" e o tom acusatório no parágrafo de abertura contradizem o padrão pastoral acolhedor dos outros 15 estudos. Em contexto hospitalar com pacientes vulneráveis de múltiplas tradições cristãs, pode romper o vínculo pastoral.

**Solução:**  
Reformular o parágrafo de abertura mantendo o conteúdo teológico mas com tom pastoral:

> *"Ao estudar os Dez Mandamentos, surgem algumas perguntas naturais: todos eles ainda se aplicam hoje? Foram dados apenas para os judeus? É possível que ao longo da história algum deles tenha sido modificado por autoridade humana? Este estudo vai examinar dois mandamentos que frequentemente levantam essas questões: o segundo, sobre a singularidade da adoração a Deus, e o quarto, sobre o dia de descanso santificado."*

**Arquivos:** `index.html` (STUDIES[13], `type:'opening'`, parágrafo de abertura)

**Critérios de teste:**
- [ ] Ler o parágrafo de abertura revisado — não deve conter linguagem acusatória ou julgamento de tradições específicas
- [ ] Confirmar que o conteúdo teológico (2º e 4º mandamentos) permanece intacto
- [ ] Confirmar que as citações bíblicas de suporte permanecem
- [ ] Pedir ao Mentor IA avaliação do tom do estudo 14 (teste qualitativo)

**Definição de pronto:** o parágrafo de abertura do Estudo 14 é acolhedor, convidativo e não usa linguagem de julgamento sobre tradições específicas.

---

### F2.4 — Contextualizar 3 João 1.2 (Estudo 8)

**Origem:** THEOLOGICAL_REVIEW.md §3.3  
**Severidade:** 🟡 Média — uso descontextualizado pode sugerir teologia da prosperidade

**Problema:**  
3 João 1.2 (*"faço votos que te vá bem em tudo, e que sejas saudável"*) é uma saudação epistolar pessoal de João ao seu amigo Gaio, não uma promessa teológica universal de saúde física a todos os crentes.

**Solução:**  
Adicionar uma nota de contexto:
> *"João, ao escrever a seu amigo Gaio, expressa um desejo que reflete o coração de Deus pelo ser humano integral — espírito, mente e corpo. Não é uma promessa de ausência de doença, mas uma afirmação de que Deus se preocupa com o nosso bem-estar completo."*

**Arquivos:** `index.html` (STUDIES[7], `type:'insight'`, próximo à citação de 3 João 1.2, linha ~1293)

**Critérios de teste:**
- [ ] Ler o Estudo 8 completo — o uso de 3 João 1.2 deve ter nota contextual
- [ ] Verificar que a nota não contradiz o argumento principal do estudo sobre saúde como mordomia
- [ ] Confirmar que a versão editada não soa como defesa da "teologia da prosperidade"

**Definição de pronto:** 3 João 1.2 é usado com contexto exegético que evita má interpretação.

---

### F2.5 — Fortalecer o Perdão Recebido no Estudo 4

**Origem:** THEOLOGICAL_REVIEW.md §11 (Estudo 4)  
**Severidade:** 🟡 Média — lacuna pastoral para usuários carregando culpa própria

**Problema:**  
O Estudo 4 enfatiza o perdão concedido a outros, mas dá menos atenção ao perdão que Deus concede ao próprio crente. Um usuário carregando culpa de algo que fez pode sentir que o estudo fala para ele *perdoar*, mas não aborda ser *perdoado*.

**Solução:**  
Adicionar, na seção `type:'content'`, um parágrafo e versículo sobre o perdão divino recebido:
> *"Mas há uma outra dimensão do perdão que precisa ser recebida antes de poder ser doada: o perdão que Deus tem por você. Não importa o que você carrega — a Cruz é a prova de que Deus já pagou a dívida."*

Adicionar: `V('1 João 1.9', '"Se confessarmos os nossos pecados, ele é fiel e justo para nos perdoar os pecados e nos purificar de toda injustiça."')`

**Arquivos:** `index.html` (STUDIES[3], `type:'content'`)

**Critérios de teste:**
- [ ] Ler o Estudo 4 — deve abordar tanto o perdão recebido de Deus quanto o perdão concedido a outros
- [ ] 1 João 1.9 deve estar presente no estudo
- [ ] A aplicação prática ainda deve incluir o perdão a terceiros
- [ ] Verificar que a adição não cria repetição excessiva com o conteúdo já existente

**Definição de pronto:** o Estudo 4 apresenta o perdão divino recebido e o perdão humano concedido como dimensões complementares.

---

**Relatório de conclusão da Fase 2:**
Ao final desta fase: numeração correta em todos os 16 estudos, título do Estudo 14 preciso, tom do Estudo 14 pastoral, contexto bíblico corrigido em dois estudos, lacuna do perdão resolvida.

---

## FASE 3 — PWA & Performance

**Objetivo:** garantir que o app funcione completamente offline, carregue rápido na primeira visita, e se comporte como PWA de primeira classe.  
**Pode ser implementada em:** 1–2 sessões.  
**Não depende de nenhuma outra fase.**

---

### F3.1 — Cachear Google Fonts no Service Worker

**Origem:** PERFORMANCE_REVIEW.md §10.2, §14.1  
**Severidade:** 🟠 Alta — sem cache das fontes, o app offline perde completamente a identidade visual

**Problema:**  
O Service Worker atual não cacheia `fonts.googleapis.com` nem `fonts.gstatic.com`. Offline, Lora e Source Sans 3 ficam indisponíveis — o app renderiza com fontes genéricas do sistema.

**Solução:**  
Adicionar estratégia `staleWhileRevalidate` para fontes no `service-worker.js`:

```javascript
// Em service-worker.js, no handler fetch:
const isFontRequest = url.hostname === 'fonts.gstatic.com' ||
                      url.hostname === 'fonts.googleapis.com';

if (isFontRequest) {
  event.respondWith(
    caches.open('amj-fonts-v1').then(cache =>
      cache.match(event.request).then(cached => {
        const networkFetch = fetch(event.request).then(response => {
          cache.put(event.request, response.clone());
          return response;
        });
        return cached || networkFetch;
      })
    )
  );
  return;
}
```

Adicionar `'amj-fonts-v1'` à lista de caches preservados no `activate`.

**Arquivos:** `service-worker.js`

**Critérios de teste:**
- [ ] Com conexão: carregar o app e verificar que fontes são cacheadas em DevTools → Application → Cache Storage
- [ ] Colocar em modo offline (DevTools → Network → Offline) e recarregar — Lora e Source Sans 3 devem renderizar corretamente
- [ ] Inspecionar tipografia offline — identificar que a fonte não é a fallback do sistema
- [ ] Verificar que o cache de fontes não está crescendo indefinidamente (versão do cache controlada)
- [ ] Testar atualização da fonte: nova versão do Google Fonts deve ser baixada em background

**Definição de pronto:** o app renderiza com Lora e Source Sans 3 em modo completamente offline após a primeira visita com conexão.

---

### F3.2 — Adicionar Resource Hints para Google Fonts

**Origem:** PERFORMANCE_REVIEW.md §2.4, §14.1  
**Severidade:** 🟠 Alta — 200–500ms de FCP desperdiçados em lookups DNS evitáveis

**Problema:**  
O HTML não inclui `preconnect` para os servidores de fontes. O navegador só descobre os domínios quando encontra o `<link>` da fonte durante o parse — iniciando DNS lookup + TCP + TLS de forma tardia.

**Solução:**  
Adicionar antes do `<link>` das fontes (linha 18):
```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="dns-prefetch" href="https://solitary-meadow-26e0.prwladi.workers.dev">
```

**Arquivos:** `index.html` (seção `<head>`, antes da linha 18)

**Critérios de teste:**
- [ ] DevTools → Network com 3G throttling: medir FCP antes e depois (esperado: -200 a 500ms)
- [ ] Verificar no painel Network que `fonts.googleapis.com` já tem conexão estabelecida quando o CSS da fonte é requisitado
- [ ] Confirmar que `dns-prefetch` para o AI Proxy aparece no timing do Network
- [ ] Lighthouse: score de Performance deve aumentar (esperado: +5–15 pontos)

**Definição de pronto:** DevTools mostra que o tempo de `TTFB` para `fonts.googleapis.com` foi reduzido pela preconexão.

---

### F3.3 — Corrigir Paths dos Ícones no `<head>`

**Origem:** PERFORMANCE_REVIEW.md §14.2  
**Severidade:** 🟠 Alta — URLs absolutas externas quebram se o domínio mudar

**Problema:**  
```html
<!-- Linhas 12–16 — atual -->
<link rel="apple-touch-icon" href="https://capelaniahospitalar.github.io/jornada-discipular/icon-192.png">
```
Os arquivos `icon-192.png` e `icon-512.png` existem localmente no repositório. Usar URLs absolutas cria dependência de domínio desnecessária.

**Solução:**  
```html
<link rel="apple-touch-icon" sizes="180x180" href="./icon-192.png">
<link rel="apple-touch-icon" sizes="152x152" href="./icon-192.png">
<link rel="apple-touch-icon" sizes="120x120" href="./icon-192.png">
<link rel="icon" type="image/png" sizes="192x192" href="./icon-192.png">
<link rel="icon" type="image/png" sizes="512x512" href="./icon-512.png">
```

**Arquivos:** `index.html` (linhas 12–16)

**Critérios de teste:**
- [ ] Inspecionar o favicon no browser tab — deve aparecer o ícone correto
- [ ] DevTools → Network: requests de ícones devem ir para o mesmo domínio (sem redirect)
- [ ] Testar "Adicionar à tela inicial" no Android — ícone correto deve aparecer
- [ ] Testar "Adicionar à tela inicial" no iOS — ícone correto deve aparecer

**Definição de pronto:** todos os ícones no `<head>` usam paths relativos e carregam sem requisições externas.

---

### F3.4 — Corrigir `"purpose"` dos Ícones no Manifest

**Origem:** PERFORMANCE_REVIEW.md §9.2  
**Severidade:** 🟡 Média — ícone pode ser cortado em adaptive icons do Android

**Problema:**  
```json
{ "src": "icon-192.png", "purpose": "any maskable" }
```
O valor `"any maskable"` combinado pode causar corte do ícone em adaptive icons. O correto é ter dois ícones separados ou um ícone com propósito único.

**Solução:**  
```json
{
  "icons": [
    { "src": "icon-192.png", "sizes": "192x192", "type": "image/png", "purpose": "any" },
    { "src": "icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "any" },
    { "src": "icon-192.png", "sizes": "192x192", "type": "image/png", "purpose": "maskable" }
  ]
}
```
**Nota:** Idealmente, o ícone maskable deveria ser uma versão específica com zona segura (safe zone) de 40% — se o design do ícone atual já respeita a zona segura, o mesmo arquivo pode ser usado. Verificar com a ferramenta [maskable.app](https://maskable.app) antes de implementar.

**Arquivos:** `manifest.json`

**Critérios de teste:**
- [ ] Validar o manifest em [web.dev/pwa](https://web.dev/measure/) — sem warnings sobre ícones
- [ ] Testar instalação no Android Chrome — verificar que o adaptive icon não corta o conteúdo
- [ ] Lighthouse PWA audit — ícone deve passar na verificação
- [ ] DevTools → Application → Manifest — sem erros de ícone

**Definição de pronto:** Lighthouse PWA audit passa na checagem de ícones sem warnings.

---

### F3.5 — Minificação do HTML

**Origem:** PERFORMANCE_REVIEW.md §13  
**Severidade:** 🟠 Alta — 347KB não minificados vs. ~170KB minificados (+50% de parse time desnecessário)

**Problema:**  
O arquivo `index.html` contém comentários extensos, whitespace generoso e código JS legível. Gzip reduz o tamanho de transferência, mas não reduz o tempo de parse do JavaScript no dispositivo.

**Solução:**  
Criar um processo de build com GitHub Actions:

```yaml
# .github/workflows/build.yml
name: Build & Deploy
on:
  push:
    branches: [main]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm install html-minifier-terser
      - run: |
          npx html-minifier-terser index.html \
            --collapse-whitespace \
            --remove-comments \
            --minify-css true \
            --minify-js true \
            -o dist/index.html
      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./dist
```

**Arquivos:** `.github/workflows/build.yml` (novo), `package.json` (novo), `dist/` (gerado)

**Critérios de teste:**
- [ ] Build executa sem erros no GitHub Actions
- [ ] `dist/index.html` tem tamanho ≤ 180KB (vs. 347KB original)
- [ ] App funciona identicamente no arquivo minificado (todas as 12 telas, todos os 16 estudos)
- [ ] Service Worker cacheia `dist/index.html` corretamente
- [ ] Lighthouse Performance score ≥ 85 na versão minificada

**Definição de pronto:** pipeline de build automático produz `dist/index.html` ≤ 180KB que passa em todos os testes funcionais.

---

**Relatório de conclusão da Fase 3:**
Ao final desta fase: fontes cacheadas offline, FCP reduzido em 200–500ms, ícones com paths corretos, manifest válido, build minificado automatizado.

---

## FASE 4 — UX & Acessibilidade

**Objetivo:** eliminar padrões de UX que quebram o contexto espiritual do app, melhorar a experiência em dispositivos móveis e criar feedback visual consistente.  
**Pode ser implementada em:** 2–3 sessões.  
**Não depende de nenhuma outra fase.**

---

### F4.1 — Mover Botão do WhatsApp para o Lado Direito

**Origem:** UX_REVIEW.md §FAB, §uso com uma mão  
**Severidade:** 🟠 Alta — botão na parte inferior esquerda é inacessível para 90% dos usuários destros

**Problema:**  
O botão flutuante do WhatsApp está posicionado em `bottom: .85rem; left: .85rem`. A maioria dos usuários de smartphone usa o polegar direito — o canto inferior esquerdo é a área de menor alcance.

**Solução:**  
```css
/* Mudar de left para right */
#fixed-tutor-btn {
  bottom: .85rem;
  right: .85rem; /* ← era left */
}
```

**Arquivos:** `index.html` (CSS do `#fixed-tutor-btn`)

**Critérios de teste:**
- [ ] Verificar posicionamento em dispositivo físico (ou DevTools mobile) — botão no canto inferior direito
- [ ] Confirmar que o botão não sobrepõe botões de ação principais na tela de estudo
- [ ] Verificar que a animação de ocultamento durante o estudo (`hidden-on-study`) ainda funciona
- [ ] Testar em telas pequenas (320px de largura) — botão não deve sair da viewport

**Definição de pronto:** o botão do WhatsApp está no canto inferior direito em todos os tamanhos de tela.

---

### F4.2 — Corrigir Altura da Tela de IA no iOS

**Origem:** UX_REVIEW.md §tela de IA, §iOS keyboard  
**Severidade:** 🟠 Alta — teclado virtual do iOS cobre o campo de input na tela de IA

**Problema:**  
```css
#screen-ai { height: calc(100vh - 56px); }
```
No iOS, `100vh` inclui a barra de endereço — quando o teclado virtual abre, ele empurra o conteúdo para baixo, escondendo o campo de input.

**Solução:**  
```css
#screen-ai {
  height: 100%;
  /* Usar dvh (dynamic viewport height) com fallback */
  height: calc(100dvh - 56px);
}
```
Adicionar listener para `visualViewport.resize` no iOS:
```javascript
if (window.visualViewport) {
  window.visualViewport.addEventListener('resize', () => {
    const aiScreen = document.getElementById('screen-ai');
    if (aiScreen && aiScreen.classList.contains('active')) {
      aiScreen.style.height = (window.visualViewport.height - 56) + 'px';
    }
  });
}
```

**Arquivos:** `index.html` (CSS da tela de IA e JS de adaptação ao viewport)

**Critérios de teste:**
- [ ] Em iPhone (físico ou simulador): abrir tela de IA, tocar no campo de texto — o campo deve ficar visível com teclado aberto
- [ ] Fechar o teclado — a tela deve voltar ao tamanho original
- [ ] Testar em Android — comportamento não deve ser afetado
- [ ] O histórico de mensagens deve rolar corretamente com teclado aberto

**Definição de pronto:** o campo de input da IA permanece visível quando o teclado virtual do iOS está aberto.

---

### F4.3 — Substituir `alert()` e `confirm()` por Modais Estilizados

**Origem:** UX_REVIEW.md §3 usos de alert/confirm, DATA_REVIEW.md §14  
**Severidade:** 🟠 Alta — dialogs nativos quebram o contexto visual e espiritual do app

**Problema:**  
3 ocorrências de `alert()` e 1 de `confirm()` nativo:
1. `alert('Escolha um tutor para começar!')` — em `startJourney()`
2. `confirm()` — em `confirmRestart()`
3. Possivelmente outros (verificar grep completo)

**Solução:**  
Criar sistema de modal/toast estilizado:

```javascript
// Sistema de modal estilizado
function showModal({ title, message, actions }) {
  // Remove modal anterior se existir
  document.getElementById('app-modal')?.remove();
  
  const overlay = document.createElement('div');
  overlay.id = 'app-modal';
  overlay.style.cssText = `
    position:fixed;inset:0;background:rgba(0,0,0,.5);
    display:flex;align-items:center;justify-content:center;z-index:9999
  `;
  overlay.innerHTML = `
    <div style="background:var(--card);border-radius:var(--radius);
                padding:1.5rem;max-width:320px;width:90%;margin:1rem">
      <p style="font-weight:600;font-size:17px;color:var(--text);margin-bottom:.75rem">${escapeHTML(title)}</p>
      <p style="color:var(--text2);font-size:15px;line-height:1.6">${escapeHTML(message)}</p>
      <div style="display:flex;gap:.75rem;margin-top:1.25rem;justify-content:flex-end">
        ${actions.map(a => `<button onclick="${a.onclick}" 
          style="padding:.6rem 1.2rem;border-radius:var(--radius-sm);
                 border:none;cursor:pointer;font-size:15px;font-family:var(--ff-body);
                 background:${a.primary ? 'var(--blue)' : 'transparent'};
                 color:${a.primary ? '#fff' : 'var(--text3)'};
                 border:1px solid ${a.primary ? 'transparent' : 'var(--border)'}">
          ${escapeHTML(a.label)}</button>`).join('')}
      </div>
    </div>`;
  document.body.appendChild(overlay);
}

function closeModal() { document.getElementById('app-modal')?.remove(); }
```

Substituir `confirmRestart()`:
```javascript
function confirmRestart() {
  showModal({
    title: 'Recomeçar a jornada?',
    message: 'Seus estudos, XP e conquistas serão zerados. Seu diário e dados de tutor serão mantidos.',
    actions: [
      { label: 'Cancelar', onclick: 'closeModal()', primary: false },
      { label: 'Sim, recomeçar', onclick: 'doRestart();closeModal()', primary: true }
    ]
  });
}
function doRestart() { /* lógica de reset atual */ }
```

**Arquivos:** `index.html` (todas as chamadas a `alert()`, `confirm()`, e nova função `showModal`)

**Critérios de teste:**
- [ ] Tentar iniciar jornada sem tutor — modal estilizado deve aparecer (não `alert()` nativo)
- [ ] Clicar em "Recomeçar" — modal estilizado de confirmação deve aparecer
- [ ] Confirmar restart pelo modal — progresso zerado conforme Fase F1.3
- [ ] Cancelar restart pelo modal — nenhuma alteração no estado
- [ ] Verificar que nenhum `alert()` ou `confirm()` nativo aparece em nenhum fluxo do app

**Definição de pronto:** zero chamadas a `alert()` ou `confirm()` no código JS — todos substituídos por modais estilizados consistentes com o design do app.

---

### F4.4 — Adicionar Transições entre Telas

**Origem:** UX_REVIEW.md §animações, §zero transições  
**Severidade:** 🟡 Média — transições de `display:none` para `display:flex` são abruptas e prejudicam o senso de calma

**Problema:**  
`showScreen(n)` alterna telas instantaneamente via `classList.remove/add('active')`. Sem animação, o app parece montar e desmontar elementos abruptamente.

**Solução:**  
Adicionar transição CSS simples de `opacity`:
```css
.screen {
  opacity: 0;
  transition: opacity 0.2s ease;
  pointer-events: none;
}
.screen.active {
  opacity: 1;
  pointer-events: auto;
}
```
Ajustar a lógica de exibição para não usar `display:none` diretamente, mas controlar por `opacity` e `pointer-events` — ou manter o `display` e adicionar `requestAnimationFrame` para o fade.

**Arquivos:** `index.html` (CSS de `.screen` e `.screen.active`, função `showScreen`)

**Critérios de teste:**
- [ ] Navegar entre telas — transição de 200ms deve ser visível e suave
- [ ] A transição não deve criar "flash" ou estado inconsistente durante a troca
- [ ] Testar em dispositivo de baixo desempenho — a transição não deve causar jank
- [ ] Verificar que `resetScroll()` ainda funciona corretamente após a transição
- [ ] O botão "voltar" não deve ser clicável durante a transição (pointer-events: none)

**Definição de pronto:** todas as trocas de tela têm fade-in/fade-out de 200ms, sem estados visuais intermediários inconsistentes.

---

### F4.5 — Aumentar Área de Toque do Botão de Envio da IA

**Origem:** UX_REVIEW.md §`.ai-send`, §Apple HIG 44×44px  
**Severidade:** 🟡 Média — 38×38px abaixo do mínimo de acessibilidade (44×44px)

**Problema:**  
O botão `.ai-send` tem `width: 38px; height: 38px` — abaixo do mínimo de 44×44px recomendado pelo Apple HIG e Google Material Design.

**Solução:**  
```css
.ai-send {
  width: 44px;
  height: 44px;
  min-width: 44px;
  min-height: 44px;
}
```

**Arquivos:** `index.html` (CSS de `.ai-send`)

**Critérios de teste:**
- [ ] Medir o botão em DevTools — deve ter pelo menos 44×44px
- [ ] Testar com dedo em dispositivo físico — acerto no primeiro toque sem precisar de precisão excessiva
- [ ] Layout da barra de input da IA não deve ser distorcido pelo aumento do botão

**Definição de pronto:** `.ai-send` tem área de toque de pelo menos 44×44px.

---

**Relatório de conclusão da Fase 4:**
Ao final desta fase: botão WhatsApp no lugar correto, IA funcional no iOS, zero dialogs nativos, transições suaves entre telas, botão de envio acessível.

---

## FASE 5 — Mentor IA

**Objetivo:** corrigir bugs no comportamento do Mentor IA e melhorar a qualidade da experiência conversacional.  
**Pode ser implementada em:** 1–2 sessões.  
**Não depende de nenhuma outra fase.**

---

### F5.1 — Renderizar Markdown nas Respostas da IA

**Origem:** MENTOR_AI_REVIEW.md §formato de resposta  
**Severidade:** 🟠 Alta — asteriscos aparecem literalmente em vez de negrito/itálico

**Problema:**  
As respostas da IA retornam com formatação Markdown (`**negrito**`, `*itálico*`, `- lista`), mas são inseridas via `textContent`, exibindo os caracteres literalmente.

**Solução:**  
Criar parser Markdown mínimo seguro (sem biblioteca externa — para manter o arquivo monolítico por ora):

```javascript
function renderMarkdown(text) {
  return escapeHTML(text) // escapar primeiro para evitar XSS
    .replace(/\*\*(.+?)\*\*/g, '<strong>$1</strong>')
    .replace(/\*(.+?)\*/g, '<em>$1</em>')
    .replace(/^- (.+)$/gm, '<li>$1</li>')
    .replace(/(<li>.*<\/li>)/gs, '<ul>$1</ul>')
    .replace(/\n\n/g, '</p><p>')
    .replace(/\n/g, '<br>');
}

// Em sendAI(), substituir:
// msgEl.textContent = text;
// Por:
msgEl.innerHTML = '<p>' + renderMarkdown(text) + '</p>';
```

**Atenção:** o `escapeHTML` deve ser aplicado **antes** do regex de Markdown para garantir que tags HTML não sejam injetadas via resposta da API.

**Arquivos:** `index.html` (função `sendAI`, inserção da resposta no DOM)

**Critérios de teste:**
- [ ] Perguntar à IA algo que provoque resposta com negrito — deve renderizar corretamente
- [ ] Perguntar algo que provoque lista — deve renderizar como `<ul><li>`
- [ ] Inspecionar DOM — resposta deve usar `innerHTML` com HTML seguro
- [ ] Testar com resposta que contenha `<script>` — deve aparecer como texto, não executar
- [ ] Testar com resposta que contenha `&` e `<` — deve ser escapada corretamente

**Definição de pronto:** respostas da IA renderizam formatação Markdown básica sem vulnerabilidade XSS.

---

### F5.2 — Corrigir Dessincronia do Histórico em Caso de Erro

**Origem:** MENTOR_AI_REVIEW.md §erro, §histórico  
**Severidade:** 🟠 Alta — mensagem do usuário fica no histórico mesmo quando a IA não respondeu

**Problema:**  
```javascript
// sendAI() — ordem atual
aiHistory.push({ role:'user', content: userMsg }); // ← push ANTES do fetch
const resp = await fetch(AI_PROXY_URL, ...);
// Se fetch falhar, a mensagem do usuário fica no histórico sem resposta correspondente
```

**Solução:**  
Mover o push para o histórico para após a resposta bem-sucedida:
```javascript
async function sendAI() {
  const userMsg = input.value.trim();
  if (!userMsg) return;
  input.value = '';
  
  // Exibir mensagem do usuário na UI (apenas visual, ainda não no histórico)
  appendMessage('user', userMsg);
  showLoadingIndicator();
  
  try {
    const historyForRequest = [...aiHistory, { role:'user', content: userMsg }];
    const resp = await fetch(AI_PROXY_URL, {
      body: JSON.stringify({ messages: historyForRequest }),
      ...
    });
    const data = await resp.json();
    const aiText = data.content?.[0]?.text || '';
    
    // Só adiciona ao histórico se a requisição foi bem-sucedida
    aiHistory.push({ role:'user', content: userMsg });
    aiHistory.push({ role:'assistant', content: aiText });
    aiHistory = aiHistory.slice(-20);
    
    appendMessage('assistant', aiText);
  } catch (e) {
    // Histórico permanece íntegro — a mensagem do usuário não foi adicionada
    showErrorMessage('Não consegui responder agora. Tente novamente.');
  } finally {
    hideLoadingIndicator();
  }
}
```

**Arquivos:** `index.html` (função `sendAI`, linhas ~1946–2071)

**Critérios de teste:**
- [ ] Desativar a rede (offline) e enviar mensagem à IA — a mensagem deve aparecer na UI, mas a mensagem de erro deve aparecer
- [ ] Reativar a rede e enviar nova mensagem — o histórico deve conter apenas as mensagens com resposta correspondente
- [ ] Verificar que após erro o campo de input não está bloqueado
- [ ] Simular timeout do Cloudflare Worker — comportamento deve ser igual ao de erro de rede

**Definição de pronto:** o `aiHistory` sempre contém pares user/assistant completos — nenhuma mensagem do usuário sem resposta correspondente.

---

### F5.3 — Melhorar o Indicador de Carregamento da IA

**Origem:** MENTOR_AI_REVIEW.md §latência, §loading  
**Severidade:** 🟡 Média — "..." estático por 3–20 segundos não comunica que algo está acontecendo

**Problema:**  
O indicador atual é apenas `<em>...</em>` em itálico — sem animação, sem contexto, sem estimativa de tempo.

**Solução:**  
Substituir por indicador animado com bolinha pulsante:
```css
.ai-typing { display: flex; gap: 4px; align-items: center; padding: .5rem 0; }
.ai-typing span {
  width: 8px; height: 8px; border-radius: 50%;
  background: var(--blue); opacity: .4;
  animation: aiPulse 1.2s infinite;
}
.ai-typing span:nth-child(2) { animation-delay: .2s; }
.ai-typing span:nth-child(3) { animation-delay: .4s; }
@keyframes aiPulse {
  0%, 80%, 100% { opacity: .4; transform: scale(1); }
  40% { opacity: 1; transform: scale(1.2); }
}
```
Exibir como bolha de mensagem da IA:
```html
<div class="ai-typing"><span></span><span></span><span></span></div>
```

**Arquivos:** `index.html` (CSS e função `showLoadingIndicator`/`hideLoadingIndicator` em `sendAI`)

**Critérios de teste:**
- [ ] Enviar mensagem à IA — três bolinhas animadas devem aparecer imediatamente
- [ ] Resposta da IA chega — bolinhas devem desaparecer e a resposta deve aparecer no lugar
- [ ] Em caso de erro — bolinhas devem desaparecer e mensagem de erro deve aparecer
- [ ] Testar em iOS (animação CSS deve funcionar)
- [ ] Bolinhas devem estar dentro de uma bolha de mensagem visualmente consistente com as outras mensagens

**Definição de pronto:** o estado de carregamento da IA é visualmente claro, animado e consistente com o design das mensagens.

---

### F5.4 — Popular os Chips de Sugestão da IA

**Origem:** MENTOR_AI_REVIEW.md §chips de sugestão (`#ai-sugs`)  
**Severidade:** 🟡 Média — CSS e HTML do `#ai-sugs` existe mas nunca é populado

**Problema:**  
O elemento `#ai-sugs` tem CSS definido mas nenhuma função JS que o popula com sugestões. O espaço fica vazio.

**Solução:**  
Definir sugestões contextuais por estudo e exibi-las quando o usuário abre a tela da IA:

```javascript
const AI_SUGGESTIONS = {
  default: [
    'O que a Bíblia diz sobre ansiedade?',
    'Como posso orar quando não tenho palavras?',
    'O que é graça?'
  ],
  0: ['Quem foi João Batista?', 'O que significa ser discípulo de Jesus?'],
  3: ['Como posso perdoar alguém que me magoou muito?', 'E se eu não sentir vontade de perdoar?'],
  // ... um array por estudo
};

function renderAISuggestions() {
  const suggestions = AI_SUGGESTIONS[ST.lastStudy] || AI_SUGGESTIONS.default;
  const container = document.getElementById('ai-sugs');
  if (!container) return;
  container.innerHTML = suggestions.map(s =>
    `<button class="ai-sug-btn" onclick="fillAIInput(${JSON.stringify(s)})">${escapeHTML(s)}</button>`
  ).join('');
}

function fillAIInput(text) {
  document.getElementById('ai-input').value = text;
  document.getElementById('ai-input').focus();
}
```

**Arquivos:** `index.html` (nova função `renderAISuggestions`, chamada na abertura da tela de IA)

**Critérios de teste:**
- [ ] Abrir tela de IA após concluir o Estudo 3 — chips contextuais sobre perdão devem aparecer
- [ ] Abrir tela de IA sem nenhum estudo concluído — chips padrão devem aparecer
- [ ] Clicar em um chip — o texto deve preencher o campo de input, não enviar automaticamente
- [ ] Chips devem desaparecer após o primeiro envio (ou permanecer, conforme decisão editorial)
- [ ] Verificar que chips não são exibidos em modo landscape com pouco espaço vertical

**Definição de pronto:** o `#ai-sugs` exibe 3 sugestões contextuais relevantes para o momento do usuário na jornada.

---

**Relatório de conclusão da Fase 5:**
Ao final desta fase: Markdown renderizado nas respostas, histórico sempre consistente, loading animado, chips de sugestão funcionais.

---

## FASE 6 — Diário Espiritual

**Objetivo:** transformar o diário em uma ferramenta completa de reflexão — com edição, exportação, backup e busca.  
**Pode ser implementada em:** 2–3 sessões.  
**Não depende de nenhuma outra fase** (depende de F0.1 para segurança, mas pode ser implementada independentemente com a garantia de que F0.1 já terá sido aplicada).

---

### F6.1 — Datas em Formato ISO 8601

**Origem:** DATA_REVIEW.md §5, JOURNAL_REVIEW.md §modelo de dados  
**Severidade:** 🟠 Alta — `"26/06/2026"` não é ordenável e é dependente de locale

**Problema:**  
```javascript
date: new Date().toLocaleDateString('pt-BR') // → "26/06/2026"
```
Impossível ordenar entradas cronologicamente, fazer buscas por período, ou comparar datas entre dispositivos com locales diferentes.

**Solução:**  
```javascript
// Ao criar nova entrada
date: new Date().toISOString() // → "2026-06-26T14:30:00.000Z"

// Ao exibir (formatar para o usuário)
function formatJournalDate(isoString) {
  try {
    return new Date(isoString).toLocaleDateString('pt-BR', {
      weekday: 'long', year: 'numeric', month: 'long', day: 'numeric'
    });
  } catch {
    return isoString; // fallback para entradas antigas
  }
}
```

Migrar entradas antigas no esquema de migração (F1.1):
```javascript
// Em migrateSchema(), versão 2:
if (st._v === 1) {
  st.journal = (st.journal || []).map(e => ({
    ...e,
    date: e.date ? tryConvertDate(e.date) : new Date().toISOString()
  }));
  st._v = 2;
}
```

**Arquivos:** `index.html` (função `saveJournalEntry`, `renderJournal`, e migração em F1.1)

**Critérios de teste:**
- [ ] Criar nova entrada de diário — `date` no localStorage deve ser ISO 8601
- [ ] Exibir entrada no diário — data deve ser formatada em português (ex: "sexta-feira, 26 de junho de 2026")
- [ ] Entradas antigas com formato `dd/mm/yyyy` devem ser migradas e exibidas corretamente
- [ ] Ordenar entradas por data (mais recentes primeiro) — deve funcionar corretamente com ISO

**Definição de pronto:** todas as entradas novas usam ISO 8601; entradas antigas são migradas na carga do app.

---

### F6.2 — Edição e Exclusão de Entradas

**Origem:** JOURNAL_REVIEW.md §CRUD completo  
**Severidade:** 🟡 Média — usuário não pode corrigir erros de digitação ou remover entradas

**Problema:**  
O diário é write-only. Uma vez salva, uma entrada não pode ser editada nem excluída.

**Solução:**  
Adicionar botões de edição e exclusão a cada entrada renderizada:
```javascript
// Em renderJournal() — adicionar a cada entry card:
`<div class="journal-actions">
  <button onclick="editJournalEntry(${i})" aria-label="Editar entrada">✏️</button>
  <button onclick="deleteJournalEntry(${i})" aria-label="Excluir entrada">🗑️</button>
</div>`

function editJournalEntry(idx) {
  // Preencher o formulário do diário com os dados da entrada
  // Mudar o botão de "Salvar" para "Atualizar"
  // Ao salvar, substituir ST.journal[idx] em vez de push
}

function deleteJournalEntry(idx) {
  showModal({
    title: 'Excluir esta entrada?',
    message: 'Esta ação não pode ser desfeita.',
    actions: [
      { label: 'Cancelar', onclick: 'closeModal()', primary: false },
      { label: 'Excluir', onclick: `doDeleteEntry(${idx});closeModal()`, primary: true }
    ]
  });
}
function doDeleteEntry(idx) {
  ST.journal.splice(idx, 1);
  save(ST);
  renderJournal();
}
```

**Arquivos:** `index.html` (funções `renderJournal`, `editJournalEntry`, `deleteJournalEntry`, `doDeleteEntry`)

**Critérios de teste:**
- [ ] Clicar em "Editar" numa entrada — formulário deve abrir com os dados pré-preenchidos
- [ ] Editar um campo e salvar — a entrada deve ser atualizada no diário e no localStorage
- [ ] Clicar em "Excluir" — modal de confirmação deve aparecer
- [ ] Confirmar exclusão — entrada removida, diário atualizado
- [ ] Cancelar exclusão — nenhuma alteração
- [ ] Verificar que edição não cria uma nova entrada (substitui a existente no índice correto)

**Definição de pronto:** cada entrada do diário pode ser editada ou excluída com confirmação, e o estado é persistido corretamente.

---

### F6.3 — Exportação do Diário

**Origem:** DATA_REVIEW.md §11, JOURNAL_REVIEW.md §exportação  
**Severidade:** 🟠 Alta — único mecanismo de backup do conteúdo mais íntimo do usuário

**Problema:**  
Não há forma de o usuário exportar seu diário. Limpar o navegador ou trocar de dispositivo resulta em perda permanente de todos as reflexões.

**Solução:**  
Implementar exportação em dois formatos:

**Formato 1 — JSON (backup técnico):**
```javascript
function exportJournalJSON() {
  const data = {
    exportDate: new Date().toISOString(),
    userName: ST.userName,
    entries: ST.journal
  };
  downloadFile(
    'diario-espiritual-' + new Date().toLocaleDateString('pt-BR').replace(/\//g,'-') + '.json',
    JSON.stringify(data, null, 2),
    'application/json'
  );
}
```

**Formato 2 — Texto legível (compartilhamento):**
```javascript
function exportJournalText() {
  const lines = ST.journal.map((e, i) => [
    `--- Entrada ${i + 1} — ${formatJournalDate(e.date)} ---`,
    `Estudo: ${e.title}`,
    `Versículo: ${e.verse}`,
    `O que aprendi: ${e.learned}`,
    `Como aplicar: ${e.apply}`,
    `Minha oração: ${e.prayer}`,
    `Escolha que faço: ${e.chosen}`
  ].join('\n')).join('\n\n');
  
  downloadFile(
    'meu-diario-espiritual.txt',
    `Diário Espiritual — ${ST.userName}\n\n${lines}`,
    'text/plain;charset=utf-8'
  );
}

function downloadFile(filename, content, type) {
  const blob = new Blob([content], { type });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url; a.download = filename;
  document.body.appendChild(a); a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);
}
```

Adicionar botão de exportação na tela do diário.

**Arquivos:** `index.html` (tela `screen-journal`, funções `exportJournalJSON`, `exportJournalText`, `downloadFile`)

**Critérios de teste:**
- [ ] Clicar em "Exportar JSON" — arquivo `.json` válido deve ser baixado
- [ ] Abrir o JSON — deve conter todas as entradas com campos completos
- [ ] Clicar em "Exportar Texto" — arquivo `.txt` legível deve ser baixado
- [ ] O arquivo de texto deve ser legível sem conhecimento técnico
- [ ] Com diário vazio — exportação deve criar arquivo com mensagem apropriada ("Nenhuma entrada ainda")
- [ ] O aviso de armazenamento cheio (F1.4) deve ter link para a exportação

**Definição de pronto:** o usuário pode exportar todo o conteúdo do seu diário em JSON e em texto simples com um único toque.

---

### F6.4 — Busca no Diário

**Origem:** JOURNAL_REVIEW.md §busca, THEOLOGICAL_REVIEW.md (mencionado como ausência)  
**Severidade:** 🟡 Média — com 50+ entradas, encontrar um estudo específico torna-se difícil

**Solução:**  
Adicionar campo de busca na tela do diário:
```javascript
function searchJournal(query) {
  const q = query.toLowerCase().trim();
  if (!q) { renderJournal(); return; }
  
  const filtered = (ST.journal || []).filter((e, i) => {
    return [e.title, e.learned, e.apply, e.prayer, e.verse, e.chosen]
      .some(field => field && field.toLowerCase().includes(q));
  });
  renderJournalEntries(filtered);
}
```

**Arquivos:** `index.html` (tela `screen-journal`, função `searchJournal`, input de busca)

**Critérios de teste:**
- [ ] Buscar por título de estudo — entradas correspondentes devem aparecer
- [ ] Buscar por palavra no conteúdo — entradas com essa palavra devem aparecer
- [ ] Buscar por texto não existente — mensagem "Nenhuma entrada encontrada" deve aparecer
- [ ] Limpar a busca — todas as entradas devem voltar a aparecer
- [ ] A busca não deve alterar o ST.journal — apenas filtrar a visualização

**Definição de pronto:** o usuário pode buscar no diário por texto livre e ver apenas as entradas que correspondem.

---

**Relatório de conclusão da Fase 6:**
Ao final desta fase: datas em ISO 8601, edição e exclusão de entradas, exportação em dois formatos, busca funcional. O diário é uma ferramenta completa de reflexão espiritual com proteção de dados.

---

## FASE 7 — Aprofundamento Teológico

**Objetivo:** corrigir as lacunas teológicas identificadas na THEOLOGICAL_REVIEW — adicionando dois estudos ausentes e aprofundando pontos existentes.  
**Pode ser implementada em:** 3–5 sessões (cada estudo novo é substancial).  
**Não depende de nenhuma outra fase.**  
**Requer revisão e aprovação do capelão antes da implementação.**

---

### F7.1 — Novo Estudo: O Espírito Santo

**Origem:** THEOLOGICAL_REVIEW.md §L1  
**Prioridade:** 🟠 Alta — O Espírito é o agente central da Nova Aliança, oração, santificação e guarda da lei — temas presentes em toda a jornada sem estudo dedicado

**Posição sugerida:** Entre o Estudo 3 (Oração) e o Estudo 4 (Perdão) — ou como Estudo 4.5.  
**Alternativa:** Após o Estudo 11 (Permanecendo em Cristo), como aprofundamento pneumatológico.

**Temas a cobrir:**
- O Espírito como Consolador (João 14.16–17, 14.26)
- O Espírito na oração (Romanos 8.26–27)
- O Espírito como agente da Nova Aliança (Jeremias 31.33 → Atos 2.1–4)
- Os frutos do Espírito como evidência da vida em Cristo (Gálatas 5.22–23)
- A diferença entre esforço moral próprio e vida no Espírito (Gálatas 5.16–17)

**Estrutura sugerida:**
- `opening`: A promessa do Consolador — Jesus partindo, o Espírito chegando
- `question`: Você já sentiu a presença de Deus de uma forma que não consegue explicar racionalmente?
- `content`: Quem é o Espírito Santo — pessoa, não força
- `deeper`: O Espírito e a oração — ele intercede por nós em gemidos inexprimíveis
- `apply`: Oração de entrega ao Espírito Santo
- `close`: Romanos 8.14 — os que são guiados pelo Espírito são filhos de Deus
- `question`: Recapitulação e decisão

**Arquivos:** `index.html` (inserção em STUDIES[], atualização de LEVELS[], BADGES[], missões relacionadas)

**Critérios de teste:**
- [ ] O novo estudo aparece na lista correta de estudos (renumerar estudos subsequentes)
- [ ] Percorrer todo o novo estudo — todas as fases funcionam
- [ ] XP do novo estudo somado corretamente ao total
- [ ] Badge correspondente ao nível desbloqueado corretamente
- [ ] O Mentor IA conhece o conteúdo do novo estudo (verificar se o system prompt precisa ser atualizado)
- [ ] O recap do novo estudo exibe o número correto

**Definição de pronto:** o Estudo sobre o Espírito Santo está disponível na jornada, com estrutura completa, versículos, aplicação prática e recapitulação.

---

### F7.2 — Novo Estudo: O Batismo

**Origem:** THEOLOGICAL_REVIEW.md §L2  
**Prioridade:** 🟠 Alta — a jornada culmina em convite ao batismo sem ensiná-lo

**Posição sugerida:** Estudo 15 (deslocar "Jesus e o Juízo Final" para Estudo 16 e o atual Estudo 16 para Estudo 17).  
**Alternativa:** Como Estudo 16.5 (módulo Plus) — entre a jornada principal e a decisão final.

**Temas a cobrir:**
- O batismo de Jesus como modelo (Mateus 3.13–17) — e por que Jesus foi batizado se era sem pecado
- O significado do batismo: morte ao pecado, ressurreição em Cristo (Romanos 6.3–11)
- A forma do batismo: imersão como símbolo da sepultura e ressurreição
- O batismo como passo público de compromisso, não de salvação por mérito
- O batismo na história da igreja primitiva (Atos 2.38–41, Atos 8.35–39)
- A diferença entre batismo infantil (dedicação) e batismo por decisão pessoal (confissão de fé)

**Arquivos:** `index.html` (inserção em STUDIES[], renumeração)

**Critérios de teste:**
- [ ] O novo estudo aparece na posição correta na jornada
- [ ] Romanos 6.3–11 está presente e contextualizado
- [ ] O estudo conecta explicitamente ao convite do Estudo 16/17 ("próximos passos: batismo")
- [ ] A aplicação prática inclui contato com pastor/capelão para conversa sobre o batismo
- [ ] Toda a numeração de estudos subsequentes foi atualizada

**Definição de pronto:** o Estudo sobre o Batismo ensina o significado teológico, histórico e prático do batismo, preparando o discípulo para a decisão final.

---

### F7.3 — Atualizar Prompt do Mentor IA com Novos Conteúdos

**Origem:** MENTOR_AI_REVIEW.md §system prompt, THEOLOGICAL_REVIEW.md §L1, §L2  
**Prioridade:** 🟡 Média — o system prompt menciona "17 estudos" quando há 16 (ou 18 após F7.1 e F7.2)

**Problema (já identificado):**  
O system prompt da IA cita "17 estudos" (linha ~1958) quando a jornada tem 16. Após F7.1 e F7.2, terá 18.

**Solução:**  
Atualizar o system prompt para:
1. Corrigir o número de estudos
2. Incluir resumo do Estudo sobre o Espírito Santo
3. Incluir resumo do Estudo sobre o Batismo
4. Adicionar instrução de que "imagens para adoração e o sábado são abordados no Estudo 14 — o Mentor IA não deve usar linguagem confrontadora com usuários de outras tradições ao discutir esses temas"

**Arquivos:** `index.html` (system prompt inline em `sendAI`, linhas ~1958–2048)

**Critérios de teste:**
- [ ] Perguntar ao Mentor IA quantos estudos tem a jornada — resposta deve ser o número correto
- [ ] Perguntar sobre o Espírito Santo — a IA deve referenciar o estudo correto
- [ ] Perguntar sobre batismo — a IA deve referenciar o estudo correto
- [ ] Perguntar sobre o sábado de forma gentil (tradição diferente) — IA deve responder com pastoral, não com confronto

**Definição de pronto:** o system prompt reflete com precisão o número e conteúdo de todos os estudos da jornada.

---

**Relatório de conclusão da Fase 7:**
Ao final desta fase: lacunas teológicas mais críticas resolvidas (Espírito Santo e Batismo), sistema de IA alinhado com o conteúdo atualizado.

---

## FASE 8 — Arquitetura

**Objetivo:** separar o código em arquivos distintos para habilitar caching de bytecode no V8, facilitar manutenção e permitir testes unitários automatizados.  
**Pode ser implementada em:** 4–6 sessões.  
**Não depende de nenhuma outra fase, mas é a mais arriscada — requer testes exaustivos.**

---

### F8.1 — Externalizar JavaScript em `app.js`

**Origem:** PERFORMANCE_REVIEW.md §7.2, §13  
**Severidade:** 🟢 Baixa/Planejamento — melhora parse time em 50–80% em visitas subsequentes

**Problema:**  
Scripts inline em HTML não são elegíveis para V8 code cache. A cada visita, mesmo com SW ativo, o motor JS reparseia e recompila ~200KB de código.

**Solução:**  
Extrair todo o JavaScript para `app.js`. O HTML passa a ter apenas:
```html
<script src="app.js" defer></script>
```
O SW passa a cachear `app.js` como asset estático, e o V8 cacheia o bytecode compilado após a segunda visita.

**Sequência de trabalho:**
1. Extrair o JS para `app.js`
2. Extrair o CSS para `styles.css`
3. Manter o HTML com apenas estrutura e referências
4. Atualizar o SW para cachear os novos arquivos
5. Atualizar o pipeline de build (F3.5) para minificar cada arquivo separadamente

**Arquivos:** `index.html` (reduzido a estrutura), `app.js` (novo), `styles.css` (novo), `service-worker.js` (atualizar ASSETS)

**Critérios de teste:**
- [ ] O app funciona identicamente após a separação (todos os 16 estudos, todas as 12 telas)
- [ ] DevTools → Sources: `app.js` e `styles.css` devem aparecer como recursos separados
- [ ] Segunda visita: V8 code cache ativo (verificável via `chrome://tracing`)
- [ ] Service Worker cacheia `app.js` corretamente
- [ ] Pipeline de build minifica `app.js` separadamente

**Definição de pronto:** o código JavaScript está em `app.js` separado, com parse time reduzido em visitas subsequentes e cache de bytecode ativo.

---

### F8.2 — Lazy Loading das Telas Não-Frequentes

**Origem:** PERFORMANCE_REVIEW.md §11.1  
**Severidade:** 🟢 Baixa — 12 telas no DOM desde o início, incluindo o painel administrativo

**Solução:**  
Telas de acesso raro (`screen-tutorpanel`, `screen-panel`) são geradas dinamicamente na primeira abertura:

```javascript
function showScreen(n) {
  // Lazy-generate tela se ainda não foi criada
  if (n === 'tutorpanel' && !document.getElementById('screen-tutorpanel').innerHTML.trim()) {
    document.getElementById('screen-tutorpanel').innerHTML = generateTutorPanelHTML();
  }
  // ... lógica de toggle existente
}
```

**Arquivos:** `index.html` (função `showScreen` e templates das telas lazy)

**Critérios de teste:**
- [ ] Acessar o painel do tutor pela primeira vez — gerado e funcionando corretamente
- [ ] Acessar uma segunda vez — não deve ser gerado novamente (verificar que o conteúdo já existe)
- [ ] Funcionalidades do painel (listar tutores, autenticar) funcionando normalmente

**Definição de pronto:** as telas de uso raro só existem no DOM após serem acessadas pela primeira vez.

---

### F8.3 — Testes Automatizados

**Origem:** todas as auditorias — ausência total de testes identificada como risco sistêmico  
**Severidade:** 🟢 Baixa — sem testes, regressões são identificadas apenas manualmente

**Solução:**  
Criar suite de testes com Playwright (end-to-end) ou Vitest (unitários para funções JS):

```javascript
// Exemplos de testes unitários (Vitest)
describe('escapeHTML', () => {
  it('deve escapar tags HTML', () => {
    expect(escapeHTML('<script>alert(1)</script>')).toBe('&lt;script&gt;alert(1)&lt;/script&gt;');
  });
});

describe('save/load', () => {
  it('deve persistir e recuperar o ST', () => {
    const st = { done: [0,1], xp: 160, userName: 'João' };
    save(st);
    const loaded = load();
    expect(loaded.done).toEqual([0,1]);
    expect(loaded.xp).toBe(160);
  });
});

describe('migrateSchema', () => {
  it('deve normalizar ST sem _v', () => {
    const st = migrateSchema({});
    expect(st._v).toBe(1);
    expect(st.done).toEqual([]);
    expect(st.journal).toEqual([]);
  });
});
```

```javascript
// Testes E2E (Playwright)
test('percorrer Estudo 1 completo', async ({ page }) => {
  await page.goto('/');
  // preencher nome, escolher tutor, iniciar jornada
  // avançar todas as fases do Estudo 1
  // verificar que o Estudo 1 aparece como concluído na home
  // verificar XP aumentou
});
```

**Arquivos:** `tests/unit/` (novo), `tests/e2e/` (novo), `package.json` (atualizar scripts)

**Critérios de teste (dos próprios testes):**
- [ ] Todos os testes unitários passam em `npm test`
- [ ] Testes E2E passam em `npm run test:e2e`
- [ ] CI executa testes a cada push (GitHub Actions)
- [ ] Teste de regressão XSS: payload malicioso no diário não executa código
- [ ] Teste de persistência: dados sobrevivem ao reload da página

**Definição de pronto:** suíte de testes automatizados com cobertura das funções críticas, executada automaticamente no CI a cada push.

---

**Relatório de conclusão da Fase 8:**
Ao final desta fase: JS externalizado com bytecode caching, telas lazy-loaded, suite de testes automatizados cobrindo funções críticas e fluxos principais.

---

## Mapa de Dependências entre Fases

```
F0 (Segurança)          ─── independente, deve ser a PRIMEIRA fase
F1 (Dados)              ─── independente
F2 (Conteúdo)           ─── independente
F3 (PWA)                ─── independente
F4 (UX)                 ─── requer F0.1 (escapeHTML, modal) já disponível
F5 (IA)                 ─── requer F0.1 (escapeHTML para Markdown)
F6 (Diário)             ─── requer F0.1 (XSS), beneficia de F1.1 (migração de datas)
F7 (Teologia)           ─── independente, requer aprovação do capelão
F8 (Arquitetura)        ─── deve ser última — reestrutura o arquivo base
```

**Regra de ouro:** F0 deve preceder todas as outras. F8 deve ser feita por último. As demais (F1–F7) podem ser implementadas em qualquer ordem, em paralelo se houver múltiplos colaboradores.

---

## Cronograma Sugerido

| Semana | Fases | Entregável |
|--------|-------|-----------|
| 1 | F0 | App seguro — XSS corrigido, senhas protegidas, erro científico removido |
| 2 | F1 + F2 | Dados íntegros + conteúdo corrigido em todos os 16 estudos |
| 3 | F3 | PWA funciona offline com fontes, performance melhorada |
| 4 | F4 | UX consistente — sem dialogs nativos, transições, botões acessíveis |
| 5 | F5 | Mentor IA com Markdown, histórico robusto, sugestões contextuais |
| 6–7 | F6 | Diário completo com edição, exportação e busca |
| 8–10 | F7 | Dois novos estudos (Espírito Santo + Batismo) revisados e aprovados |
| 11–12 | F8 | Arquitetura separada, testes automatizados, CI/CD completo |

---

## Métricas de Sucesso por Fase

| Fase | Métrica principal | Alvo |
|------|------------------|------|
| F0 | Vulnerabilidades críticas | 0 XSS, senhas protegidas |
| F1 | Perda de dados por bugs | 0 incidentes de corrupção em restart |
| F2 | Erros de conteúdo | 0 erros de numeração ou título |
| F3 | Lighthouse Performance | ≥ 85 · Offline: app funcional com fonte correta |
| F4 | alert()/confirm() nativos | 0 ocorrências |
| F5 | Markdown literais na IA | 0 asteriscos visíveis em resposta normal |
| F6 | Exportação de dados | Usuário consegue backup completo em < 3 toques |
| F7 | Cobertura teológica | 0 lacunas críticas identificadas na THEOLOGICAL_REVIEW |
| F8 | Parse time (v. subsequentes) | Redução ≥ 40% medida via DevTools Performance |

---

## Notas Finais

**Sobre estabilidade:** cada fase é uma entrega independente e testável. O app nunca deve ficar em estado incompleto ou quebrado entre fases. Se uma fase for abandonada no meio, deve ser possível reverter para o estado anterior com um `git revert`.

**Sobre regressões:** antes de qualquer fase, criar um commit de tag `v-pre-fase-N` para permitir rollback rápido.

**Sobre aprovação:** as Fases F2 (conteúdo) e F7 (novos estudos) requerem aprovação do Capelão Wladimir antes da implementação — o conteúdo teológico e pastoral é responsabilidade ministerial, não técnica.

**Sobre testes manuais:** até a Fase 8 implementar testes automatizados, cada fase deve ser testada manualmente percorrendo todos os 16 estudos no início e no final da fase para detectar regressões.

---

*Plano elaborado com base nas auditorias: SPIRITUAL_JOURNEY_REVIEW · MENTOR_AI_REVIEW · UX_REVIEW · JOURNAL_REVIEW · DATA_REVIEW · PERFORMANCE_REVIEW · THEOLOGICAL_REVIEW.*  
*Nenhuma alteração foi implementada neste documento.*
