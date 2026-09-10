# Patagônia · João Rock

Site de portfólio e loja do fotógrafo **João Rock** para a expedição fotográfica pela Patagônia Argentina. Inclui:

- Hero com scrollytelling (foto de capa → mosaico de 6 cards)
- Roteiro editorial em 5 capítulos · 13 destinos
- Loja com livro de estreia, caneca, camiseta, boné, pôster, cartões postais e calendário

## Stack

| Camada       | Ferramenta                                          |
| ------------ | --------------------------------------------------- |
| HTML         | Estático, single-page (`index.html`)                |
| Estilização  | Tailwind CSS v4 (compilado em `src/output.css`)     |
| Animações    | GSAP 3.12.5 + ScrollTrigger (via CDN)               |
| Smooth scroll| Lenis 1.3.26 (via CDN)                              |
| Fontes       | `system-ui` + Georgia (serif editorial)             |
| Hosting      | Vercel (static)                                     |

Sem build step, sem framework JS, sem dependências de runtime (apenas CDNs de GSAP/Lenis).

## Como rodar localmente

```bash
# Qualquer servidor estático serve — o site é puro HTML/CSS
python3 -m http.server 8000
# ou
npx serve .

# Abra http://localhost:8000
```

### Recompilar o CSS (se editar `src/input.css`)

```bash
# Watch mode
npm run watch:css

# Build único (minificado)
npm run build:css
```

> **Importante:** o arquivo commitado é `src/output.css` (compilado, ~26 KB minificado). Edite `src/input.css` e regere antes de deployar.

## Estrutura de arquivos

```
patagonia/
├── index.html                  # Página única (~700 linhas, tudo inline)
├── src/
│   ├── input.css               # Tailwind source (@import "tailwindcss")
│   └── output.css              # Tailwind compilado (commitado)
├── package.json                # Scripts de dev/build/deploy
├── vercel.json                 # Config de hosting (clean URLs, headers)
├── .vercelignore               # Exclui node_modules, lixo, .agent/
├── README.md                   # Este arquivo
├── HANDOFF.md                  # Documentação técnica detalhada
└── .agent/skills/              # Skills locais do projeto (não vai pro deploy)
```

## Deploy

### Vercel (recomendado)

```bash
# 1. Instalar CLI globalmente (se ainda não tiver)
npm i -g vercel

# 2. Login (abre o browser) ou usar token
vercel login
# ou
export VERCEL_TOKEN=seu_token_aqui

# 3. Deploy
vercel              # preview
npm run deploy      # produção
```

A `vercel.json` já está configurada para:
- Servir como static site (sem build)
- `cleanUrls` ativo (sem `.html` na URL)
- `Cache-Control: public, max-age=31536000, immutable` em `/src/output.css`
- Headers de segurança (`X-Content-Type-Options`, `Referrer-Policy`)

## Imagens e assets

Todas as fotos são carregadas do **Unsplash CDN** em runtime (`https://images.unsplash.com/photo-...`). Não há imagens bundled no repositório. Se alguma foto quebrar no futuro (Unsplash remove/expirar), basta trocar o ID no `src` da `<img>` correspondente.

## Skills locais (não-deploy)

A pasta `.agent/skills/` contém documentação de design e padrões usados durante o desenvolvimento. São ignorados no deploy via `.vercelignore`. Não afetam o site em produção.

## Licença

Conteúdo (textos, fotos do projeto) © João Rock.  
Código aberto para fins de estudo/portfólio.
