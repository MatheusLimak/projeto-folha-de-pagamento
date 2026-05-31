# Estrutura de Dados Simulada

Dicionário de dados fictício representando integração entre **HRIS**, **sistema de folha (ERP)** e **REP de ponto eletrônico**.

Volume simulado sugerido para demonstração:

| Tabela | Registros | Período |
|--------|-----------|---------|
| `DimColaborador` | ~850 | 2020–2025 |
| `FatoFolha` | ~120.000 | 24 competências |
| `FatoPonto` | ~450.000 | ~530 dias × 850 colaboradores (parcial) |
| `DimData` | ~3.650 | 2020–2030 |

---

## DimColaborador

| Coluna | Tipo | Exemplo | Descrição |
|--------|------|---------|-----------|
| `ColaboradorID` | Int | 1001 | PK surrogate |
| `Matricula` | Text | "EMP-004521" | Matrícula funcional |
| `Nome` | Text | "Ana Silva" | Nome completo |
| `CPF` | Text | "123.456.789-00" | Documento (mascarado no BI) |
| `CargoID` | Int | 12 | FK → DimCargo |
| `CentroCustoID` | Int | 305 | FK → DimCentroCusto |
| `DataAdmissao` | Date | 2021-03-15 | Data de admissão |
| `DataDemissao` | Date | BLANK | Preenchido se desligado |
| `Status` | Text | "Ativo" | Ativo / Desligado / Afastado |
| `TipoContrato` | Text | "CLT" | CLT, PJ, Estágio, Temporário |
| `JornadaHoras` | Decimal | 8.00 | Carga horária diária contratual |
| `Genero` | Text | "F" | Para análises de diversidade |
| `FaixaEtaria` | Text | "30-39" | Calculada na origem |

---

## DimCargo

| Coluna | Tipo | Exemplo | Descrição |
|--------|------|---------|-----------|
| `CargoID` | Int | 12 | PK |
| `Cargo` | Text | "Analista de DP Pleno" | Nome do cargo |
| `FamiliaCargo` | Text | "Recursos Humanos" | Agrupamento |
| `Area` | Text | "Administrativo" | Área funcional |
| `Nivel` | Text | "Pleno" | Junior / Pleno / Senior / Coordenação |
| `SalarioReferencia` | Decimal | 6500.00 | Faixa média de mercado interno |

---

## DimCentroCusto

| Coluna | Tipo | Exemplo | Descrição |
|--------|------|---------|-----------|
| `CentroCustoID` | Int | 305 | PK |
| `CentroCusto` | Text | "DP-Operações" | Nome |
| `Departamento` | Text | "Departamento Pessoal" | |
| `Gerencia` | Text | "Gestão de Pessoas" | |
| `Diretoria` | Text | "Administrativa" | |
| `Unidade` | Text | "Matriz SP" | Filial / unidade |
| `UF` | Text | "SP" | Estado |

---

## DimData

| Coluna | Tipo | Exemplo | Descrição |
|--------|------|---------|-----------|
| `DataID` | Int | 20250115 | PK (YYYYMMDD) |
| `Data` | Date | 2025-01-15 | |
| `Ano` | Int | 2025 | |
| `Mes` | Int | 1 | |
| `MesNome` | Text | "Janeiro" | |
| `AnoMes` | Text | "202501" | |
| `Trimestre` | Text | "Q1" | |
| `DiaSemana` | Text | "Quarta-feira" | |
| `NumeroSemana` | Int | 3 | ISO week |
| `EhFimDeSemana` | Int | 0 | 0/1 |
| `EhFeriado` | Int | 0 | 0/1 (nacional) |

---

## DimCalendarioTrabalho

| Coluna | Tipo | Exemplo | Descrição |
|--------|------|---------|-----------|
| `DataID` | Int | 20250115 | FK → DimData |
| `Unidade` | Text | "Matriz SP" | Calendário por unidade |
| `EhDiaUtil` | Int | 1 | Considera feriados locais |
| `TipoDia` | Text | "Util" | Util / Feriado / Ponto Facultativo |

---

## DimRubrica

| Coluna | Tipo | Exemplo | Descrição |
|--------|------|---------|-----------|
| `RubricaID` | Int | 45 | PK |
| `CodigoRubrica` | Text | "HE50" | Código interno |
| `Rubrica` | Text | "Hora Extra 50%" | Descrição |
| `Categoria` | Text | "Hora Extra" | Salario Base / Beneficio / Encargo / Desconto |
| `TipoLancamento` | Text | "Provento" | Provento / Desconto |
| `IncideEncargo` | Int | 1 | Base para encargos patronais |

---

## FatoFolha

| Coluna | Tipo | Exemplo | Descrição |
|--------|------|---------|-----------|
| `FolhaID` | Int | 900001 | PK |
| `ColaboradorID` | Int | 1001 | FK |
| `CompetenciaID` | Int | 202501 | FK → DimData (1º dia do mês) |
| `RubricaID` | Int | 45 | FK |
| `Valor` | Decimal | 320.50 | Positivo provento, negativo desconto |
| `TipoLancamento` | Text | "Provento" | Redundante para filtros rápidos |
| `DataPagamentoID` | Int | 20250205 | Data crédito em conta |
| `ProcessamentoID` | Text | "FOL-202501-01" | Lote de processamento |

---

## FatoPonto

| Coluna | Tipo | Exemplo | Descrição |
|--------|------|---------|-----------|
| `PontoID` | Int | 5000001 | PK |
| `ColaboradorID` | Int | 1001 | FK |
| `DataID` | Int | 20250115 | FK |
| `HorasContratuais` | Decimal | 8.00 | Esperado no dia |
| `HorasTrabalhadas` | Decimal | 8.75 | Efetivamente trabalhado |
| `HorasExtras` | Decimal | 0.75 | Acima da jornada |
| `HorasCompensadas` | Decimal | 0.00 | Abatidas do banco |
| `MinutosAtraso` | Int | 12 | Atraso na entrada |
| `MinutosSaidaAntecipada` | Int | 0 | |
| `Falta` | Int | 0 | 1 = falta injustificada |
| `Atestado` | Int | 0 | 1 = ausência justificada |
| `Origem` | Text | "REP" | REP / Manual / Ajuste RH |

---

## FatoPontoDetalhe (opcional)

Marcações individuais para drill-through:

| Coluna | Tipo | Exemplo |
|--------|------|---------|
| `MarcacaoID` | Int | 80000001 |
| `PontoID` | Int | 5000001 |
| `TipoMarcacaoID` | Int | 1 |
| `HorarioMarcacao` | DateTime | 2025-01-15 08:12:00 |
| `TipoMarcacao` | Text | "Entrada" |

---

## ParametrosKPI

| Parametro | Valor | Unidade |
|-----------|-------|---------|
| Meta Turnover Mensal | 0.02 | % |
| Meta Horas Extras % | 0.05 | % |
| Meta Taxa Atraso | 0.03 | % |
| Tolerancia Divergencia Folha-Ponto (horas) | 0.25 | horas |

---

## Regras de qualidade de dados (Power Query)

Simulação de transformações aplicadas na camada de ingestão:

```powerquery
// Exemplo M — padronização de status
= Table.TransformColumns(
    FonteColaboradores,
    {{"Status", each Text.Proper(_), type text}}
)

// Exemplo M — filtrar competências futuras inválidas
= Table.SelectRows(FatoFolha, each [CompetenciaID] <= Date.Year(DateTime.LocalNow()) * 100 + Date.Month(DateTime.LocalNow()))
```

### Validações implementadas

| Regra | Ação |
|-------|------|
| CPF duplicado ativo | Flag de erro na carga |
| Valor folha = 0 | Excluir ou revisar |
| Horas trabalhadas > 16h/dia | Flag outlier |
| Colaborador desligado com ponto posterior | Excluir registros |
| Rubrica sem categoria | Mapear para "Não Classificado" |

---

## Amostra de dados fictícios

### DimColaborador (3 registros)

```
ColaboradorID | Matricula   | Nome          | CargoID | CentroCustoID | DataAdmissao | Status  | TipoContrato
1001          | EMP-004521  | Ana Silva     | 12      | 305           | 2021-03-15   | Ativo   | CLT
1002          | EMP-004522  | Bruno Costa   | 8       | 210           | 2019-07-01   | Ativo   | CLT
1003          | EMP-003890  | Carla Mendes  | 15      | 110           | 2018-01-10   | Desligado | CLT
```

### FatoFolha — competência 202501, colaborador 1001

```
FolhaID | ColaboradorID | CompetenciaID | RubricaID | Valor   | TipoLancamento
900101  | 1001          | 202501        | 001       | 5500.00 | Provento
900102  | 1001          | 202501        | 010       | 220.00  | Provento
900103  | 1001          | 202501        | 045       | 187.50  | Provento
900104  | 1001          | 202501        | 101       | -605.00 | Desconto
900105  | 1001          | 202501        | 201       | 1210.00 | Provento
```

> Rubrica 201 = encargo patronal (INSS patronal simulado), armazenado como linha separada.

### FatoPonto — colaborador 1001, jan/2025 (amostra)

```
DataID   | HorasContratuais | HorasTrabalhadas | HorasExtras | MinutosAtraso | Falta
20250102 | 8.00             | 8.50             | 0.50        | 0             | 0
20250103 | 8.00             | 9.25             | 1.25        | 15            | 0
20250106 | 8.00             | 8.00             | 0.00        | 0             | 0
20250107 | 8.00             | 0.00             | 0.00        | 0             | 1
```
