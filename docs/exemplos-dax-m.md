# Exemplos Reais — DAX e Linguagem M (Editor Avançado)

Exemplos práticos aplicados ao contexto de **RH, Folha e Ponto**, como aparecem no **Editor Avançado** do Power Query e no **Editor DAX** do Power BI.

---

## Parte 1 — Power Query M (Editor Avançado)

No Power BI: **Transformar dados → Editor do Power Query → Exibir → Editor Avançado**.

Cada bloco abaixo é um script M completo ou trecho pronto para colar e adaptar.

---

### Exemplo 1 — Limpeza de cadastro de colaboradores (HRIS)

**Cenário:** exportação do HRIS com nomes inconsistentes, CPF sujo, status em caixa alta e datas como texto.

```powerquery
let
    // 1. Conexão com a origem (simula CSV do HRIS)
    Fonte = Csv.Document(
        File.Contents("C:\Dados\RH\colaboradores_export.csv"),
        [Delimiter = ";", Encoding = 65001, QuoteStyle = QuoteStyle.Csv]
    ),

    // 2. Promover cabeçalho e definir tipos iniciais
    ComCabecalho = Table.PromoteHeaders(Fonte, [PromoteAllScalars = true]),
    TiposIniciais = Table.TransformColumnTypes(
        ComCabecalho,
        {
            {"ColaboradorID", Int64.Type},
            {"Matricula", type text},
            {"Nome", type text},
            {"CPF", type text},
            {"DataAdmissao", type text},
            {"Status", type text},
            {"TipoContrato", type text}
        }
    ),

    // 3. LIMPEZA DE TEXTO — trim, proper, remover caracteres inválidos
    LimparTexto = Table.TransformColumns(
        TiposIniciais,
        {
            {"Nome", each Text.Trim(Text.Proper(_)), type text},
            {"Status", each Text.Proper(Text.Trim(_)), type text},
            {"TipoContrato", each Text.Upper(Text.Trim(_)), type text}
        }
    ),

    // 4. LIMPEZA DE CPF — manter só dígitos e formatar
    LimparCPF = Table.AddColumn(
        LimparTexto,
        "CPF_Limpo",
        each
            let
                SoDigitos = Text.Select([CPF], {"0".."9"}),
                ComPadding = Text.PadStart(SoDigitos, 11, "0"),
                Formatado =
                    if Text.Length(ComPadding) = 11 then
                        Text.Combine({
                            Text.Start(ComPadding, 3), ".",
                            Text.Middle(ComPadding, 3, 3), ".",
                            Text.Middle(ComPadding, 6, 3), "-",
                            Text.End(ComPadding, 2)
                        })
                    else
                        null
            in
                Formatado,
        type text
    ),

    // 5. LIMPEZA DE DATA — converter texto BR (dd/MM/yyyy) com tratamento de erro
    ConverterData = Table.AddColumn(
        LimparCPF,
        "DataAdmissao_Date",
        each
            try Date.FromText(_, "pt-BR")
            otherwise try #date(Number.From(Text.End(_, 4)), Number.From(Text.Middle(_, 3, 2)), Number.From(Text.Start(_, 2)))
            otherwise null,
        type date
    ),

    // 6. Padronizar matrícula (EMP-000000)
    PadronizarMatricula = Table.AddColumn(
        ConverterData,
        "Matricula_Padrao",
        each
            let
                Numero = Text.Select([Matricula], {"0".."9"}),
                Padded = Text.PadStart(Numero, 6, "0")
            in
                "EMP-" & Padded,
        type text
    ),

    // 7. Remover linhas inválidas e duplicatas
    FiltrarValidos = Table.SelectRows(
        PadronizarMatricula,
        each [ColaboradorID] <> null and [CPF_Limpo] <> null and [DataAdmissao_Date] <> null
    ),
    SemDuplicatas = Table.Distinct(FiltrarValidos, {"ColaboradorID"}),

    // 8. Selecionar colunas finais (descartar colunas sujas)
    ColunasFinais = Table.SelectColumns(
        SemDuplicatas,
        {"ColaboradorID", "Matricula_Padrao", "Nome", "CPF_Limpo", "DataAdmissao_Date", "Status", "TipoContrato"}
    ),

    // 9. Renomear para o modelo
    Renomear = Table.RenameColumns(
        ColunasFinais,
        {
            {"Matricula_Padrao", "Matricula"},
            {"CPF_Limpo", "CPF"},
            {"DataAdmissao_Date", "DataAdmissao"}
        }
    )
in
    Renomear
```

**Funções M usadas:** `Text.Trim`, `Text.Proper`, `Text.Upper`, `Text.Select`, `Text.PadStart`, `Date.FromText`, `try ... otherwise`, `Table.SelectRows`, `Table.Distinct`, `Table.TransformColumnTypes`.

---

### Exemplo 2 — Limpeza de folha de pagamento (ERP)

**Cenário:** valores com vírgula decimal, rubricas desconhecidas e competência como `202501` (inteiro).

```powerquery
let
    Fonte = Excel.Workbook(File.Contents("C:\Dados\Folha\eventos_folha.xlsx"), null, true),
    AbaEventos = Fonte{[Item = "Eventos", Kind = "Sheet"]}[Data],
    Cabecalho = Table.PromoteHeaders(AbaEventos, [PromoteAllScalars = true]),

    // Substituir valores nulos e strings vazias
    SubstituirNulos = Table.ReplaceValue(Cabecalho, null, 0, Replacer.ReplaceValue, {"Valor"}),
    SubstituirVazio = Table.ReplaceValue(SubstituirNulos, "", null, Replacer.ReplaceValue, {"RubricaCodigo"}),

    // Converter valor monetário BR: "1.234,56" → 1234.56
    ValorNumerico = Table.AddColumn(
        SubstituirVazio,
        "Valor_Num",
        each
            let
                Texto = Text.Replace(Text.Replace(Text.From(_[Valor]), ".", ""), ",", "."),
                Numero = try Number.From(Texto) otherwise 0
            in
                Numero,
        type number
    ),

    // Competência YYYYMM → data (1º dia do mês)
    CompetenciaDate = Table.AddColumn(
        ValorNumerico,
        "CompetenciaID",
        each
            let
                s = Text.From([Competencia]),
                ano = Number.FromText(Text.Start(s, 4)),
                mes = Number.FromText(Text.End(s, 2))
            in
                ano * 100 + mes,
        Int64.Type
    ),

    // Classificar provento/desconto por sinal e código
    TipoLancamento = Table.AddColumn(
        CompetenciaDate,
        "TipoLancamento",
        each if [Valor_Num] >= 0 then "Provento" else "Desconto",
        type text
    ),

    // Remover erros de conversão
    SemErros = Table.RemoveRowsWithErrors(CompetenciaDate, {"Valor_Num"}),

    // Filtrar competências futuras inválidas
    CompetenciaValida = Table.SelectRows(
        SemErros,
        each [CompetenciaID] <= Date.Year(DateTime.LocalNow()) * 100 + Date.Month(DateTime.LocalNow())
    )
in
    CompetenciaValida
```

**Funções M usadas:** `Table.ReplaceValue`, `Text.Replace`, `Number.From`, `Table.RemoveRowsWithErrors`, `Table.AddColumn`, `Table.SelectRows`.

---

### Exemplo 3 — Limpeza de ponto eletrônico (REP)

**Cenário:** marcações com horas em texto `"8:30"`, outliers e colaboradores desligados.

```powerquery
let
    Fonte = Sql.Database("servidor-rh", "PontoDB"),
    ConsultaBruta = Value.NativeQuery(
        Fonte,
        "SELECT ColaboradorID, DataRef, HorasTrabalhadas, HorasExtras, MinutosAtraso, Falta FROM vw_PontoDiario",
        null,
        [EnableFolding = true]
    ),

    // Função customizada: converter "H:MM" ou "HH:MM" em decimal
    fnHoraDecimal = (horaTexto as text) as nullable number =>
        let
            partes = Text.Split(Text.Trim(horaTexto), ":"),
            horas = try Number.From(partes{0}) otherwise null,
            minutos = try Number.From(partes{1}) otherwise 0,
            resultado = if horas = null then null else horas + minutos / 60
        in
            resultado,

    ConverterHoras = Table.TransformColumns(
        ConsultaBruta,
        {
            {"HorasTrabalhadas", each fnHoraDecimal(Text.From(_)), type number},
            {"HorasExtras", each fnHoraDecimal(Text.From(_)), type number}
        }
    ),

    // Data como inteiro YYYYMMDD
    DataID = Table.AddColumn(
        ConverterHoras,
        "DataID",
        each Date.Year([DataRef]) * 10000 + Date.Month([DataRef]) * 100 + Date.Day([DataRef]),
        Int64.Type
    ),

    // Flag de outlier: mais de 16h trabalhadas no dia
    FlagOutlier = Table.AddColumn(
        DataID,
        "OutlierJornada",
        each [HorasTrabalhadas] > 16,
        type logical
    ),

    // Remover outliers e registros sem colaborador
    Filtrar = Table.SelectRows(
        FlagOutlier,
        each [ColaboradorID] <> null
            and [HorasTrabalhadas] <> null
            and [OutlierJornada] = false
    ),

    RemoverColunaAux = Table.RemoveColumns(Filtrar, {"OutlierJornada", "DataRef"})
in
    RemoverColunaAux
```

**Funções M usadas:** função customizada (`fnHoraDecimal`), `Text.Split`, `Number.From`, `Table.TransformColumns`, `Table.AddColumn` com lógica condicional.

---

### Exemplo 4 — Merge (JOIN) entre folha e dimensão de rubricas

**Cenário:** enriquecer fato com descrição da rubrica — equivalente ao relacionamento no modelo, feito ainda no ETL.

```powerquery
let
    FatoFolha = FatoFolha_Limpo,
    DimRubrica = DimRubrica_Limpo,

    // Left Outer Join — mantém todos os eventos de folha
    MergeRubrica = Table.NestedJoin(
        FatoFolha,
        {"RubricaCodigo"},
        DimRubrica,
        {"CodigoRubrica"},
        "DimRubrica",
        JoinKind.LeftOuter
    ),

    ExpandirRubrica = Table.ExpandTableColumn(
        MergeRubrica,
        "DimRubrica",
        {"RubricaID", "Rubrica", "Categoria", "IncideEncargo"},
        {"RubricaID", "Rubrica", "Categoria", "IncideEncargo"}
    ),

    // Tratar rubricas não mapeadas
    CategoriaPadrao = Table.ReplaceValue(
        ExpandirRubrica,
        null,
        "Nao Classificado",
        Replacer.ReplaceValue,
        {"Categoria"}
    )
in
    CategoriaPadrao
```

**Funções M usadas:** `Table.NestedJoin`, `Table.ExpandTableColumn`, `JoinKind.LeftOuter`, `Table.ReplaceValue`.

---

### Exemplo 5 — Função reutilizável de limpeza (padrão de projeto)

Centraliza regras para aplicar em várias queries:

```powerquery
// Query: fn_LimparTexto (carregada como Connection Only)
(texto as nullable text) as nullable text =>
    let
        resultado =
            if texto = null then
                null
            else
                Text.Trim(Text.Clean(Text.Proper(texto)))
    in
        resultado
```

Uso em outra query:

```powerquery
let
    Fonte = ...,
    AplicarLimpeza = Table.TransformColumns(
        Fonte,
        {
            {"Nome", fn_LimparTexto, type text},
            {"Departamento", fn_LimparTexto, type text}
        }
    )
in
    AplicarLimpeza
```

**Funções M usadas:** `Text.Clean` (remove caracteres de controle), função parametrizada reutilizável.

---

### Exemplo 6 — Calendário gerado em M (DimData)

```powerquery
let
    DataInicio = #date(2020, 1, 1),
    DataFim = #date(2030, 12, 31),
    TotalDias = Duration.Days(DataFim - DataInicio) + 1,

    ListaDatas = List.Dates(DataInicio, TotalDias, #duration(1, 0, 0, 0)),
    TabelaDatas = Table.FromList(ListaDatas, Splitter.SplitByNothing(), {"Data"}),

    ComColunas = Table.AddColumn(TabelaDatas, "DataID", each Date.Year([Data]) * 10000 + Date.Month([Data]) * 100 + Date.Day([Data]), Int64.Type),
    ComAnoMes = Table.AddColumn(ComColunas, "AnoMes", each Text.From(Date.Year([Data])) & Text.PadStart(Text.From(Date.Month([Data])), 2, "0"), type text),
    ComMesNome = Table.AddColumn(ComAnoMes, "MesNome", each Date.ToText([Data], "MMMM", "pt-BR"), type text),
    ComTrimestre = Table.AddColumn(ComMesNome, "Trimestre", each "Q" & Text.From(Date.QuarterOfYear([Data])), type text),
    ComFimSemana = Table.AddColumn(ComTrimestre, "EhFimDeSemana", each if Date.DayOfWeek([Data], Day.Monday) >= 5 then 1 else 0, Int64.Type)
in
    ComFimSemana
```

---

## Parte 2 — DAX (Editor de Fórmulas)

No Power BI: **Modelagem → Nova medida / Nova coluna** ou clique direito na tabela.

---

### Exemplo 7 — RELATED (coluna calculada na fato)

**Uso:** trazer atributo da dimensão para a fato **sem expandir o modelo visualmente**. Só funciona no sentido **Many → One** (fato → dimensão).

```dax
// Coluna calculada em FatoFolha
Departamento =
RELATED(DimCentroCusto[Departamento])
```

```dax
// Coluna calculada em FatoPonto
Nome Colaborador =
RELATED(DimColaborador[Nome])
```

```dax
// Coluna calculada em FatoFolha — nome da rubrica
Descricao Rubrica =
RELATED(DimRubrica[Rubrica])
```

```dax
// Coluna calculada em DimColaborador — cargo via relacionamento
Nome Cargo =
RELATED(DimCargo[Cargo])
```

> **Regra:** se o filtro vier da dimensão para a fato, prefira medidas com `CALCULATE`. Use `RELATED` em colunas quando precisar do valor estático na linha da fato (exportação, RLS auxiliar, agrupamentos específicos).

---

### Exemplo 8 — RELATEDTABLE (contagem inversa One → Many)

**Uso:** contar quantos registros filhos existem para cada linha da dimensão.

```dax
// Coluna calculada em DimColaborador — quantos eventos de folha o colaborador tem
Qtd Eventos Folha =
COUNTROWS(
    RELATEDTABLE(FatoFolha)
)
```

```dax
// Coluna calculada em DimColaborador — dias registrados no ponto
Dias Com Ponto =
COUNTROWS(
    RELATEDTABLE(FatoPonto)
)
```

```dax
// Coluna calculada em DimCentroCusto — headcount de ativos
Headcount Ativos CC =
COUNTROWS(
    FILTER(
        RELATEDTABLE(DimColaborador),
        DimColaborador[Status] = "Ativo"
    )
)
```

---

### Exemplo 9 — CALCULATE (filtros explícitos e contexto modificado)

**Uso:** alterar o contexto de filtro de uma expressão de agregação.

```dax
// Medida — proventos apenas da competência selecionada
Total Proventos =
CALCULATE(
    SUM(FatoFolha[Valor]),
    FatoFolha[TipoLancamento] = "Provento"
)
```

```dax
// Medida — folha só de CLT (filtro na dimensão)
Folha CLT =
CALCULATE(
    SUM(FatoFolha[Valor]),
    DimColaborador[TipoContrato] = "CLT"
)
```

```dax
// Medida — ignorar filtro de departamento mas manter competência
Folha Todos Departamentos =
CALCULATE(
    SUM(FatoFolha[Valor]),
    ALL(DimCentroCusto[Departamento])
)
```

```dax
// Medida — horas extras só da gerência selecionada
Horas Extras Gerencia =
CALCULATE(
    SUM(FatoPonto[HorasExtras]),
    NOT(ISBLANK(DimCentroCusto[Gerencia]))
)
```

```dax
// Medida — usar relacionamento inativo (data de pagamento)
Folha Por Data Pagamento =
CALCULATE(
    SUM(FatoFolha[Valor]),
    USERELATIONSHIP(DimData[DataID], FatoFolha[DataPagamentoID])
)
```

---

### Exemplo 10 — SUMX (iteração linha a linha com cálculo)

**Uso:** quando a agregação **não é uma simples soma** — precisa calcular algo por linha e depois somar.

```dax
// Medida — custo total iterando cada linha da folha (equivalente a SUM com filtro)
Total Folha SUMX =
SUMX(
    FatoFolha,
    FatoFolha[Valor]
)
```

```dax
// Medida — soma de proventos positivos apenas
Total Proventos Positivos =
SUMX(
    FILTER(FatoFolha, FatoFolha[Valor] > 0),
    FatoFolha[Valor]
)
```

```dax
// Medida — horas extras × valor hora médio POR COLABORADOR (cálculo composto)
Custo Estimado HE =
SUMX(
    VALUES(DimColaborador[ColaboradorID]),
    VAR HorasExtraColab =
        CALCULATE(SUM(FatoPonto[HorasExtras]))
    VAR SalarioBase =
        CALCULATE(
            SUM(FatoFolha[Valor]),
            DimRubrica[Categoria] = "Salario Base"
        )
    VAR HorasContratuais =
        CALCULATE(SUM(FatoPonto[HorasContratuais]))
    VAR ValorHora =
        DIVIDE(SalarioBase, HorasContratuais, 0)
    RETURN
        HorasExtraColab * ValorHora * 1.5
)
```

```dax
// Medida — banco de horas por colaborador agregado
Saldo Banco Horas SUMX =
SUMX(
    VALUES(DimColaborador[ColaboradorID]),
    VAR Extras = CALCULATE(SUM(FatoPonto[HorasExtras]))
    VAR Compensadas = CALCULATE(SUM(FatoPonto[HorasCompensadas]))
    RETURN Extras - Compensadas
)
```

```dax
// Medida — custo médio por colaborador com SUMX (alternativa explícita)
Custo Medio SUMX =
DIVIDE(
    SUMX(
        VALUES(DimColaborador[ColaboradorID]),
        CALCULATE([Custo Total Empresa])
    ),
    DISTINCTCOUNT(DimColaborador[ColaboradorID]),
    0
)
```

---

### Exemplo 11 — AVERAGEX, COUNTX, MAXX (mesma lógica iterativa)

```dax
// Média de minutos de atraso por colaborador (primeiro soma por colab, depois média)
Media Atraso Por Colaborador =
AVERAGEX(
    VALUES(DimColaborador[ColaboradorID]),
    CALCULATE(SUM(FatoPonto[MinutosAtraso]))
)
```

```dax
// Quantos colaboradores tiveram falta no período
Colaboradores Com Falta =
COUNTX(
    FILTER(
        VALUES(DimColaborador[ColaboradorID]),
        CALCULATE(SUM(FatoPonto[Falta])) > 0
    ),
    DimColaborador[ColaboradorID]
)
```

```dax
// Maior custo individual de folha no contexto atual
Maior Custo Individual =
MAXX(
    VALUES(DimColaborador[ColaboradorID]),
    CALCULATE([Custo Total Empresa])
)
```

---

### Exemplo 12 — CALCULATE + SUMX combinados

```dax
// Folha do departamento DP com cálculo linha a linha de encargos
Encargos Departamento DP =
CALCULATE(
    SUMX(
        FatoFolha,
        IF(
            RELATED(DimRubrica[IncideEncargo]) = 1,
            FatoFolha[Valor] * 0.20,
            0
        )
    ),
    DimCentroCusto[Departamento] = "Departamento Pessoal"
)
```

> **Nota:** `RELATED` dentro de `SUMX` sobre `FatoFolha` funciona porque cada linha da fato tem relacionamento com `DimRubrica`. Em medidas, alternativa mais idiomática:

```dax
Encargos Estimados 20pct =
SUMX(
    FatoFolha,
    FatoFolha[Valor] * RELATED(DimRubrica[IncideEncargo]) * 0.20
)
```

---

### Exemplo 13 — FILTER + CALCULATE (subconjuntos complexos)

```dax
// Colaboradores ativos com horas extras acima de 10h no mês
Colaboradores HE Critico =
COUNTROWS(
    FILTER(
        ADDCOLUMNS(
            VALUES(DimColaborador[ColaboradorID]),
            "@TotalHE", CALCULATE(SUM(FatoPonto[HorasExtras]))
        ),
        [@TotalHE] > 10
            && CALCULATE(MAX(DimColaborador[Status])) = "Ativo"
    )
)
```

```dax
// Folha de colaboradores com tempo de casa > 5 anos
Folha Veteranos =
CALCULATE(
    SUM(FatoFolha[Valor]),
    FILTER(
        DimColaborador,
        DimColaborador[TempoCasaMeses] > 60
    )
)
```

---

### Exemplo 14 — VAR / RETURN (legibilidade e performance)

```dax
// Medida completa de conformidade folha × ponto
Conformidade Detalhada =
VAR ColaboradoresNoContexto =
    VALUES(DimColaborador[ColaboradorID])
VAR Tolerancia = 0.25
VAR TabelaAnalise =
    ADDCOLUMNS(
        ColaboradoresNoContexto,
        "@HEPonto",
            CALCULATE(SUM(FatoPonto[HorasExtras])),
        "@HEFolha",
            CALCULATE(
                SUM(FatoFolha[Valor]),
                DimRubrica[CodigoRubrica] IN {"HE50", "HE100"}
            )
    )
VAR Conformes =
    COUNTROWS(
        FILTER(
            TabelaAnalise,
            ABS([@HEPonto] - [@HEFolha]) <= Tolerancia
        )
    )
VAR Total =
    COUNTROWS(ColaboradoresNoContexto)
RETURN
    DIVIDE(Conformes, Total, 0)
```

---

### Exemplo 15 — Time Intelligence com CALCULATE

```dax
// Custo folha mês anterior
Folha Mes Anterior =
CALCULATE(
    [Custo Total Empresa],
    DATEADD(DimData[Data], -1, MONTH)
)
```

```dax
// Variação percentual MoM
Variacao MoM =
VAR Atual = [Custo Total Empresa]
VAR Anterior = [Folha Mes Anterior]
RETURN
    DIVIDE(Atual - Anterior, Anterior, BLANK())
```

```dax
// Acumulado no ano
Folha YTD =
TOTALYTD(
    [Custo Total Empresa],
    DimData[Data]
)
```

```dax
// Mesmo período ano anterior
Folha SPLY =
CALCULATE(
    [Custo Total Empresa],
    SAMEPERIODLASTYEAR(DimData[Data])
)
```

---

## Parte 3 — Quando usar o quê

| Necessidade | Use |
|-------------|-----|
| Limpar texto, datas, tipos na carga | **Power Query M** |
| Padronizar antes de entrar no modelo | **M** (`Table.TransformColumns`, `try/otherwise`) |
| Trazer coluna da dimensão para a fato | **RELATED** (coluna calculada) |
| Contar filhos de uma dimensão | **RELATEDTABLE** |
| Agregação com filtro extra | **CALCULATE** |
| Cálculo linha a linha depois agregado | **SUMX / AVERAGEX** |
| KPI pronto para visual | **Medida** (não coluna) |
| Contexto sem filtro de slicer | **ALL**, **ALLEXCEPT**, **REMOVEFILTERS** |

---

## Parte 4 — Anti-padrões comuns neste projeto

| Evitar | Preferir |
|--------|----------|
| Coluna calculada com SUM na fato | Medida com CALCULATE |
| RELATED na direção errada (dim → fato sem RELATEDTABLE) | RELATEDTABLE ou medida |
| Limpeza de CPF no DAX | Limpeza no Power Query M |
| SUMX em tabela inteira sem necessidade | SUM simples quando não há cálculo por linha |
| Múltiplos RELATED em medidas | Manter atributos na dimensão e filtrar por lá |

---

## Referências cruzadas

- Modelo e relacionamentos: [`modelo-dados.md`](modelo-dados.md)
- Medidas de negócio: [`formulas-dax.md`](formulas-dax.md)
- Colunas das tabelas: [`estrutura-dados.md`](estrutura-dados.md)
