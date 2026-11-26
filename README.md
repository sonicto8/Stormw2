# STORMWATCH

Meteorologia em tempo real para Portugal — previsão, avisos oficiais do IPMA, radar de precipitação animado, assistente STORM AI, comunidade em tempo real e partilha de emergência por WhatsApp. Interface **inteiramente em português europeu (pt-PT)**, instalável no iPhone como app (PWA) e com notificações push (iOS 16.4+).

**Sem subtítulos, sem marcas extra — o nome é apenas STORMWATCH.**

**Documentação:** [DEPLOY.md](DEPLOY.md) — publicar sem custos (Vercel + Turso, Cloudflare Tunnel) · `download/guia-stormwatch.pdf` — guia rápido para utilizadores (uso, instalação iPhone/Android, publicação).

---

## Identidade visual

Paleta inspirada no céu e no clima, com **cantos retos (0 px)** em toda a interface — estética técnica, profissional e de leitura imediata:

| Papel | Cor | Hex |
| --- | --- | --- |
| Primária (Azul Céu) | `#3498DB` |
| Secundária (Azul Escuro) | `#1E3A5F` |
| Fundo (Branco Gelo) | `#F5F9FC` |
| Destaque (Amarelo Sol) | `#F4C542` |
| Aviso moderado (Laranja) | `#FF8C42` |
| Alerta crítico (Vermelho) | `#E63946` |
| Texto (Cinzento Escuro) | `#2D3436` |

Gradiente de assinatura: `linear-gradient(135deg, #3498DB → #1E3A5F)`. Os mesmos tokens alimentam botões, cartões, menus, gráficos (Recharts) e ícones meteorológicos animados. Tema escuro derivado da mesma identidade (navy profundo + azul céu claro).

Regras de superfície:

- **Modo claro/escuro** com 3 opções nas Definições (Claro · Escuro · Automático) e alternação rápida no botão lua/sol da barra superior. A escolha persiste no dispositivo e a cor da barra do sistema acompanha o tema ativo.
- **Contornos reservados à navegação** — cartões, botões e painéis de conteúdo não têm contorno (fundo + sombra); o contorno visível (`.nav-outline`) fica apenas no menu lateral/barra de abas e nos separadores ativos. Severidade mantém barra lateral esquerda (border-l) nos avisos IPMA.

---

## Requisitos

- **Bun** 1.1+ (runtime e gestor de pacotes)
- Portas livres: **3000** (app Next.js), **3003** (alerts-service), **3010** (comment-service, socket.io) e **3004** (interno, apenas localhost)

## Executar localmente (3 terminais)

### Terminal 1 — dependências e base de dados (Prisma)

```bash
bun install
bun run db:push
```

O esquema Prisma (utilizadores do dispositivo, contactos de emergência, localidades favoritas, comentários da comunidade) é aplicado à base SQLite em `db/`.

### Terminal 2 — aplicação Next.js (porta 3000)

```bash
bun run dev
```

```
▲ Next.js 16.1.x (Turbopack)
- Local: http://localhost:3000
✓ Ready
```

### Terminal 3 — serviço de alertas (porta 3003)

```bash
cd mini-services/alerts-service
bun run dev
```

```
[STORMWATCH alerts-service] listening on port 3003
```

Sonda o IPMA a cada **5 minutos**, deteta avisos novos (diff), mantém um stream **SSE** para a app e envia **Web Push (VAPID)** aos dispositivos subscritos. As chaves VAPID são geradas localmente — ver `mini-services/alerts-service/README.md`.

> O **comment-service** (socket.io na porta **3010**, difusão instantânea dos comentários) arranca da mesma forma, em `mini-services/comment-service`. Sem ele, a comunidade continua a funcionar por atualização periódica.

### Variáveis de ambiente (opcionais)

| Variável | Efeito |
| --- | --- |
| `DATABASE_URL` | Caminho da base de dados Prisma (predefinição: `file:../db/stormwatch.db`). |
| `OPEN_METEO_API_KEY` | **Chave de API do Open-Meteo** (grátis em open-meteo.com → conta → API key). Recomendado em servidores com IP partilhado: sem chave, todos os utilizadores desse IP partilham a mesma quota diária e, quando ela se esgota, a app mostra os últimos dados em cache com um aviso até à reposição (meia-noite UTC). Com chave, a quota passa a ser própria e independente. A app funciona sem ela — a chave só evita o limite. |
| Chaves VAPID (alerts-service) | Ver `mini-services/alerts-service/README.md`; sem elas, o push fica «Não configurado» nas Definições. |

### O aviso «servidos de cache — limite de utilização» 

Quando a fonte externa (Open-Meteo) devolve `429 Daily API request limit exceeded`, a aplicação **não falha**: mostra os últimos dados válidos (memória + base de dados) com um aviso discreto e volta a tentar sozinha — imediatamente após a reposição da quota ou a cada **15 minutos** (disjuntor interno que evita pedidos inúteis). Se o aviso aparecer com frequência no teu servidor, define `OPEN_METEO_API_KEY`.

## Ver no iPhone (HTTPS obrigatório)

O GPS e o Web Push exigem HTTPS. No local, expõe a app com um túnel:

```bash
# Túnel rápido (URL temporário)
cloudflared tunnel --url http://localhost:3000

# Domínio fixo (requer túnel já criado no Cloudflare)
cloudflared tunnel run stormwatch
```

Alternativas: `npx localtunnel --port 3000` ou `ngrok http 3000`.

No iPhone (Safari — o iOS só permite instalação PWA pelo Safari):

1. Abre o URL **https** do túnel.
2. Ecrã de boas-vindas → **«Ativar localização»** (se recusares a permissão, usa **«Pesquisar cidade»**).
3. Partilhar → **Adicionar ao Ecrã Inicial** → abre pelo ícone STORMWATCH.
4. Em **Definições → Notificações push → Ativar notificações** (três condições do iOS 16.4+: app instalada no ecrã principal, autorização explícita e app/serviço ativos em segundo plano).

## Deploy

> Guia detalhado de publicação gratuita (Vercel + Turso, Cloudflare Tunnel, checklist pós-publicação): **[DEPLOY.md](DEPLOY.md)**.

### Vercel

```bash
bunx vercel            # deteta Next.js automaticamente
bunx vercel --prod
```

- Base de dados: usa um SQLite persistente num volume (Railway/Fly) ou troca o provider Prisma por Postgres/Turso.
- O alerts-service não corre na Vercel — hospeda-o num serviço separado (Railway/Fly) e define o host público nos clientes (`alerts-service-client.ts`).

### Railway

```bash
bunx @railway/cli init
bunx @railway/cli up
```

Define `start` como `bun run start` (ou usa o Dockerfile gerado) e adiciona um volume para `db/`. Railway suporta múltiplos serviços no mesmo projeto — cria um segundo serviço para `mini-services/alerts-service`.

### Fly.io

```bash
fly launch                # gera fly.toml (Next.js)
fly deploy
fly volumes create stormwatch_data --size 1 --mount-path /app/db
```

Corre o alerts-service como processo adicional (`[processes]` no `fly.toml`) ou como app separada na mesma rede interna.

## Arquitetura

```
Next.js 16 (App Router, Turbopack) ── porta 3000
  ├─ Route handlers /api/* ─ cache 10 min + Zod (respostas upstream validadas)
  ├─ Prisma (SQLite) ─ perfil, contactos de emergência, favoritas, comentários
  └─ Cliente socket.io → gateway → porta 3010 (difusão ao vivo)

mini-services/alerts-service ── porta 3003
  └─ Poll IPMA (5 min) · diff de avisos · SSE /api/stream · Web Push VAPID

mini-services/comment-service ── porta 3010 (socket.io) · 3004 (interno, localhost)
  └─ Difusão `comment:new` a todos os clientes ligados
```

**Fontes de dados (apenas estas):** Open-Meteo (forecast, air-quality, geocoding), IPMA (avisos oficiais + distritos), RainViewer (radar). Nenhuma chamada externa direta do browser — tudo passa por route handlers com cache de 10 minutos e validação Zod.

## Testes

```bash
bunx vitest run          # testes unitários (motores, adaptadores, schemas)
```

## Estrutura

```
src/app/api/*            route handlers (forecast, air-quality, geocoding, ipma,
                         radar, comments, reports, storm-ai, push…)
src/components/*         UI por separador (Início, Alertas, Assistente, Radar,
                         Comunidade, Planeador, Definições)
src/lib/engines/*        regras determinísticas (alertas, índices, sugestões,
                         contagem de chuva, intenções do assistente, emergência)
mini-services/*          alerts-service (3003) e comment-service (3010/3004)
prisma/schema.prisma     modelo de dados (`bun run db:push`)
public/                  manifest PWA (ícone raio STORMWATCH) + service worker
```
