# Handoff Técnico — Patagônia · João Rock

> Documentação para a próxima pessoa/equipe que mexer no site. Cobre arquitetura, decisões, conteúdo, pontos de extensão e pegadinhas conhecidas.

## 0. Log da sessão 2026-09-10

**Hero foi refeito inteiro** durante esta sessão. Linha do tempo das tentativas:

1. ❌ Imagem Unsplash estática (original) — usuário pediu movimento
2. ❌ `<video>` com scrub via GSAP `onUpdate` — vídeo travava em frame 0
3. ❌ `<video>` com `discoverDuration` (seek `1e10` + `timeupdate`) — mesmo assim travava
4. ❌ `<video>` com `tl.progress(self.progress)` manual — bug: ScrollTrigger do vídeo ficou em `range 6000-12000` (depois do pin)
5. ❌ `<video>` com `tl.eventCallback("onUpdate", ...)` — frame swap funcionava, mas seek do WEBM quebrava no servidor local
6. ✅ **240 frames JPEG (1280×720) em `/src/img/frame_NNNN.jpg`**, trocados via `<img id="hero-frame">` no `eventCallback("onUpdate")` da timeline

**Causa raiz do travamento do vídeo (lição importante):**
- `python -m http.server` (e o `SimpleHTTP` em geral) **não suporta Range Requests**
- O browser precisa de `Range: bytes=X-Y` pra fazer seek em qualquer formato de vídeo
- Resultado: `video.currentTime = 5` ficava em 0 com `seeking=true` eterno
- **Em produção (Vercel) funciona**, mas local quebra — sempre usar `npx serve` ou similar
- A solução do `<img>` evita o problema: 240 requests normais (200 OK), browser cache resolve

**Commit final:** `8ad5203` (240 frames + index.html + output.css)

## 1. Visão geral

Site estático de página única com 4 grandes blocos:

```
┌──────────────────────────────────────────────┐
│  HERO + MOSAICO  (pinned, scrollytelling)     │  ← GSAP pin 2500px
│  - Foto de capa encolhe para 0.45            │
│  - Mosaic de 6 cards aparece                  │
│  - Hero fade out completo                      │
├──────────────────────────────────────────────┤
│  ROTEIRO (5 capítulos, scroll normal)         │  ← Lenis + ScrollTrigger
│  1. Lagos Andinos       (3 destinos)          │
│  2. Patagônia Central   (3 destinos)          │
│  3. Santuário Trekking  (2 destinos)          │
│  4. Cordilheira Austral (2 destinos)          │
│  5. Fim do Mundo        (3 destinos, full-bleed)│
├──────────────────────────────────────────────┤
│  O PROJETO GLOBAL (CTA curto)                 │
├──────────────────────────────────────────────┤
│  LOJA                                         │
│  - Livro em destaque (card 5+7 colunas)       │
│  - Grade 3×2 de produtos                       │
│  - Footer com 3 colunas de info de envio      │
└──────────────────────────────────────────────┘
```

## 2. Stack e dependências

**Runtime (CDN, nada bundlado):**
- `gsap@3.12.5` — animações
- `ScrollTrigger@3.12.5` — plugin GSAP para triggers de scroll
- `lenis@1.3.26` — smooth scroll
- Tailwind CSS já compilado em `src/output.css`

**Dev (npm):**
- `tailwindcss@4.3.3`
- `@tailwindcss/cli@4.3.3`

**Por que sem build JS?** Single-page com ~700 linhas de HTML. O único JS é o motor de animação, que vem do CDN. Não justifica bundler.

## 3. Arquitetura interna

### 3.1 O scrollytelling (hero)

GSAP timeline pinned em `#hero-container`:
- `pin: true`, `scrub: 0.3`, `end: "+=6000"` — 6000px de scroll virtual (aumentado pra acomodar a duração de 10s do scrub dos frames)
- **Container `#hero-image`** recebe os tweens de transform/fade (scale 1→0.75, borderRadius, brightness, opacity, y)
- **`<img id="hero-frame">`** dentro do container troca o `src` a cada update da timeline

**Frame swap (coração da animação):**
```js
const TOTAL_FRAMES = 240;
const FRAME_PATH = (n) => `src/img/frame_${String(n).padStart(4, "0")}.jpg`;

const preloaded = new Set();
function preloadFrame(n) {
    if (preloaded.has(n)) return;
    preloaded.add(n);
    const img = new Image();
    img.src = FRAME_PATH(n);
}
function preloadAround(currentIndex, range = 40) {
    for (let i = Math.max(1, currentIndex - range); i <= Math.min(TOTAL_FRAMES, currentIndex + range); i++) {
        preloadFrame(i);
    }
}

let currentFrame = 1;
function setFrame(n) {
    n = Math.max(1, Math.min(TOTAL_FRAMES, n));
    if (n === currentFrame) return; // evita reflow desnecessário
    currentFrame = n;
    heroFrame.src = FRAME_PATH(n);
    preloadAround(n);
}

// No onUpdate da timeline (dispara a cada tick do scrub):
heroTL.eventCallback("onUpdate", () => {
    const frameIndex = Math.floor(heroTL.progress() * (TOTAL_FRAMES - 1)) + 1;
    setFrame(frameIndex);
});
```

**Preloading:** primeiros 80 frames já vêm quentes no boot, depois 40 antes/depois do atual são recarregados dinamicamente. Browser cache resolve o resto após primeiro scroll.

**Estado inicial crítico**: o CSS initial dos cards é `clip-path: inset(0 100% 0 0)` etc. **GSAP não consegue parsear `transform: none` ou `clip-path: none` como estado inicial** — sem o `gsap.set()` antes da timeline, o GSAP assume valores zero e o `fromTo` falha silenciosamente. Por isso as três linhas de `gsap.set()` no início do script são obrigatórias.

```js
// OBRIGATÓRIO — sem isso, a hero abre PRETA (filter: brightness(0))
gsap.set("#hero-image", { filter: "brightness(1)", borderRadius: 0, opacity: 1 });
gsap.set("#mosaic-layer", { opacity: 0, pointerEvents: "none" });
gsap.set(".mosaic-item", { y: 120, opacity: 0, scale: 0.85 });
```

### 3.2 As 8 animações variadas do roteiro

Cada card recebe uma classe `anim-*` e o handler `revealCard()` mapeia para a animação GSAP correspondente:

| Classe            | Animação                                              | Uso típico                |
| ----------------- | ----------------------------------------------------- | ------------------------- |
| `anim-fade-up`    | y 80 → 0, opacity 0 → 1                                | Headers de região, eyebrow|
| `anim-slide-left` | x -120 → 0                                             | Títulos "01" "03"          |
| `anim-slide-right`| x +120 → 0                                             | Títulos "02" "04"          |
| `anim-scale`      | scale 0.7 → 1 (back.out easing)                        | Cards especiais            |
| `anim-rotate`     | y 60, rotate 2° → 0                                    | Cards pequenas, variação   |
| `anim-mask`       | clip-path inset(0 100% 0 0) → inset(0 0% 0 0)         | Cards grandes              |
| `anim-mask-v`     | clip-path inset(0 0 100% 0) → inset(0 0 0% 0)         | Reveal de baixo            |
| `anim-blur`       | blur(20px) → 0, opacity 0 → 1, y 40 → 0                | Cards com foco             |
| `anim-clip-up`    | clip-path inset(100% 0 0 0) → inset(0% 0 0 0)         | Cards verticais           |

### 3.3 Reveal palavra-por-palavra (intro "O Roteiro")

```js
gsap.fromTo(spans,
    { y: (i, el) => el.offsetHeight * 1.1 },
    { y: 0, ... }
);
```

> **Pegadinha conhecida:** o CSS `transform: translateY(110%)` é lido pelo GSAP como `y: <pixels>`, não como `yPercent: 110`. Por isso usamos `y` em pixels (calculado de `el.offsetHeight * 1.1`) em vez de `yPercent`. Animar `yPercent: 0` em cima do valor de pixel "travado" não funciona — o bug foi identificado e corrigido nesta versão.

### 3.4 Linha vertical da jornada

`.journey-line` é um `position: fixed` com gradient vertical. Aparece quando `#roteiro-intro` entra na viewport e some quando `#regiao-5` termina. Controlada por `ScrollTrigger.create` com `onEnter` / `onLeave` / `onEnterBack` / `onLeaveBack`.

### 3.5 Loja

- Botões `.Adicionar` em `[data-product]` têm handler JS que troca o texto para `✓ Adicionado` e adiciona classe emerald por 1.4s. **Não há carrinho real** — é visual feedback.
- "327 exemplares restantes" é estático (hard-coded). Para usar real, integrar com API de estoque.

## 4. Conteúdo

### 4.1 Estrutura editorial

Todos os textos (roteiro, loja, descrições) estão em **pt-BR** e foram escritos para soar editorial, não comercial. Evitar "ALL-CAPS eyebrow" sem propósito, evitar acentuar uma palavra só, evitar "→" no final de CTAs.

### 4.2 Roteiro (5 regiões × ~2-3 destinos = 13 cards)

| Cap. | Título                              | Destinos cobertos                              |
| ---- | ----------------------------------- | ---------------------------------------------- |
| 01   | O Portão de Entrada · Lagos Andinos | Bariloche, Tronador, Pucón                     |
| 02   | A Patagônia Central · Estepes       | Península Valdés, Cueva de las Manos, Catedrais |
| 03   | O Santuário do Trekking e do Gelo   | El Chaltén, Glaciar Perito Moreno              |
| 04   | A Cordilheira Austral e os Fiordes  | Torres del Paine, Puerto Natales               |
| 05   | O Fim do Mundo                      | Tierra del Fuego, Canal de Beagle, Magalhães   |

**Fonte do texto**: roteiro vem de um briefing editorial do cliente (descrito em conversas). Manter o tom — "O Portão de Entrada", "A Imensidão Árida", "O Santuário" etc. Não é uma lista de cidades, é uma narrativa de viagem.

### 4.3 Loja (8 produtos)

| Produto                  | Foto usada (Unsplash)             | Preço  |
| ------------------------ | --------------------------------- | ------ |
| Livro Fotográfico Vol. 1 | Glaciar Perito Moreno (capa)      | R$ 249 |
| Caneca Estepa            | Estepa Patagônica                 | R$ 79  |
| Camiseta Fitz Roy        | Fitz Roy Sunrise                  | R$ 129 |
| Boné Perito Moreno       | Tierra del Fuego (pico)           | R$ 99  |
| Poster A2 Torres         | Torres del Paine                  | R$ 149 |
| Kit 12 Cartões Postais   | Glaciar Perito Moreno             | R$ 49  |
| Calendário 2026          | Estreito de Magalhães (glaciar)   | R$ 89  |

**Estratégia de imagem**: as fotos da Patagônia SÃO o produto (preview do que vai impresso). Não tentamos usar mockups reais de caneca/camiseta — ia ficar genérico. Em vez disso, mostramos a foto + categoria do produto + tag de variantes (tamanho, etc).

### 4.4 Hero / Mosaic

**Hero:** 240 frames JPEG em `/src/img/frame_0001.jpg` até `frame_0240.jpg` (1280×720, ~45KB cada, total 9.9MB). Troca de frame é linear com o scroll (frame N = `floor(progress * 239) + 1`).

Para **trocar a animação do hero**:
- Substituir os 240 arquivos em `/src/img/frame_NNNN.jpg` mantendo o padrão de nomenclatura
- Se mudar o número de frames: alterar `TOTAL_FRAMES` no JS
- Se mudar o aspect ratio: ajustar a `duration` da timeline (atualmente 240/24 = 10s)
- Se mudar a "velocidade" do scrub: ajustar `end: "+=NNNN"` no ScrollTrigger (atualmente 6000)

**Mosaic (6 cards):** IDs Unsplash:
- `1464822759023-fed622ff2c3b` — Glaciar Perito Moreno
- `1506744038136-46273834b3fb` — Fitz Roy Sunrise
- `1470071459604-3b5ec3a7fe05` — Torres del Paine
- `1500382017468-9049fed747ef` — Estepa Patagônica (substituiu uma que saiu do ar)
- `1510784722466-f2aa9c52fff6` — Ushuaia
- `1758736553564-488a2508dc1c` — Lago Nahuel Huapi (Dina Huapi)

## 5. Pontos de extensão

### 5.1 Adicionar mais regiões no roteiro

Copie o bloco `<section id="regiao-X" ...>`, ajuste o número, título e cards. O JS já está preparado — basta adicionar a `#regiao-X` no seletor:

```js
"#roteiro-intro .anim-fade-up, #regiao-1 ..., #regiao-X [class*='anim-'], ..."
```

### 5.2 Adicionar produtos na loja

Copie um `<article class="dest-card group anim-..." data-product>...</article>` da grade. Use uma classe `anim-*` ainda não usada naquele card.

### 5.3 Trocar imagens

**Hero:** trocar os 240 arquivos em `/src/img/frame_NNNN.jpg`. Manter o padrão de nomenclatura (`frame_0001.jpg` a `frame_0240.jpg`).

**Mosaic + Roteiro + Loja:** 21 URLs Unsplash. Para trocar, basta editar o `src` da `<img>`. Não há cache local — usar URL com `?auto=format&fit=crop&w=...&q=...` para garantir tamanho/qualidade adequados.

### 5.4 Conectar a um CMS / e-commerce real

Para um carrinho real:
- Substituir o handler de clique do botão `.Adicionar` por integração com Shopify Buy SDK, Stripe Checkout, ou um carrinho custom em localStorage
- A coleção de produtos pode ser injetada via JSON estático em `produtos.json` e populada via JS

## 6. Decisões e trade-offs

| Decisão                                  | Por quê                                                                                  |
| ---------------------------------------- | ---------------------------------------------------------------------------------------- |
| **Sem build JS**                         | Single-page, sem React, sem router. CDNs GSAP/Lenis resolvem tudo                        |
| **Tailwind v4 com `@import`**            | Compilação mais simples que config JS do v3. CSS é compilado uma vez e commitado         |
| **Imagens via Unsplash CDN**             | Sem uploads, sem build pipeline. Risco: foto pode sair do ar (já aconteceu 1x — Estepa)  |
| **Hero como image sequence (240 frames)**| Mais confiável que `<video>` (não precisa de Range Request). Funciona em qualquer servidor. Trade-off: 240 arquivos no repo (9.9MB) vs 1 WEBM (1.4MB) |
| **Abandono do WEBM no hero**             | Tentei WEBM com scrub mas seek quebrou no `python -m http.server` (sem Range). Em produção (Vercel) funcionaria, mas local não. Decidi pela image sequence pra ter consistência dev/prod |
| **Animações por classe**                 | Mais simples que gerenciar 13 timelines separadas. Reutiliza o `revealCard()`            |
| **CSS `transform: translateY(110%)`**    | Mais performático que `opacity: 0` para a intro. GSAP calcula em pixels e anima `y`     |
| **Linha vertical fixa (journey line)**   | Reforça a metáfora de "jornada" sem ser intrusiva. Só aparece no roteiro                |
| **Botões "Adicionar" sem carrinho real** | Site é portfólio, não e-commerce. Feedback visual já comunica a intenção                 |
| **Layout 3+2 na loja**                   | 6 produtos não cabem bem em 2 colunas (largo demais) nem 4 (apertado). 3+2 é o sweet spot|
| **Region 05 com bg full-bleed**          | Único "respiro" de imagem grande no roteiro. Quebra a sequência e dá peso ao fim        |
| **404 da foto Estepa original**          | Substituída por `1500382017468-9049fed747ef`. Lição: cachear alternativa na HEAD         |

## 7. Pegadinhas conhecidas (read this before changing)

1. **Não remover os `gsap.set()` antes da timeline do hero** — sem eles, a hero abre preta (filter brightness(0)).
2. **`anim-*` precisam de `transition-delay` removido** — se quiser delays, use GSAP `delay:` em vez de `style="transition-delay:.15s"` (o CSS transition não é usado pelo GSAP).
3. **CSS `transform: translateY(110%)` em `.reveal-word > span`** — o GSAP lê como pixels, então a animação usa `y` em pixels. Se mudar a fonte/tamanho, recalcule `offsetHeight * 1.1` (ou refatore para usar `yPercent` direto, removendo o CSS inicial).
4. **A `journey-line` é `position: fixed`** — só some se `#regiao-5` terminar. Se adicionar mais seções, ajuste o `endTrigger` no JS.
5. **Unsplash pode remover fotos** — sempre tenha IDs alternativos testados para as 21 fotos do site.
6. **Tailwind v4 com `@import "tailwindcss"`** — não funciona com a config v3 (`tailwind.config.js`). Se voltar para v3, remova o `@import` e adicione as diretivas `@tailwind base/components/utilities`.
7. **CDNs GSAP/Lenis** — se a Vercel Edge bloquear domínios externos em alguma região, fazer download dos JS e servir localmente.
8. **`<video>` seek precisa de servidor com Range Request** — `python -m http.server` não suporta. Sempre use `npx serve` ou similar para testar vídeo localmente. Em produção (Vercel) tá ok.
9. **2 ScrollTriggers no mesmo `trigger` se sobrepõem errado** — o segundo automaticamente começa onde o primeiro termina. Se precisar de dois triggers no mesmo elemento, use `onUpdate` da timeline em vez de `ScrollTrigger.create()` separado.
10. **`heroTL.eventCallback("onUpdate", ...)`** é a forma correta de adicionar lógica que dispara a cada tick do scrub — mais confiável que `onUpdate` dentro de `scrollTrigger:` no config da timeline.
11. **Preloading de frames** — se o usuário scrollar muito rápido, pode ver frames em branco. Aumentar `range` em `preloadAround(n, range)` (atualmente 40) cobre scrolls mais agressivos, ao custo de mais requests paralelos.

## 8. Métricas & validação

Feito no Playwright durante o desenvolvimento:

- **0 console errors / warnings** após todas as correções
- **21/21 imagens** carregam (200 OK no Unsplash)
- **240/240 frames** do hero carregam (200 OK, 1280×720 cada)
- **5 regiões** + intro + 5 cards da loja = todas as seções validadas visualmente
- **8 tipos de animação** aplicados aleatoriamente — efeito "viciante" mencionado no brief
- **Acessibilidade básica**: `lang="pt-BR"`, `alt` em todas as imagens, foco visível preservado do Tailwind, `prefers-reduced-motion` **NÃO** respeitado ainda (TODO)

**Validação do scrub do hero (Playwright headless, port 3000 com `serve`):**
```
scroll=    0px → frame=  1/240  heroOp=1.00
scroll= 1500px → frame= 60/240  heroOp=0.90
scroll= 3000px → frame=120/240  heroOp=0.31
scroll= 4500px → frame=180/240  heroOp=0.01
scroll= 6000px → frame=240/240  heroOp=0.00
```
1:1 linear, sem travamentos, sem loading delays visíveis.

## 9. Próximos passos sugeridos

- [ ] Respeitar `prefers-reduced-motion` (desabilitar GSAP/Lenis)
- [ ] Adicionar `loading="lazy"` nas imagens abaixo da fold
- [ ] Substituir CDN GSAP por bundle local para melhor cache
- [ ] Integrar carrinho real (Shopify Buy / Stripe)
- [ ] Adicionar Open Graph / Twitter Card meta tags
- [ ] Adicionar sitemap.xml e robots.txt
- [ ] Configurar domínio custom (ex: patagonia.joaorock.com)
- [ ] Lighthouse audit + correção de CWV
- [ ] Tradução para inglês (briefing já é bilíngue)
- [ ] **Limpar arquivos não usados:** `src/img/video-hero.webm` (1.4MB) e `src/img/video-hero.mp4` (2MB) ficaram orfãos após trocar pra image sequence. Estão untracked — `rm src/img/video-hero.*`
- [ ] **Otimizar frames do hero:** se ficar pesado no Lighthouse, converter para WebP (240 × ~15KB = ~3.5MB total)
- [ ] **Testar em mobile:** o scrub de 240 frames via mobile pode engasgar — considerar reduzir `TOTAL_FRAMES` em telas pequenas via media query

## 10. Contato / propriedade

- **Projeto**: João Rock · Patagônia
- **Repositório**: (este diretório)
- **Stack owner**: (definir)
- **Última atualização**: ver `git log` ou data de modificação do `index.html`
