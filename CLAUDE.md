# Sentinela Prime — guia rápido para o Claude

Leia ESTE arquivo primeiro. Depois: `docs/REGRAS.md` (regras de negócio), `docs/DECISOES.md` (o que foi decidido e quando)
e `docs/MAPA.md` (índice função → linha; os scripts têm até ~5.000 linhas, **não leia inteiros**: use `grep -n` / `sed -n 'A,Bp'`).

## O que é
Dashboard de monitoramento das visitas dos agentes de campo (e-Visita) de Ponta Porã/MS. Idioma: **português do Brasil**
(código, commits, textos). Publicado como site estático no GitHub Pages.

## Fluxo (GitHub Actions `update-dashboard.yml`: dom/seg/ter 07:00 UTC + botão "Run workflow")
```
e-Visita ──Selenium──▶ data/semanas/NNN.xlsx (cache por semana)
   scripts/coletar_evisita.py ─▶ data/Analise_Consolidada.xlsx (acumulado) + data/historico/semanas_AAA-BBB.xlsx (1 por mês)
   scripts/generate_dashboard.py + scripts/templates/ (Jinja2) ─▶ index.html, acumulado.html, historico/*.html, partes/
Editores (validar/deteccao/cronograma/home .html) ─▶ Worker Cloudflare (worker/index.js) ─▶ repository_dispatch
   ─▶ scripts/aplicar_decisao.py / lancar_ausencia.py / lancar_atestado.py ─▶ data/*.json
```

## Onde fica cada coisa
| Quero mexer em… | Arquivo (função/constante) |
|---|---|
| Pontuação e pesos (PONTOS_*) | `scripts/coletar_evisita.py` → `calcular_pontuacao`, bloco `PONTOS_*` (~L1440) |
| Turno sem lançamento / Possível Evento | `coletar_evisita.py` → `_preparar_ausencias`, `LIMITE_AGENTES_EVENTO_TURNO`, `EVENTO_COLETIVO_A_PARTIR_DE` |
| Ranking e dias esperados | `coletar_evisita.py` → `_preparar_ranking`, `RANKING_SEM_NORMALIZACAO_A_PARTIR_DE` |
| Detecção automática férias/chuva | `coletar_evisita.py` → `detectar_ausencias_e_paralisacoes`, `atualizar_deteccoes_pendentes`, `CONFIRMAR_*_AUTOMATICAMENTE` |
| Cronograma (justificativas) | `data/cronograma_ausencias.json`, `_obter_motivo_cronograma`, `MOTIVOS_CRONOGRAMA` |
| Agentes e equipes | `AGENTES_POR_EQUIPE` (coletar_evisita.py); nomes em `data/agentes_nomes.json` (automático); depois rode `python3 scripts/gerar_form_atestado.py` |
| Painel Custos & Produtividade | `scripts/generate_dashboard.py` (`build_kpi_custos`, `build_custos_*`) + `scripts/templates/sections/custos.html` |
| Visual (CSS / JS) | `scripts/style.css`, `scripts/bootstrap.js`, `scripts/templates/charts.js` |
| Abas do dashboard | `scripts/templates/sections/*.html` (uma por aba), `base.html` |
| Meses fechados/abertos | `data/meses_arquivados.json`, `data/historico/manifest.json`, `scripts/fechar_mes.py` |

## Comandos
```bash
./scripts/rodar_testes.sh                 # 10 testes do CI, uma linha cada (~20 s). Rode antes de todo commit.
python3 scripts/generate_dashboard.py data/historico/semanas_327-330.xlsx _teste.html   # página de teste (Outubro)
python3 tools/shot.py _teste.html custos  # screenshot claro/escuro + erros de JS (precisa playwright)
python3 scripts/gerar_mapa.py             # atualiza docs/MAPA.md depois de mexer em funções
```
Depois de gerar `_teste.html`, **apague** `_teste.html` e `partes/_teste*` (não commitar).

## Convenções
- **Nunca edite à mão** arquivos gerados: `index.html`, `acumulado.html`, `transparencia.html`, `historico/*.html`, `partes/*`,
  `assets/agentes_dados.js`, `data/Analise_Consolidada.xlsx`, `data/historico/*.xlsx`, `data/semanas/*`, `data/csv/*`.
  Mudou o código/template → o workflow regenera. O bot faz commits "Atualiza coleta e dashboard automaticamente": sempre `git pull --rebase` antes de push.
- Agente é identificado por **ID**, nunca por nome, em qualquer lógica (nomes podem repetir).
- Regra nova → constante com nome claro + comentário do porquê + teste em `scripts/test_*.py` + linha em `docs/DECISOES.md`.
- Dados públicos: nunca gravar CID/diagnóstico (atestado = "Ausência justificada").
- Token do GitHub: usar só em URL temporária de push (`https://x-access-token:TOKEN@github.com/...`), nunca gravar no remote. Alterar `.github/workflows/*` exige permissão "Workflows" no token.

## Armadilhas conhecidas (documentação/código desatualizados — confie neste arquivo e em docs/REGRAS.md)
- Mês ≠ mês civil: Outubro = semanas 327–330 (a semana de 04/10 abre o mês; 01–02/10 ainda são da semana 326/Setembro).
  Maio 305–308 · Junho 309–312 · Julho 313–317 · Agosto 318–321 · Setembro 322–326 · Outubro 327–330.
- `RANKING_SEM_NORMALIZACAO_A_PARTIR_DE = date(2026,1,1)`: o método novo vale p/ todo período de até 45 dias (todos os meses). O comentário
  do código ainda cita "Outubro/2026"; o Acumulado (>45 dias) segue no método antigo (normalizado).
- Detecção de férias dispara com `LIMITE_DIAS_UTEIS_FERIAS = 7` dias úteis (o README ainda diz 5). Desde 02/11/2026 as detecções novas entram
  **pendentes** (comentários do código, `deteccao_editor.html`, `validar.html` e README seção "Detecção automática" ainda dizem "confirmadas automaticamente").
- Cron real: dom/seg/ter 07:00 UTC (o README cita seg/ter/qui).
- Equipes sob demanda (`EQUIPES_SEM_DETECCAO_AUTOMATICA`: Bloqueio, Ponto Estratégico) ficam fora das regras de evento; a Equipe Bloqueio também fica fora do painel de custos.
- Nas planilhas, a 1ª tabela de cada aba termina na primeira linha em branco (as legendas vêm depois): leia só até aí.
