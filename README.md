# Dashboard de Monitoramento de Visitas — Ponta Porã (S E N T I N E L A - HUB)

Este repositório agora faz o processo **inteiro sozinho**: coleta os dados no
e-Visita, gera o `index.html` do dashboard e publica no GitHub Pages — tudo
automaticamente, segunda/terça/quinta de madrugada, sem precisar rodar nada na sua máquina nem
subir planilha manualmente.

Continua funcionando também o jeito antigo (subir uma `Analise_Consolidada.xlsx`
manualmente em `data/`) como alternativa/backup, caso a coleta automática
falhe ou você precise reprocessar um dado específico.

**Página inicial configurável + aba "Acumulado":** o `index.html` (home)
mostra o período definido em `data/config_home.json` — por padrão, Outubro/2026
(`{"arquivo": "data/historico/semanas_327-330.xlsx", "label": "Outubro/2026"}`).
Separado da home, `acumulado.html` sempre mostra a SOMA de TODAS as semanas
já coletadas até agora (não depende de configuração nenhuma, é sempre o
`data/Analise_Consolidada.xlsx` puro) — os dois aparecem lado a lado no
seletor de período do topo ("🏠 Início" e "📊 Acumulado"), junto com cada mês
arquivado em `historico/`.

Pra trocar qual período aparece na home, tem duas formas:
  - **Editor visual** (recomendado): abra **`home_editor.html`** no
    navegador, cole o conteúdo de `data/historico/manifest.json`, escolha o
    período no menu e copie o JSON gerado pra `data/config_home.json` no
    GitHub. Mesmo padrão do `cronograma_editor.html` — sem precisar mexer em
    código.
  - **Na mão**: edite `data/config_home.json` direto, com o caminho
    (relativo à raiz do repo) e o rótulo do arquivo que deve virar home.

Se `data/config_home.json` não existir ou apontar pra um arquivo que sumiu,
a home cai automaticamente pro acumulado, pra nunca gerar uma home quebrada.

**Coleta semana a semana:** a coleta não busca mais um intervalo de semanas
de uma vez só — cada semana do e-Visita (HTML ID) é buscada e cacheada
SEPARADAMENTE em `data/semanas/<html_id>.xlsx` (+ `_pe.xlsx` pro Ponto
Estratégico, `_pendencia.json` com os números de fechados/recusados). A cada
execução (agora domingo/segunda/terça às 3h):

  1. Recoleta a última semana já salva (os dados do e-Visita ainda podem mudar).
  2. Busca a(s) semana(s) nova(s), se já estiver(em) disponível(is) no site.
  3. Consolida TODO o cache disponível num único `data/Analise_Consolidada.xlsx`.
  4. Reconstrói `data/historico/` a partir do cache — **apenas páginas
     MENSAIS** (Julho, Agosto, Setembro, Outubro, Novembro...). As páginas
     semanais soltas (semanas 318-326) foram arquivadas em
     `data/historico/arquivo_semanal/` (fora do menu, dados preservados); o
     detalhe por semana continua disponível pelo filtro de semana dentro de
     cada página mensal. Pra voltar a gerar uma página por semana, troque
     `GERAR_PAGINAS_SEMANAIS` pra `True` em `scripts/coletar_evisita.py`.

Na primeira execução (`data/semanas/` vazio), o script coleta da semana
18/2026 (HTML 305, início de Maio) até a semana atual, uma planilha por
semana. O `data/semanas/` é commitado no repositório junto com o resto — é
ele que guarda "até onde já coletamos" de uma execução pra outra.

**Ciclos diferentes por semana:** o `id_ciclo` usado na consulta ao e-Visita
não é fixo — muda em pontos específicos do ano. Isso já está mapeado em
`CICLOS_TRATAMENTO`, em `scripts/coletar_evisita.py`:
```python
CICLOS_TRATAMENTO = [
    (305, 312, 162),  # semanas 305-312 (maio-junho/2026) usam id_ciclo 162
    (322, 326, 164),  # semanas 322-326 (setembro/2026) usam id_ciclo 164
    (327, 330, 164),  # semanas 327-330 (outubro/2026)  usam id_ciclo 164
    (331, 334, 165),  # semanas 331-334 (novembro/2026) usam id_ciclo 165
    # faixas sem entrada aqui usam o ID_CICLO padrão (163)
]
```
Se o site mudar de ciclo de novo no futuro, adicione uma nova linha aqui — o
formato é `(semana_inicio_html, semana_fim_html, id_ciclo)`.

**Limites de mês (histórico):** os "meses" do histórico (Maio/2026,
Junho/2026 etc.) NÃO batem exatamente com o calendário civil — são os
limites reais confirmados na URL do e-Visita, cadastrados em
`MESES_OFICIAIS`, em `scripts/coletar_evisita.py`:
```python
MESES_OFICIAIS = [
    (305, 308, "Maio/2026"),
    (309, 312, "Junho/2026"),
    (313, 317, "Julho/2026"),
    (318, 321, "Agosto/2026"),
    (322, 326, "Setembro/2026"),
    (327, 330, "Outubro/2026"),
    (331, 334, "Novembro/2026"),
]
```
(Maio e Junho ficam em `MESES_ARQUIVADOS`, preservados como estavam.) Agosto a Novembro/2026 também estão em `MESES_OFICIAIS` (o nome do arquivo usa
sempre a faixa oficial do mês, ex.: `semanas_327-330.xlsx`, mesmo com o mês
incompleto — assim o link da home nunca quebra). Ao cadastrar um mês
novo aqui, a próxima coleta já reagrupa e renomeia os arquivos certinho; planilhas
órfãs são apagadas, exceto as semanais (`semana_<N>.xlsx`), que vão para
`data/historico/arquivo_semanal/`.

**Novo mês (passo a passo):** quando o e-Visita abrir um ciclo novo, copie da
URL de cada consulta o `id_ciclo` e as semanas, e cadastre 3 linhas em
`scripts/coletar_evisita.py`: `MESES_OFICIAIS` (nome do mês),
`CICLOS_TRATAMENTO` (atividade 4) e `CICLOS_PONTO_ESTRATEGICO` (atividade 3).
Depois, se quiser, aponte a home em `data/config_home.json` (ou use o
`home_editor.html`). Já estão cadastrados: Outubro (Tratamento 164 / P.E. 157,
semanas 327-330) e Novembro (Tratamento 165 / P.E. 158, semanas 331-334).
Trava de segurança: a partir da semana 327, semana SEM ciclo de Tratamento
cadastrado não é coletada (erro claro no log) em vez de usar o ciclo padrão
(que devolve dados vazios).

**Reprocessamento automático e seguro:** a cada semana coletada com sucesso a
partir da 327, o ciclo usado fica registrado em
`data/semanas/_ciclos_coletados.json`. Toda execução reprocessa sozinha as
semanas disponíveis cujo registro não bate com o ciclo configurado (ou que
nunca foram coletadas direito) — foi assim que Outubro (327-330) é recuperado
na primeira execução depois desta mudança, e depois disso para de ser
reprocessado. A recoleta SUBSTITUI os dados da semana (nunca soma, então não
duplica) e nunca piora o cache: coleta vazia não apaga nada, e agente que
falhar ou vier com páginas faltando mantém os dados que já estavam salvos.

Pra recoletar manualmente só uma semana específica (ex.: corrigir um dado):
```
python scripts/coletar_evisita.py --semana 305
```
(305 é o HTML ID da semana — ver `scripts/semanas_evisita.py` pra converter
entre HTML ID e "Semana do Ano".)

**Ranking — sem fator de normalização a partir de Outubro/2026:** a regra
mora em `RANKING_SEM_NORMALIZACAO_A_PARTIR_DE` (`scripts/coletar_evisita.py`,
hoje `date(2026, 10, 4)` = início da semana 327). Períodos que COMEÇAM nessa
data ou depois (Outubro, Novembro...) usam o método novo: Peso Efetivo = Peso
Nominal para todos, pontos = 100 + Σ(quantidade × peso), e **Dias Úteis
Esperados** = dias úteis do próprio período − dias de férias/atestado/folga/
chuva/reunião do cronograma em que o agente não trabalhou. Períodos que
começam antes (Maio–Setembro e o Acumulado) seguem exatamente como eram, com
fator de normalização. Pra desligar a regra nova, coloque `None`. Testes:
`python3 scripts/test_ranking.py`.

**Aba "Média Mensal":** tabela agente × mês com a pontuação final de cada
mês e a média simples delas (`build_media_mensal` em
`scripts/generate_dashboard.py`). Não recalcula nada: lê a aba Ranking já
pronta de cada mês de `data/historico/manifest.json` (só meses "Mês/Ano"),
então meses fechados nunca mudam, e um mês novo entra sozinho na coleta
seguinte.

**Feriados nacionais:** `FERIADOS_NACIONAIS` (em `coletar_evisita.py`) traz os feriados nacionais de 2026 e 2027 (sem pontos facultativos como Carnaval e Corpus Christi; acrescente os estaduais/municipais que a equipe não trabalha). Só valem a partir de `FERIADOS_VALEM_A_PARTIR_DE` (01/07/2026): descontam os dias úteis de Julho em diante (Julho e Agosto não têm feriado nacional; Setembro 21, Outubro 19, Novembro 18) — no cálculo dos dias esperados e na detecção de férias. Meses anteriores (Maio–Junho) seguem como foram calculados. Pra valer desde sempre, coloque `None`.

**Ranking — regras do método novo (Outubro/2026 em diante):**
- **Bônus proporcionais:** os bônus únicos (+5 sem visita rápida, +5 sem turno sem lançamento, +3 pontualidade) só são concedidos a quem trabalhou pelo menos `BONUS_MIN_FRACAO_DIAS_ESPERADOS` (50%) dos dias esperados; o Detalhamento/Extrato explica quando o bônus não vem. Meses antigos mantêm a regra antiga.
- **Observação "possível afastamento não registrado":** aparece na tabela do Ranking pra quem trabalhou pouco (< 70% dos dias esperados) e tem muitos turnos sem lançamento (≥ 4) — é sinal de férias/licença/atestado que falta lançar em `data/cronograma_ausencias.json` (ou confirmar em `deteccao_editor.html`). Parâmetros `AVISO_AFASTAMENTO_*` em `coletar_evisita.py`. Só em períodos de ~1 mês (não no Acumulado).
- **Extrato por agente:** botão **🧾 Extrato** em cada linha do Ranking abre a folha com a conta completa (dias, bônus, perdas, limite 0–100), pronta pra imprimir ou salvar em PDF (1 página A4).
- **Dias esperados de períodos que atravessam a virada** (ex.: Setembro, 31/08 a 02/10) usam o mês que domina o período, não o do último dia.
- **Média Mensal** agora mostra a **tendência** (diferença do último mês pro anterior) e um mini-gráfico por agente; dá pra ordenar pela tendência.

**Conferência automática de ciclos:** `scripts/validar_ciclos.py` roda a cada execução do workflow. Semana já iniciada sem `id_ciclo` (Tratamento ou P.E.) ou fora de `MESES_OFICIAIS` → o passo final da execução FALHA (o GitHub avisa por e-mail) com o motivo no resumo da execução; ciclos das próximas semanas faltando viram só um aviso. A coleta também escreve no resumo da execução os agentes ausentes/páginas faltando. Testes: `python3 scripts/test_ciclos.py`.

**Impressão do relatório completo:** o botão "🖨️ Imprimir relatório completo do mês" imprime todas as abas (A4 paisagem), sem páginas em branco — a 1ª tabela de cada aba não quebra página, legendas recolhidas saem abertas e o estado é restaurado depois.

**Tamanho das páginas:** `enxugar_html()` (em `generate_dashboard.py`) colapsa a indentação do HTML fora de `<script>/<style>/<pre>`; as páginas ficaram ~45% menores (Acumulado 6,9 → 3,8 MB) sem mudar o que aparece.

**Semana do e-Visita vs. Semana do Ano:** o e-Visita usa dois números
diferentes pra semana — o HTML ID (só pra consultar o sistema) e a "Semana
do Ano" (só pra mostrar pro usuário). A conversão entre os dois
(`html_id = semana_ano + 287`, válida só pra 2026) mora inteira em
`scripts/semanas_evisita.py`. Se o offset mudar em outro ano, valide um par
(html_id, semana_ano) real no site e atualize `OFFSETS_VALIDADOS` nesse
arquivo — o script trava com um erro claro em vez de converter errado.

**Filtro por semana no dashboard:** a aba "Resumo Completo por Agente"
agora tem um seletor de semana (só aparece quando o período carregado cobre
mais de uma semana) — junto com os filtros de equipe/classificação/nome já
existentes. Selecionar uma semana troca os números da tabela pros daquela
semana específica, sem precisar trocar de página. A aba "Ponto Estratégico"
já tinha esse filtro por semana antes.


O gerador do HTML usa **Jinja2**: o `generate_dashboard.py` só carrega a
planilha e prepara os dados; todo o HTML fica separado em templates dentro de
`scripts/templates/`, um arquivo por aba do dashboard. Isso deixa o código
bem mais curto e fácil de mexer — pra ajustar o layout de uma aba, edita só
o `.html` dela, não precisa tocar no Python.

## Estrutura do repositório

```
.
├── data/
│   ├── Analise_Consolidada.xlsx      ← acumulado de tudo, gerado automaticamente pela coleta
│   ├── config_home.json              ← qual período aparece na home (edite com home_editor.html)
│   ├── cronograma_ausencias.json     ← chuva/folga/atestado/reunião/férias (edite com cronograma_editor.html)
│   ├── agentes_nomes.json            ← ID → nome de cada agente, salvo automaticamente pelo coletor
│   ├── deteccoes_pendentes.json      ← férias/paralisações detectadas automaticamente (confirme com deteccao_editor.html)
│   ├── semanas/                     ← cache de cada semana coletada (uma planilha por semana)
│   └── historico/                   ← planilhas + manifest.json de cada mês arquivado
├── index.html                       ← home (período de data/config_home.json)
├── acumulado.html                   ← soma de TODAS as semanas já coletadas
├── home_editor.html                 ← editor visual pra escolher o período da home (abra no navegador)
├── cronograma_editor.html           ← editor visual do cronograma (abra no navegador)
├── deteccao_editor.html             ← confirmar/rejeitar férias/paralisações detectadas (abra no navegador)
├── assets/
│   └── logo_ponta_pora.png
├── scripts/
│   ├── coletar_evisita.py           ← faz login e coleta os dados do e-Visita (novo)
│   ├── generate_dashboard.py        ← carrega a planilha e prepara os dados
│   ├── xlsx_scan.py                 ← separa tabelas empilhadas dentro de uma aba
│   ├── style.css
│   ├── bootstrap.js
│   ├── chart.umd.js
│   └── templates/                   ← todo o HTML do dashboard mora aqui
│       ├── base.html                ← estrutura geral da página
│       ├── macros.html              ← componentes reutilizáveis (kpi, gráfico, tabela de alerta)
│       ├── charts.js                ← instancia os gráficos Chart.js com os dados
│       └── sections/                ← um arquivo por aba
│           ├── visao_geral.html
│           ├── agentes.html
│           ├── equipes.html
│           ├── areas.html
│           ├── alertas.html
│           ├── ausencias.html
│           ├── pendencias.html
│           ├── ponto_estrategico.html
│           └── custos.html
├── .github/workflows/
│   └── update-dashboard.yml         ← coleta + gera + publica, tudo em um workflow
├── requirements.txt
└── index.html                       ← gerado automaticamente, não edite à mão
```

## Passo a passo — configurar pela primeira vez

### 1. Criar o repositório no GitHub
1. Entre em [github.com/new](https://github.com/new).
2. Dê um nome, ex.: `dashboard-visitas`.
3. Deixe **Public** (necessário para o GitHub Pages gratuito) ou **Private**
   se você tiver GitHub Pro/Team/Enterprise.
4. Não marque nenhuma opção de inicialização (README, .gitignore etc.) —
   vamos enviar os arquivos prontos.
5. Clique em **Create repository**.

### 2. Enviar estes arquivos para o repositório
A forma mais simples, sem instalar nada:
1. Na página do repositório recém-criado, clique em **uploading an existing file**.
2. Arraste **todas** as pastas e arquivos deste pacote (mantendo a estrutura
   de pastas: `data/`, `assets/`, `scripts/`, `.github/`, `requirements.txt`).
   O GitHub aceita arrastar pastas inteiras direto do seu computador.
3. Escreva uma mensagem de commit (ex.: "Primeira versão") e clique em
   **Commit changes**.

*(Se preferir usar Git pela linha de comando, veja a seção "Via linha de
comando" mais abaixo.)*

### 3. Cadastrar o CPF e a senha do e-Visita como Secrets

**Nunca coloque CPF/senha direto no código** — o repositório é público, então
qualquer texto commitado fica visível pra qualquer pessoa. O script já lê as
credenciais de variáveis de ambiente; no GitHub isso vem dos **Secrets**:

1. No repositório, vá em **Settings → Secrets and variables → Actions**.
2. Clique em **New repository secret**.
3. Crie um secret chamado `EVISITA_CPF` com o CPF de login.
4. Crie outro chamado `EVISITA_SENHA` com a senha.

> ⚠️ Se essa senha já foi compartilhada em algum chat, e-mail ou documento
> fora de um cofre de senhas, o ideal é trocá-la no e-Visita antes de seguir
> — trate-a como exposta.

### 4. Dar permissão de escrita para o workflow
1. No repositório, vá em **Settings → Actions → General**.
2. Em **Workflow permissions**, marque **Read and write permissions**.
3. Clique em **Save**.

*(Isso é necessário porque o workflow precisa commitar a planilha nova e o
`index.html` atualizado de volta no repositório.)*

### 5. Ativar o GitHub Pages
1. Vá em **Settings → Pages**.
2. Em **Source**, selecione **GitHub Actions** (não "Deploy from a branch").
3. Pronto — não precisa configurar mais nada aqui.

### 6. Rodar a primeira vez
1. Vá na aba **Actions** do repositório.
2. Clique no workflow **Coletar e Atualizar Dashboard** na lista à esquerda.
3. Clique em **Run workflow**. Deixe a opção "Coletar dados novos no e-Visita"
   marcada e confirme.
4. Aguarde — a coleta de todos os agentes pode levar alguns minutos (é a
   mesma coleta que já era feita, só que rodando no GitHub em vez da sua
   máquina). Quando os passos `build` e `deploy` ficarem verdes, vá em
   **Settings → Pages** — o link do seu dashboard vai aparecer no topo, algo
   como: `https://SEU-USUARIO.github.io/dashboard-visitas/`

## Ranking normalizado "por mês" — funciona igual em qualquer período

`PONTOS_INICIAIS` (100) foi calibrado pra um período de ~`DIAS_UTEIS_MES`
(22) dias úteis trabalhados. Sem correção, avaliar um agente num período
maior (o Acumulado, por exemplo, somando vários meses) acumula 3-4x mais
ocorrências brutas de cada critério — mais dias sem meta, mais visitas fora
do expediente etc. — e a pontuação desabava pro chão mesmo com o
desempenho diário exatamente igual (comprovei: mesmo padrão de desempenho
dava 102 pontos avaliado como 1 mês, e virava 0 avaliado como 4 meses
seguidos, sem essa correção).

Agora cada contagem é projetada pra uma "taxa equivalente por mês" antes
de aplicar o peso — então o tamanho do período avaliado (1 semana, 1 mês,
ou o histórico inteiro acumulado) não muda mais o resultado, só a TAXA de
ocorrências por dia trabalhado importa. Reproduzi o cenário com dados
sintéticos e confirmei: mesmo padrão de desempenho dá a mesma pontuação
tanto em 1 mês quanto em 4 meses seguidos.

**Teto de segurança (`TETO_FATOR_NORMALIZACAO`, padrão 2x):** sem um
limite, o efeito inverso acontecia com quem trabalhou POUCOS dias no
período — por exemplo, alguém de férias quase o mês inteiro, com só 2 dias
realmente trabalhados. Sem teto, o fator seria 22/2 = 11x, e um punhado de
visitas rápidas nesses 2 dias virava um mês inteiro péssimo (comprovado:
6 visitas rápidas — 1,8 ponto normal — viravam quase 20 pontos perdidos).
Com o teto em 2x, o mesmo caso vira só 3,6 pontos — ainda descontado, mas
proporcional à amostra pequena, sem extrapolar demais.

**Contagem de ausências por ID, não por nome:** a pontuação contava
ausências agrupando pelo NOME do agente — se dois agentes tivessem o
mesmo nome (comum em equipes grandes), as ausências de um contaminavam a
pontuação do outro. Corrigido pra agrupar por ID (que é único de verdade).
Testei com dois agentes chamados "Maria Silva" (IDs diferentes, uma sem
nenhuma falta e outra com falta frequente) e confirmei: pontuações
completamente independentes agora. O mesmo bug existia na lista "Top 15
com mais ausências" do dashboard — corrigido junto (a planilha agora tem
uma coluna "ID Agente" na aba Ausências, só pra isso).

**Decisões antigas que ficaram desatualizadas por uma correção:** se você
já tinha CONFIRMADO uma detecção (férias ou paralisação) antes de alguma
dessas correções, e o padrão que gerou ela não é mais detectado do jeito
certo, o sistema não desfaz a decisão sozinho (pode ser arriscado) — em
vez disso, marca com `"revisao_necessaria": true` e um aviso explicando o
motivo. No `deteccao_editor.html`, esses itens aparecem destacados no topo
da lista, com um botão "🗑️ Remover (não era válido)". A exclusão continua
valendo até você decidir algo — só fica sinalizada pra você conferir.

## Como funciona dali pra frente

**Automático:** o workflow roda sozinho segunda, terça e quinta às 3h (horário de
Ponta Porã) — coleta no e-Visita, gera o `index.html` e publica. Você não
precisa fazer nada.

Pra mudar o horário, edite a linha `cron` no topo de
`.github/workflows/update-dashboard.yml`. O formato é `minuto hora * * dias`
em **UTC** (Ponta Porã/MS está em UTC-4, então some 4 horas ao horário local
desejado para calcular o valor em UTC).

**Manual, sob demanda:** aba **Actions → Coletar e Atualizar Dashboard → Run
workflow**. Dá pra desmarcar "Coletar dados novos" se você só quer forçar
uma nova geração do HTML sem coletar de novo (por exemplo, depois de editar
um template).

**Fallback antigo (upload manual da planilha):** ainda funciona. Suba um
`Analise_Consolidada.xlsx` novo em `data/` pela interface do GitHub — isso
dispara o workflow, mas **sem** rodar a coleta automática (só regenera o
dashboard a partir do arquivo que você subiu).

Você pode acompanhar qualquer execução na aba **Actions** — uma bolinha
amarela girando indica que está processando; verde é sucesso; vermelho é
erro (clique no workflow pra ver o log; erros de coleta geralmente são login
que falhou ou o e-Visita fora do ar; erros de geração geralmente são aba ou
coluna da planilha renomeada).

## Detecção automática de possíveis férias e paralisação coletiva

**Cada agente é avaliado a partir da PRÓPRIA primeira visita, nunca da
data mínima do sistema inteiro.** Sem isso, um agente (ou equipe inteira)
que começou a trabalhar depois do início do período rastreado — ex.: uma
atividade sazonal — teria todo o tempo ANTES de existir contado como
"dias sem lançamento", gerando férias/paralisações falsas logo no início
do período de cada um (e, como vários agentes da mesma equipe têm essa
mesma falsa ausência se sobrepondo desde o mesmo dia, isso também inflava
paralisações coletivas em bolhas de meses de duração, juntando gente que
não tinha nada a ver uma com a outra). Testei com um cenário reproduzindo
exatamente esse padrão (equipe inteira começando 8 semanas depois do
início do período) e confirmei: zero eventos falsos, tanto individuais
quanto coletivos. **Essa mesma correção também estava faltando na aba
"Ausências" em si** (não só na detecção de férias) — corrigido junto.

**Agrupamento de paralisação coletiva por interseção real, não corrente:**
o agrupamento antigo juntava um agente ao grupo se ele se sobrepusesse com
QUALQUER outro membro já no grupo — isso forma uma CORRENTE (as férias de
A tocam nas de B, as de B tocam nas de C, e o grupo vira uma bolha de
meses mesmo que A e C nunca tivessem ficado ausentes ao mesmo tempo).
Corrigido pra exigir que o grupo inteiro tenha uma interseção real (pelo
menos um dia em comum entre TODOS os membros) — testei com o cenário
exato desse bug (A toca B, B toca C, A nunca toca C) e confirmei: A agora
aparece como férias individual separada, e só B+C (que realmente se
sobrepuseram) formam um evento coletivo pequeno e preciso.

**Exclusão individual completa, mesmo dentro de um grupo coletivo:** a
correção acima trouxe um efeito colateral que também corrigi — quando 2+
agentes formam um grupo, o período do EVENTO (pra rotular/exibir) é a
interseção (o dia em comum), mas cada agente é excluído dos cálculos pelo
período INDIVIDUAL completo dele, não só pela interseção. Sem isso, um
agente cuja férias de 20 dias só tocasse a de outro por 2 dias teria só
esses 2 dias excluídos — o resto ficaria contando como ausência normal por
engano (era a causa da maioria das férias automáticas não aparecerem
direito). Testei com esse cenário exato (agente com férias de 19 dias,
outro de 11 dias, se tocando só em 5) e confirmei: cada um agora tem o
período completo excluído.

**Equipes sem detecção automática:** cadastre em
`EQUIPES_SEM_DETECCAO_AUTOMATICA`, em `scripts/coletar_evisita.py`, o nome
de equipes que não devem passar pela detecção — por padrão, "Equipe
Bloqueio" (time pequeno, dinâmica diferente do resto, com férias
cadastradas na mão em `data/cronograma_ausencias.json`).
```python
EQUIPES_SEM_DETECCAO_AUTOMATICA = ["Equipe Bloqueio"]
```

Além do cronograma manual (acima), o sistema também **detecta sozinho**
quando um agente fica muitos dias úteis seguidos sem lançar nenhuma visita
— sem precisar que ninguém cadastre nada primeiro.

Como funciona, por agente, contando só dias úteis (pula sábado, domingo e
os feriados cadastrados em `FERIADOS_NACIONAIS`, em `scripts/coletar_evisita.py`):

| Dias seguidos sem lançamento | O que acontece |
|---|---|
| 1 a 4 dias | Aparece como "possível ausência" na aba Ausências de sempre — informativo, nada muda nos cálculos |
| 5 dias | Continua só monitorando — ainda nada muda |
| Mais de 5 dias (`LIMITE_DIAS_SEM_LANCAMENTO_FERIAS`) | **Confirmado automaticamente** como Férias — já sai de todos os cálculos sozinho, sem precisar de nenhuma ação |

**Antes de confirmar férias individuais automaticamente**, o sistema
verifica se **2 ou mais agentes da mesma equipe** pararam de lançar no
mesmo período — nesse caso, em vez de confirmar sozinho como férias, vira
**"possível paralisação coletiva"** (🌧️ Chuva, 📚 Capacitação, 🏢
Reunião/atividade coletiva, 📋 Outra atividade coletiva), que **fica
pendente esperando você escolher o motivo** — o sistema não adivinha
sozinho nesse caso, porque o padrão sugere outra coisa (chuva, reunião
etc.), não o agente estar de férias.

**Se não for férias/atestado de verdade** (ex.: é uma falta mesmo, não
justificada), abra **`deteccao_editor.html`**, cole o conteúdo de
`data/deteccoes_pendentes.json`, encontre o evento e clique em **"🚫 Não é
férias — marcar como falta"**. A partir daí os dias voltam a contar
normalmente (aparecem na Ausências, pesam na pontuação) — documentado como
falta, não como uma detecção esquecida. Pra desfazer, é só clicar em
"✅ Voltar a Férias" de novo.

Pra paralisação coletiva, o fluxo continua igual: escolher o motivo e
clicar em confirmar (ou "🚫 Marcar como falta", se não for nada disso).

Depois de decidir algo no editor, copie o JSON gerado e cole em
`data/deteccoes_pendentes.json` no GitHub. A decisão fica salva pra sempre
— mesmo que o período se estenda (o agente continue sem lançar), a próxima
coleta só atualiza a data final do evento, sem te pedir pra decidir de novo.

Exemplo, exatamente como pedido:
```
Agente A: 20 dias úteis no período, 5 dias de férias confirmadas
→ ranking e todos os indicadores consideram 15 dias trabalhados, não 20
```
Isso já funciona assim por como o cálculo é feito desde sempre (médias e
ranking sempre dividiram pelos dias que o agente realmente trabalhou, nunca
por um total fixo de dias do mês) — a confirmação só evita que esses dias
apareçam como "ausência não explicada" pesando na pontuação.

Na aba **Ausências**, uma tabela "🌴 Quem está de Férias, Atestado ou
Afins" mostra de forma rápida quem tem um período confirmado agora, sem
precisar abrir o Cronograma inteiro pra procurar.

## Editar cronograma — chuva, reunião, índice, ovitrampa, recuperação de casas (sem mexer em código)

**Não cadastre férias/atestado individual aqui** — o sistema já detecta e
confirma isso sozinho (ver seção acima). Use `data/cronograma_ausencias.json`
só pra coisas que o sistema NÃO tem como adivinhar sozinho: chuva, reunião,
levantamento de índice, instalação/recolha de ovitrampa, recuperação de
casas — atividades coletivas ou específicas que não são "ausência", mas
também não têm como ser inferidas automaticamente. Durante
esses períodos, o(s) agente(s) afetado(s) não contam como possível ausência
a investigar (não pesa na pontuação) e o registro aparece listado na aba
"Cronograma" do dashboard — em cima dos dias que o próprio programa já
detecta sozinho (ninguém de nenhuma equipe trabalhou naquele turno).

Cada registro tem: **quem** é afetado (um agente específico, uma equipe
inteira, ou todos os agentes — útil pra chuva ou reunião geral), o
**motivo** (sempre um destes cinco: Chuva, Folga, Atestado,
Reunião/Treinamento, Férias) e o **período** (só o dia de entrada e o dia
de saída — pra um dia só, repete a mesma data nos dois campos).

Pra editar sem tocar em código: abra **`cronograma_editor.html`** (baixe do
repositório e abra no navegador, com dois cliques — não precisa de internet
nem instalar nada). Escolha quem é afetado, o motivo (menu fixo, sem digitar
nada errado) e o período, e um botão já monta o JSON certinho pra colar em
`data/cronograma_ausencias.json` no GitHub (abrir o arquivo → ✏️ ícone de
lápis → apagar tudo → colar → Commit changes). Se o arquivo já tiver
registros antigos, cole o conteúdo dele na caixa "Carregar o que já existe"
no topo do editor antes de adicionar/remover — assim você edita a lista em
vez de começar do zero toda vez.

**Nomes de verdade no menu, em vez de só o ID:** o coletor salva
automaticamente `data/agentes_nomes.json` (ID → nome + equipe) toda vez que
roda de verdade — é a única fonte confiável dos nomes, já que eles só
existem dentro do e-Visita (o `AGENTES_POR_EQUIPE` do script só tem os IDs).
Cole o conteúdo desse arquivo na caixa "Carregar nomes dos agentes" no topo
do `cronograma_editor.html` e o menu passa a mostrar "João da Silva —
Equipe 1" em vez de "ID 116 — Equipe 1". Antes da primeira coleta rodar,
esse arquivo ainda não existe — o menu simplesmente mostra os IDs até lá.

## Planilha antiga / fora do sistema de coleta (ex.: Jan-Maio, coletada com outros parâmetros)

Se você já tem uma planilha no mesmo formato do `Analise_Consolidada.xlsx`,
mas gerada por fora do fluxo normal (outro período, outros parâmetros — por
exemplo, Janeiro a Maio/2026, rodado manualmente antes de o sistema semanal
existir), dá pra deixá-la disponível pra consulta **sem que ela entre em
nenhum cálculo** (Acumulado, filtro de semana, custos, médias — nada). Ela
fica só como uma página fixa, separada.

Passo a passo:
1. Coloque o arquivo `.xlsx` dentro de `data/historico/` (ex.:
   `data/historico/jan_a_maio_2026.xlsx`).
2. Abra `data/historico/manifest.json` no GitHub e acrescente uma entrada
   com `"origem": "manual"` — é esse campo que protege a entrada de ser
   apagada na próxima coleta automática:
   ```json
   {
     "arquivo": "jan_a_maio_2026.xlsx",
     "label": "Janeiro a Maio/2026",
     "origem": "manual"
   }
   ```
3. Pronto — na próxima geração do dashboard, ela aparece no seletor de
   período ("📅 Janeiro a Maio/2026"), com sua própria página, sem afetar o
   Acumulado nem nada mais.

**Por que fica isolada automaticamente:** o Acumulado (`acumulado.html`) só
soma o que está no cache de semanas (`data/semanas/`) — nunca lê arquivos de
`data/historico/` diretamente. E a rotina que reconstrói o histórico a cada
coleta automática (`atualizar_historico_do_cache()`) **nunca toca** em
entradas marcadas `"origem": "manual"`, só nas que ela mesma gerou
(`"origem": "cache"`) — então sua planilha antiga nunca é sobrescrita nem
apagada, mesmo rodando a coleta automática infinitas vezes depois.

## Teto de duração por visita — cada visita conta no máximo 20 minutos nos cálculos

Visitas com mais de 20 minutos de duração (`TETO_DURACAO_MEDIA_MIN`, em
`scripts/coletar_evisita.py`) são comuns quando o lançamento é feito pela
web em vez de no momento real da visita — a duração fica artificialmente
longa e distorce médias, estatísticas e gráficos. Caso real que motivou
esse teto: um lançamento do agente Gabriel de **2.888 minutos** numa única
visita, que estava entrando "normalmente" nos cálculos. Pra neutralizar
isso, qualquer visita acima do teto conta como se tivesse durado
exatamente o teto (ex.: 2.888min → considerado como 20min) — visitas de
até 20min entram com o valor real, sem alteração nenhuma.

Isso afeta o **Tempo Médio/Mediana** (Resumo Geral, Por Área e Ponto
Estratégico), as **Horas Trab/Dia**, **Horas Trab/Semana** e o **Custo
Hora Útil**, e qualquer gráfico que use esses números — todos calculados a
partir do valor CAPADO. O lançamento original nunca é alterado nem
excluído: a aba/alerta "Acima de 20min (auditoria)" continua mostrando a
duração REAL de cada visita acima do teto (é justamente a lista pra
auditar um lançamento manipulado, então precisa do valor de verdade, não
do capado). Já "Visitas Suspeitas" é um alerta diferente (sequência de
lançamentos muito próximos no tempo) e não usa esse teto.

Esse teto é recalculado a cada consolidação a partir da duração bruta
guardada no cache (`data/semanas/`) — se o valor de `TETO_DURACAO_MEDIA_MIN`
mudar de novo no futuro, basta rodar a coleta/geração de novo (sem precisar
recoletar nada do e-Visita) que TODO o histórico já cacheado é
recalculado com o teto novo.

## Importante: não mude os nomes das abas/colunas da planilha

O script (`scripts/xlsx_scan.py` + `scripts/generate_dashboard.py`) espera
estas abas:

- `Resumo Geral` — Agente, Equipe, Classificação, Imóveis Abertos/Fechados/Recusados, Total Geral, % Pendência, horas, custo, etc.
- `Por Área` — mesma lógica de Abertos/Fechados/Recusados, por área/agente.
- `Visitas Duplicadas`, `Visitas Suspeitas`, `Visitas Negativas`, `Fora do Expediente`, `Visitas no Almoço` — alertas (agora com coluna `Região`).
- `Ausências`
- `Cronograma` — período, motivo (Chuva/Folga/Atestado/Reunião-Treinamento/Férias), quem foi afetado. Aparece como aviso no topo da Visão Geral.
- `Recusados e Pendências` — tem **duas tabelas na mesma aba** (resumo de pendência por agente + detalhe dos imóveis recusados). Vira a aba **Pendências** do dashboard.
- `Ponto Estratégico` — acompanhamento semanal de um agente/local especial. Vira a aba **Ponto Estratégico** do dashboard. Linhas com "Selecione" no campo Agente (placeholder de dropdown vazio) são ignoradas automaticamente.

Como a coleta agora é automática e gera a planilha com a mesma lógica de
sempre (`scripts/coletar_evisita.py` é o mesmo script de antes, só ajustado
pra rodar sem interface gráfica), isso não deve mudar no dia a dia. Vale só
se você decidir editar `coletar_evisita.py` e sem querer renomear uma coluna
ou aba de saída — a geração quebra com um erro do tipo
`KeyError: 'Nome da Coluna'`.

## Testar localmente antes de subir (opcional)

Se tiver Python instalado:

```bash
pip install -r requirements.txt

# Coleta (precisa das credenciais como variável de ambiente):
export EVISITA_CPF="seu-cpf"
export EVISITA_SENHA="sua-senha"
python scripts/coletar_evisita.py --saida data

# Geração do HTML:
python scripts/generate_dashboard.py
```

Isso gera o `index.html` na raiz do projeto — só abrir no navegador para
conferir antes de enviar para o GitHub.

## Via linha de comando (alternativa ao passo 2)

```bash
git init
git add .
git commit -m "Primeira versão"
git branch -M main
git remote add origin https://github.com/SEU-USUARIO/dashboard-visitas.git
git push -u origin main
```

---

## Ajustes manuais de pontuação (`data/ajustes_ranking.json`)

Para dar ou tirar pontos de um agente com justificativa (aparece no Extrato e na
coluna de detalhes como "Ajuste manual: motivo"), adicione itens à lista:

```json
[
  {"id_agente": 116, "data": "15/10/2026", "pontos": 2.0, "motivo": "Ação extra autorizada pela coordenação"}
]
```

- `pontos` positivo soma, negativo desconta (sem fator de normalização).
- O ajuste só vale no mês cuja janela contém a `data`.
- Itens inválidos são ignorados com aviso no log; `validar_json.py` também aponta.
- **Só afeta meses regenerados** (Outubro/Novembro). Meses arquivados não são recalculados.

## Sessões 8 e 9 (31/10/2026) — coordenação e agentes

**Pré-requisito:** Cloudflare configurada (seção acima). Sem ela, `validar.html` avisa e o formulário de contestação não aparece.
Copie para o repositório: `.github/`, `partes/`, `assets/` (agora com `agentes_dados.js` e ícones), `worker/` (cole o novo `index.js` no Cloudflare e **publique**), e os arquivos novos `agente.html`, `contestacao.html`, `manifest.webmanifest`, `sw.js`, `data/meses_arquivados.json`, `data/metas_equipes.json`.

### `validar.html` — abas novas
- **Ajuste de pontos**: agente, dia, pontos (até ±20), motivo curto. Pede confirmação com resumo; aparece no Extrato como "Ajuste manual: motivo" (**público**: o sistema recusa motivo com palavras de saúde/atestado). Lista os ajustes lançados, cada um com **Desfazer**. Mês fechado não aceita ajuste.
- **Equipe parada**: chuva, reunião/treinamento/capacitação, levantamento de índice, ovitrampa, recuperação de casas; uma equipe ou todas; datas (até 30 dias); dia inteiro, manhã ou tarde. Grava no cronograma (substitui o `cronograma_editor.html`). Cada parada tem **Remover**.
- **Fechar mês**: lista de conferência automática (semanas coletadas e mês terminado; próximo mês e ciclo cadastrados; nenhuma férias esperando validação; nenhum dia sem classificação; cópia de segurança automática). O botão só liga com tudo ✅. Fechar congela o mês e as semanas dele. **Reabrir** exige digitar o nome do mês e faz cópia antes (em `data/historico/backup_congelado/`).
- **Contestações**: pedidos dos agentes; **Aceitar** (com pontos opcionais, que viram um ajuste) ou **Recusar**, sempre com resposta curta.
- **?**: ajuda com o que cada botão faz.
- **Aplicativo no celular**: "Adicionar à tela inicial" instala como "Sentinela — validação". O PIN só vale enquanto o app está aberto; o service worker guarda só a página e nunca o que vem do Worker.

### Para os agentes
- **`agente.html?id=…` / `?nome=…`**: nota por mês e por semana, dias trabalhados × esperados, o que mais tirou e deu pontos. Só dados públicos, sem saúde; respeita o modo apresentação. Há link 📈 no Ranking e no Extrato.
- **Extratos em lote** (aba Ranking): "Imprimir da equipe" e "Imprimir de todos do mês" — uma folha por agente, sem página em branco; use "Salvar como PDF" na impressão.
- **Contestação**: no Extrato (só mês aberto), "Algo errado? Peça uma revisão" → critério + uma frase → o agente recebe um **código** e consulta a resposta em `contestacao.html`. Tudo fica só no Worker (nada em arquivo público); limites: 4 pedidos/hora por rede, 3 por dia por agente, campo-isca anti-robô, sem links no texto.
- **Comparativo semanal por equipe** (aba Equipes do mês): nota média de cada equipe por semana; semana parcial em tom mais claro.
- **Metas por equipe** (opcional): `data/metas_equipes.json`, ex.: `{"Equipe 1": 25}`. Quem não aparece usa a meta padrão (20). Aparece em "Como o ranking é calculado". Hoje o arquivo está vazio.

### Testes
`python scripts/test_sessao8.py`, `python scripts/test_sessao9.py` e `node worker/test_worker.mjs` rodam no workflow. Os de navegador (`scripts/test_sessao8_navegador.py`, `scripts/test_sessao9_navegador.py`) exigem Playwright e rodam só no computador de desenvolvimento.

## Identidade visual "painel de monitoramento" (31/10/2026)

Todas as páginas (dashboard, histórico, semanas, acumulado, `validar.html`, `agente.html`, `contestacao.html`, transparência) usam o mesmo visual: fundo azul-marinho profundo com as cores do logo da prefeitura (amarelo, vermelho e verde-água) e a faixa do topo que repete o sublinhado do logo. Números e títulos em **Barlow Semi Condensed**, texto em **IBM Plex Sans** — as fontes ficam em `assets/fonts/` (sem internet externa; licença OFL) e são carregadas por `assets/fontes.css`.
- **Impressão continua em papel branco** (relatórios, extratos e lote).
- No celular o cabeçalho deixa de ficar fixo no topo e as abas rolam para o lado.
- Para ajustar cores: no dashboard, `scripts/style.css` (bloco `:root`); nas outras páginas, o bloco `:root` no começo de cada arquivo.
- Copie para o repositório também: `assets/fonts/`, `assets/fontes.css` e os ícones novos `assets/icone-*.png`.

## Meses congelados (Maio a Setembro/2026)

Os meses fechados ficam em `data/meses_arquivados.json` (lido por `scripts/meses_arquivados.py`;
o botão **Fechar mês** da página `validar.html` grava ali) e marcados
`origem: "manual"` no `manifest.json`: o pipeline **não** recalcula nem reescreve
essas planilhas, então os números publicados não mudam.

- Observação: recalcular Setembro com as regras atuais (paradas coletivas abatidas
  dos dias esperados) mudaria a pontuação de ~37 dos 42 agentes (até 17 pts).
- Para reabrir um mês fechado pela página: aba **Fechar mês → Reabrir** (digite o nome do mês;
  cópia de segurança automática). Maio–Setembro seguem congelados; só o desenvolvedor reabre.
- **Depois da semana 330:** feche Outubro pela aba **Fechar mês** (ou `python scripts/fechar_mes.py fechar "Outubro/2026"`).

---

## Lançar atestado pelo GitHub (formulário pré-preenchido)

**Jeito mais fácil:** abra `atestado.html` no site publicado
(`https://SEU-USUARIO.github.io/dashboard-visitas/atestado.html`), escolha o agente e as
datas (já vêm com hoje) e clique em **Abrir formulário já preenchido**. O GitHub abre com
tudo preenchido; basta clicar em *Submit new issue*.

Também dá para ir em **Issues → New issue → Lançar atestado**.

O que acontece depois (`.github/workflows/lancar-atestado.yml`):
1. Confere agente e datas. Se algo estiver errado, comenta na issue e **não grava nada**.
2. Acrescenta `{"alvo":"agente","id_agente":N,"motivo":"Atestado",...}` em
   `data/cronograma_ausencias.json` (repetir o mesmo pedido não duplica).
3. Faz o commit, dispara a atualização do site (sem recoletar no e-Visita), comenta e fecha a issue.

Regras e cuidados:
- Só dono, membros e colaboradores do repositório conseguem lançar (outras issues são ignoradas).
- O formulário **não pede CID nem diagnóstico**; só ficam datas e o rótulo "Atestado".
  Se o repositório for público, o nome do agente fica visível na issue — prefira repositório privado.
- Atestado lançado tira os dias de "Turno sem lançamento" e dos dias esperados: **não desconta ponto**.
- Entrou ou saiu agente? Rode `python scripts/gerar_form_atestado.py` e envie o
  `.github/ISSUE_TEMPLATE/atestado.yml` atualizado.
- Para desfazer um lançamento errado, remova a linha em `data/cronograma_ausencias.json`
  (ou use o `cronograma_editor.html`).

---

## Regras de ausência e férias (atualização de Outubro/2026)

- **Dia inteiro sem lançamento** (os 2 turnos vazios) perde 1,0 ponto a mais, além dos 2 turnos
  (−1,5 cada): o dia custa −4,0. Vale só do método novo (Outubro/2026 em diante); meses antigos e o
  Acumulado antigo não mudam. Constante: `PONTOS_PERDA_DIA_INTEIRO_SEM_LANCAMENTO`.
- **Feriado nacional não conta como ausência** (antes contava como 2 turnos sem lançamento).
- **Agente que já trabalhava antes do mês** é avaliado desde o 1º dia do mês, mesmo que a primeira visita
  dele no mês tenha sido tarde (antes esses dias eram ignorados). Só quem de fato começou depois do
  início do mês é poupado. Equipes sob demanda (`EQUIPES_SEM_DETECCAO_AUTOMATICA`, ex.: Bloqueio) seguem
  a regra antiga e não recebem o desconto extra de dia inteiro.
- **Férias têm no máximo 30 dias corridos.** Na detecção automática só o 1º pedaço (até 30 dias) vira férias;
  o que passar disso entra como **falta** (`status: "falta"`).
- **Agentes sem abono automático:** `data/agentes_sem_abono.json` →
  `{"2951": {"desde": "01/10/2026", "motivo": "texto sem dado de saúde"}}`. A detecção desses agentes vira
  **falta** a partir da data; só vira abono quando a coordenação lançar à mão.

### Turno sem Lançamento × Possível Evento Externo/Interno (regra de 02/11/2026)

- **Padrão:** toda ausência sem lançamento é **Turno sem Lançamento** e desconta pontos (−1,5 por turno).
- **Antes de descontar** o sistema confere o cronograma: chuva, treinamento (`Reunião/Treinamento`), atestado
  (`Ausência justificada`), férias, folga e reunião. Turno coberto por qualquer um deles não desconta.
- **Evento coletivo, classificado sozinho como "Possível Evento Externo ou Interno" (sem desconto):**
  1. `LIMITE_AGENTES_EVENTO_TURNO` (20) ou mais agentes, de qualquer equipe, sem lançamento no mesmo dia e turno;
  2. todos os agentes ativos de uma equipe (2 ou mais) sem lançamento no turno. "Ativo" = não coberto pelo
     cronograma naquele turno (quem está de férias não conta).
  Equipes sob demanda (Bloqueio) ficam fora das duas regras. A aba Ausências mostra a classificação no começo
  da Observação. Se o evento for de fato falta, lance um ajuste em `data/ajustes_ranking.json`.
- **Detecção automática** (`data/deteccoes_pendentes.json`): detecções NOVAS de férias e de paralisação coletiva
  entram como `pendente` e continuam contando como turno sem lançamento até a coordenação confirmar (antes,
  entravam já confirmadas). Decisões antigas não mudam. Voltar ao modo antigo:
  `CONFIRMAR_FERIAS_AUTOMATICAMENTE` / `CONFIRMAR_CHUVA_AUTOMATICAMENTE = True`.
- **Desde quando vale:** `EVENTO_COLETIVO_A_PARTIR_DE = date(2026, 10, 4)` — Outubro/2026 em diante (a semana de 04/10 abre o mês); os meses fechados
  (Maio a Setembro) não mudam. `None` aplica a todos os períodos.
- **Painel de custos:** os dias esperados do custo real já descontam férias, atestado, folga e paralisações.
- Testes: `python3 scripts/test_turno_evento.py`.

### Lançar, ajustar ou cancelar férias (formulário no GitHub)

Em `atestado.html` escolha **Férias — definir ou ajustar** ou **Férias — cancelar** (ou vá em
Issues → New issue → "Férias (definir, ajustar ou cancelar)"). O workflow `lancar-atestado.yml`:

- **Definir/ajustar:** o período informado vira o período final das férias (máx. 30 dias). Se já havia férias
  detectadas tocando o período, são ajustadas; senão cria uma nova. O ajuste fica protegido
  (`ajustado_manual`): a detecção automática não estica nem encolhe mais.
- **Cancelar:** os dias informados viram **falta**; o que sobrar fora do período continua como férias.
- O tipo é reconhecido pelo próprio formulário (não depende de labels existirem no repositório).
- Entrada de férias ajustada à mão sempre vence a detecção; dias fora dela, sem trabalho, contam como falta.
- Entrou/saiu agente? Rode `python scripts/gerar_form_atestado.py` e envie os dois `.yml` de `.github/ISSUE_TEMPLATE/`.

### Cuidado ao copiar para o GitHub

`.github` é pasta oculta. Os arquivos devem ficar exatamente em `.github/workflows/*.yml` e
`.github/ISSUE_TEMPLATE/*.yml`. Cópias em `.github/workflows/workflows/` ou
`.github/workflows/ISSUE_TEMPLATE/` **não funcionam** (apague).

## Validação pela página (`validar.html`) — para a coordenação, sem mexer no GitHub

Página com botões grandes: **Férias** (✅ está certo / ✏️ ajustar datas / ❌ não é férias),
**Dias sem trabalho** (sugestões: Ausência justificada, Folga, Meia folga ou Falta),
**Novo lançamento** e **Histórico**. Cada pessoa entra com o **próprio PIN**.

Como funciona: `validar.html` → Cloudflare Worker (confere o PIN, grava o histórico) →
GitHub (`repository_dispatch`) → `aplicar-decisao.yml` → commit → site atualizado em poucos minutos.
O Worker é gratuito (plano Free) e guarda os dados sensíveis (lista de sugestões, faltas, histórico);
o repositório público só recebe o rótulo neutro "Ausência justificada" — **nunca digite CID/diagnóstico**.

### Configurar (uma vez, ~20 min)
1. Crie conta grátis em cloudflare.com → **Workers & Pages → Create → Worker**. Cole o conteúdo de `worker/index.js`.
2. **KV**: Workers → KV → crie um namespace e ligue ao Worker com o nome `SENTINELA_KV` (Settings → Bindings).
3. Em **Settings → Variables and Secrets** do Worker, crie:
   - `PINS` = `{"4821":"Maria","7305":"João"}` (PIN → nome de quem usa; um por pessoa)
   - `GH_TOKEN` = token do GitHub (Settings → Developer settings → Fine-grained token, só este repositório, permissão **Contents: Read and write** e **Actions: Read and write**)
   - `GH_REPO` = `SEU-USUARIO/NOME-DO-REPO`
   - `PUBLISH_TOKEN` = uma senha longa inventada por você
   - `ALLOWED_ORIGIN` = `https://SEU-USUARIO.github.io`
4. No GitHub: **Settings → Secrets and variables → Actions** → crie `SENTINELA_WORKER_URL` (endereço do Worker, ex.: `https://sentinela.xxx.workers.dev`) e `SENTINELA_PUBLISH_TOKEN` (o mesmo valor de `PUBLISH_TOKEN`).
5. Edite `data/config_worker.json` e coloque `{"url": "https://sentinela.xxx.workers.dev"}`.
6. Rode o workflow *Atualizar dashboard* uma vez (envia as primeiras sugestões) e abra `validar.html`.

Segurança: PIN e token ficam só como segredos do Worker; 8 PINs errados bloqueiam o IP por 15 min;
limite de 40 pedidos/min. Para trocar/remover alguém, edite `PINS`.
Testes: `node worker/test_worker.mjs` e `python scripts/test_decisao.py` (rodam no workflow).
Limites: meia folga tira só o turno sem lançamento e conta 0,5 dia esperado (não perdoa atraso/“dia sem meta” desse turno).

## Novidades de 29–30/10/2026

**Visões semanais (mês vigente).** Outubro e Novembro/2026 (e os próximos meses) ganham uma barra
`Semana 1 | Semana 2 | … | Mês completo` no topo da página do mês. Cada semana é uma página completa, com as
**mesmas regras** do ranking aplicadas só àqueles dias (veja a aba Equipes para o desempenho por equipe).
A semana em andamento aparece como **parcial**. Quando o mês é arquivado (`MESES_ARQUIVADOS`), as semanas dele
congelam junto na execução seguinte. Meses anteriores a Outubro não ganham semanais
(`PRIMEIRA_SEMANA_VISAO_SEMANAL`). Teste: `python scripts/test_semanal.py`.

**Presença mínima (nota proporcional).** Presença = dias trabalhados ÷ dias esperados (os esperados já descontam férias,
folga e ausência justificada). Abaixo de 50% (`PRESENCA_MINIMA_FRACAO`) a nota é multiplicada por presença ÷ 50%.
Só vale de Outubro/2026 em diante e não vale para Equipe Bloqueio / Ponto Estratégico. Aparece como linha no Extrato.

**Rótulo neutro.** O repositório é público: "Atestado" passa a ser sempre "Ausência justificada" e a detecção
automática grava só "Férias". Entradas antigas são convertidas na leitura (`_motivo_neutro`).

**Dia em andamento.** O dia de hoje nunca gera ausência, e quem ainda não lançou nada hoje não "deve" o dia de hoje nos dias esperados.

**Setembro/2026 reaberto e arquivado de novo (29/10).** Foi refeito com as regras atuais (dias esperados corrigidos,
feriado de 07/09). A versão anterior está em `data/historico/backup_congelado/`. Julho e Agosto continuam congelados.

**Transparência e peso das páginas.** `transparencia.html` explica o cálculo e é gerada das constantes do código
(`python scripts/gerar_transparencia.py`); tem link no cabeçalho e no Extrato. As páginas ficaram bem mais leves:
o Chart.js é um arquivo compartilhado (`assets/chart.umd.js`) e as tabelas de alertas e os extratos ficam em
`partes/` e carregam só quando você abre a aba/extrato (funciona também com o arquivo aberto do disco).
Ao subir arquivos à mão, suba também as pastas `partes/` e `assets/`.

**Atualização de 30/10/2026 — meses antigos recalculados.** Maio a Setembro foram reabertos, recalculados com as regras novas
(sem fator de normalização, feriados em todos os meses, dia inteiro sem lançamento, presença mínima, sem abono) e arquivados de novo.
As planilhas anteriores estão em `data/historico/backup_congelado/`. O **Acumulado** continua no método antigo (período longo).
`data/agentes_sem_abono.json` agora inclui Alana (2951) e Teonilda (3035) desde 01/01/2026: férias viram falta; só ausência
justificada lançada pela coordenação abona. Para tirar alguém da lista, apague a entrada.

**Acumulado = só a média.** Na página Acumulado a aba Ranking passa a ser "Ranking (média dos meses)": a média simples das notas
mensais de cada agente (com a tendência e a evolução). O ranking detalhado e o Extrato ficam na página de cada mês.

## Blindagem da coleta (30/10/2026)
- **Timeouts:** toda chamada HTTP usa (conexão 15 s, leitura 60 s); o Selenium tem tempo máximo de carga de página (60 s) e de espera por elemento (30 s).
- **Novas tentativas:** espera crescente com jitter (1 s, 2 s, 4 s… até 30 s) nas páginas, no login (3 tentativas) e na verificação de saúde.
- **e-Visita fora do ar:** antes de coletar, o coletor testa a página de login. Se não responder, **pula a coleta e publica com os dados que já existem**; as semanas são tentadas de novo na próxima execução (aviso no resumo da execução).
- **Uma semana com problema não derruba as outras:** o erro vai para o resumo (GITHUB_STEP_SUMMARY) e a coleta segue. Sessão caída = abre sessão nova e refaz a semana uma vez. Duas semanas seguidas falhando por inteiro = para (site instável) e deixa o resto para a próxima execução.
- **Tempo:** o coletor não começa semana nova depois de 170 min; o passo tem teto de 240 min e o job de 330 min. Se estourar, o site é publicado com o que já foi salvo.
- **Cache nunca piora:** um agente que volta sem nenhuma visita, ou com menos da metade do que já estava salvo, mantém os dados antigos.
- Versões das ações do GitHub fixadas (checkout 4.2.2, setup-python 5.3.0, upload-pages-artifact 3.0.1, deploy-pages 4.0.5) e do `requirements.txt` já eram fixas.
- Teste: `python scripts/test_coleta.py`.

## Sessão 10 — visual claro, tela de reunião e exportação (01/11/2026)

- **Tema claro é o padrão.** Em todas as páginas há o botão "Tema escuro / Tema claro" no cabeçalho; a escolha fica salva só no navegador de quem clicou (`assets/tema.js`). A impressão e o PDF saem sempre em fundo branco.
- **Ícones próprios** (`scripts/icones.py`) no lugar dos emojis de enfeite. Os emojis de situação (🟢🟡🟠🔴 ✅ ❌ ⛔) continuam, porque fazem parte da informação.
- **Gráfico por equipe, semana a semana** com as cores do logo e o valor escrito em cada barra.
- **Animação de entrada nos números** (contagem até o valor, 0,6 s). Caixa "Animações" no cabeçalho desliga; respeita a opção "reduzir movimento" do aparelho.
- **Baixar o ranking**: na aba Ranking, botão "Planilha (CSV, abre no Excel)" (sem nomes se o Modo apresentação estiver ligado) e link para o .xlsx completo do período.
- **Tela de reunião** (`projecao.html`, botão "Tela de reunião" no cabeçalho): alterna sozinha entre resumo do mês, top 10, maior evolução, equipes semana a semana e média por mês. Teclas: espaço pausa, ← → trocam, F tela cheia, N mostra/oculta nomes. Nomes começam ocultos.
- Teste novo: `python3 scripts/test_visual_navegador.py` (só na máquina de desenvolvimento).
- **Arquivos novos para copiar ao repositório:** `assets/tema.js`, `projecao.html`, `scripts/icones.py` (e, como sempre, as pastas `.github`, `partes` e `assets`).

## Sessão 11 — comparar meses, cartão do agente e linha do tempo (01/11/2026)

- **Comparar mês com mês** (`comparar.html`, botão "Comparar meses" no cabeçalho): escolha dois meses e veja a média geral, quem subiu, quem caiu e quem ficou estável (até 0,5 ponto), agrupado por equipe, com quantas posições cada um ganhou ou perdeu. Abas "Quem subiu / Quem caiu". Avisa quando um dos meses ainda está aberto.
- **Cartão do agente em estilo carta de jogador (com o logo novo em tom sobre tom, `assets/logo_carta.png`)** (topo de `agente.html`): nota do mês em destaque, equipe, posição e seis medidas de 0 a 99 (ASS assiduidade, TUR turnos cobertos, CON consistência, EVO evolução, POS posição, MED média dos meses). Nível pela nota: ouro ≥ 90, prata ≥ 75, bronze abaixo; "destaque" para top 3 com nota ≥ 90. Sem foto (usa as iniciais) e sem marcas de terceiros. No Modo apresentação nome e iniciais ficam borrados. A explicação das medidas aparece ao lado do cartão.
- **Linha do tempo do agente** (`agente.html`): mês a mês (e semana a semana dentro do mês aberto) com um quadradinho por dia esperado — cheio = trabalhado, vazio = sem lançamento — e etiquetas das **paradas da equipe** (chuva, reunião, mutirão…), que não contam como falta. Fica de fora, de propósito, qualquer lista dia a dia de ausência por agente: isso é sensível e o repositório é público.
- `scripts/gerar_agentes.py` passou a gravar `paradas` (vindas de `data/cronograma_ausencias.json`) em cada mês e semana.
- Testes novos: `scripts/test_sessao10.py` (entra no workflow) e `scripts/test_sessao10_navegador.py` (só na máquina de desenvolvimento).
- **Copiar para o repositório:** `comparar.html`, `agente.html`, `scripts/` e `.github` (e as pastas `partes` e `assets`, como sempre).

### Álbum de cartinhas (`album.html`)
Todas as cartinhas dos agentes, da maior nota para a menor, com o número do lugar. Passar o mouse (ou tocar no celular) amplia a cartinha; clicar abre a evolução do agente. Filtros por mês, equipe, nível (destaque/ouro/prata/bronze) e busca por nome; respeita o Modo apresentação e o tema claro/escuro. Botão "Álbum de cartinhas" no cabeçalho do painel.
O desenho da cartinha agora vive em `assets/carta.css` e `assets/carta.js` (usados pelo álbum e pela página do agente) e o logo em `assets/logo_carta.png`.
**Copiar:** `album.html`, `agente.html`, `assets/` (carta.css, carta.js, logo_carta.png) e `scripts/`.

### Cartinha como imagem / WhatsApp (página do agente)
Dois botões ao lado da cartinha: **Baixar imagem** (PNG) e **Enviar pelo WhatsApp**. No celular abre a lista de compartilhamento já com a imagem anexada; no computador baixa a imagem e abre o WhatsApp com a frase pronta (basta anexar). No Modo apresentação a frase sai sem o nome e a imagem sai com o nome coberto por bolinhas. A imagem é desenhada no próprio navegador (`Carta.png` em `assets/carta.js`), então só funciona pelo endereço do site (não abrindo o arquivo direto do computador). Copiar: `agente.html` e `assets/carta.js`.
