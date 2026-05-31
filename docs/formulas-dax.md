# Fórmulas DAX — Medidas e Colunas

Todas as medidas abaixo assumem o modelo descrito em [`modelo-dados.md`](modelo-dados.md).

---

## Folha de Pagamento

### Total Proventos

```dax
Total Proventos =
CALCULATE(
    SUM(FatoFolha[Valor]),
    FatoFolha[TipoLancamento] = "Provento"
)
```

### Total Descontos

```dax
Total Descontos =
CALCULATE(
    SUM(FatoFolha[Valor]),
    FatoFolha[TipoLancamento] = "Desconto"
)
```

### Custo Líquido Folha

```dax
Custo Liquido Folha = [Total Proventos] + [Total Descontos]
```

> Descontos já são negativos na origem; a soma representa o custo efetivo.

### Total Encargos Patronais

```dax
Total Encargos =
CALCULATE(
    SUM(FatoFolha[Valor]),
    DimRubrica[Categoria] = "Encargo Patronal"
)
```

### Custo Total Empresa

```dax
Custo Total Empresa = [Total Proventos] + [Total Encargos]
```

### Custo Médio por Colaborador

```dax
Custo Medio Colaborador =
DIVIDE(
    [Custo Total Empresa],
    [Headcount Ativo],
    0
)
```

### Custo Médio MoM (variação)

```dax
Custo Medio MoM % =
VAR CustoAtual = [Custo Medio Colaborador]
VAR CustoAnterior =
    CALCULATE(
        [Custo Medio Colaborador],
        DATEADD(DimData[Data], -1, MONTH)
    )
RETURN
    DIVIDE(CustoAtual - CustoAnterior, CustoAnterior, 0)
```

### % Composição — Benefícios

```dax
% Beneficios Custo Total =
DIVIDE(
    CALCULATE(
        [Total Proventos],
        DimRubrica[Categoria] = "Beneficio"
    ),
    [Custo Total Empresa],
    0
)
```

### Folha YTD

```dax
Folha YTD =
TOTALYTD(
    [Custo Total Empresa],
    DimData[Data]
)
```

---

## Headcount e Turnover

### Headcount Ativo

```dax
Headcount Ativo =
CALCULATE(
    COUNTROWS(DimColaborador),
    DimColaborador[Status] = "Ativo"
)
```

### Headcount por Data (snapshot)

Conta colaboradores ativos em uma data específica do calendário:

```dax
Headcount Snapshot =
VAR DataRef = MAX(DimData[Data])
RETURN
    CALCULATE(
        COUNTROWS(DimColaborador),
        DimColaborador[DataAdmissao] <= DataRef,
        OR(
            ISBLANK(DimColaborador[DataDemissao]),
            DimColaborador[DataDemissao] > DataRef
        )
    )
```

### Admissões no Período

```dax
Admissoes =
CALCULATE(
    COUNTROWS(DimColaborador),
    NOT(ISBLANK(DimColaborador[DataAdmissao]))
)
```

### Desligamentos no Período

```dax
Desligamentos =
CALCULATE(
    COUNTROWS(DimColaborador),
    NOT(ISBLANK(DimColaborador[DataDemissao]))
)
```

### Taxa de Turnover Mensal

```dax
% Turnover Mensal =
VAR Deslig = [Desligamentos]
VAR HC medio =
    AVERAGEX(
        VALUES(DimData[Data]),
        [Headcount Snapshot]
    )
RETURN
    DIVIDE(Deslig, HC medio, 0)
```

### Turnover YTD

```dax
Turnover YTD =
TOTALYTD([% Turnover Mensal], DimData[Data])
```

### Turnover vs Meta

```dax
Turnover vs Meta =
[% Turnover Mensal] - SELECTEDVALUE(ParametrosKPI[Meta Turnover Mensal], 0.02)
```

---

## Gestão de Ponto

### Total Horas Trabalhadas

```dax
Total Horas Trabalhadas = SUM(FatoPonto[HorasTrabalhadas])
```

### Total Horas Contratuais

```dax
Total Horas Contratuais = SUM(FatoPonto[HorasContratuais])
```

### Total Horas Extras

```dax
Total Horas Extras = SUM(FatoPonto[HorasExtras])
```

### % Horas Extras

```dax
% Horas Extras =
DIVIDE(
    [Total Horas Extras],
    [Total Horas Contratuais],
    0
)
```

### Dias com Atraso

```dax
Dias com Atraso =
CALCULATE(
    COUNTROWS(FatoPonto),
    FatoPonto[MinutosAtraso] > 0
)
```

### Taxa de Atraso

```dax
% Taxa Atraso =
VAR DiasUteisRegistrados =
    CALCULATE(
        COUNTROWS(FatoPonto),
        FatoPonto[Falta] = 0
    )
RETURN
    DIVIDE([Dias com Atraso], DiasUteisRegistrados, 0)
```

### Taxa de Absenteísmo

```dax
% Absenteismo =
DIVIDE(
    CALCULATE(COUNTROWS(FatoPonto), FatoPonto[Falta] = 1),
    COUNTROWS(FatoPonto),
    0
)
```

### Saldo Banco de Horas

```dax
Saldo Banco Horas =
SUMX(
    VALUES(DimColaborador[ColaboradorID]),
    VAR HorasExtras = CALCULATE(SUM(FatoPonto[HorasExtras]))
    VAR HorasCompensadas =
        CALCULATE(
            SUM(FatoPonto[HorasCompensadas]),
            ALL(DimData)
        )
    RETURN HorasExtras - HorasCompensadas
)
```

### Média Minutos Atraso por Colaborador

```dax
Media Atraso Minutos =
AVERAGEX(
    VALUES(DimColaborador[ColaboradorID]),
    CALCULATE(SUM(FatoPonto[MinutosAtraso]))
)
```

### Horas Extras vs Meta

```dax
Horas Extras vs Meta =
[% Horas Extras] - SELECTEDVALUE(ParametrosKPI[Meta Horas Extras %], 0.05)
```

---

## Integração Folha × Ponto

### Horas Extras Pagas na Folha

Converte valor monetário em horas usando valor/hora médio do colaborador:

```dax
Horas Extras Pagas Folha =
VAR ValorHE =
    CALCULATE(
        SUM(FatoFolha[Valor]),
        DimRubrica[CodigoRubrica] = "HE50"
    )
VAR ValorHoraMedio =
    DIVIDE(
        CALCULATE([Total Proventos], DimRubrica[Categoria] = "Salario Base"),
        [Total Horas Contratuais],
        0
    )
RETURN
    DIVIDE(ValorHE, ValorHoraMedio, 0)
```

### Divergência Folha vs Ponto (horas)

```dax
Divergencia HE Folha Ponto =
[Horas Extras Pagas Folha] - [Total Horas Extras]
```

### % Conformidade Folha-Ponto

```dax
% Conformidade Folha Ponto =
VAR Tolerancia = SELECTEDVALUE(ParametrosKPI[Tolerancia Divergencia Folha-Ponto (horas)], 0.25)
VAR ColaboradoresConformes =
    COUNTROWS(
        FILTER(
            ADDCOLUMNS(
                VALUES(DimColaborador[ColaboradorID]),
                "@Div", [Divergencia HE Folha Ponto]
            ),
            ABS([@Div]) <= Tolerancia
        )
    )
VAR TotalColaboradores = COUNTROWS(VALUES(DimColaborador[ColaboradorID]))
RETURN
    DIVIDE(ColaboradoresConformes, TotalColaboradores, 0)
```

---

## Time Intelligence

### Custo Folha Mês Anterior

```dax
Custo Folha Mes Anterior =
CALCULATE(
    [Custo Total Empresa],
    DATEADD(DimData[Data], -1, MONTH)
)
```

### Variação MoM Custo Folha

```dax
Variacao MoM Custo Folha =
VAR Atual = [Custo Total Empresa]
VAR Anterior = [Custo Folha Mes Anterior]
RETURN
    DIVIDE(Atual - Anterior, Anterior, 0)
```

### Headcount Same Period Last Year

```dax
Headcount SPLY =
CALCULATE(
    [Headcount Snapshot],
    SAMEPERIODLASTYEAR(DimData[Data])
)
```

### Crescimento Headcount YoY

```dax
Crescimento HC YoY =
DIVIDE(
    [Headcount Snapshot] - [Headcount SPLY],
    [Headcount SPLY],
    0
)
```

---

## KPIs compostos (cards executivos)

### Score Saúde RH

Índice composto normalizado (0–100) penalizando turnover, absenteísmo e horas extras:

```dax
Score Saude RH =
VAR ScoreTurnover =
    1 - MIN([% Turnover Mensal] / 0.05, 1)
VAR ScoreAbsenteismo =
    1 - MIN([% Absenteismo] / 0.10, 1)
VAR ScoreHE =
    1 - MIN([% Horas Extras] / 0.10, 1)
RETURN
    ROUND((ScoreTurnover + ScoreAbsenteismo + ScoreHE) / 3 * 100, 1)
```

### Semáforo Turnover

```dax
Semaforo Turnover =
SWITCH(
    TRUE(),
    [% Turnover Mensal] <= 0.015, "Verde",
    [% Turnover Mensal] <= 0.025, "Amarelo",
    "Vermelho"
)
```

---

## Medidas de formatação dinâmica

### Formato Moeda BRL

```dax
Custo Total Formatado =
FORMAT([Custo Total Empresa], "#,##0.00") & " BRL"
```

### Título dinâmico do período

```dax
Titulo Periodo =
"Competência: " &
FORMAT(MIN(DimData[Data]), "MMM/YYYY") &
IF(
    HASONEVALUE(DimData[Data]),
    "",
    " a " & FORMAT(MAX(DimData[Data]), "MMM/YYYY")
)
```

---

## Boas práticas aplicadas neste projeto

1. **Medidas base + medidas derivadas** — reuso de `[Custo Total Empresa]`, `[Headcount Ativo]`, etc.
2. **DIVIDE com fallback 0** — evita erro de divisão por zero
3. **Variáveis (VAR/RETURN)** — legibilidade e performance
4. **Filtros via CALCULATE** — sem dependência de slicers implícitos
5. **Tabelas desconectadas para metas** — `ParametrosKPI` com SELECTEDVALUE
