# PERFORMANCE_REVIEW.md — Revisão de Performance
**Aplicativo:** Aos Pés do Mestre Jesus  
**Versão analisada:** index.html (bed2252) | 4069 linhas | 347 KB (não minificado)  
**Analistas:** Arquiteto de Software Sênior · Especialista em PWA · QA Engineer · Revisor de Código  
**Data:** 26/06/2026  
**Status:** Somente análise — nenhuma alteração implementada  

---

## Índice

1. [Visão Geral de Performance](#1-visão-geral-de-performance)
2. [Tempo de Carregamento](#2-tempo-de-carregamento)
3. [DOM — Tamanho e Estrutura](#3-dom--tamanho-e-estrutura)
4. [Memória](#4-memória)
5. [Reflows e Repaints](#5-reflows-e-repaints)
6. [Renderizações](#6-renderizações)
7. [Uso de CPU](#7-uso-de-cpu)
8. [Uso de RAM](#8-uso-de-ram)
9. [PWA — Progressive Web App](#9-pwa--progressive-web-app)
10. [Service Worker e Cache](#10-service-worker-e-cache)
11. [Lazy Loading](#11-lazy-loading)
12. [Compressão](#12-compressão)
13. [Minificação](#13-minificação)
14. [Fonts e Assets Externos](#14-fonts-e-assets-externos)
15. [Padrões JavaScript com Impacto em Performance](#15-padrões-javascript-com-impacto-em-performance)
16. [Tabela de Problemas Identificados](#16-tabela-de-problemas-identificados)
17. [Recomendações Prioritárias](#17-recomendações-prioritárias)

---

## 1. Visão Geral de Performance

O aplicativo é uma **SPA monolítica** — um único arquivo `index.html` de 347 KB não minificado que contém todo o CSS, HTML, dados e JavaScript. Essa arquitetura tem implicações diretas e medidas em cada dimensão de performance analisada.

**Diagnóstico rápido:**

| Dimensão | Status | Gravidade |
|----------|--------|-----------|
| Tempo de carregamento | ⚠️ HTML 347KB, sem minificação | Alta |
| DOM | ⚠️ 12 telas sempre no DOM | Média |
| Memória JS | ⚠️ STUDIES[] 1039 linhas carregadas de vez | Média |
| Reflows | 🔴 `save()` por keystroke + reflow por troca de tela | Alta |
| Repaints | ✅ Transições CSS simples, sem animações pesadas | Baixa |
| CPU | ⚠️ Parse de 347KB + JSON em cada keystroke | Média |
| RAM | ✅ Uso total estimado < 8 MB | Baixa |
| PWA | ✅ Manifest válido + Service Worker funcional | — |
| Service Worker | ✅ Existe, bem implementado | — |
| Cache | ✅ Cache-First para assets, Network-First para HTML | — |
| Lazy Loading | 🔴 Ausente — tudo carregado na inicialização | Alta |
| Compressão | ✅ GitHub Pages serve gzip automaticamente | — |
| Minificação | 🔴 Ausente — 347KB legível vs ~120KB minificado | Alta |

---

## 2. Tempo de Carregamento

### 2.1 Tamanho do payload

| Arquivo | Tamanho no disco | Estimativa comprimido (gzip) |
|---------|-----------------|------------------------------|
| `index.html` | 347 KB | ~90–110 KB |
| `service-worker.js` | ~2 KB | < 1 KB |
| `manifest.json` | < 1 KB | < 1 KB |
| `icon-192.png` | ~25–40 KB (PNG) | ~25–40 KB (já comprimido) |
| `icon-512.png` | ~60–100 KB (PNG) | ~60–100 KB |
| Google Fonts CSS | ~15–20 KB | ~5 KB |
| Google Fonts woff2 | ~50–80 KB (Lora + Source Sans 3) | ~50–80 KB (já comprimido) |

**Total estimado na primeira visita:** ~230–340 KB transferidos  
**Total estimado em visitas subsequentes (cache ativo):** ~0 KB para assets locais  

### 2.2 Caminho crítico de renderização (Critical Rendering Path)

```
1. Navegador solicita index.html
2. Recebe HTML (90-110 KB gzip)
3. Parser HTML encontra <link> para Google Fonts → BLOQUEIA renderização
   └── DNS lookup: fonts.googleapis.com
   └── TCP connect + TLS handshake
   └── Baixa CSS (~5 KB gzip)
   └── CSS do Google Fonts referencia fonts.gstatic.com
       └── Segundo DNS lookup + handshake
       └── Baixa woff2 (~50-80 KB)
4. Parser continua → encontra <style> (inline, não bloqueia)
5. Parser processa HTML (12 telas simultâneas no DOM)
6. Parser executa <script> inline (347 KB brutos de JS+dados)
   └── Inicializa STUDIES[] (1039 linhas de objetos)
   └── Inicializa TUTORS[], LEVELS[], BADGES[], OBSTACLES[]
   └── Chama load() → JSON.parse do localStorage
   └── Chama initApp() → showScreen() → renderHome()
7. Primeira pintura visual (FCP)
8. Service Worker registrado (window load event)
```

**Gargalo principal:** Google Fonts bloqueia a renderização. Sem `preconnect`, o navegador não inicia o DNS até encontrar o `<link>`. Em redes lentas (3G hospitalar, Wi-Fi congestionado), isso pode atrasar o FCP em **0,5–2 segundos**.

**Nota positiva:** `display=swap` está presente na URL do Google Fonts (`&display=swap`), o que evita FOIT (Flash of Invisible Text) — o texto é exibido com fonte fallback enquanto Lora/Source Sans 3 carregam.

### 2.3 Estimativas de tempo por cenário

| Cenário | FCP estimado | Interativo (TTI) |
|---------|-------------|-----------------|
| 4G + primeira visita (sem cache SW) | 1,5–3 s | 2–4 s |
| 4G + segunda visita (cache SW ativo) | 0,3–0,8 s | 0,5–1,2 s |
| 3G hospitalar + primeira visita | 4–8 s | 6–12 s |
| 3G hospitalar + cache SW ativo | 0,5–1,5 s | 1–2 s |
| Offline + cache SW ativo | ~0,3 s | ~0,5 s |

### 2.4 Ausência de Resource Hints

O HTML não contém nenhum dos seguintes hints de performance:

```html
<!-- Ausentes — reduziriam FCP em 200-500ms -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="dns-prefetch" href="https://solitary-meadow-26e0.prwladi.workers.dev">
```

---

## 3. DOM — Tamanho e Estrutura

### 3.1 Inventário das telas no DOM

O app usa SPA por toggle de classe CSS (`.screen.active`). Todas as 12 telas existem no DOM simultaneamente desde o carregamento:

| Tela | ID | Sempre no DOM |
|------|-----|--------------|
| Boas-vindas | `screen-welcome` | ✅ Sim |
| Home | `screen-home` | ✅ Sim |
| Estudo | `screen-study` | ✅ Sim |
| Completo | `screen-complete` | ✅ Sim |
| Reflexão | `screen-reflection` | ✅ Sim |
| Bônus | `screen-bonus` | ✅ Sim |
| Tutor | `screen-tutor` | ✅ Sim |
| Diário | `screen-journal` | ✅ Sim |
| IA | `screen-ai` | ✅ Sim |
| Compartilhar | `screen-share` | ✅ Sim |
| Painel Admin | `screen-panel` | ✅ Sim |
| Painel Tutor | `screen-tutorpanel` | ✅ Sim |

**Impacto:** o navegador constrói e calcula o layout de **todas as 12 telas** durante o carregamento inicial, mesmo que o usuário veja apenas 1. Isso aumenta o tempo de parse e o custo do layout inicial.

### 3.2 Estimativa de nós DOM

| Seção | Nós DOM estimados |
|-------|-----------------|
| Tela de boas-vindas (onboarding completo) | ~120 |
| Home (com cards de estudos renderizados) | ~80–200 (dinâmico) |
| Tela de estudo (conteúdo de fase) | ~60–100 (dinâmico) |
| 10 telas estáticas restantes | ~300 |
| **Total estimado** | **~600–800 nós** |

Para um app mobile, esse volume é **aceitável**. Não há risco de performance por tamanho de DOM neste contexto.

### 3.3 Injeção dinâmica via innerHTML

Múltiplas funções de renderização injetam HTML dinâmico via `innerHTML`:

```javascript
// renderHome() — regenera cards de estudo inteiros
// renderJournal() — regenera lista de entradas
// renderPhase() — regenera conteúdo da fase atual
```

Cada chamada causa um **reparse de HTML + recálculo de layout** na subárvore injetada. Não há virtual DOM ou diff — substituição completa a cada render.

---

## 4. Memória

### 4.1 Dados carregados em memória na inicialização

| Dado | Localização no código | Tamanho estimado em memória |
|------|----------------------|----------------------------|
| `STUDIES[]` (16 estudos, fases, textos) | Linhas 873–1912 | ~500–800 KB |
| `TUTORS[]` | ~100 linhas | ~10 KB |
| `LEVELS[]`, `BADGES[]`, `OBSTACLES[]` | ~100 linhas | ~15 KB |
| `FASE_LABELS`, `PHASE_TAGS`, etc. | ~50 linhas | ~5 KB |
| Objeto `ST` (estado do usuário) | localStorage desserializado | ~2–50 KB |
| Closures JS e protótipos | — | ~2–5 MB (overhead V8) |

**Total estimado em heap JS:** 3–8 MB — bem dentro da capacidade de dispositivos modernos.

### 4.2 Vazamentos de memória potenciais

**`window._homeCountdownTimer`** (linha 3477):

```javascript
if (window._homeCountdownTimer) clearInterval(window._homeCountdownTimer);
window._homeCountdownTimer = setInterval(() => { ... }, 30000);
```

O timer é cancelado corretamente ao re-entrar na home. Sem vazamento aqui.

**`aiHistory[]` (em memória, max 20 mensagens):**  
Capped em 20 mensagens com `slice(-20)`. Sem crescimento ilimitado.

**`reflectionTimer`** (linha 3664):  
Verificar se é cancelado ao sair da tela de reflexão — risco de timer ativo em tela inativa.

**innerHTML em `renderJournal()`:**  
Cada chamada destrói e recria nós DOM. Event listeners em elementos filhos (se houver) seriam vazados — mas a análise mostra que o app usa atributos `onclick` inline, não `addEventListener`. Sem vazamento aqui.

---

## 5. Reflows e Repaints

### 5.1 Problema crítico: `save()` disparado por cada keystroke

**Localização:** linha 483

```html
<input oninput="ST.userName=this.value.trim();save(ST)">
```

A função `save()` é chamada a cada caractere digitado no campo de nome. Cada chamada executa:

1. `JSON.stringify(ST)` — serialização do objeto completo
2. `localStorage.setItem('amj3', data)` — escrita síncrona em localStorage
3. `sessionStorage.setItem('amj3', data)` — segunda escrita síncrona
4. `document.cookie = 'amj3=' + encodeURIComponent(data) + ...` — terceira escrita
5. `syncToTutorBackend()` — inicia debounce de 1.500ms

`JSON.stringify` e `setItem` são operações síncrona no thread principal. Para um ST pequeno (< 10 KB) o impacto é negligível. Mas com diário extenso (50+ entradas, ~100 KB), cada tecla pode bloquear o thread por **2–10ms**, causando UI jank perceptível.

**Severidade:** Média. Improvável em uso normal inicial, mas cresce com o tempo de uso.

### 5.2 Troca de tela causa reflow completo

**Localização:** linha 3933

```javascript
function showScreen(n) {
  document.querySelectorAll('.screen').forEach(s => s.classList.remove('active'));
  document.getElementById('screen-' + n).classList.add('active');
  resetScroll();
}
```

`classList.remove/add` em elementos com `display: none → flex` força recálculo de layout. Com 12 telas no DOM, o navegador percorre todos os nós para aplicar CSS. Impacto: **<5ms** por transição em dispositivos modernos — aceitável.

### 5.3 `resetScroll()` com `requestAnimationFrame`

**Localização:** linha 3955

```javascript
function resetScroll() {
  requestAnimationFrame(() => {
    const el = document.querySelector('.screen.active');
    if (el) el.scrollTop = 0;
  });
}
```

Uso correto de `requestAnimationFrame` — não causa reflow extra, apenas lê e escreve `scrollTop` no próximo frame. ✅

### 5.4 `renderHome()` — reflow na navegação

`renderHome()` atualiza múltiplos elementos DOM com `textContent` e `style.width`:

```javascript
document.getElementById('h-level').textContent = lv.name;
document.getElementById('h-xpbar').style.width = pct + '%';
// ... 10+ operações DOM
```

Cada operação intercalada de leitura+escrita pode causar layout thrashing. No entanto, como todas são escritas simples (sem leitura de `offsetWidth` entre elas), o navegador pode batchar — impacto baixo.

---

## 6. Renderizações

### 6.1 Estratégia de renderização

O app usa **renderização imperativa** — funções JS atualizam o DOM diretamente. Não há framework reativo, virtual DOM ou sistema de diff.

| Função | Trigger | Custo DOM |
|--------|---------|-----------|
| `renderHome()` | Toda navegação à home + a cada 30s (timer) | Médio |
| `renderPhase()` | Cada avanço de fase | Médio (innerHTML) |
| `renderJournal()` | Abertura do diário | Alto (lista completa) |
| `renderBadges()` | Abertura de conquistas | Baixo |
| `renderFixedTutorBtn()` | Inicialização + mudança de tutor | Baixo |

### 6.2 Ausência de memoization ou cache de renderização

`renderHome()` é chamada:
- Ao navegar para a home
- Após cada estudo concluído
- Após `confirmRestart()`
- A cada ciclo do `_homeCountdownTimer` (30s, quando há lock)

Não há verificação de "algo mudou?". Cada chamada regenera toda a home. Para o volume atual, isso é aceitável — mas é um padrão que não escala.

### 6.3 Emojis como "ícones"

O app usa emojis para ícones (🙏, ✝️, 🎯, 📖). Emojis são renderizados pelo sistema operacional como texto — zero requisições HTTP, zero memória de imagem. Escolha excelente para performance. ✅

---

## 7. Uso de CPU

### 7.1 Inicialização — parse e execução JS

Na primeira execução (ou após limpeza de cache), o motor JS (V8/JavaScriptCore) deve:

1. **Parse:** tokenizar e parsear 347 KB de JS+HTML+dados
2. **Compilar:** compilar o código para bytecode
3. **Executar:** inicializar todos os arrays de dados e funções

**Estimativa de tempo de parse (mid-range Android, 2023):**

| Fase | Tempo estimado |
|------|---------------|
| Parse do HTML (inclui CSS inline) | 30–60 ms |
| Parse do JS (STUDIES[], lógica) | 80–150 ms |
| Execução de `initApp()` | 5–15 ms |
| **Total estimado** | **115–225 ms** |

Em dispositivos de entrada (Android < 2 GB RAM), pode atingir **400–800 ms** de bloqueio do thread principal.

### 7.2 V8 Code Caching

Em visitas subsequentes com Service Worker ativo:
- O `index.html` é servido do cache → parse é necessário novamente (HTML não tem bytecode cached)
- O V8 pode usar **code cache** para scripts externos (`.js`), mas **não para scripts inline** em HTML
- **Impacto:** parse JS completo a cada visita, mesmo com SW ativo

Se o JS estivesse em arquivo separado (`app.js`), o V8 cacheia o bytecode compilado após a segunda visita, reduzindo o parse em **50–80%**.

### 7.3 STUDIES[] — parse desnecessário

O array `STUDIES[]` (1039 linhas de objetos literais JS) é parseado e mantido em memória integralmente mesmo que o usuário esteja no Estudo 1 e nunca chegue ao Estudo 16. Não há carregamento progressivo ou lazy.

---

## 8. Uso de RAM

### 8.1 Estimativa de consumo

| Componente | RAM estimada |
|-----------|-------------|
| Tab overhead (Chrome mobile) | ~15–25 MB |
| Heap JS (dados + closures + engine) | ~5–10 MB |
| DOM (12 telas + elementos dinâmicos) | ~2–5 MB |
| Imagens (base64 logo inline) | ~70–90 KB (decodificado: ~300–500 KB) |
| Fontes (Lora + Source Sans 3) | ~2–4 MB |
| **Total estimado** | **~25–45 MB** |

**Avaliação:** totalmente dentro da capacidade de qualquer smartphone moderno (mínimo 2 GB RAM). Sem risco de OOM (Out of Memory).

### 8.2 Logo como base64 inline

**Localização:** linha 409

```html
<img src="data:image/png;base64,iVBORw0KGgoA..." style="width:52px;height:52px">
```

A string base64 da logo do Hospital Adventista Silvestre ocupa **~40 KB** no HTML. Em memória, a imagem decodificada (~52×52px PNG) ocupa apenas ~10 KB de pixels. Vantagem: zero requisição HTTP extra. Desvantagem: inflaciona o HTML em ~40 KB antes da compressão gzip.

---

## 9. PWA — Progressive Web App

### 9.1 Checklist PWA

| Requisito PWA | Status | Detalhe |
|---------------|--------|---------|
| HTTPS | ✅ GitHub Pages serve HTTPS | — |
| manifest.json | ✅ Presente e válido | — |
| Service Worker | ✅ Existe e está registrado | — |
| Ícones (192×192) | ✅ icon-192.png | — |
| Ícones (512×512) | ✅ icon-512.png | — |
| `start_url` | ✅ `./index.html` | — |
| `display: standalone` | ✅ Configurado | — |
| `theme_color` | ✅ `#042C53` | — |
| `background_color` | ✅ `#042C53` | — |
| `orientation: portrait` | ✅ Configurado | — |
| Funciona offline | ✅ Via SW Cache-First | — |
| Instalável (Add to Home Screen) | ✅ Critérios atendidos | — |
| `maskable` icons | ⚠️ Ambos com `"purpose": "any maskable"` | Deveria ser separado |

### 9.2 Problema: `"purpose": "any maskable"` combinado

**Localização:** `manifest.json` linhas 16 e 23

```json
{ "src": "icon-192.png", "purpose": "any maskable" }
```

O valor `"any maskable"` é tecnicamente aceito mas não recomendado. O correto é ter dois ícones separados:
- Um com `"purpose": "any"` — para uso geral (sem máscara)
- Um com `"purpose": "maskable"` — para uso em adaptive icons (Android)

Com `"any maskable"` combinado, alguns navegadores Android podem cortar partes do ícone ao aplicar a máscara, se a zona segura não estiver corretamente configurada na imagem.

### 9.3 Ausência de Screenshots no manifest

O manifest não inclui `"screenshots"` — impede que a "mini infobar" de instalação no Chrome mostre preview do app. Sem impacto funcional.

---

## 10. Service Worker e Cache

### 10.1 Implementação atual

**Arquivo:** `service-worker.js` (74 linhas)  
**Cache name:** `amj-v3`

```javascript
const ASSETS = ['./', './index.html', './manifest.json', './icon-192.png', './icon-512.png'];
```

**Estratégias implementadas:**

| Tipo de request | Estratégia | Comportamento |
|----------------|-----------|---------------|
| POST, non-GET | Network only | AI Proxy, Google Sheets sync passam direto |
| `api.anthropic.com` | Network only | IA sempre online |
| Navegação (HTML) | **Network First** | Busca versão mais recente; cai para cache offline |
| Todos os outros | **Cache First** | Assets servidos do cache, rede como fallback |

**Avaliação:** implementação correta e bem comentada. A estratégia Network-First para HTML garante que atualizações do app cheguem ao usuário. Cache-First para assets garante performance offline. ✅

### 10.2 Limitações do cache atual

**Google Fonts não é cacheado:**  
O Service Worker intercepta requests do mesmo domínio/origem. Requisições para `fonts.googleapis.com` e `fonts.gstatic.com` são domínios externos — o SW atual não as cacheia explicitamente.

Resultado: **em uso offline, as fontes Lora e Source Sans 3 não estarão disponíveis**. O navegador usa fontes fallback (serif/sans-serif genéricas), alterando completamente a identidade visual do app.

**Correção possível:** adicionar estratégia `staleWhileRevalidate` para fonts.gstatic.com no SW.

**`self.skipWaiting()` no install:**  
Garante que a nova versão do SW assume imediatamente. Combinado com `self.clients.claim()` no activate, isso significa que usuários com o app aberto recebem a nova versão do SW sem precisar recarregar. ✅

**Limpeza de caches antigos:**  
O activate handler remove todos os caches que não sejam `amj-v3`. Correto. ✅

### 10.3 Atualização do app

Quando `index.html` é atualizado no servidor:
1. Network-First busca a nova versão
2. Salva no cache substituindo a anterior
3. Usuário recebe conteúdo atualizado imediatamente

Não há necessidade de mudar o `CACHE_NAME` para forçar atualização do HTML — apenas para assets estáticos como ícones.

---

## 11. Lazy Loading

**Status:** 🔴 **Ausente em todas as dimensões**

### 11.1 Telas: sem lazy loading

Todas as 12 telas são renderizadas no DOM na inicialização. Telas como `screen-tutorpanel` (painel administrativo) são carregadas mesmo para usuários que nunca acessarão esse recurso.

**Alternativa:** gerar o HTML das telas menos frequentes apenas ao serem acessadas pela primeira vez (`innerHTML` sob demanda), limpando do DOM ao sair.

### 11.2 STUDIES[]: sem lazy loading

O array completo de 16 estudos (1039 linhas de objetos) é inicializado na startup. Cada objeto de estudo tem `phases[]` com textos completos, versículos, perguntas.

**Estimativa de dados por estudo:** ~3–8 KB de dados literais.  
**Total do array:** ~50–100 KB de objetos JS em memória.

Para o uso atual (usuário acessa estudos sequencialmente), seria possível carregar apenas os estudos 1–3 inicialmente e os demais sob demanda.

### 11.3 Imagens: sem lazy loading

Os ícones dos tutores são emojis (sem HTTP). A logo é base64 (sem HTTP). Não há imagens externas para lazy-load. **Sem impacto real neste caso.**

### 11.4 Mentor IA: sem pré-carga inteligente

O componente de IA não tem pré-aquecimento de conexão. A primeira chamada ao Cloudflare Worker pode sofrer latência de cold start (50–500ms). Um `dns-prefetch` ou `preconnect` reduziria esse overhead.

---

## 12. Compressão

### 12.1 GitHub Pages — gzip automático

O app é hospedado no GitHub Pages, que **serve automaticamente gzip** para HTML, CSS, JS e JSON quando o cliente suporta (`Accept-Encoding: gzip`). Todos os navegadores modernos suportam.

**Ganho estimado de compressão:**

| Arquivo | Tamanho original | Tamanho comprimido | Redução |
|---------|-----------------|-------------------|---------|
| `index.html` | 347 KB | ~90–110 KB | ~70% |
| `service-worker.js` | ~2 KB | ~1 KB | ~50% |
| `manifest.json` | <1 KB | <1 KB | — |

**Avaliação:** compressão de transferência está bem coberta pelo GitHub Pages. ✅

### 12.2 Brotli — não disponível no GitHub Pages

GitHub Pages suporta apenas gzip, não Brotli. Brotli oferece 15–25% melhor compressão que gzip para texto. Não é possível ativar no GitHub Pages padrão — requereria migrar para Cloudflare Pages ou Netlify.

**Impacto:** com Brotli, `index.html` seria ~75–85 KB em vez de 90–110 KB. Ganho real mas não crítico.

---

## 13. Minificação

**Status:** 🔴 **Completamente ausente**

O arquivo `index.html` contém:
- Comentários extensos (ex.: `// ══════════════════════════════════════`, linhas de seção)
- Nomes de variáveis longos e descritivos (`pendingJournalIdx`, `welcomeTutorSelected`)
- Whitespace generoso (indentação, linhas em branco)
- CSS não minificado com seletores verbosos

**Estimativa de ganho com minificação:**

| Técnica | Tamanho antes | Tamanho depois | Redução |
|---------|--------------|----------------|---------|
| HTML minify (remoção de espaços, comentários) | 347 KB | ~280 KB | ~20% |
| CSS minify | ~30 KB (linhas 18–399) | ~18 KB | ~40% |
| JS minify (remoção de comentários, espaços, var shortening) | ~200 KB | ~110 KB | ~45% |
| **Total após minificação** | **347 KB** | **~170–190 KB** | **~47%** |
| **Após gzip do arquivo minificado** | ~100 KB | **~55–65 KB** | **~83% vs original** |

**Impacto real:** reduz o tempo de parse em ~40–50% em dispositivos de entrada. Melhora o FCP em 0,5–1,5s em 3G.

**Limitação da arquitetura monolítica:** ferramentas de minificação como Terser (JS), cssnano (CSS) e html-minifier operam em arquivos separados. Um HTML monolítico requer ferramentas capazes de tratar os três em conjunto, o que é mais complexo.

---

## 14. Fonts e Assets Externos

### 14.1 Google Fonts — análise detalhada

**Declaração atual** (linha 18):
```html
<link href="https://fonts.googleapis.com/css2?family=Lora:ital,wght@0,400;0,500;0,600;1,400;1,500&family=Source+Sans+3:wght@300;400;500;600&display=swap" rel="stylesheet">
```

**Positivo:**
- `display=swap` ✅ — texto visível durante carregamento das fontes
- Subsetting implícito via Google Fonts API ✅ — apenas os pesos/estilos necessários

**Negativo:**
- Sem `<link rel="preconnect">` para `fonts.googleapis.com` e `fonts.gstatic.com`
- Fontes não incluídas no cache do Service Worker — indisponíveis offline
- 2 round-trips de rede (googleapis → gstatic) antes de iniciar download

**Fontes carregadas:**
- Lora: regular, medium, semibold (3 pesos) + italic em 3 pesos = 6 arquivos woff2
- Source Sans 3: light, regular, medium, semibold = 4 arquivos woff2
- Total: **10 arquivos woff2** (~5–8 KB cada) = ~50–80 KB

### 14.2 Ícones externos via URL absoluta

**Localização:** linhas 12–16

```html
<link rel="apple-touch-icon" href="https://capelaniahospitalar.github.io/jornada-discipular/icon-192.png">
```

Os ícones `icon-192.png` e `icon-512.png` também existem como arquivos locais no repositório (confirmado via Glob). As referências `<head>` usam URL absoluta externa, enquanto o SW cacheia os arquivos locais com path relativo (`./icon-192.png`).

**Problema:** se o domínio do GitHub Pages mudar, os ícones do `<head>` quebram. Deveria usar path relativo (`./icon-192.png`) como o manifest.json já faz corretamente.

---

## 15. Padrões JavaScript com Impacto em Performance

### 15.1 `querySelectorAll` em cada troca de tela

**Localização:** linha 3934

```javascript
document.querySelectorAll('.screen').forEach(s => s.classList.remove('active'));
```

Chamado a cada `showScreen()`. Com 12 elementos `.screen`, isso percorre o DOM completo. Impacto: negligível (< 1ms). Poderia ser otimizado mantendo referência em array, mas não é prioritário.

### 15.2 `setInterval` no timer da home

**Localização:** linha 3480

```javascript
window._homeCountdownTimer = setInterval(() => { ... }, 30000);
```

Intervalo de 30 segundos para atualizar countdown de reflexão. Impacto em CPU: mínimo. Limpeza correta com `clearInterval`. ✅

### 15.3 Ausência de `debounce` no campo de nome

**Localização:** linha 483

```html
oninput="ST.userName=this.value.trim();save(ST)"
```

`save()` chama `JSON.stringify + setItem` síncronos a cada keystroke. Não há debounce. Para nomes longos (> 20 chars) ou dispositivos lentos com ST grande, isso pode causar stuttering.

**Solução simples:** debounce de 300ms antes de chamar `save()` no campo de texto.

### 15.4 Uso correto de `requestAnimationFrame`

**Localização:** linhas 3534 e 3955

```javascript
requestAnimationFrame(() => { el.scrollTop = 0; });
```

Uso correto — leitura/escrita de propriedades de layout dentro do RAF. ✅

### 15.5 `async/await` na função de IA

**Localização:** linha 1946

```javascript
async function sendAI() { ... }
```

Uso de `async/await` evita bloqueio do thread durante a requisição ao AI Proxy. ✅

### 15.6 Inline event handlers (`onclick=` no HTML)

O app usa extensivamente `onclick="funcao()"` nos elementos HTML. Isso:
- Cria closures implícitos para cada elemento
- Aumenta levemente o parse time do HTML
- Não causa vazamentos de memória neste padrão
- É uma escolha válida para app monolítico sem framework

Sem impacto mensurável de performance. Nota arquitetural apenas.

---

## 16. Tabela de Problemas Identificados

| # | Problema | Severidade | Categoria | Impacto Mensurável |
|---|----------|-----------|-----------|-------------------|
| P01 | HTML 347KB sem minificação | 🔴 Alta | Minificação | +40-50% parse time vs minificado |
| P02 | Google Fonts bloqueante sem preconnect | 🔴 Alta | Loading | +200-500ms no FCP |
| P03 | Fontes não cacheadas pelo SW offline | 🟠 Alta | PWA/Cache | Layout visual quebrado offline |
| P04 | `save()` disparado por keystroke sem debounce | 🟠 Alta | CPU/Reflow | Jank em ST > 50KB |
| P05 | STUDIES[] carregado integralmente na startup | 🟠 Alta | Memória/CPU | +80-150ms de parse inicial |
| P06 | 12 telas no DOM simultaneamente | 🟠 Média | DOM/Layout | Layout completo calculado desnecessariamente |
| P07 | Ausência de `rel="preconnect"` para fontes | 🟡 Média | Loading | +100-300ms por lookup de DNS |
| P08 | Ícones `<head>` com URL absoluta externa | 🟡 Média | Confiabilidade | Quebra se domínio mudar |
| P09 | `"purpose": "any maskable"` combinado no manifest | 🟡 Média | PWA | Ícone cortado em alguns Android |
| P10 | Brotli indisponível (GitHub Pages) | 🟢 Baixa | Compressão | +15-25KB vs Brotli |
| P11 | V8 não cacheia bytecode de scripts inline | 🟢 Baixa | CPU | +50-150ms reparseo em visitas subsequentes |
| P12 | `renderHome()` sem memoization | 🟢 Baixa | Renderização | Re-render completo desnecessário |
| P13 | Ausência de screenshots no manifest | 🟢 Info | PWA | Sem preview na mini infobar |

---

## 17. Recomendações Prioritárias

> **Atenção:** este documento é exclusivamente analítico. Nenhuma alteração deve ser implementada sem revisão e aprovação explícita.

### P0 — Impacto imediato, baixo risco de implementação

**P0.1 — Adicionar `preconnect` para Google Fonts**  
Adicionar 2 linhas no `<head>`, antes do link das fontes:
```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
```
Ganho esperado: **-200 a 500ms no FCP**.

**P0.2 — Debounce no campo de nome**  
Adicionar debounce de 300ms no `oninput` do campo de nome para evitar `save()` por keystroke.

**P0.3 — Corrigir paths de ícones no `<head>`**  
Substituir URLs absolutas por paths relativos: `href="./icon-192.png"`.

### P1 — Alta prioridade

**P1.1 — Cachear Google Fonts no Service Worker**  
Adicionar estratégia `staleWhileRevalidate` para `fonts.gstatic.com` no SW:
```javascript
if (url.hostname === 'fonts.gstatic.com') {
  event.respondWith(staleWhileRevalidate(event.request));
  return;
}
```
Ganho: **fontes disponíveis offline**, identidade visual preservada sem rede.

**P1.2 — Separar ícones `"purpose"` no manifest.json**  
```json
{ "src": "icon-192.png", "sizes": "192x192", "purpose": "any" },
{ "src": "icon-192-maskable.png", "sizes": "192x192", "purpose": "maskable" }
```

### P2 — Média prioridade

**P2.1 — Minificação do arquivo index.html**  
Usar ferramenta como `html-minifier-terser` em build step (GitHub Actions). Ganho: **~47% de redução no tamanho do arquivo**, **~40% menos parse time**.

**P2.2 — Externalizar JS em arquivo separado**  
Mover o bloco `<script>` principal para `app.js`. Isso permite que o V8 cacheia o bytecode compilado após a segunda visita — **redução de 50-80% no tempo de parse em visitas subsequentes**.

**P2.3 — Lazy loading de telas menos frequentes**  
Gerar HTML do painel do tutor e painel administrativo apenas quando acessados pela primeira vez.

### P3 — Baixa prioridade / longo prazo

**P3.1 — Lazy loading do STUDIES[]**  
Carregar metadados dos estudos na inicialização e conteúdo completo apenas ao abrir o estudo.

**P3.2 — Migrar para Cloudflare Pages ou Netlify**  
Para suporte a Brotli, headers customizados (`Cache-Control`, `Content-Security-Policy`) e Edge Functions.

**P3.3 — `dns-prefetch` para o AI Proxy**  
```html
<link rel="dns-prefetch" href="https://solitary-meadow-26e0.prwladi.workers.dev">
```

---

## Sumário Executivo

O aplicativo tem uma **base de performance sólida para um app monolítico**: o Service Worker está corretamente implementado com estratégias híbridas, o GitHub Pages serve gzip automaticamente, `display=swap` está configurado nas fontes, e o uso de emojis como ícones elimina requisições HTTP desnecessárias.

O **maior problema imediato é a ausência de minificação**: 347 KB de código legível que poderia ser servido em ~55–65 KB (gzip + minificação). Em redes 3G hospitalares, isso representa a diferença entre 4–8 segundos e 1–2 segundos de carregamento.

O **segundo maior problema é a ausência de `preconnect`** para Google Fonts, que bloqueia o Critical Rendering Path e atrasa o FCP em 200–500ms — um custo evitável com 2 linhas de HTML.

O **terceiro problema** é que as fontes não são cacheadas pelo Service Worker, fazendo com que a identidade visual do app quebre completamente quando usado offline — justamente quando o cache torna o app funcionalmente disponível.

As três correções de P0 e P1 podem ser implementadas em poucas linhas de código, sem risco arquitetural, e produziriam ganho mensurável e imediato para os usuários em ambiente hospitalar.

---

*Relatório gerado como análise estática do código-fonte. Nenhuma alteração foi implementada.*  
*Próximo passo sugerido: SECURITY_REVIEW.md (consolidação de todas as vulnerabilidades identificadas nos 6 reviews) ou implementação das correções P0.*
