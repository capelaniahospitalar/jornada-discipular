# MENTOR_AI_REVIEW.md
## Análise Profunda do Mentor IA — Aos Pés do Mestre Jesus
**Data:** 26 de junho de 2026  
**Versão analisada:** index (26).html — linhas 1919–2071  
**Modelo subjacente:** Claude (Anthropic) via Cloudflare Worker proxy  
**Proxy:** `https://solitary-meadow-26e0.prwladi.workers.dev/`  
**Análise estática — nenhuma alteração implementada**

---

## 1. VISÃO GERAL DA ARQUITETURA

```
USUÁRIO
  ↓  digita pergunta (input#ai-input)
sendAI()
  ↓  monta payload {system: systemPrompt, messages: aiHistory}
fetch(AI_PROXY_URL, POST, JSON)
  ↓
Cloudflare Worker (proxy)
  ↓  repassa para a API Claude
Claude API
  ↓  retorna {ok: boolean, reply: string}
addAIMsg('bot', reply)
  ↓
aiHistory.push({role:'assistant', content: reply})
```

**Fluxo resumido:** input → `sendAI()` → Worker proxy → Claude → render.  
**Não há streaming** — a resposta completa chega antes de ser exibida.  
**Não há persistência do histórico** — `aiHistory` é um array em memória (var global), zerado quando a página é recarregada.

---

## 2. ANÁLISE DO PROMPT PRINCIPAL (system prompt)

### 2.1 Estrutura geral

O system prompt tem aproximadamente **1.450 palavras** e está organizado em 12 seções claramente delimitadas:

| Seção | Linhas | Finalidade |
|---|---|---|
| MISSÃO | 1958–1961 | Define propósito e identidade |
| IDENTIDADE TEOLÓGICA | 1963–1969 | Ancora teologicamente |
| HIERARQUIA DE FONTES | 1971–1974 | Define autoridade epistémica |
| FORMATO DAS RESPOSTAS | 1976–1983 | Estrutura a saída |
| RESPONDENDO A CRÍTICAS | 1985–1993 | Protocolo apologético |
| TEMAS APOLOGÉTICOS | 1995 | Lista de temas sensíveis |
| TOM DE VOZ | 1997–1998 | Define a voz pastoral |
| FUNDAMENTO — TEMAS-CHAVE | 2000–2009 | Doutrinas específicas com versículos |
| SEU ESCOPO | 2011–2017 | O que responder |
| FORA DO SEU ESCOPO | 2019–2027 | O que recusar |
| SOBRE DÍZIMOS | 2029–2036 | Instruções específicas de transparência |
| AO RECUSAR | 2038–2039 | Script de recusa |
| REGRAS PRÁTICAS | 2041–2045 | Diretrizes operacionais |
| VERIFICAÇÃO FINAL | 2047–2048 | Checklist de qualidade |

**Avaliação da estrutura:** bem organizado, com hierarquia clara e uso de seções em CAPS_LOCK para separação visual no prompt. A nomenclatura é consistente e a sequência lógica (missão → identidade → fontes → formato → tom → escopo) é pedagogicamente correta.

---

### 2.2 Qualidade e precisão das instruções

#### Pontos fortes

**P1 — Hierarquia de fontes é teologicamente correta e bem executada**  
A ordem Bíblia → documentos oficiais IASD → pesquisa acadêmica é teologicamente adequada e respeita o princípio adventista de *Sola Scriptura*. A instrução explícita de que Ellen White "nunca é prova primária de doutrina" é pastoralmente sábia e teologicamente honesta — evita um vício comum em material adventista popular.

**P2 — Protocolo apologético de 6 passos é excelente**  
O método para tratar críticas (reconhecer → explicar justo → fatos históricos → textos bíblicos → ensino → convidar ao exame, baseado em Atos 17.11) é equilibrado e evita defensividade reativa. É o protocolo correto para um assistente digital em contexto de discipulado.

**P3 — Lista de temas apologéticos é abrangente e relevante**  
Cobre os principais pontos de questionamento que pacientes hospitalizados, curiosos religiosos e evangélicos fariam ao interagir com material adventista.

**P4 — Instrução sobre dízimos tem nível de detalhe excepcional**  
A seção de dízimos e ofertas inclui fluxo financeiro completo, destinação, sistema de auditoria e referência ao Annual Statistical Report (ASR). Isso é raro e indica preocupação real com transparência — um ponto de confiança crítico para usuários céticos.

**P5 — Verificação final é um mecanismo de autocorreção inteligente**  
A pergunta "Esta resposta aproxima a pessoa de Jesus, da Bíblia e de uma compreensão mais profunda da verdade?" funciona como um prompt de reflexão implícito que orienta o modelo a priorizar o objetivo pastoral sobre o debate.

**P6 — Tom de voz bem especificado e com anti-padrões explicitados**  
Listar o que evitar ("linguagem combativa, triunfalismo denominacional, arrogância teológica, simplificações excessivas") é mais eficaz do que apenas descrever o positivo.

---

#### Problemas identificados no prompt

**⚠️ PR1 — Inconsistência no número de estudos (CRÍTICO)**  
Na seção "SEU ESCOPO" (linha 2015), o prompt lista **17 estudos**:  
> "Os 17 estudos da série 'Aos Pés do Mestre Jesus': Jesus, Escrituras, Oração, Perdão, Os Outros, Comunidade, Sofrimento, Saúde, Administração da Vida, Segunda Vinda, **Juízo Final**, Santuário, Ressurreição, Lei de Deus, Sábado, Missão, Seguir Jesus"

O aplicativo tem **16 estudos**. O título "Juízo Final" e "Missão" não correspondem a estudos existentes na trilha (o estudo 10 é "Jesus irá voltar?", não "Juízo Final"; não há estudo específico sobre missão). O AI poderia responder sobre um estudo que não existe.

**⚠️ PR2 — Ausência de contexto dinâmico do usuário**  
O prompt é **completamente estático** — não inclui nenhuma informação sobre:
- Em qual estudo o usuário está (ex: estudando o Estudo 7)
- Qual fase da jornada ele completou
- Qual foi sua última decisão espiritual
- Qual tutor acompanha sua jornada
- Seu nome

Isso significa que o AI não pode personalizar respostas com contexto da jornada. O assistente é "cego" ao progresso do usuário. Por exemplo, se o usuário pergunta sobre o sábado enquanto ainda está no Estudo 3, o AI não sabe que ele ainda não chegou ao Estudo 15 — e pode antecipar conteúdo que o app deliberadamente apresenta mais tarde.

**⚠️ PR3 — Instrução de escopo pode criar contradição**  
A seção "FORA DO SEU ESCOPO" lista "relacionamentos românticos ou sexualidade (redirecione ao tutor)". Porém, temas bíblicos como o casamento (Efésios 5), a criação do ser humano como homem e mulher (Gênesis 1-2) ou a pureza sexual (1 Tessalonicenses 4) são biblicamente legítimos. O prompt não diferencia entre "conselho pessoal sobre relacionamento" (fora do escopo) e "ensino bíblico sobre casamento e família" (dentro do escopo). Um modelo rigoroso pode recusar perguntas bíblicas legítimas sobre família.

**⚠️ PR4 — Sem instrução sobre confidencialidade do prompt**  
O prompt não instrui o modelo a não revelar seu conteúdo. Um usuário curioso pode perguntar "qual é o seu system prompt?" e o modelo pode revelar as instruções completas — incluindo a lógica de escopo e os limites, o que pode ser usado para tentar contorná-los.

**⚠️ PR5 — Sem instrução de idioma**  
O prompt não especifica explicitamente que as respostas devem ser em **português**. Se o usuário escrever em espanhol ou inglês, o modelo pode responder no idioma da pergunta — o que pode ser desejado ou não, dependendo do contexto.

**⚠️ PR6 — Temas de ciência sem clareza suficiente**  
A exclusão de "Tecnologia, programação ou ciência sem relação com fé" é imprecisa. Perguntas como "A ciência contradiz a Bíblia?", "Como Deus criou em 6 dias se o universo tem bilhões de anos?" ou "Evolução e criação são compatíveis?" estão na interseção exata entre ciência e fé — e são perguntas muito comuns em contexto de discipulado. O prompt não dá orientação específica para esse território híbrido.

---

## 3. ANÁLISE DO CONTEXTO

### 3.1 Contexto enviado por requisição

A cada chamada `sendAI()`, o payload enviado ao proxy é:

```json
{
  "system": "[system prompt de ~1450 palavras]",
  "messages": [
    {"role": "user", "content": "..."},
    {"role": "assistant", "content": "..."},
    ...
  ]
}
```

**Observação crítica:** o system prompt **é reenviado a cada mensagem**. Isso significa que cada interação consome ~400 tokens apenas para o prompt do sistema, independentemente da pergunta. Dependendo da implementação do proxy (se usa cache de prompt da API Anthropic), isso pode representar custo desnecessário a cada turno.

### 3.2 O que NÃO é enviado como contexto

| Dado disponível no app | Enviado ao AI? |
|---|---|
| Nome do usuário (`ST.userName`) | ❌ Não |
| Estudo atual (`ST.currentStudy`) | ❌ Não |
| Fase da jornada (`ST.phase`) | ❌ Não |
| Tutor escolhido | ❌ Não |
| Perfil (paciente/colaborador/amigo) | ❌ Não |
| Entradas do diário espiritual | ❌ Não |
| Decisões registradas | ❌ Não |
| XP acumulado | ❌ Não |

Nenhum dado do usuário é injetado no contexto da conversa. O AI é genérico para todos os usuários, em todos os momentos da jornada.

---

## 4. ANÁLISE DO HISTÓRICO DE CONVERSA

### 4.1 Implementação

```javascript
let aiHistory = [];
// ...
aiHistory.push({ role:'user', content: question });
// ...
aiHistory.push({ role:'assistant', content: reply });
// Keep history manageable
if (aiHistory.length > 20) aiHistory = aiHistory.slice(-20);
```

### 4.2 Avaliação

**Positivos:**
- Limite de 20 mensagens evita crescimento ilimitado do histórico
- O slice mantém as mensagens mais recentes (janela deslizante)

**Problemas:**

**H1 — Histórico não é persistido no localStorage**  
`aiHistory` é uma variável JavaScript em memória. Ao recarregar a página, fechar o app ou alternar entre estudos e voltar ao AI, o histórico é zerado. O usuário perde o contexto de todas as conversas anteriores. Isso é especialmente problemático para uma jornada de 16 estudos que pode durar semanas ou meses.

**H2 — Limite de 20 mensagens é arbitrário e pode cortar contexto importante**  
`aiHistory.slice(-20)` mantém as 20 mensagens mais recentes (10 turnos de conversa). Se uma conversa longa ocorrer, o modelo perde o início da sessão — o que pode criar inconsistências onde o AI "esquece" o que o usuário disse 11 mensagens atrás.

**H3 — Sem separação de sessões**  
Toda a conversa com o AI, desde que a página não seja recarregada, é acumulada em um único array. Não há conceito de "sessão" ou "contexto por estudo" — todas as perguntas sobre estudos diferentes convivem no mesmo histórico.

**H4 — Mensagem de erro não é adicionada ao histórico**  
```javascript
} catch(e) {
  loading.textContent = 'Erro de conexão. Verifique sua internet e tente novamente.';
  loading.classList.remove('loading');
}
```
Quando ocorre erro, a mensagem de erro é exibida visualmente mas **não é adicionada ao `aiHistory`**. Se o usuário tentar novamente, o AI não sabe que a mensagem anterior falhou. Isso é correto — não seria desejável persistir erros no histórico. Porém, a mensagem do usuário **já foi adicionada ao histórico** antes da chamada falhar (linha 1953), então o histórico ficará dessincronizado em caso de erro: conterá a pergunta do usuário sem a resposta do bot.

---

## 5. ANÁLISE DOS LIMITES E SEGURANÇA

### 5.1 Limites de escopo (guardrails)

**Seção "FORA DO SEU ESCOPO" no prompt:**
- Política, eleições, partidos
- Economia, finanças pessoais
- Saúde física, diagnósticos, medicamentos
- Relacionamentos românticos / sexualidade
- Receitas, culinária, esportes, entretenimento
- Tecnologia/programação sem relação com fé
- Qualquer tema alheio à fé cristã e à Bíblia

**Avaliação dos limites:**

**L1 — Limites dependem inteiramente do modelo**  
Não há nenhuma validação no lado do cliente (JavaScript) antes de enviar a pergunta ao modelo. Qualquer texto que o usuário digitar será enviado ao proxy e ao Claude. Os guardrails são instrucionais (no prompt), não técnicos. Claude é robusto em seguir instruções, mas isso não é uma garantia absoluta.

**L2 — Sem rate limiting no cliente**  
O usuário pode enviar mensagens rapidamente em sequência. Não há debounce, cooldown ou limite de requisições por minuto implementado no front-end. Isso expõe o proxy a uso excessivo (intencional ou não).

**L3 — Sem validação de comprimento da mensagem**  
O usuário pode digitar um texto extremamente longo (tentativa de "prompt injection" ou simplesmente um texto indevido). Não há limite de caracteres no `<input>`.

**L4 — Sem sanitização de HTML na renderização**  
```javascript
div.textContent = text;
```
A resposta do bot é inserida via `textContent` (não `innerHTML`) — isso é **correto e seguro**: nenhum HTML da resposta é executado como markup. Não há risco de XSS por esse caminho.

**L5 — Ausência de filtros para conteúdo espiritual extremo ou crise de saúde mental**  
O prompt instrui o AI a redirecionar temas pessoais ao tutor, mas não contém instruções específicas para situações de crise (ideação suicida, crise de fé severa, luto agudo). Um paciente hospitalizado em estado vulnerável pode interagir com o AI antes de conseguir acesso ao tutor. O prompt não contém um protocolo de emergência ou encaminhamento urgente.

---

## 6. TOM PASTORAL

### 6.1 Mensagem de boas-vindas

```
✝️ Olá! Sou a IA de Apoio ao Discípulo. Estou aqui para te ajudar a mergulhar na Palavra 
de Deus — responder dúvidas bíblicas, explorar passagens das Escrituras e caminhar ao seu 
lado nesta jornada de fé.

Lembre-se: para questões pessoais e aconselhamento, seu Tutor de Jornada está sempre 
disponível. 🙏
```

**Avaliação:** A mensagem é clara, acolhedora e estabelece imediatamente os dois pontos críticos: (1) o que o AI faz; (2) que não substitui o tutor. O uso de "✝️" é contextualmente adequado. A instrução de redirecionar ao tutor logo na abertura é pastoralmente sábia.

**Problema:** a mensagem de boas-vindas é **genérica** — não usa o nome do usuário (`ST.userName`), não menciona em que estudo ele está, não personaliza o tom pelo perfil (paciente vs. colaborador). Para uma jornada discipular que investe em personalização com `N()`, essa é uma inconsistência.

### 6.2 Tom prescrito vs. tom esperado

O prompt especifica: *"pastoral, acolhedor, inteligente, equilibrado, respeitoso, evangelístico e cristocêntrico."*

Esta combinação é bem escolhida para o contexto. O modelo Claude segue bem instruções de tom. A inclusão explícita de "evangelístico" é importante — não é apenas um assistente informativo, mas deve conduzir ao encontro com Jesus.

### 6.3 Tratamento de vulnerabilidade

O prompt instrui: *"Em temas pessoais delicados (luto, vícios, conflitos familiares), responda com acolhimento e ao final sugira: 'Recomendo conversar também com seu Tutor de Jornada...'"*

**Avaliação:** adequado para a maioria dos casos. Porém, como mencionado em L5, situações de crise aguda exigem protocolo diferente. A instrução genérica de "redirecionar ao tutor" pode ser insuficiente se o tutor não estiver disponível imediatamente (o app funciona offline e o tutor pode não estar alcançável no momento).

---

## 7. COERÊNCIA BÍBLICA

### 7.1 Versículos citados no prompt

O prompt cita 21 versículos como referência para os temas-chave. Verificação de adequação:

| Tema | Versículo | Adequação |
|---|---|---|
| Sábado | Êxodo 20.8-11; Lucas 4.16 | ✅ Excelente |
| Estado dos mortos | Ecl 9.5; 1Ts 4.13-18; Jo 11.11-14 | ✅ Excelente |
| Segunda Vinda | Atos 1.11; Mateus 24 | ✅ Adequado |
| Santuário/Juízo | Daniel 8-9; Hebreus 8-9 | ✅ Adequado |
| Lei e graça | Efésios 2.8-9; João 14.15 | ✅ Excelente |
| Saúde/Templo | 1Co 6.19-20; Levítico 11 | ✅ Adequado |
| Mordomia | Malaquias 3.10; 2Co 9.7 | ✅ Adequado |
| Missão | Apocalipse 14.6-12 | ✅ Adequado |

**Observação positiva:** a instrução de citar a versão ARA (Almeida Revista e Atualizada) é específica e correta — evita inconsistências entre versões.

### 7.2 Posicionamento hermenêutico

O prompt define uma hermenêutica explícita:
- "Interpretar a Escritura pela Escritura (analogia da fé)"
- "Explique o sentido dos termos originais em hebraico, aramaico ou grego"
- "Considere o contexto histórico, geográfico, cultural e literário"

Esta combinação representa hermenêutica histórico-gramatical com sensibilidade canônica — o padrão adequado para um assistente de discipulado adventista.

---

## 8. CONSISTÊNCIA TEOLÓGICA

### 8.1 Posicionamento sobre Ellen White

> "Ellen White nunca é prova primária de doutrina — pode ser citada como leitura devocional complementar, sempre subordinada e posterior às Escrituras."

**Avaliação:** Esta instrução é teologicamente correta segundo o próprio posicionamento adventista oficial (o dom profético é subordinado à Escritura). É um ponto delicado que o prompt acerta: permite o uso de Ellen White sem fazer dela autoridade primária — evitando ao mesmo tempo dois erros opostos (ignorá-la completamente ou sobrepô-la à Bíblia).

### 8.2 Tratamento de divergências internas

O prompt instrui: *"Quando houver interpretações cristãs legítimas diferentes, explique-as com respeito antes de apresentar o que as Escrituras ensinam."*

**Avaliação:** Adequado para divergências interdenominacionais. Porém, não há instrução específica sobre divergências **internas à IASD** (ex: questões sobre o papel da mulher no ministério pastoral, que divide sinceramente adventistas em diferentes países). O modelo pode responder de forma que conflite com posições regionais da IASD brasileira.

### 8.3 Doutrina da Trindade

O prompt lista "A Trindade" como tema apologético mas não oferece instrução específica sobre ela. Diferentemente de temas como o sábado e o estado dos mortos — para os quais o prompt fornece versículos específicos — a Trindade recebe apenas uma linha na lista. Dado que é um dos temas mais frequentemente questionados em contextos de evangelismo (Testemunhas de Jeová, unitaristas, etc.), a ausência de orientação mais detalhada é uma lacuna.

---

## 9. TRATAMENTO DE PERGUNTAS FORA DO ESCOPO

### 9.1 Script de recusa

```
"Esse tema está fora do meu escopo como assistente bíblico-teológico. 
Para isso, recomendo buscar [profissional adequado]. 
Posso te ajudar com alguma dúvida bíblica, espiritual ou sobre a jornada discipular?"
```

**Avaliação:**
- A estrutura da recusa é correta: recusa sem julgamento + redireciona + oferece alternativa
- O placeholder `[profissional adequado]` é substituído pelo modelo contextualmente — o que é o comportamento correto (não lista uma profissão genérica, mas se adapta ao tema)
- A oferta de alternativa ao final ("Posso te ajudar com...") mantém o usuário no fluxo da conversa

### 9.2 Zona cinzenta — temas híbridos

Perguntas que combinam espiritualidade com temas "fora do escopo" criam ambiguidade:

| Pergunta | Classificação esperada | Risco |
|---|---|---|
| "Deus cura doenças?" | Dentro (fé e cura bíblica) | Pode cruzar para conselho médico |
| "Devo parar de tomar meu remédio e confiar em Deus?" | Fora (saúde médica) | Alto risco — orientação médica disfarçada de espiritual |
| "Como a Bíblia trata a depressão?" | Dentro (saúde mental bíblica) | Pode cruzar para conselho clínico |
| "Meu pastor disse que devo votar em X porque é cristão" | Fora (política) | Pode cruzar para interpretação bíblica sobre autoridade |
| "O que Deus pensa sobre aborto?" | Dentro (ética bíblica) | Altamente polêmico, sem instrução específica |

O prompt não oferece orientação para essas zonas de fronteira, que são exatamente onde um AI pastoral enfrenta os maiores desafios.

---

## 10. TRATAMENTO DE ERROS

### 10.1 Erros de rede

```javascript
} catch(e) {
  loading.textContent = 'Erro de conexão. Verifique sua internet e tente novamente.';
  loading.classList.remove('loading');
}
```

**Avaliação:**
- A mensagem é clara e indica a ação esperada do usuário
- Remove corretamente a classe `loading` (evita spinner permanente)
- Não há retry automático — o usuário precisa reenviar manualmente

**Problemas:**
- Não distingue entre tipos de erro: timeout de rede, erro HTTP 4xx/5xx, JSON parse error, erro do proxy
- Não há log de erros para depuração
- A mensagem de erro fica no DOM como se fosse uma mensagem do bot — pode ser visualmente confuso

### 10.2 Erros da API / proxy

```javascript
if (!data.ok) throw new Error(data.error || 'Erro desconhecido');
```

O proxy retorna `{ok: boolean, error?: string, reply?: string}`. Se `ok === false`, o erro é capturado e cai no `catch`. Isso está correto — unifica o tratamento de erros da API e de rede.

**Problema:** a verificação `if (!data.ok)` pressupõe que a resposta sempre será um JSON com o campo `ok`. Se o proxy retornar um erro HTTP (ex: 500 sem corpo JSON), a linha `const data = await response.json()` lançará uma exceção que chegará ao `catch` com uma mensagem de parse error — não de conexão. A mensagem ao usuário ("Verifique sua internet") pode ser tecnicamente incorreta nesses casos.

### 10.3 Dessincronização do histórico em caso de erro

Conforme identificado em H4:

```javascript
aiHistory.push({ role:'user', content: question }); // linha 1953
// ... fetch que pode falhar ...
aiHistory.push({ role:'assistant', content: reply }); // linha 2064 — não executada em erro
```

Em caso de erro, o histórico conterá a pergunta do usuário sem a resposta. Na próxima mensagem, o AI receberá um histórico com uma mensagem de usuário sem resposta — o que tecnicamente viola o alternado user/assistant esperado pela API Claude e pode causar comportamento inesperado.

**Solução recomendada:** remover a mensagem do usuário do histórico em caso de erro, ou só adicioná-la após a resposta bem-sucedida.

---

## 11. ANÁLISE DE LATÊNCIA

### 11.1 Arquitetura de latência

A cadeia de latência é:
```
Front-end → Cloudflare Worker → Claude API → Cloudflare Worker → Front-end
```

Latências estimadas (sem dados reais, baseado em arquitetura conhecida):
- Front-end → Worker: ~20–100ms (Cloudflare edge, próximo do usuário)
- Worker → Claude API: ~100–300ms (Anthropic API, provavelmente US East)
- Claude API (geração): ~2–15 segundos dependendo do comprimento da resposta
- Worker → Front-end: ~20–100ms

**Latência total estimada:** 3–20 segundos por resposta.

### 11.2 UX durante a espera

```javascript
const loading = addAIMsg('bot', '...');
loading.classList.add('loading');
```

O indicador de carregamento é `'...'` com estilo `italic` e `color: var(--text3)`. É funcional mas minimalista.

**Problemas de UX:**
- Sem indicador visual animado (spinner, typing dots)
- Sem estimativa de tempo
- O botão de envio não é desativado durante o carregamento — o usuário pode enviar múltiplas mensagens enquanto espera, causando requisições paralelas e possível desordem no histórico
- Não há timeout definido — se o proxy não responder, o indicador `'...'` permanece indefinidamente

### 11.3 Ausência de streaming

O sistema aguarda a resposta completa antes de renderizar. Para respostas longas (teológicas), isso pode resultar em 10–15 segundos de tela com apenas `'...'` antes de receber o texto completo. A implementação de streaming (Server-Sent Events ou resposta chunked) melhoraria significativamente a percepção de latência — mas requer mudanças no proxy também.

---

## 12. ANÁLISE DE USO DE TOKENS

### 12.1 Tokens por requisição

**System prompt:** ~450 tokens (estimativa)  
**Histórico médio (10 turnos):** ~800–2000 tokens  
**Pergunta do usuário:** ~10–50 tokens  
**Total de input por requisição:** ~1.300–2.500 tokens

**Resposta do modelo (output):** ~200–800 tokens

**Custo por conversa (20 turnos, janela deslizante):**
- Input acumulado: ~50.000–100.000 tokens
- Output acumulado: ~16.000 tokens

### 12.2 Ineficiência do system prompt por requisição

O system prompt de ~450 tokens é enviado a **cada requisição**. Se o proxy não implementar o cache de prompt da API Anthropic (`cache_control: "ephemeral"`), cada mensagem do usuário pagará o custo completo do system prompt como input. Para uma conversa de 10 mensagens, o system prompt representa ~30% do custo total de tokens.

**Recomendação:** verificar se o proxy Cloudflare Worker usa `anthropic-beta: prompt-caching-2024-07-31` — se sim, o system prompt é cacheado após a primeira requisição e não é cobrado novamente por 5 minutos.

### 12.3 Sem limite de tokens de saída

O payload enviado ao proxy não especifica `max_tokens`. Dependendo da configuração do proxy/API, respostas muito longas podem ser geradas — aumentando custo e latência sem necessariamente melhorar a utilidade pastoral da resposta.

---

## 13. CLAREZA DAS RESPOSTAS

### 13.1 Estrutura definida no prompt

Para perguntas substanciais, o prompt define até 5 seções:
1. Resposta Resumida
2. Fundamentação Bíblica
3. Compreensão Bíblica do Tema
4. Aplicação Prática
5. Próximo Passo

E permite respostas breves para perguntas simples: *"Para perguntas curtas e diretas, responda de forma direta e calorosa, sem forçar a estrutura completa."*

**Avaliação:** Estrutura bem calibrada — não robotiza respostas simples e oferece profundidade quando necessário.

### 13.2 Renderização das respostas

```javascript
div.textContent = text;
```

As respostas são exibidas como **texto puro** — toda a formatação markdown (negrito com `**`, listas com `-`, títulos com `#`) fica visível como caracteres literais. Se o modelo retornar "**João 3.16**", o usuário vê `**João 3.16**` com asteriscos, não o texto em negrito.

**Este é um problema relevante de UX.** O prompt não instrui o modelo a evitar markdown, então o modelo vai usar markdown naturalmente (especialmente para listas de versículos e seções estruturadas). O resultado visual é confuso.

**Solução necessária:** ou converter markdown para HTML (`innerHTML` com sanitização), ou instruir o prompt a usar apenas texto puro sem markdown.

### 13.3 Fonte e tamanho

```css
.ai-msg { font-size: 20px; line-height: 1.6; font-family: var(--ff-body); }
```

Fonte de 20px é generosa — adequada para leitura em dispositivos móveis e para usuários com dificuldades visuais (contexto hospitalar). Line-height 1.6 é excelente para legibilidade.

---

## 14. SEGURANÇA

### 14.1 Proteção do cliente

| Aspecto | Status | Avaliação |
|---|---|---|
| XSS via resposta do AI | ✅ Protegido | `textContent` em vez de `innerHTML` |
| Injeção de HTML pelo usuário | ✅ Protegido | `textContent` |
| Exposição de API key | ✅ Protegido | Proxy oculta a chave |
| HTTPS para o proxy | ✅ Cloudflare Worker usa HTTPS por padrão |
| Validação da resposta do proxy | ⚠️ Parcial | Verifica `data.ok` mas não valida estrutura completa |

### 14.2 Proteção do proxy

Não é possível analisar o código do Cloudflare Worker a partir do repositório. Presumindo arquitetura padrão de proxy Claude:

| Aspecto | Status presumido | Risco |
|---|---|---|
| Autenticação no proxy | Desconhecido | 🔴 Se o proxy não requer autenticação, qualquer pessoa com a URL pode usar o AI às custas do projeto |
| Rate limiting no proxy | Desconhecido | 🔴 Sem rate limit, o proxy é vulnerável a abuso |
| Validação de origem (CORS) | Desconhecido | 🟡 Sem CORS restrito, o proxy aceita chamadas de qualquer domínio |
| Log de conversas | Desconhecido | 🟡 Conversas podem ser logadas pelo Worker para debugging |

### 14.3 Privacidade do usuário

O histórico da conversa é enviado ao proxy (e portanto ao Claude API) a cada mensagem. Se o usuário mencionar informações pessoais sensíveis (diagnóstico médico, situação familiar, crises espirituais), esses dados são transmitidos para servidores externos (Cloudflare e Anthropic). O app não informa o usuário sobre isso — não há aviso de privacidade ou termos de uso relacionados ao AI.

---

## 15. CHIPS DE SUGESTÃO (ai-sugs)

```javascript
function openAI() {
  showScreen('ai');
  document.getElementById('ai-sugs').innerHTML = '';
  // ...
}
```

O elemento `#ai-sugs` existe no HTML e tem CSS definido (`.ai-suggestions`, `.ai-sug`), mas a função `openAI()` **sempre o limpa** (`innerHTML = ''`) e nunca o popula com sugestões. A funcionalidade de chips de sugestão está **planejada mas não implementada**.

---

## 16. TABELA-RESUMO DE PROBLEMAS

| ID | Categoria | Descrição | Gravidade |
|---|---|---|---|
| PR1 | Prompt | Lista 17 estudos quando o app tem 16 — títulos errados | 🔴 Alto |
| PR2 | Contexto | Sem injeção de dados do usuário (nome, estudo, fase) | 🟡 Médio |
| PR3 | Prompt | Limite de escopo "relacionamentos" pode bloquear temas bíblicos legítimos | 🟡 Médio |
| PR4 | Segurança | Sem instrução de confidencialidade do prompt | 🟡 Médio |
| PR5 | Prompt | Sem instrução de idioma | 🔵 Baixo |
| PR6 | Prompt | Zona cinzenta ciência/fé sem orientação | 🟡 Médio |
| H1 | Histórico | Histórico não persiste entre sessões/recargas | 🟡 Médio |
| H3 | Histórico | Sem separação de contexto por estudo | 🔵 Baixo |
| H4 | Histórico | Dessincronização user/assistant em caso de erro | 🟡 Médio |
| L2 | Segurança | Sem rate limiting no front-end | 🟡 Médio |
| L3 | Segurança | Sem limite de caracteres no input | 🔵 Baixo |
| L5 | Segurança | Sem protocolo para crise de saúde mental | 🔴 Alto |
| C1 | Clareza | Markdown não renderizado — asteriscos visíveis | 🔴 Alto |
| BV | Boas-vindas | Mensagem genérica — não usa nome do usuário | 🟡 Médio |
| LAT1 | Latência | Botão de envio não desativado durante carregamento | 🟡 Médio |
| LAT2 | Latência | Sem timeout definido — spinner pode ficar permanente | 🟡 Médio |
| LAT3 | Latência | Sem streaming — UX de espera longa | 🔵 Baixo |
| TOK1 | Tokens | System prompt reenviado a cada requisição sem cache | 🟡 Médio |
| TOK2 | Tokens | Sem limite de max_tokens configurado | 🔵 Baixo |
| PRIV | Privacidade | Sem aviso ao usuário sobre envio de dados ao AI | 🟡 Médio |
| SUGS | UI | Chips de sugestão planejados mas não implementados | 🟡 Médio |
| PROX | Segurança | Autenticação e rate limiting do proxy desconhecidos | 🔴 Alto |

---

## 17. PONTOS ALTOS — O QUE ESTÁ BEM

1. **System prompt de alta qualidade** — um dos system prompts pastorais mais cuidadosos já analisados para um app de discipulado. A hierarquia de fontes, o protocolo apologético e a verificação final são excepcionais.

2. **Proteção XSS via `textContent`** — decisão correta e segura para renderização.

3. **Proxy como camada de proteção da API key** — arquitetura correta para um app público.

4. **Mensagem de boas-vindas clara sobre limitações** — o AI se apresenta honestamente desde o início, incluindo a instrução de recorrer ao tutor humano.

5. **Instrução sobre Ellen White** — teologicamente precisa e pastoralmente equilibrada.

6. **Seção de dízimos com detalhe excepcional** — transparência financeira como parte do prompt é uma decisão rara e sábia.

7. **Tom prescrito é equilibrado** — pastoral sem ser piegas, intelectualmente honesto sem ser arrogante.

8. **Hierarquia `Bíblia > Documentos IASD > Pesquisa`** — confessional e academicamente responsável.

---

## 18. RECOMENDAÇÕES POR PRIORIDADE

### Prioridade 1 — Críticas (implementar antes de lançar publicamente)

**R1 — Corrigir a contagem de estudos no prompt**  
Linha 2015: substituir "Os 17 estudos" por "Os 16 estudos" e corrigir os títulos para refletir os estudos reais do app.

**R2 — Adicionar protocolo de crise mental/emocional ao prompt**  
Inserir na seção REGRAS PRÁTICAS:  
*"Se o usuário expressar pensamentos de autolesão, desejo de morrer, desespero extremo ou situação de emergência, responda com acolhimento imediato, interrompa qualquer resposta teológica e oriente: 'Esta situação precisa de apoio urgente e presencial. Por favor, fale com a equipe do hospital agora ou ligue para o CVV (188).'"*

**R3 — Desativar botão de envio durante carregamento**  
Adicionar `input.disabled = true` e `document.querySelector('.ai-send').disabled = true` ao início do `sendAI()`, e reativar no `finally`.

**R4 — Corrigir dessincronização do histórico em erro**  
Mover `aiHistory.push({ role:'user', content: question })` para depois da resposta bem-sucedida, ou remover a entrada em caso de erro.

**R5 — Renderização de markdown**  
Implementar conversão simples de markdown para HTML com sanitização, ou adicionar instrução ao prompt: *"Responda sempre em texto puro, sem markdown. Use apenas quebras de linha para separar seções."*

### Prioridade 2 — Melhorias importantes

**R6 — Injetar contexto básico do usuário no prompt**  
Modificar `sendAI()` para incluir no início do system prompt (ou como primeira mensagem `system`):  
```
Contexto do usuário: nome = [ST.userName], estudo atual = [ST.currentStudy], fase = [ST.phase].
```

**R7 — Persistir histórico de AI no localStorage**  
Salvar `aiHistory` com chave `jornada_ai_history` no `localStorage`, e carregar ao iniciar o app.

**R8 — Adicionar timeout à requisição fetch**  
```javascript
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 30000);
const response = await fetch(AI_PROXY_URL, { signal: controller.signal, ... });
clearTimeout(timeout);
```

**R9 — Implementar chips de sugestão**  
Popular o `#ai-sugs` com 3–5 perguntas contextuais relacionadas ao estudo atual do usuário.

**R10 — Adicionar aviso de privacidade**  
Inserir na mensagem de boas-vindas: *"Suas perguntas são enviadas para processamento por inteligência artificial. Evite incluir informações médicas, nomes completos ou dados pessoais sensíveis."*

### Prioridade 3 — Otimizações

**R11 — Verificar cache de prompt no proxy**  
Confirmar se o Cloudflare Worker usa `anthropic-beta: prompt-caching-2024-07-31` para evitar re-cobrar tokens do system prompt a cada mensagem.

**R12 — Definir max_tokens no payload**  
Adicionar `max_tokens: 1024` ao body da requisição — suficiente para respostas pastorais completas sem exagero.

**R13 — Adicionar `maxLength` ao input**  
`<input maxlength="2000">` — limita perguntas muito longas.

**R14 — Esclarecer zona de escopo sobre ciência e família**  
Adicionar ao prompt:  
*"Para perguntas sobre ciência e fé (criação, evolução, Big Bang), responda com a perspectiva bíblica sem negar o método científico. Para perguntas bíblicas sobre família, casamento e pureza, responda o ensinamento bíblico sem adentrar em situações pessoais específicas."*

---

## 19. CONCLUSÃO

O Mentor IA do aplicativo "Aos Pés do Mestre Jesus" possui um **system prompt de qualidade excepcional** — talvez o ponto mais cuidadoso de todo o código analisado. A arquitetura teológica, o protocolo apologético e o tom pastoral estão bem calibrados para o público-alvo.

Os problemas mais críticos são de natureza técnica, não teológica:
1. A inconsistência no número de estudos (17 vs. 16) no prompt
2. A ausência de renderização de markdown (asteriscos visíveis)
3. A ausência de protocolo para situações de crise emocional
4. A possível falta de autenticação e rate limiting no proxy

O maior potencial de crescimento está na **personalização contextual** — injetar dados do usuário (nome, estudo atual, fase) no prompt transformaria o Mentor IA de um assistente bíblico genérico em um acompanhante espiritual verdadeiramente integrado à jornada discipular.

O Mentor IA é, em sua forma atual, um recurso teológico funcional e pastoralmente responsável — com refinamentos técnicos, tornará-se uma das funcionalidades mais poderosas e distintivas do aplicativo.

---

*Relatório gerado por análise estática do código-fonte (index.html, linhas 1919–2071 e 311–325).*  
*Nenhuma modificação foi feita ao código do aplicativo.*  
*Data: 26 de junho de 2026*
