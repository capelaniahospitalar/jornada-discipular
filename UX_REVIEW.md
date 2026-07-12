# UX_REVIEW.md
## Revisão Completa da Experiência do Usuário — Aos Pés do Mestre Jesus
**Data:** 26 de junho de 2026  
**Versão analisada:** index (26).html  
**Escopo:** experiência completa do usuário — 12 telas, fluxos, visual, responsividade  
**Análise estática — nenhuma alteração implementada**

---

## 1. IDENTIDADE VISUAL E ATMOSFERA

### 1.1 Paleta de cores

```
Navy principal:   #0D2137  (fundo do header, botões primários)
Navy secundário:  #1A3A56  (hover, gradientes)
Azul médio:       #2E7BB5  (progress, destaques)
Ouro/dourado:     #D4A017  (XP, badges, destaques espirituais)
Teal:             #0E7B6C  (confirmações, Fase Iniciante)
Roxo:             #5C3D8F  (insights, Fase Discípulo)
Coral:            #C0392B  (avisos, tag insight)
Fundo:            #F7F5F0  (creme suave — não é branco puro)
Cartões:          #FFFFFF
Texto principal:  #0D0D1A
Texto secundário: #4A5868
Borda:            rgba(0,0,0,0.09)
```

**Avaliação:**  
A paleta é espiritualmente coerente — o navy escuro remete a profundidade e seriedade; o dourado transmite valor e conquista; o teal cria um senso de vida e crescimento. O fundo creme `#F7F5F0` em vez de branco puro reduz a fadiga visual e transmite aconchego — decisão excelente para leitura contemplativa.

**Problema:** A borda `rgba(0,0,0,0.09)` é quase imperceptível em modo light e pode desaparecer completamente em telas com baixo contraste ou brilho reduzido. Os cartões podem parecer "flutuando" sem separação visual clara.

### 1.2 Tipografia

| Fonte | Uso | Tamanho médio |
|---|---|---|
| Lora (serif) | Títulos, citações, nomes de estudos, versículos | 20–35px |
| Source Sans 3 | Corpo, rótulos, botões, inputs | 14–21px |

A combinação Lora (contemplativa, literária) + Source Sans 3 (legível, neutra) é uma das escolhas mais acertadas do projeto. Lora carrega a dimensão espiritual; Source Sans 3 garante funcionalidade. A hierarquia é clara e coerente.

**Problema:** fonte carregada via Google Fonts — sem fallback adequado. Em modo offline ou com conexão lenta, o app exibirá `sans-serif` (Arial/Helvetica) antes do carregamento, causando um flash de layout. Não há `font-display: swap` ou pré-carregamento via `<link rel="preload">`.

### 1.3 Transmite calma, simplicidade e foco espiritual?

**Sim, em grande medida.** O design é silencioso — não há elementos piscando, banners, pop-ups intrusivos ou cores gritantes. O fundo creme, os espaços generosos e a tipografia Lora criam uma atmosfera contemplativa genuína.

**Tensões identificadas:**
- Os elementos de gamificação (XP, badges, bosses, níveis RPG) contrastam com a atmosfera contemplativa — funcionam bem, mas podem parecer "jogosos" demais para usuários que buscam apenas uma experiência devocional.
- O botão flutuante verde do WhatsApp (posicionado no canto inferior esquerdo) é funcionalmente útil mas visualmente disruptivo — o verde WhatsApp `#25D366` não pertence à paleta espiritual do app.

---

## 2. PRIMEIRO ACESSO — WELCOME SCREEN

### 2.1 Hero visual

A tela de boas-vindas tem um gradiente vertical rico:
`#D6EAF8 → #5BA4CF → #1A3A56 → #0D2137` (céu ao amanhecer descendo para noite profunda)

Este gradiente é uma das escolhas mais belas do projeto — metaforicamente perfeito para o início de uma jornada espiritual (aurora → profundidade). O logo do Hospital Adventista Silvestre com `drop-shadow` está bem integrado.

**Problema:** o elemento `.life-stamp` ("Arquitetura da Vida") com rotação `rotate(8deg)` e borda dourada é visualmente atraente, mas sua posição ao lado do título principal cria competição visual. Em telas de 320px, o título pode comprimir.

### 2.2 Mensagem de boas-vindas

O texto de abertura é genuinamente bom:  
> "Você está prestes a iniciar uma aventura que já transformou milhões de vidas ao longo da história."

É evangelístico sem ser agressivo, convidativo sem ser superficial. A estrutura tripartite em negrito ("Cada passo... Cada desafio... Cada decisão...") é ritmicamente eficaz.

### 2.3 Grid de funcionalidades

O grid 2×3 de feature cards (ícones emoji + título + descrição) é uma solução padrão de onboarding. Funciona, mas tem algumas limitações:
- 6 cartões em grid 2 colunas cria 3 linhas que o usuário precisa scrollar para ver
- Em telas menores, as descrições de 14px ficam pequenas
- Não há hierarquia entre as 6 funcionalidades — todas parecem igualmente importantes

### 2.4 Formulário de identificação

O formulário de nome + perfil (paciente/colaborador/amigo) está bem estruturado:
- Inputs com label uppercase + placeholder claro
- Feedback visual imediato ao selecionar perfil (highlight verde-teal)
- Persistência automática via `oninput` (salva no localStorage enquanto digita)

**Problemas:**
- O input de nome não tem `autocomplete="name"` — perda de conveniência para usuários que retornam
- O formulário não tem `required` visual explícito — não há asterisco ou indicação de obrigatoriedade
- Os cartões de perfil não exibem check de confirmação após seleção — apenas highlight de fundo; em ambientes de alta luminosidade pode não ser perceptível

### 2.5 Seleção de tutor

Os 4 cartões de tutor (capelães) + 1 cartão de tutor convidado (dashed border) formam uma lista clara. O estado selecionado (teal + check "✓") é visualmente inequívoco.

**Problema:** o tutor convidado com `border-style: dashed` é uma boa diferenciação visual, mas o formulário inline (nome + WhatsApp) que aparece ao selecionar pode ser confuso — o usuário não sabe de imediato que precisa preencher mais campos para prosseguir.

### 2.6 Botão de início

```css
.start-journey-btn {
  background: linear-gradient(135deg, #0D2137, #185FA5);
  box-shadow: 0 4px 15px rgba(13,33,55,0.3);
  transition: all .2s;
}
.start-journey-btn:hover { transform: translateY(-1px); }
.start-journey-btn:disabled { background: #ccc; }
```

O único botão com gradiente e `box-shadow` na tela — boa hierarquia de ação. O estado `disabled` (cinza) é claro. O `hover: translateY(-1px)` é um toque de qualidade.

**Problema:** o botão usa `alert()` nativo para validação (`"Escolha um tutor para começar!"`). Em mobile, `alert()` quebra o contexto visual do app — parece erro de sistema, não mensagem do app.

### 2.7 Fluxo do onboarding

```
Nome → Perfil → Tutor → Iniciar
```

**Sem progressão visual** — não há indicador de etapas (ex: "Passo 1 de 3"). O usuário não sabe quanto falta para começar. Para uma tela longa (estimativa: 2–3 scrolls completos), a ausência de progressão pode causar abandono.

**Não há validação em tempo real** do nome — o usuário pode iniciar a jornada sem nome. O nome é personalizador crítico (usado via `N()` em todos os estudos).

---

## 3. HOME SCREEN

### 3.1 Header (`.hdr`)

O header navy com título em Lora, level badge, barra XP e hint de próximo nível é funcionalmente completo. Os dois círculos decorativos (`:before` e `:after`) como ornamento sutil são elegantes.

**Densidade de informação:** o header contém em apenas 3 linhas: nome do app, ícone de nível com nome, barra XP e hint. Para um primeiro uso, essa densidade pode ser avassaladora — o usuário ainda não sabe o que é XP, nível ou a barra.

### 3.2 Stats grid (3 cartões)

Os 3 cards de estatísticas (estudos concluídos, dias de streak, XP total) em grid 3 colunas funcionam bem visualmente. O problema é semântico: no primeiro acesso, todos os valores são zero — "0 estudos, 0 dias, 0 XP" — transmite vazio ao invés de convite.

### 3.3 Streak banner

O banner azul claro de streak ("🔥 sua sequência de estudo") é bem posicionado e agradável visualmente. Em 0 dias (primeiro uso), exibe "0 dias de estudo consecutivos" — fria introdução ao conceito.

### 3.4 Próximo estudo (Next Card)

Este é o elemento mais importante da home e também o mais bem executado: cartão com header navy (fase + XP tag), título em Lora, badge da fase colorido e botão "Iniciar Estudo" proeminente.

**Ponto forte:** hierarquia visual clara — o botão `start-btn` em azul é a única ação de destaque na tela. Impossível não encontrar.

**Problema:** o botão "Iniciar Estudo" tem ícone SVG de chevron, mas o tamanho do ícone (14×14px) é muito pequeno em relação ao botão (font-size: 20px). Visualmente desbalanceado.

### 3.5 Progress track

A barra de progresso geral da jornada (0–16 estudos) com percentual em azul é informativa e motivadora. Bem posicionada logo abaixo do próximo estudo.

### 3.6 Badges scroll

A faixa horizontal de badges com scroll horizontal é uma solução elegante para mostrar conquistas sem ocupar muito espaço. Os estados `locked` (opacity .3, grayscale), `current` (navy gradient) e `unlocked` (dourado) são visualmente distintos.

**Problema:** scrollbar oculta (`::-webkit-scrollbar: none`) sem qualquer indicador visual de que há mais conteúdo além da borda — nem seta, nem gradiente de fade. O usuário não descobre intuitivamente que pode scrollar.

### 3.7 Atributos espirituais (barras de progresso)

O card de "Atributos" com 6 barras (Sabedoria, Oração, Amor, Perseverança, Saúde, Missão) é visualmente rico e diferenciado. A transição `width .6s ease` é suave.

### 3.8 Radar chart

O SVG do "Mapa de Discipulado" (radar/spider chart) é uma adição sofisticada. O SVG é gerado dinamicamente com `viewBox="0 0 260 230"` e labels de 9px.

**Problema:** labels de 9px no radar são muito pequenos para leitura em mobile — especialmente em telas de 320px onde o `max-width: 280px` do SVG pode render os rótulos ilegíveis.

### 3.9 Botão flutuante do tutor (WhatsApp)

```css
.tutor-btn-fixed {
  position: fixed;
  bottom: .85rem;
  left: .85rem;
  background: #25D366;
  border-radius: 24px;
  z-index: 100;
}
```

**Problema crítico de uso com uma mão:** o botão está no canto inferior **esquerdo** — a posição menos acessível para usuários destros (maioria). O polegar direito não alcança confortavelmente o canto inferior esquerdo sem reposicionar a mão. A posição convencional para FABs é o canto inferior direito.

**Problema de cobertura:** o botão tem `z-index: 100` e `position: fixed` — em telas pequenas, pode cobrir conteúdo importante da home, especialmente os botões na parte inferior da lista.

### 3.10 FAB do AI (`.ai-fab`)

```css
.ai-fab {
  position: fixed;
  bottom: 5rem;
  right: 1rem;
  z-index: 99;
}
```

O FAB do AI existe no CSS mas não é renderizado na home — `renderAIArea()` cria um botão inline, não o FAB. Isso significa que o estilo `.ai-fab` não é utilizado na versão atual.

---

## 4. ESTUDOS — STUDY SCREEN

### 4.1 Header de estudo

O header compacto (back button + "Estudo N" + título em Lora + XP tag) em navy é consistente com os demais headers do app. A barra de progresso de 3px com transição `width .35s ease` é discreta e eficaz.

### 4.2 Phase dots

```css
.pdot { width: 6px; height: 6px; }
.pdot.active { width: 22px; border-radius: 3px; }
```

Os dots de fase (bolinhas que se transformam em "pílula" na fase ativa) é um padrão reconhecível e bem implementado. A transição `all .25s` é suave.

**Problema:** dots de 6px são muito pequenos para usuários com dificuldades visuais. O contrast ratio dos dots `.done` (teal) sobre o fundo white pode ser insuficiente para baixa visão.

### 4.3 Content cards

Os cartões de conteúdo (`content-card`) com fundo branco, borda sutil e `padding: 1.15rem 1rem` são clean e focados. O `body-text` em 21px com `line-height: 1.7` é uma das melhores escolhas de legibilidade do projeto — generoso, respirado, contemplativo.

**21px é o tamanho correto** para leitura espiritual — maior que o padrão de 16px da web, permitindo foco e meditação sem esforço visual.

### 4.4 Verse buttons

```css
.verse-btn {
  background: var(--blue-pale);
  border: 1.5px solid var(--blue-light);
  border-radius: 8px;
  font-size: 17.5px;
  font-weight: 600;
}
```

Os botões de versículo expandíveis são um dos elementos mais bem executados do app. O ícone de livro + referência + hint "toque para expandir" + chevron animado é intuitivo. A expansão com border-left de 3px em azul cria uma citação visual clara.

**Problema:** `font-size: 17.5px` nos botões de versículo é menor que o corpo de texto (21px), criando uma inconsistência hierárquica — o versículo parece secundário ao corpo, mas deveria ser primário.

### 4.5 Caixas temáticas (apply, insight, decision)

| Caixa | Cor | Uso |
|---|---|---|
| `.apply-box` | Dourado pálido | Aplicação prática |
| `.insight-box` | Roxo claro | Insights/reflexões profundas |
| `.decision-box` | Navy escuro | Decisão final |

A diferenciação cromática por tipo de conteúdo é excelente — o usuário aprende visualmente o que cada cor representa ao longo dos 16 estudos.

**Problema:** a `.decision-box` (navy escuro com texto branco) é excelente visualmente, mas o texto `font-size: 19px` em italic sobre fundo escuro pode ser difícil para usuários com baixa visão.

### 4.6 Opções de resposta (`.opt-btn`)

```css
.opt-btn {
  font-size: 20px;
  padding: .75rem .9rem;
  border-radius: var(--radius-sm);
}
```

Botões de resposta com 20px e padding generoso são ideais para toque preciso. O estado `.chosen` (teal) e `.chosen-warn` (amarelo) são distintos e claros.

**Problema:** não há estado de loading/disabled entre o toque na opção e o feedback. Em redes lentas ou processamento, o botão pode parecer não ter respondido — o usuário pode tocar duas vezes.

### 4.7 Botão de navegação principal

```css
.nav-btn {
  width: 100%;
  padding: .85rem;
  font-size: 20px;
}
```

Botão de ação principal full-width, 20px, padding generoso — perfeito para uso mobile com uma mão. Sempre ao final do conteúdo, sem necessidade de scroll para encontrá-lo.

### 4.8 Navegação entre fases

A navegação entre fases de um estudo (ex: Opening → Question → Content → Apply → Close) não tem botão "voltar para fase anterior" — apenas "avançar". Isso é uma decisão pedagógica válida (não retroceder durante um estudo), mas pode frustrar usuários que querem reler a fase anterior antes de responder.

---

## 5. TELA DE CONCLUSÃO — COMPLETE SCREEN

### 5.1 Estrutura visual

```css
.complete-wrap {
  padding: 2rem 1.5rem 2.5rem;
  text-align: center;
  justify-content: center;
}
```

A tela de conclusão é centrada vertical e horizontalmente — criando um momento de "pausa" após o estudo. O ícone de 65px, título em Lora, XP ganho em dourado e preview do próximo estudo formam uma sequência narrativa satisfatória.

**Ponto forte:** é a única tela verdadeiramente centrada — transmite a sensação de "chegada" e celebração.

### 5.2 Bloqueio do botão home (diário obrigatório)

```javascript
btn.disabled = true;
btn.textContent = 'Registre seu diário para continuar';
btn.onclick = () => openJournal();
```

O bloqueio do retorno à home até registrar no diário é uma decisão pedagógica corajosa. Visualmente, o botão fica com `opacity: 0.5` e cursor `not-allowed`.

**Problema de UX:** o usuário pode não entender imediatamente por que o botão está desativado. O texto "Registre seu diário para continuar" é claro, mas sem uma animação de atenção ou destaque visual, o usuário pode ignorar e tentar clicar repetidamente.

**Ausência de prompt visual:** não há nenhuma seta ou indicação visual explicando o que aconteceu. Para usuários mais velhos ou menos digitais (contexto hospitalar), isso pode gerar confusão.

---

## 6. TELA DE REFLEXÃO — REFLECTION SCREEN

O countdown timer de 24h (`font-size: 30px; font-weight: 700; color: #fff` sobre fundo navy) é visualmente impactante. O formato `--:--:--` indica tempo restante de forma clara.

**Problema:** a tela de reflexão (24h lock) não está descrita com clareza suficiente. O usuário pode não entender o motivo do bloqueio. Um parágrafo breve sobre "por que esperar 24h" seria pedagogicamente valioso.

---

## 7. DIÁRIO ESPIRITUAL — JOURNAL SCREEN

### 7.1 Estrutura

A tela do diário tem duas abas: "Registros" e "+ Novo". A aba ativa fica navy (fundo) com texto branco — estado ativo claramente diferenciado.

### 7.2 Entradas existentes

```css
.journal-entry-hdr { background: var(--navy); }
.journal-entry-hdr span { font-family: var(--ff-serif); font-size: 16px; }
```

O cabeçalho dos registros em navy com título em Lora e data em texto opaco é visualmente elegante e coerente com o tom espiritual do app.

**Problema:** `font-size: 16px` para o título do registro é menor que o corpo do conteúdo dos estudos (21px). Lendo os próprios registros, o usuário encontra texto menor que o que estava lendo durante o estudo.

### 7.3 Textarea

```css
textarea {
  font-size: 17.5px;
  line-height: 1.6;
  resize: none;
}
```

`resize: none` é correto para mobile (evita resize acidental). `font-size: 17.5px` é adequado para escrita. `line-height: 1.6` garante boa legibilidade.

**Problema:** `resize: none` em desktop pode frustrar usuários em tablets que desejam ampliar a área de escrita. Uma solução seria `resize: vertical` com `min-height` definido.

### 7.4 Ausência de contador de caracteres

Não há indicador de comprimento mínimo ou máximo para as entradas do diário. O usuário não sabe se uma entrada muito curta ("ok") é suficiente para liberar a home.

---

## 8. MENTOR IA — AI SCREEN

### 8.1 Layout

```css
.ai-screen { height: calc(100vh - 56px); }
.ai-messages { flex: 1; overflow-y: auto; }
.ai-input-row { background: var(--card); border-top: 0.5px solid var(--border); }
```

O layout de chat (mensagens acima, input fixo embaixo) é o padrão correto e esperado para um chat. A altura `calc(100vh - 56px)` garante que o input nunca desapareça atrás do teclado virtual.

**Problema crítico:** em iOS, quando o teclado virtual aparece, ele não empurra o viewport para cima — reduz o viewport. O `height: calc(100vh - 56px)` pode não funcionar corretamente porque `100vh` é o viewport completo (incluindo a área do teclado). O input pode ficar atrás do teclado em alguns modelos de iPhone.

### 8.2 Balões de mensagem

```css
.ai-msg.user { background: var(--navy); align-self: flex-end; }
.ai-msg.bot  { background: var(--card); align-self: flex-start; }
```

A diferenciação visual usuário/bot (navy/white, right/left) é padrão e imediatamente reconhecível. O `border-bottom-right-radius: 3px` (user) e `border-bottom-left-radius: 3px` (bot) cria o detalhe visual de "cauda" de balão.

**Problema:** `font-size: 20px` nos balões é consistente com o corpo dos estudos, mas para um chat com múltiplas mensagens pode aumentar muito o scroll necessário para reler a conversa.

### 8.3 Loading indicator

```css
.ai-msg.bot.loading { color: var(--text3); font-style: italic; }
```

O indicador de carregamento é apenas `'...'` em itálico cinza. É funcionalmente mínimo — não há animação, spinner ou typing dots. Para uma espera de 3–15 segundos (conforme análise de latência no MENTOR_AI_REVIEW.md), este indicador é insuficiente.

### 8.4 Input field

```css
.ai-input { border-radius: 20px; font-size: 20px; }
.ai-send { border-radius: 50%; width: 38px; height: 38px; }
```

Input arredondado (20px border-radius) é moderno e adequado para chat. O botão de envio circular é standard.

**Problema:** botão de envio de 38×38px é menor que o mínimo recomendado de 44×44px (Apple HIG) para alvos de toque. Em telas pequenas com dedos grandes, pode ser difícil tocar precisamente.

**Chips de sugestão (ai-sugs):** o elemento existe no HTML e CSS, mas a função `openAI()` limpa seu conteúdo sem populá-lo. Funcionalidade planejada não implementada — oportunidade perdida de onboarding no chat.

---

## 9. PROGRESSO E GAMIFICAÇÃO

### 9.1 Sistema de níveis

8 níveis (Buscador → Enviado) com XP progressivo. A representação via `level-badge` no header home é compacta. O `level-card` com gradiente navy e descrição do nível é motivador.

**Problema:** o sistema de XP não tem "preview" do que pode ser ganho — o usuário não sabe quantos XP vai ganhar antes de iniciar um estudo (exceto pela tag `xp-tag` no header da study screen). Na home, o próximo estudo mostra o XP na tag dourada — isso está bem posicionado.

### 9.2 Obstáculos/Bosses

A seção de bosses com barras de progresso por obstáculo espiritual (oito obstáculos) é conceitualmente interessante mas visualmente complexa — 8 barras empilhadas, cada uma com ícone + label + barra + status, em uma estrutura densa.

### 9.3 Missões

A lista de missões com checkbox circular (teal quando concluída) é visualmente limpa. A transição `all .15s` nos checkboxes é suave.

**Problema:** missions e bosses estão na home mas não têm hierarquia visual clara em relação ao "próximo estudo" — que é a ação principal. Usuários podem ficar distraídos com missões ao invés de prosseguir com os estudos.

---

## 10. CONFIGURAÇÕES / PAINEL DO TUTOR

### 10.1 Painel do discípulo (panel screen)

O painel exibe estatísticas do discípulo para o tutor ver. Funcionalmente correto, mas sem hierarquia clara entre dados importantes e secundários.

### 10.2 Painel do tutor (tutorpanel screen)

O sistema de login do tutor com seleção de nome + senha é funcionalmente adequado.

**Problema de visibilidade:** o acesso ao painel do tutor está no rodapé da home em um botão de texto pequeno (`font-size: 16px; color: var(--text2)`). Não há distinção visual clara entre "área do discípulo" e "área do tutor" — um discípulo pode acidentalmente descobrir e acessar esta área.

### 10.3 Ausência de "Configurações" explícitas

Não há uma tela de configurações dedicada. Ações como:
- Mudar o nome do usuário
- Trocar o tutor
- Alterar o perfil (paciente/colaborador/amigo)
- Preferências de notificação

...estão espalhadas pela interface (trocar tutor aparece na home; o nome só pode ser alterado na welcome screen). Não há um local único para configurar o app.

---

## 11. MODO OFFLINE

### 11.1 Service Worker

O service-worker.js está referenciado na linha 4062, mas **o arquivo não existe no repositório**. Isso significa que o app **não tem funcionalidade offline real** — apesar de se declarar PWA.

**Impacto:** toda a funcionalidade offline depende apenas do caching automático do navegador (que não é garantido). Em uma primeira visita offline, o app não carregará. Mesmo após carregamento, recursos como Google Fonts não estarão disponíveis offline.

### 11.2 Comportamento online-only

| Funcionalidade | Offline? |
|---|---|
| Estudos bíblicos (conteúdo) | ✅ (embutido no HTML) |
| Progresso do usuário | ✅ (localStorage) |
| Diário espiritual | ✅ (localStorage) |
| Mentor IA | ❌ (requer proxy + Claude API) |
| Contato com tutor (WhatsApp) | ❌ (requer WhatsApp) |
| Painel do tutor | ❌ (requer Google Sheets API) |
| Google Fonts (Lora + Source Sans) | ❌ (requer internet) |
| Ícone/Logo do hospital | ✅ (Base64 embutido) |

### 11.3 Ausência de feedback offline

Quando o app está offline e o usuário tenta acessar o Mentor IA, recebe a mensagem de erro de rede. Não há indicação proativa de que o app está offline nem quais funcionalidades estão disponíveis sem conexão.

---

## 12. FLUXO ENTRE TELAS

### 12.1 Função `showScreen()`

```javascript
function showScreen(n) {
  document.querySelectorAll('.screen').forEach(s => s.classList.remove('active'));
  document.getElementById('screen-' + n).classList.add('active');
  resetScroll();
}
```

A navegação é puramente baseada em `display: none / flex` via classe `.active`. Não há:
- Animação de transição entre telas
- Indicação visual de "carregando próxima tela"
- Estado intermediário

### 12.2 `resetScroll()`

```javascript
function resetScroll() {
  window.scrollTo(0, 0);
  requestAnimationFrame(() => { window.scrollTo(0, 0); });
}
```

A implementação com `requestAnimationFrame` para garantir scroll ao topo após renderização async é tecnicamente correta — mas não garante que funcione em todos os casos de conteúdo gerado dinamicamente via `innerHTML`.

### 12.3 Ausência de animações de transição

A troca entre telas é **imediata** — sem fade, slide ou qualquer transição. Para um app de caráter espiritual e contemplativo, transições suaves (mesmo que simples, como `opacity: 0→1` em 200ms) agregariam significativamente à sensação de calma e qualidade.

### 12.4 Botão "Voltar"

O back button (`←`) leva sempre ao home screen via `onclick="goHome()"`. Não há histórico de navegação — se o usuário está no Mentor IA e pressiona voltar, vai direto para a home, não para onde estava antes.

**Problema:** o botão voltar do sistema (Android back gesture) não é interceptado — pode fechar o app ou navegar para uma URL diferente. Sem service worker, não há controle sobre o comportamento do histórico do browser.

---

## 13. FEEDBACK VISUAL

### 13.1 Estados interativos

| Elemento | Hover | Active | Disabled | Loading |
|---|---|---|---|---|
| `.nav-btn` | background .15s | — | opacity/cursor | — |
| `.opt-btn` | background .15s | — | — | — |
| `.start-btn` | background .15s | — | — | — |
| `.verse-btn` | background .15s | — | — | — |
| `.tutor-card` | background .15s | — | — | — |
| `.ai-send` | background .15s | — | — | — |
| AI input | — | — | não implementado | — |

**Padrão consistente:** quase todos os elementos interativos usam `transition: background .15s` como único estado de feedback. É funcional mas minimalista.

**Ausências críticas:**
- Sem estado `:active` (toque visual instantâneo em mobile — o `hover` não funciona em touch)
- Sem feedback tátil (não usa `navigator.vibrate()` que seria opcional mas impactante)
- Sem estado de loading em botões (o usuário não sabe se a ação foi registrada)

### 13.2 Confirmações

| Ação | Confirmação atual |
|---|---|
| Selecionar tutor | highlight teal + check "✓" |
| Responder questão | `.chosen` teal ou amarelo |
| Avançar fase | dots atualizam |
| Completar estudo | tela complete |
| Registrar diário | botão home desbloqueia |
| Recomeçar jornada | `confirm()` nativo |
| Copiar link | `alert()` nativo |
| Tutor inválido | `alert()` nativo |

**3 usos de `alert()` / `confirm()` nativos** — quebram o contexto visual, aparecem como diálogos do sistema operacional, não do app. Especialmente problemático em dispositivos iOS onde `alert()` pode bloquear o thread de UI.

---

## 14. ANIMAÇÕES E LOADING

### 14.1 Animações existentes

| Elemento | Animação | Duração |
|---|---|---|
| XP fill bar | `width .7s ease` | 700ms |
| Track progress | `width .5s ease` | 500ms |
| Phase dots | `all .25s` | 250ms |
| Study prog bar | `width .35s ease` | 350ms |
| Attr bars | `width .6s ease` | 600ms |
| Boss bars | `width .6s ease` | 600ms |
| Verse chevron | `transform .2s` | 200ms |
| Botão tutor WA | `transform: scale(1.03)` | .15s |
| Start journey btn | `translateY(-1px)` | .2s |

As animações de largura das barras de progresso são o ponto mais forte do sistema de feedback visual — suaves, proporcionadas e funcionalmente informativas.

**Ausências:**
- Sem animação de entrada para telas (fade in)
- Sem animação de conclusão de estudo (poderia ter uma pequena confeti, pulse, ou flash dourado)
- Sem animação de conquista de badge
- Sem animação de unlock (desbloqueio do próximo estudo)
- Sem shimmer/skeleton loading para conteúdo gerado dinamicamente

### 14.2 Loading states

| Operação | Loading state |
|---|---|
| Navegação entre telas | Nenhum |
| Carregamento de estudos | Nenhum (instantâneo — conteúdo embutido) |
| Mentor IA aguardando | `'...'` em italic cinza |
| Painel do tutor carregando | "Carregando lista de discípulos..." (texto) |
| Tutor panel API error | Mensagem de erro em texto |

O único loading real é o do Mentor IA e do painel do tutor. O texto "Carregando lista de discípulos..." no tutor panel sem indicador visual (spinner, skeleton) parece primitivo.

---

## 15. TEMPO DE RESPOSTA

| Operação | Tempo estimado | Feedback durante espera |
|---|---|---|
| Troca de tela | < 50ms (instantâneo) | Nenhum (desnecessário) |
| Renderização da home | < 100ms | Nenhum (aceitável) |
| Carregamento de fontes (1ª vez) | 500ms–3s | Flash de fonte fallback |
| Mentor IA (resposta) | 3–20s | `'...'` italic — insuficiente |
| Painel do tutor (API) | 1–5s | Texto de "Carregando..." |
| WhatsApp (link externo) | Instantâneo | n/a |

---

## 16. USO COM UMA MÃO

### 16.1 Mapeamento de alcance (polegar direito, 375px de largura)

```
ZONA VERDE (fácil alcance):    bottom 0–40% da tela
ZONA AMARELA (alcance médio):  40–70% da tela
ZONA VERMELHA (difícil):       top 30% da tela
```

| Elemento | Posição | Acessibilidade |
|---|---|---|
| Botão flutuante WhatsApp | Bottom-left | 🔴 Difícil (destros) |
| Input do Mentor IA | Bottom | ✅ Fácil |
| Botão de envio (AI) | Bottom-right | ✅ Fácil |
| Botão "Iniciar Estudo" (home) | Meio-baixo | ✅ Fácil |
| Back button (← ) | Top-left | 🔴 Difícil |
| XP tag do estudo | Top-right | 🔴 Difícil |
| Verse buttons (durante estudo) | Meio variável | ✅ Adequado |
| `.nav-btn` (avançar estudo) | Bottom | ✅ Fácil |
| Badges scroll | Meio | 🟡 Médio |

**Problema maior:** o back button (`←`) no canto superior esquerdo é inacessível com o polegar direito sem reposicionar a mão. Em telas de 6"+ (iPhone Pro Max, Samsung Galaxy), isso é especialmente problemático. Muitos apps modernos colocam o back button no canto inferior esquerdo ou usam gesture navigation.

---

## 17. USO EM CELULARES PEQUENOS (320px — iPhone SE, Moto G)

### 17.1 Layout

`max-width: 420px` com `margin: 0 auto` — o app tem largura máxima fixada. Em telas de 320px de largura, o conteúdo ocupa 100% da largura com `padding: 0 1rem` (16px de cada lado = 288px de área útil).

**Problemas em 320px:**

1. **Título da welcome screen:** `font-size: 35px` com `font-weight: 600` para o título "Aos Pés do Mestre Jesus" pode quebrar em 3 linhas em 288px de área útil.

2. **Grid de features 2×3:** `gap: 8px` com `padding: .9rem .75rem` nos cards — em 288px úteis, cada card tem ~136px de largura. O `feature-title` em `font-size: 16px` com wrap pode ficar em 3 linhas, aumentando muito a altura do grid.

3. **Stats grid (3 colunas):** em 288px, cada card tem ~88px. O `scard-v` em 21px é legível. O `scard-l` em 12.5px é no limite do legível.

4. **Radar chart:** `max-width: 280px` — em 320px de tela o chart ocupa quase 100% da largura. Os labels de `font-size: 9px` ficam ilegíveis.

5. **Botão flutuante WA:** `max-width: calc(100vw - 1.7rem)` com `text-overflow: ellipsis` — trata bem a largura. Em 320px, o nome do tutor será truncado.

6. **Tags de fase + XP no header do estudo:** na mesma linha (`display: flex`), em 320px podem comprimir o título do estudo além do que o `text-overflow: ellipsis` consegue mostrar com utilidade.

### 17.2 Inputs e alvos de toque

`padding: .6rem 1rem` nos inputs = altura de ~40px. Aceitável em 320px.  
`padding: .85rem` nos `.nav-btn` = altura de ~52px. Bom.  
`padding: 2px 7px` nas XP tags = muito pequeno para toque — mas não são interativas.

---

## 18. USO EM TABLETS (768px+)

O app tem `max-width: 420px; margin: 0 auto` — em tablets, o conteúdo fica centralizado em uma coluna estreita com margens laterais. Isso é uma decisão válida para um app de discipulado (foco e leitura), mas:

1. **Desperdício de espaço:** em um iPad de 1024px de largura, 604px das laterais ficam vazios (fundo cinza `#F7F5F0`). Não há nenhum conteúdo ou decoração nas laterais.

2. **Fontes grandes em tela grande:** o `body-text` de 21px e `welcome-title` de 35px em um iPad 10" ficam proporcionalmente pequenos para a tela — o usuário lê conteúdo de smartphone numa tela de tablet.

3. **Botão flutuante WhatsApp:** em tablets, o `bottom: .85rem; left: .85rem` fica fora da coluna de conteúdo (que vai de `cx - 210px` a `cx + 210px`). O botão aparece na lateral esquerda da tela, longe do conteúdo.

4. **Radar chart:** em tablets, o `max-width: 280px` é bem menor que o card que o contém — não escala para aproveitar o espaço disponível.

5. **AI input:** em tablets com teclado físico, pressionar Enter envia a mensagem (`onkeydown="if(event.key==='Enter')sendAI()"`). Isso é correto e apreciado.

---

## 19. TABELA-RESUMO DE PROBLEMAS UX

| ID | Área | Descrição | Gravidade |
|---|---|---|---|
| UX01 | Onboarding | Sem progressão visual (passo 1/3) | 🟡 Médio |
| UX02 | Onboarding | Nome não é obrigatório — pode iniciar sem nome | 🟡 Médio |
| UX03 | Onboarding | `alert()` nativo para validação de tutor | 🟡 Médio |
| UX04 | Fontes | Google Fonts sem fallback offline / font-display | 🟡 Médio |
| UX05 | Home | Badges sem indicador de scroll horizontal | 🟡 Médio |
| UX06 | Home | Radar labels de 9px ilegíveis em mobile pequeno | 🟡 Médio |
| UX07 | Home | Botão WA fixo no canto inferior esquerdo (mão direita) | 🔴 Alto |
| UX08 | Home | FAB do AI definido no CSS mas não renderizado | 🔵 Baixo |
| UX09 | Estudos | Sem botão de retorno à fase anterior | 🔵 Baixo |
| UX10 | Estudos | Verso buttons em 17.5px — menor que corpo (21px) | 🔵 Baixo |
| UX11 | Estudos | Sem estado :active em botões para toque mobile | 🟡 Médio |
| UX12 | Conclusão | Botão bloqueado sem animação de atenção | 🟡 Médio |
| UX13 | Mentor IA | Loading indicator `'...'` — insuficiente para 3-20s | 🔴 Alto |
| UX14 | Mentor IA | Botão envio 38×38px abaixo do mínimo de 44×44px | 🟡 Médio |
| UX15 | Mentor IA | iOS: input pode ficar atrás do teclado virtual | 🔴 Alto |
| UX16 | Mentor IA | Chips de sugestão não implementados | 🟡 Médio |
| UX17 | Navegação | Sem animações de transição entre telas | 🟡 Médio |
| UX18 | Navegação | Back button no topo-esquerdo — inacessível com uma mão | 🔴 Alto |
| UX19 | Navegação | 3 usos de `alert()`/`confirm()` nativos | 🟡 Médio |
| UX20 | Offline | Service Worker ausente — PWA sem funcionalidade offline | 🔴 Alto |
| UX21 | Offline | Sem feedback proativo de status offline | 🟡 Médio |
| UX22 | Configurações | Sem tela dedicada de configurações | 🟡 Médio |
| UX23 | Visual | Sem animação de conclusão de estudo / conquista | 🟡 Médio |
| UX24 | Visual | Sem skeleton/shimmer loading em conteúdo dinâmico | 🔵 Baixo |
| UX25 | Tablet | Conteúdo não escala — coluna estreita em telas grandes | 🔵 Baixo |
| UX26 | 320px | Radar ilegível; título pode quebrar em 3+ linhas | 🟡 Médio |
| UX27 | Diário | Sem contador de caracteres / indicador de suficiência | 🔵 Baixo |
| UX28 | Gamificação | Missions/bosses competem com ação principal (estudar) | 🔵 Baixo |

---

## 20. O QUE ESTÁ MUITO BEM — PONTOS ALTOS DE UX

1. **`body-text` em 21px com line-height 1.7** — a melhor decisão tipográfica do projeto. Leitura espiritual contemplativa e sem esforço.

2. **Fundo creme `#F7F5F0`** — reduz fadiga ocular e transmite acolhimento. Superior ao branco puro.

3. **Combinação Lora + Source Sans 3** — hierarquia visual clara entre contemplação (serif) e funcionalidade (sans).

4. **Verse buttons expandíveis** — interface elegante, intuitiva e educativa. O chevron animado é um micro-detalhe de qualidade.

5. **Phase dots com transformação de bolinha → pílula** — feedback visual de progresso dentro do estudo, compacto e claro.

6. **Decision box em navy com texto branco e Lora italic** — o momento de decisão mais impactante visualmente de toda a trilha.

7. **Botão flutuante do WhatsApp oculto durante estudos** (`.hidden-on-study`) — atenção aos detalhes, evita distrações durante o estudo.

8. **Apply box dourada / Insight box roxa / Decision box navy** — sistema de cores semântico que o usuário aprende ao longo da jornada. Excelente UX progressiva.

9. **Gradiente welcome screen** (azul claro → profundo navy) — metaforicamente perfeito para o início da jornada.

10. **`resetScroll()` com `requestAnimationFrame`** — atenção técnica ao comportamento de scroll em conteúdo gerado dinamicamente.

---

## 21. RECOMENDAÇÕES POR PRIORIDADE

### Prioridade 1 — Críticas

**R1 — Mover botão WhatsApp para canto inferior direito (UX07)**  
`left: .85rem` → `right: .85rem` e ajustar para não conflitar com o FAB do AI.

**R2 — Implementar Service Worker (UX20)**  
Criar `service-worker.js` com cache de Shell (HTML, CSS, JS) e assets. Prioridade máxima para PWA offline real.

**R3 — Corrigir comportamento do input no iOS (UX15)**  
Substituir `height: calc(100vh - 56px)` por `height: calc(100dvh - 56px)` (dynamic viewport height — suportado desde 2023 em iOS Safari) na tela de AI.

**R4 — Melhorar loading do Mentor IA (UX13)**  
Substituir `'...'` por animação CSS de typing dots:  
```css
@keyframes typing { 0%,80%,100%{opacity:0} 40%{opacity:1} }
```

**R5 — Eliminar `alert()` / `confirm()` nativos (UX19, UX03)**  
Criar componente de toast/modal inline (simples div posicionado) para todas as validações e confirmações.

### Prioridade 2 — Importantes

**R6 — Adicionar indicador de progresso no onboarding (UX01)**  
Barra de progresso ou "Passo 1/3" visível no topo da welcome screen.

**R7 — Validação do nome como obrigatório (UX02)**  
Bloquear o botão "Iniciar" se o nome estiver vazio, com mensagem inline (não alert).

**R8 — Adicionar `font-display: swap` e preconnect (UX04)**  
```html
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Lora...&display=swap" rel="stylesheet">
```

**R9 — Animação de transição entre telas (UX17)**  
Adicionar `opacity: 0 → 1` em 200ms nas telas ao ativarem (`.screen.active { animation: fadeIn .2s ease }`)

**R10 — Indicador de scroll nos badges (UX05)**  
Adicionar gradiente `::after` fade no lado direito do container de badges.

**R11 — Implementar chips de sugestão no Mentor IA (UX16)**  
Popularar `#ai-sugs` com 3–5 perguntas contextuais ao abrir a tela.

**R12 — Animação de conclusão de estudo (UX23)**  
Adicionar pulse ou shimmer dourado na `.comp-icon` e `.comp-xp` ao entrar na tela de conclusão.

### Prioridade 3 — Melhorias

**R13 — Estado `:active` em todos os botões (UX11)**  
```css
.nav-btn:active, .opt-btn:active { transform: scale(0.98); opacity: .9; }
```

**R14 — Botão de envio AI para 44×44px (UX14)**  
`width: 44px; height: 44px;`

**R15 — Aumentar labels do radar (UX06)**  
`font-size: 9px` → `font-size: 11px` com ajuste do viewBox.

**R16 — Criar tela de configurações acessível da home (UX22)**  
Ícone de engrenagem ⚙ no header da home abrindo uma tela com: nome, perfil, tutor, preferências.

---

## 22. CONCLUSÃO

O aplicativo "Aos Pés do Mestre Jesus" tem uma base visual e tipográfica de qualidade superior à média de apps religiosos. A paleta, a tipografia e a hierarquia cromática por tipo de conteúdo formam um sistema de design coerente e espiritualmente apropriado.

A experiência **transmite calma e foco espiritual** na grande maioria das telas — especialmente durante os estudos, onde o body-text generoso, os espaços brancos e as caixas semânticas (apply/insight/decision) criam um ambiente contemplativo genuíno.

Os problemas mais críticos são de natureza técnica e de acessibilidade mobile:
1. Ausência de Service Worker (PWA sem offline real)
2. Botão WhatsApp no lado errado para usuários destros
3. Input do AI que pode desaparecer atrás do teclado iOS
4. Ausência de animações de transição que confeririam "suavidade" espiritual à navegação
5. `alert()`/`confirm()` nativos que quebram o ambiente visual do app

Com as correções de prioridade 1 implementadas, o app estaria tecnicamente sólido. Com as de prioridade 2, a experiência se tornaria verdadeiramente diferenciada — digna de ser usada por milhares de pessoas durante muitos anos.

---

*Relatório gerado por análise estática de todos os CSS (linhas 18–399), HTML (linhas 401–824) e JavaScript de navegação (linhas 1913–4069).*  
*Nenhuma modificação foi feita ao código do aplicativo.*  
*Data: 26 de junho de 2026*
