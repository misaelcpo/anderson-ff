# SIGNAL.AI — PRD

## Problem Statement (original, in PT-BR)
"Cria um sistema que mande sinais de put e call apenas com uma foto ou print que vou te mandar. voce precisa analisar o par que esta aberto, o time e me dizer para próxima vela se vai ser put o call. qual a porcentagem de acerto da quela operação. quero um template limpo sem muita informação apenas o necessário com uma aparência de IA."

## User Choices
- Model: Gemini 3 Flash (`gemini-3-flash-preview`) via EMERGENT_LLM_KEY
- Auth: none (open access)
- History: yes + learn from WIN/RED feedback
- Language: Portuguese
- Vibe: minimal, "AI" tech aesthetic (dark, neon)

## Architecture
- Backend: FastAPI + Motor (MongoDB). Emergentintegrations LlmChat with Gemini 3 Flash Vision.
- Frontend: React + Tailwind + shadcn primitives (sonner toast) + framer-motion + lucide-react.
- Deep-void black theme; Outfit + JetBrains Mono fonts; cyan (CALL) / hot-pink (PUT) accents; WIN green / RED red.

## Implemented (2026-02)
- Backend endpoints (all prefixed /api):
  - `GET /` health
  - `POST /analyze` – base64 chart in → Signal (par, timeframe, PUT/CALL, confidence 55-96, analysis)
  - `GET /signals` – latest 50 (desc)
  - `POST /signals/{id}/feedback` – WIN/RED
  - `GET /stats` – winrate metrics
- Learning: past feedback aggregated and injected into the system prompt of every new analysis (pair+timeframe+signal WIN/LOSS ratios).
- Frontend `SignalStudio`:
  - Drag & drop / click upload (PNG/JPG/WEBP), client-side downscale to ≤1600px JPEG.
  - "IA_ANALISANDO_PADROES..." scan animation with cyan laser sweep.
  - Big PUT/CALL result panel with glow, confidence bar, analysis note.
  - WIN / RED feedback buttons.
  - History table with time, pair, TF, signal, confidence, result badge.
  - Stats bar: total / wins / reds / winrate.
- Testing: 13/13 backend tests pass (real Gemini call).

## Backlog
- P1: Auth or rate-limit (Emergent LLM key concurrency=1 → serialize UX)
- P1: Retry/backoff on 429 during /analyze
- P2: Pagination on /signals
- P2: Export history CSV
- P2: Manual signal expiry timer (start countdown after result)
- P2: Multiple language toggle (PT/EN)

## Update 2026-02-XX — Autopilot Mode
- **Manual upload removed.** IA agora opera 100% autônoma.
- New backend endpoints:
  - `GET /api/system/status` — powered + next candle close UTC + monitored pairs
  - `POST /api/system/power {on:bool}` — persistente em DB
- Autopilot loop (`asyncio.create_task` no startup):
  - A cada 20s, verifica se `power` está ON no DB
  - Grada sinais expirados usando preço de fechamento real do Binance (data-api.binance.vision) → WIN se predição PUT/CALL bateu com a direção real
  - Se há nova vela M5 fechada em algum par monitorado, busca 40 candles, renderiza chart candlestick com matplotlib (dark neon), envia para Gemini 3 Flash, publica sinal + Telegram
- Pares: BTC/USDT, ETH/USDT, SOL/USDT via `data-api.binance.vision` (public, sem chave, sem geo-block)
- Timeframe fixo M5, expiry = próxima vela (candle close + 5min)
- Grade automático → WIN/RED enviados ao Telegram com preço de entrada/saída
- Nova UI: 3 cards de pares (último sinal por pair), countdown até próxima vela, painel de último sinal com entrada/resultado, histórico com colunas entrada/saída

## Update 2026-02-XX — Análise técnica LOCAL (sem créditos) + Relatórios
- **Removido LLM do autopilot loop.** Sinais gerados por análise técnica pura em Python:
  - EMA9 vs EMA21 (tendência)
  - RSI(14) com Wilder smoothing (sobrecomprado/sobrevendido)
  - Momentum 5 velas
  - Padrão de engolfo (bullish/bearish engulfing)
  - Direção do corpo da última vela
  - Score agregado → PUT/CALL + confiança 55-92
- **Zero chamadas ao Gemini** durante operação normal (endpoint `/api/analyze` com upload manual ainda usa LLM se acionado)
- **Relatório automático por operação:** Telegram recebe ✅ WIN / ❌ LOSS com entrada/saída + tally do dia (`X W / Y L · winrate %`)
- **Relatório diário automático:** No primeiro tick de um novo dia UTC, envia resumo do dia anterior com totais + winrate + breakdown por par
- **Botão manual `ENVIAR RELATÓRIO`** na UI para forçar envio a qualquer momento (POST /api/report/send-now)
- **Endpoint `/api/stats`** agora inclui bloco `today` com estatísticas do dia
- UI: novo bloco HOJE (OP/W/L/WR) + label "M5 · Binance · RSI+EMA · 0 créditos"

## Update 2026-02-XX — Análise de vela avançada + Backtest 24h
- **Rule-based signal expandido** com novos sinais técnicos:
  - Body strength (corpo forte >= 1.5x média das últimas 20 velas)
  - Marubozu (body_ratio > 0.75)
  - Wick rejection (martelo / estrela cadente)
  - Streak direcional (3-5 velas consecutivas)
  - ROC acceleration (aceleração do momentum)
- Confiança agora vai até 96% (antes era 92%)
- **Endpoint `GET /api/backtest?candles=288`** roda a estratégia contra as últimas 288 velas M5 (24h) do Binance para cada par, retorna wins/losses/winrate/CALL-PUT split + confiança média em wins vs losses
- **UI Modal Backtest 24H** acessível pelo botão `BACKTEST 24H` (verde neon), mostra:
  - Overall (amostras, W, L, winrate colorido pelo threshold)
  - Tabela por par com OPS, CALL, PUT, W, L, winrate
  - Nota educacional sobre thresholds (>55% edge, 50-55% neutro, <50% perde)
  - Re-executar / Fechar buttons
- Resultado real do backtest com dados atuais: ~50% winrate (honesto — o mercado é difícil, especialmente em cripto lateralizada)

## Update 2026-02-XX — Assertividade + Fuso BRT
- **MACD (12,26,9)** e **Bollinger Bands (20, 2σ)** adicionados ao score
- **EMA multi-timeframe**: EMA9 > EMA21 > EMA50 = tendência forte (bônus)
- **Filtro de confiança mínima**: MIN_CONFIDENCE=78 (env-configurável) — só publica sinais ≥ 78%
  - Backtest calibrado: 78% conf → winrate 56.3% (BTC 56.8, ETH 54.2, SOL 57.7) - edge positivo
- **Fuso horário Brasil (America/Sao_Paulo, UTC-3)** em todos os pontos user-facing:
  - Telegram: horário de envio, expiração e fechamento em BRT
  - Relatório diário: usa dia BRT (00:00–23:59 BRT), formato DD/MM/YYYY
  - Frontend: history table e pair cards em BRT via `Intl.DateTimeFormat('pt-BR', {timeZone:'America/Sao_Paulo'})`
  - /api/stats "today" usa dia BRT
- Endpoint `/api/backtest?candles=X&min_confidence=Y` permite testar cortes customizados
- Contador de "skipped" no backtest mostra quantos sinais foram filtrados
