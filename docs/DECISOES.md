# Registro de decisões (mais recente primeiro)

Uma entrada por decisão: **data · decisão · por quê · onde está no código**. Acrescente sempre que uma regra mudar.

## 02/11/2026
- **Possível Evento vale para todos os períodos (Maio–Outubro)** (`EVENTO_COLETIVO_A_PARTIR_DE = None`). Reclassifica ~1.100 turnos
  (Maio 246, Junho 161, Julho 198, Agosto 143, Setembro 202, Outubro 157); média de pontos sobe de 6 a 8 por mês, ninguém desce.
  Para poupar o passado: `date(2026, 10, 4)`.
- **Turno sem Lançamento é o padrão; Possível Evento Externo/Interno** com 20+ agentes no mesmo turno ou equipe inteira sem lançamento
  (sem desconto). Antes de descontar, confere cronograma. Por quê: coordenação quer verificar chuva/treinamento/atestado/férias/folga/reunião
  antes de punir. → `_preparar_ausencias`, `LIMITE_AGENTES_EVENTO_TURNO`. Teste: `test_turno_evento.py`.
- **Detecções novas de férias e paralisação entram `pendente`** (antes entravam confirmadas). Decisões antigas intactas.
  → `CONFIRMAR_FERIAS_AUTOMATICAMENTE`, `CONFIRMAR_CHUVA_AUTOMATICAMENTE`.
- **Agente 3177 — Patricia Lanjer de Souza Domingues, Equipe 3** (agente nova). Treinamento lançado no cronograma de 05/10 a 30/10/2026
  (`Reunião/Treinamento`). Se continuar em novembro, estender o período em `data/cronograma_ausencias.json`.
- **Alana (2951) e Teonilda (3035)** seguem como **falta** (decisão da coordenação): desconto mantido.
- **Painel de Custos remodelado:** custo no ritmo × custo real; dias esperados já sem férias/atestado/treinamento/paralisação;
  média geral = folha ÷ horas úteis; tabela de custo mensal por agente; legendas e faixas. → `generate_dashboard.py` (`build_kpi_custos` e afins).
- Seletor de meses no tema claro: cores próprias (antes se misturava ao cabeçalho). → `scripts/style.css` (`.period-select`).
- Ranking de todos os meses conferido: Maio–Outubro regenerados com o método novo; Acumulado segue no antigo.

## Pendente de decisão (02/11/2026)
- Dias de chuva da Ana Gabriela (28 e 29/10, 14/10 tarde, 30/10 manhã): não confirmados (só havia previsão, sem medição).
- Os 7 turnos de Outubro classificados como evento (06/10 M, 08/10 M, 13/10 M e T, 14/10 M, 15/10 M, 30/10 T) aguardam confirmação.
