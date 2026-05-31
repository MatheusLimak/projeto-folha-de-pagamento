# Modelo de Dados — Relacionamentos

## Visão geral

O modelo segue um **Star Schema** com duas tabelas fato (`FatoFolha` e `FatoPonto`) e dimensões compartilhadas. A granularidade de cada fato é distinta e não deve ser unificada em uma única tabela — a análise cruzada é feita via dimensões comuns (`DimColaborador`, `DimData`, `DimCentroCusto`).

---

## Tabelas do modelo

### Dimensões

| Tabela | Chave | Descrição |
|--------|-------|-----------|
| `DimColaborador` | `ColaboradorID` | Cadastro mestre: nome, CPF, admissão, demissão, status |
| `DimCargo` | `CargoID` | Cargos, níveis e faixas salariais de referência |
| `DimCentroCusto` | `CentroCustoID` | Departamentos, diretorias e unidades de negócio |
| `DimData` | `DataID` | Calendário completo (2020–2030) para time intelligence |
| `DimCalendarioTrabalho` | `DataID` | Dias úteis, feriados e escala por unidade |
| `DimRubrica` | `RubricaID` | Tipos de provento/desconto da folha |
| `DimTipoMarcacao` | `TipoMarcacaoID` | Entrada, saída, intervalo, etc. |

### Fatos

| Tabela | Granularidade | Descrição |
|--------|---------------|-----------|
| `FatoFolha` | 1 linha por colaborador × rubrica × competência | Valores financeiros da folha |
| `FatoPonto` | 1 linha por colaborador × dia | Totais diários de jornada |

---

## Diagrama de relacionamentos

```
DimColaborador (1) ──────< (N) FatoFolha
DimColaborador (1) ──────< (N) FatoPonto
DimCargo       (1) ──────< (N) DimColaborador
DimCentroCusto (1) ──────< (N) DimColaborador
DimData        (1) ──────< (N) FatoFolha        [CompetenciaID → DataID]
DimData        (1) ──────< (N) FatoPonto        [DataID]
DimRubrica     (1) ──────< (N) FatoFolha
DimCalendarioTrabalho (1) ──< (N) FatoPonto     [DataID]
DimTipoMarcacao (1) ──────< (N) FatoPontoDetalhe [opcional]
```

---

## Relacionamentos detalhados

| De | Para | Coluna | Cardinalidade | Direção do filtro | Ativo |
|----|------|--------|---------------|-------------------|-------|
| `DimColaborador` | `FatoFolha` | `ColaboradorID` | 1:N | Single | Sim |
| `DimColaborador` | `FatoPonto` | `ColaboradorID` | 1:N | Single | Sim |
| `DimCargo` | `DimColaborador` | `CargoID` | 1:N | Single | Sim |
| `DimCentroCusto` | `DimColaborador` | `CentroCustoID` | 1:N | Single | Sim |
| `DimData` | `FatoFolha` | `DataID` → `CompetenciaID` | 1:N | Single | Sim |
| `DimData` | `FatoPonto` | `DataID` | 1:N | Single | Sim |
| `DimRubrica` | `FatoFolha` | `RubricaID` | 1:N | Single | Sim |
| `DimCalendarioTrabalho` | `FatoPonto` | `DataID` | 1:N | Single | Sim |

### Relacionamento inativo (uso avançado)

| De | Para | Uso |
|----|------|-----|
| `DimData` | `FatoFolha` via `DataPagamentoID` | Análise por data de crédito em conta (USERELATIONSHIP) |

---

## Regras de modelagem

### 1. Granularidade da folha

`FatoFolha` armazena **eventos por rubrica**, não o holerite consolidado. Um colaborador em uma competência pode ter N linhas (salário base, VT, INSS, hora extra paga, etc.).

```
ColaboradorID | CompetenciaID | RubricaID | Valor | Tipo (P/D)
--------------+---------------+-----------+-------+-----------
1001          | 202501        | 001       | 5500  | Provento
1001          | 202501        | 045       | 220   | Provento
1001          | 202501        | 101       | -605  | Desconto
```

### 2. Granularidade do ponto

`FatoPonto` consolida **totais diários** para performance. Marcações brutas ficam em `FatoPontoDetalhe` (tabela opcional, não relacionada diretamente às medidas principais).

```
ColaboradorID | DataID   | HorasContratuais | HorasTrabalhadas | HorasExtras | MinutosAtraso | Falta
--------------+----------+------------------+------------------+-------------+---------------+------
1001          | 20250102 | 8.00             | 8.50             | 0.50        | 0             | 0
1001          | 20250103 | 8.00             | 0.00             | 0.00        | 0             | 1
```

### 3. DimColaborador como hub

Toda análise por departamento ou cargo passa por `DimColaborador`, evitando snowflake nas fatos:

```
DimCentroCusto → DimColaborador → FatoFolha
DimCentroCusto → DimColaborador → FatoPonto
```

### 4. Status do colaborador

Colaboradores desligados permanecem na dimensão com `Status = "Desligado"` e `DataDemissao` preenchida. Medidas de headcount filtram ativos via DAX, preservando histórico para turnover.

---

## Hierarquias

### Organizacional
```
DimCentroCusto[Diretoria] → [Gerencia] → [Departamento] → [CentroCusto]
```

### Temporal
```
DimData[Ano] → [Trimestre] → [MesNome] → [Data]
```

### Cargo
```
DimCargo[Area] → [FamiliaCargo] → [Cargo]
```

---

## Colunas calculadas na dimensão

### DimColaborador — Tempo de Casa (meses)

```dax
TempoCasaMeses =
DATEDIFF(
    DimColaborador[DataAdmissao],
    IF(
        DimColaborador[Status] = "Desligado",
        DimColaborador[DataDemissao],
        TODAY()
    ),
    MONTH
)
```

### DimColaborador — Faixa Tempo de Casa

```dax
FaixaTempoCasa =
SWITCH(
    TRUE(),
    DimColaborador[TempoCasaMeses] <= 6,  "0-6 meses",
    DimColaborador[TempoCasaMeses] <= 12, "6-12 meses",
    DimColaborador[TempoCasaMeses] <= 36, "1-3 anos",
    DimColaborador[TempoCasaMeses] <= 60, "3-5 anos",
    "5+ anos"
)
```

### DimData — Flags de período

```dax
EhMesAtual = IF(DimData[AnoMes] = FORMAT(TODAY(), "YYYYMM"), 1, 0)

EhYTD = IF(DimData[Data] <= TODAY() && DimData[Ano] = YEAR(TODAY()), 1, 0)
```

---

## Tabela auxiliar: ParametrosKPI

Tabela desconectada para metas configuráveis:

| Parametro | Valor |
|-----------|-------|
| Meta Turnover Mensal | 0.02 |
| Meta Horas Extras % | 0.05 |
| Meta Taxa Atraso | 0.03 |
| Tolerancia Divergencia Folha-Ponto (horas) | 0.25 |

---

## Considerações de performance

- Preferir **medidas** a colunas calculadas nas fatos
- Não usar colunas de texto nas fatos — manter descritivos nas dimensões
- `FatoPonto` pode ter milhões de linhas; evitar RELATED em colunas calculadas
- Agregações automáticas (Premium) podem ser aplicadas em `FatoPonto` por mês
