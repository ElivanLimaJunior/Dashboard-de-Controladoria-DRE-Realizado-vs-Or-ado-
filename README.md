# Dashboard de Controladoria — DRE (Realizado vs Orçado)

Projeto final da formação **Xperiun: Power BI Data Analyst** (8 cursos, 40h — do básico ao intermediário). Um dashboard de DRE (Demonstrativo de Resultado do Exercício) construído em aula, simulando um case real de controladoria: monitorar receita, margem de contribuição, despesas e resultado líquido de uma empresa, comparando o realizado com orçado/previsto e com histórico desde 2021.

![Dashboard DRE](./dashboard-dre.png)

## Contexto

O ponto de partida do case foi um pedido bem realista, do tipo que aparece no dia a dia de qualquer área de controladoria:

> "Gostaria de solicitar a criação de um relatório para nossa área, que nos permita monitorar de maneira eficaz e eficiente as métricas financeiras essenciais da empresa. Aquele Excel que a gente tem está travando demais, não consigo utilizar direito! O objetivo é ter um painel que forneça uma visão clara de receita, margem de contribuição, despesas e resultado líquido em relação ao que tínhamos previsto, filtrando por filial, com histórico desde 2021, para apresentar mensalmente na Reunião de Resultados da Diretoria."

A partir disso, o projeto virou um dashboard completo de análise de resultado, com modelo de dados, medidas DAX e storytelling visual pensados para responder exatamente a essas perguntas.

## Funcionalidades

- **Cartões de resumo**: receita bruta, margem de contribuição, despesas operacionais e resultado realizado do mês, cada um com sua análise vertical (% sobre a receita) e o comparativo contra o orçado.
- **Tabela de DRE completa**: estrutura em árvore (receita bruta → deduções → receita líquida → custos → margem de contribuição → despesas → resultado), com Realizado, Meta, %, Realizado YTD, Meta YTD e % YTD lado a lado.
- **Análise horizontal e vertical**: variação percentual entre períodos e peso de cada linha sobre a receita bruta.
- **Análise de despesas**: abertura detalhada por conta (nível 1 e nível 2 do plano de contas).
- **Resultado líquido do exercício**: gráfico mês a mês mostrando em quais meses a meta foi batida, qual foi o pior mês do ano e o acumulado (YTD) frente ao orçado.
- **Filtros**: Meta (Orçado ou Previsto), Ano, Mês e Filial, com histórico completo desde 2021.
- **Bridge/Waterfall**: ponte de variação entre Realizado e Meta.

## Modelo de dados

Modelo em estrela, com três fatos e três dimensões principais (mais uma tabela auxiliar de parâmetro):

**Fatos**
| Tabela | Colunas | Descrição |
|---|---|---|
| `fLancamentos` | `competencia_data`, `planoContas_id`, `valor` | Lançamentos realizados |
| `fOrcamento` | `competencia_data`, `planoContas_id`, `valor` | Valores orçados |
| `fPrevisao` | `competencia_data`, `planoContas_id`, `valor` | Valores previstos (revisão de orçamento) |

**Dimensões**
| Tabela | Descrição |
|---|---|
| `dCalendario` | Calendário completo (Ano, Mês, Trimestre, Semestre, flag de data passada, etc.), usada como tabela de datas ativa do modelo |
| `dPlanoConta` | Plano de contas (`descricaoN1`, `descricaoN2`, `tipoLancamento` — sinal para receita/despesa, `detalharN2` — controla o nível de abertura exibido) |
| `dEstruturaDRE` | Estrutura/máscara do DRE (`contaGerencial`, `index` — ordem das linhas, `subtotal` — flag de linha de totalização, `empresa` — contexto de filial usado nos totais) |
| `auxMeta` | Tabela de parâmetro para alternar entre Orçado e Previsto no filtro "Meta" |

**Relacionamentos** (todos 1:N, filtro em uma direção, a partir das dimensões):

```
dCalendario[Data]      → fLancamentos[competencia_data]
dCalendario[Data]      → fOrcamento[competencia_data]
dCalendario[Data]      → fPrevisao[competencia_data]
dPlanoConta[id]        → fLancamentos[planoContas_id]
dPlanoConta[id]        → fOrcamento[planoContas_id]
dPlanoConta[id]        → fPrevisao[planoContas_id]
dEstruturaDRE[id]      → dPlanoConta[mascaraDRE_id]
```

## Medidas DAX principais

O núcleo do modelo são três medidas "totais", que aplicam o sinal correto (receita soma, despesa subtrai) via `tipoLancamento`:

```dax
$ Total Lançamentos =
SUMX(
    fLancamentos,
    fLancamentos[valor] * RELATED(dPlanoConta[tipoLancamento])
)
```

A mesma lógica se repete para `$ Total Orçado` (sobre `fOrcamento`) e `$ Total Previsto` (sobre `fPrevisao`).

A medida `$ Realizado` é a mais importante do modelo: ela lê o contexto da estrutura do DRE (`dEstruturaDRE`) para decidir se a linha atual deve mostrar o valor da conta ou o subtotal acumulado, e evita duplicar valores quando a tabela está expandida em diferentes níveis de detalhe:

```dax
$ Realizado =
VAR vSubtotal = SELECTEDVALUE(dEstruturaDRE[subtotal])
VAR vIndex = SELECTEDVALUE(dEstruturaDRE[index])
VAR vDetalharN2 = SELECTEDVALUE(dPlanoConta[detalharN2])
VAR vTotal = [$ Total Lançamentos]
VAR vAcumulado =
    CALCULATE(
        [$ Total Lançamentos],
        ALL(dEstruturaDRE),
        VALUES(dEstruturaDRE[empresa]),
        dEstruturaDRE[index] < vIndex
    )
RETURN
SWITCH(
    TRUE(),
    vSubtotal = 1 && ISINSCOPE(dPlanoConta[descricaoN1]), BLANK(),
    vDetalharN2 = 0 && ISINSCOPE(dPlanoConta[descricaoN2]), BLANK(),
    vSubtotal = 0, vTotal,
    vSubtotal = 1, vAcumulado,
    vTotal
)
```

`$ Orçado` segue a mesma estrutura sobre `$ Total Orçado`. `$ Meta` alterna entre os dois conforme o filtro selecionado:

```dax
$ Meta =
IF(
    SELECTEDVALUE(auxMeta[auxMeta Pedido]) = 0,
    [$ Orçado],
    [$ Previsto]
)
```

O comparativo percentual usado nos cartões e na tabela:

```dax
% Meta vs Realizado =
VAR vMeta = [$ Meta]
VAR vRealizado = [$ Realizado]
VAR vDif = vRealizado - vMeta
VAR vResultado = DIVIDE(vDif, ABS(vMeta))
RETURN
IF( vRealizado, vResultado )
```

O acumulado do ano (YTD) reaproveita `$ Realizado` e `$ Meta`, filtrando pela flag `dataPassada?` para não somar meses futuros:

```dax
$ Realizado YTD =
CALCULATE(
    [$ Realizado],
    CALCULATETABLE(
        DATESYTD(dCalendario[Data]),
        dCalendario[dataPassada?] = TRUE()
    )
)
```

A análise vertical (% sobre a receita bruta) usa a receita bruta fixa como denominador:

```dax
% AV = DIVIDE([$ Realizado], [$ Realizado Receita bruta Fixa])
```

E cada bloco do DRE (receita bruta, margem de contribuição, despesas) tem sua própria medida de recorte, filtrando pelo índice da linha correspondente em `dEstruturaDRE`:

```dax
$ Realizado Receita bruta = CALCULATE([$ Realizado], dEstruturaDRE[index] = 1)
$ Realizado Margem de contribuição = CALCULATE([$ Realizado], dEstruturaDRE[index] = 5)
$ Realizado Despesas = CALCULATE(ABS([$ Realizado]), dEstruturaDRE[index] = 6)
```

O modelo conta com 41 medidas ao todo, organizadas em três pastas de exibição: **DRE** (cálculos financeiros), **Total** (agregações base) e **Storytelling** (títulos, flags de cor e textos dinâmicos dos cartões).

## Tecnologias e habilidades aplicadas

- **Power Query**: tratamento e carga dos dados de lançamentos, orçamento e previsão.
- **Modelagem de dados**: modelo em estrela com tabela de calendário dedicada.
- **DAX**: medidas de comparação (realizado vs meta), YTD, análise vertical/horizontal e lógica de subtotal dinâmico por nível do plano de contas.
- **Storytelling e visualização de dados**: cartões com flags de cor, títulos dinâmicos e narrativa guiada (melhor mês, pior mês, acumulado do ano).
- **Design de dashboards (Figma)**: layout do painel antes da construção no Power BI.
- **Power BI Serviço**: publicação para consumo mensal pela diretoria.

## Formação

Projeto desenvolvido durante a certificação **Xperiun: Power BI Data Analyst** — Power BI Básico → Intermediário, 40 horas, concluída em 20/07/2026.

## Links

- Link do projeto:
- Link do portfólio:
- Link do projeto no GitHub:
