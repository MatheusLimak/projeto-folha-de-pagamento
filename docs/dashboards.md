# Dashboards — Wireframes e Especificação

Documentação visual conceitual das páginas do relatório Power BI.

---

## Página 1 — Visão Executiva RH

**Público:** Diretoria e gerência sênior  
**Atualização:** Mensal (competência fechada)

### Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  [Logo]  RH Analytics — Visão Executiva     [Competência ▼]     │
├────────────┬────────────┬────────────┬────────────┬─────────────┤
│ Headcount  │ Custo Total│  Turnover  │ Horas Ext. │ Score Saúde │
│    847     │ R$ 4,2 Mi  │   2,1%     │   4,8%     │    78/100   │
│  ▲ +3 MoM  │ ▲ +1,2%    │  ● Amarelo │ ▼ -0,3pp   │             │
├────────────┴────────────┴────────────┴────────────┴─────────────┤
│  [Gráfico linha] Headcount e Turnover — últimos 12 meses        │
├──────────────────────────────┬──────────────────────────────────┤
│ [Barras] Custo por Diretoria │ [Rosca] Composição do Custo      │
├──────────────────────────────┴──────────────────────────────────┤
│  [Mapa ou barras] Headcount por UF / Unidade                    │
└─────────────────────────────────────────────────────────────────┘
```

### Filtros (slicers)

- Competência (DimData[AnoMes])
- Unidade (DimCentroCusto[Unidade])
- Tipo de contrato (DimColaborador[TipoContrato])

### Medidas principais

- `[Headcount Snapshot]`
- `[Custo Total Empresa]`
- `[% Turnover Mensal]`
- `[% Horas Extras]`
- `[Score Saude RH]`

---

## Página 2 — Folha de Pagamento

**Público:** Departamento Pessoal e controllers  
**Foco:** Composição e evolução de custos

### Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  Folha de Pagamento          [Competência ▼] [Centro Custo ▼] │
├────────────┬────────────┬────────────┬─────────────────────────┤
│ Proventos  │ Descontos  │ Encargos   │ Custo Total Empresa     │
│ R$ 3,8 Mi  │ R$ 620 mil │ R$ 1,02 Mi │ R$ 4,82 Mi              │
├────────────┴────────────┴────────────┴─────────────────────────┤
│  [Waterfall] Variação MoM do Custo Total                        │
├──────────────────────────────┬──────────────────────────────────┤
│ [Matriz] Rubrica × Centro    │ [Linha] Evolução 24 meses        │
│         de Custo             │   por categoria de rubrica       │
├──────────────────────────────┴──────────────────────────────────┤
│  [Tabela] Top 20 colaboradores por custo total (drill-through)  │
└─────────────────────────────────────────────────────────────────┘
```

### Drill-through

Destino: **Detalhe Colaborador** — filtra por `ColaboradorID` e exibe holerite simulado (matriz rubricas × competências).

### Medidas principais

- `[Total Proventos]`, `[Total Descontos]`, `[Total Encargos]`
- `[Custo Total Empresa]`
- `[Variacao MoM Custo Folha]`
- `[Custo Medio Colaborador]`
- `[% Beneficios Custo Total]`

---

## Página 3 — Gestão de Ponto

**Público:** RH operacional e gestores de área  
**Foco:** Jornada, absenteísmo e horas extras

### Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  Gestão de Ponto              [Período ▼] [Departamento ▼]      │
├────────────┬────────────┬────────────┬────────────┬─────────────┤
│ Hrs Trab.  │ Hrs Extras │ % Atraso   │ Absenteísmo│ Banco Horas │
│  142.500h  │  6.840h    │   2,7%     │   1,9%     │  +320h      │
├────────────┴────────────┴────────────┴────────────┴─────────────┤
│  [Heatmap] Atrasos por dia da semana × departamento             │
├──────────────────────────────┬──────────────────────────────────┤
│ [Barras] HE por Gerencia     │ [Scatter] Atraso vs HE por equipe│
├──────────────────────────────┴──────────────────────────────────┤
│  [Tabela] Colaboradores com indicadores fora da meta              │
│  (conditional formatting vermelho/amarelo)                      │
└─────────────────────────────────────────────────────────────────┘
```

### Regras visuais (conditional formatting)

| Medida | Verde | Amarelo | Vermelho |
|--------|-------|---------|----------|
| `[% Taxa Atraso]` | ≤ 2% | ≤ 4% | > 4% |
| `[% Horas Extras]` | ≤ 4% | ≤ 7% | > 7% |
| `[% Absenteismo]` | ≤ 1,5% | ≤ 3% | > 3% |

### Medidas principais

- `[Total Horas Trabalhadas]`, `[Total Horas Extras]`
- `[% Taxa Atraso]`, `[% Absenteismo]`
- `[Saldo Banco Horas]`
- `[Media Atraso Minutos]`

---

## Página 4 — Auditoria Folha × Ponto

**Público:** Auditoria interna e DP  
**Foco:** Conformidade entre sistemas

### Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  Auditoria Folha × Ponto        [Competência ▼] [Unidade ▼]    │
├────────────────────────────┬────────────────────────────────────┤
│  % Conformidade            │  Divergência Total (horas)         │
│       94,2%                │         -128,5h                    │
├────────────────────────────┴────────────────────────────────────┤
│  [Barras empilhadas] HE Ponto vs HE Paga na Folha por dept.     │
├──────────────────────────────┬──────────────────────────────────┤
│ [Tabela] Colaboradores com   │ [KPI] Colaboradores críticos     │
│ divergência > tolerância     │      (|div| > 2h)                │
└──────────────────────────────┴──────────────────────────────────┘
```

### Medidas principais

- `[% Conformidade Folha Ponto]`
- `[Divergencia HE Folha Ponto]`
- `[Horas Extras Pagas Folha]`
- `[Total Horas Extras]`

---

## Página 5 — Detalhe Colaborador (Drill-through)

**Acesso:** Clique em colaborador nas tabelas das outras páginas

### Conteúdo

- Dados cadastrais (cargo, centro de custo, tempo de casa)
- Sparklines: custo mensal e horas extras (6 meses)
- Tabela de rubricas da competência selecionada
- Calendário de ponto do mês (matriz dia × indicadores)

---

## Segurança (RLS) — planejada

| Perfil | Escopo |
|--------|--------|
| **Diretoria** | Todos os centros de custo |
| **Gestor RH** | Filtrado por `DimCentroCusto[Gerencia]` via USERPRINCIPALNAME() |
| **DP Operacional** | Acesso total leitura, sem dados de salário individual *(opcional)* |

Exemplo de role DAX para gestor:

```dax
// Role: Gestor Area
[GestorEmail] = USERPRINCIPALNAME()
```

Filtro na tabela `DimCentroCusto`:

```dax
[EmailGestor] = USERPRINCIPALNAME()
```

---

## Paleta de cores sugerida

| Uso | Cor | Hex |
|-----|-----|-----|
| Primária | Azul corporativo | `#1F4E79` |
| Positivo | Verde | `#2E7D32` |
| Alerta | Amarelo | `#F9A825` |
| Crítico | Vermelho | `#C62828` |
| Neutro | Cinza | `#757575` |
| Fundo | Branco / cinza claro | `#FAFAFA` |

---

## Bookmarks e navegação

- **Bookmark "Mês Atual"** — filtra competência corrente
- **Bookmark "YTD"** — acumulado do ano
- **Botões de navegação** entre as 4 páginas principais no cabeçalho
- **Tooltip pages** — mini-card de turnover ao passar mouse sobre headcount
