# Visuais HTML Content — Indicadores Estratégicos de RH

Documentação de **medidas DAX** e **templates HTML5** para o visual customizado **HTML Content** (AppSource), aplicados aos KPIs estratégicos de **Folha, Ponto e People Analytics**.

> Visual: [HTML Content](https://appsource.microsoft.com/) — adicionar medidas ao campo **Values** e referenciá-las no HTML com `{{Nome da Medida}}`.

---

## Como funciona a integração DAX + HTML Content

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────────┐
│  Modelo DAX     │────►│  Medidas KPI     │────►│  HTML Content       │
│  (Star Schema)  │     │  (texto/cor/%)   │     │  {{Medida}} no HTML │
└─────────────────┘     └──────────────────┘     └─────────────────────┘
```

1. Crie medidas DAX que retornam **valor formatado**, **cor hex** ou **classe CSS**
2. Arraste as medidas para o visual HTML Content
3. Cole o HTML no painel **Content** do visual
4. Use `{{Nome Exato da Medida}}` — o nome deve coincidir com a medida no modelo

---

## Indicadores estratégicos do negócio

| KPI | Sigla | Objetivo estratégico | Meta simulada |
|-----|-------|----------------------|---------------|
| **Custo Total de Pessoal** | CTP | Controlar despesa com folha e encargos | Orçamento mensal |
| **Custo por Colaborador (FTE)** | CPC | Benchmark de eficiência de headcount | R$ 5.700/colab. |
| **Taxa de Turnover** | TO | Medir retenção e clima organizacional | ≤ 2,0% a.m. |
| **Taxa de Absenteísmo** | ABS | Impacto de faltas na operação | ≤ 1,5% |
| **Índice de Horas Extras** | IHE | Pressão de jornada e custo oculto | ≤ 5,0% |
| **Conformidade Folha × Ponto** | CFP | Integridade entre sistemas | ≥ 95% |
| **Score Saúde RH** | SSR | Índice composto executivo | ≥ 75/100 |
| **Produtividade Horas** | PH | Horas produtivas vs. contratuais | ≥ 97% |
| **Variação MoM Folha** | VMF | Tendência de custo mês a mês | ± 3% |
| **Retenção 12 meses** | RET | Colaboradores que permanecem | ≥ 88% |

---

## Medidas DAX — valores estratégicos base

### Custo Total Empresa (base)

```dax
HTML Custo Total Empresa =
VAR Valor = [Custo Total Empresa]
RETURN
    Valor
```

### Custo formatado para exibição

```dax
HTML Custo Total Texto =
VAR Valor = [Custo Total Empresa]
RETURN
    "R$ " & FORMAT(Valor, "#,##0", "pt-BR")
```

### Custo por colaborador (FTE)

```dax
HTML Custo Por FTE =
DIVIDE(
    [Custo Total Empresa],
    [Headcount Snapshot],
    0
)
```

```dax
HTML Custo Por FTE Texto =
"R$ " & FORMAT([HTML Custo Por FTE], "#,##0", "pt-BR")
```

### Variação MoM com seta Unicode

```dax
HTML Variacao MoM Texto =
VAR VarPct = [Variacao MoM Custo Folha]
VAR Seta = IF(VarPct >= 0, "▲", "▼")
VAR Cor = IF(VarPct <= 0.03, "#2E7D32", IF(VarPct <= 0.05, "#F9A825", "#C62828"))
RETURN
    Seta & " " & FORMAT(ABS(VarPct), "0.0%", "pt-BR")
```

```dax
HTML Variacao MoM Cor =
VAR VarPct = [Variacao MoM Custo Folha]
RETURN
    IF(VarPct <= 0.03, "#2E7D32", IF(VarPct <= 0.05, "#F9A825", "#C62828"))
```

### Turnover estratégico

```dax
HTML Turnover Texto =
FORMAT([% Turnover Mensal], "0.00%", "pt-BR")
```

```dax
HTML Turnover Cor =
SWITCH(
    [Semaforo Turnover],
    "Verde", "#2E7D32",
    "Amarelo", "#F9A825",
    "#C62828"
)
```

```dax
HTML Turnover Status =
SWITCH(
    [Semaforo Turnover],
    "Verde", "Dentro da meta",
    "Amarelo", "Atenção",
    "Crítico — acima da meta"
)
```

### Absenteísmo

```dax
HTML Absenteismo Texto =
FORMAT([% Absenteismo], "0.00%", "pt-BR")
```

```dax
HTML Absenteismo Cor =
IF([% Absenteismo] <= 0.015, "#2E7D32", IF([% Absenteismo] <= 0.03, "#F9A825", "#C62828"))
```

### Horas extras

```dax
HTML Horas Extras Texto =
FORMAT([% Horas Extras], "0.0%", "pt-BR")
```

```dax
HTML Horas Extras Cor =
IF([% Horas Extras] <= 0.04, "#2E7D32", IF([% Horas Extras] <= 0.07, "#F9A825", "#C62828"))
```

### Conformidade Folha × Ponto

```dax
HTML Conformidade Texto =
FORMAT([% Conformidade Folha Ponto], "0.0%", "pt-BR")
```

```dax
HTML Conformidade Cor =
IF([% Conformidade Folha Ponto] >= 0.95, "#2E7D32", IF([% Conformidade Folha Ponto] >= 0.90, "#F9A825", "#C62828"))
```

### Score Saúde RH (índice executivo)

```dax
HTML Score Saude Texto =
FORMAT([Score Saude RH], "0.0") & " / 100"
```

```dax
HTML Score Saude Cor =
IF([Score Saude RH] >= 75, "#2E7D32", IF([Score Saude RH] >= 60, "#F9A825", "#C62828"))
```

```dax
HTML Score Saude Label =
SWITCH(
    TRUE(),
    [Score Saude RH] >= 85, "Excelente",
    [Score Saude RH] >= 75, "Saudável",
    [Score Saude RH] >= 60, "Atenção",
    "Crítico"
)
```

### Headcount estratégico

```dax
HTML Headcount Texto =
FORMAT([Headcount Snapshot], "#,##0", "pt-BR") & " colaboradores"
```

```dax
HTML Headcount Variacao =
VAR Atual = [Headcount Snapshot]
VAR Anterior =
    CALCULATE([Headcount Snapshot], DATEADD(DimData[Data], -1, MONTH))
VAR Diff = Atual - Anterior
RETURN
    IF(Diff >= 0, "+", "") & FORMAT(Diff, "#,##0", "pt-BR") & " vs mês ant."
```

### Produtividade de horas

```dax
HTML Produtividade Horas =
DIVIDE(
    SUM(FatoPonto[HorasTrabalhadas]) - SUM(FatoPonto[HorasExtras]),
    SUM(FatoPonto[HorasContratuais]),
    0
)
```

```dax
HTML Produtividade Texto =
FORMAT([HTML Produtividade Horas], "0.0%", "pt-BR")
```

### Retenção 12 meses

```dax
HTML Retencao 12m =
VAR Admitidos12m =
    CALCULATE(
        COUNTROWS(DimColaborador),
        DATESINPERIOD(DimData[Data], MAX(DimData[Data]), -12, MONTH),
        NOT(ISBLANK(DimColaborador[DataAdmissao]))
    )
VAR AindaAtivos =
    CALCULATE(
        COUNTROWS(DimColaborador),
        DimColaborador[Status] = "Ativo",
        DATESINPERIOD(DimData[Data], MAX(DimData[Data]), -12, MONTH)
    )
RETURN
    DIVIDE(AindaAtivos, Admitidos12m, 0)
```

```dax
HTML Retencao Texto =
FORMAT([HTML Retencao 12m], "0.0%", "pt-BR")
```

### Período dinâmico (cabeçalho)

```dax
HTML Titulo Periodo = [Titulo Periodo]
```

```dax
HTML Data Atualizacao =
"Atualizado em " & FORMAT(TODAY(), "DD/MM/YYYY", "pt-BR")
```

---

## Medidas DAX — suporte visual (CSS e layout)

### Cor primária da marca

```dax
HTML Cor Primaria = "#1F4E79"
```

### Cor de fundo do card

```dax
HTML Cor Fundo Card = "#FFFFFF"
```

### Largura da barra de progresso (Score Saúde)

```dax
HTML Score Barra Width =
FORMAT([Score Saude RH], "0") & "%"
```

### Folha vs Orçamento (indicador estratégico financeiro)

```dax
HTML Orcamento Folha =
SELECTEDVALUE(ParametrosKPI[Orcamento Folha Mensal], 4200000)
```

```dax
HTML Folha vs Orcamento Pct =
DIVIDE([Custo Total Empresa], [HTML Orcamento Folha], 0)
```

```dax
HTML Folha vs Orcamento Texto =
FORMAT([HTML Folha vs Orcamento Pct], "0.0%", "pt-BR") & " do orçamento"
```

```dax
HTML Folha vs Orcamento Cor =
IF([HTML Folha vs Orcamento Pct] <= 1, "#2E7D32", IF([HTML Folha vs Orcamento Pct] <= 1.05, "#F9A825", "#C62828"))
```

> Adicionar na tabela `ParametrosKPI`: coluna `Orcamento Folha Mensal` = 4200000.

---

## Template 1 — Card KPI executivo (single metric)

**Medidas no visual:** `HTML Custo Total Texto`, `HTML Variacao MoM Texto`, `HTML Variacao MoM Cor`, `HTML Titulo Periodo`

```html
<div style="
  font-family: 'Segoe UI', sans-serif;
  background: linear-gradient(135deg, #1F4E79 0%, #2d6a9f 100%);
  border-radius: 12px;
  padding: 20px 24px;
  color: #fff;
  box-shadow: 0 4px 12px rgba(0,0,0,0.15);
  min-height: 120px;
">
  <div style="font-size: 11px; opacity: 0.85; text-transform: uppercase; letter-spacing: 1px;">
    Custo Total de Pessoal
  </div>
  <div style="font-size: 32px; font-weight: 700; margin: 8px 0 4px;">
    {{HTML Custo Total Texto}}
  </div>
  <div style="font-size: 14px; color: {{HTML Variacao MoM Cor}}; font-weight: 600;">
    {{HTML Variacao MoM Texto}} vs mês anterior
  </div>
  <div style="font-size: 10px; opacity: 0.7; margin-top: 12px;">
    {{HTML Titulo Periodo}}
  </div>
</div>
```

---

## Template 2 — Painel estratégico com 4 KPIs (grid)

**Medidas:** `HTML Score Saude Texto`, `HTML Score Saude Cor`, `HTML Score Saude Label`, `HTML Turnover Texto`, `HTML Turnover Cor`, `HTML Absenteismo Texto`, `HTML Absenteismo Cor`, `HTML Conformidade Texto`, `HTML Conformidade Cor`, `HTML Titulo Periodo`

```html
<style>
  .rh-panel { font-family: 'Segoe UI', sans-serif; padding: 8px; }
  .rh-header { font-size: 13px; color: #1F4E79; font-weight: 700; margin-bottom: 12px; border-bottom: 2px solid #1F4E79; padding-bottom: 6px; }
  .rh-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }
  .rh-kpi { background: #FAFAFA; border-radius: 8px; padding: 14px; border-left: 4px solid #1F4E79; }
  .rh-kpi-label { font-size: 10px; color: #757575; text-transform: uppercase; }
  .rh-kpi-value { font-size: 22px; font-weight: 700; margin: 4px 0; }
  .rh-kpi-sub { font-size: 11px; color: #757575; }
</style>
<div class="rh-panel">
  <div class="rh-header">Indicadores Estratégicos RH — {{HTML Titulo Periodo}}</div>
  <div class="rh-grid">
    <div class="rh-kpi" style="border-left-color: {{HTML Score Saude Cor}};">
      <div class="rh-kpi-label">Score Saúde RH</div>
      <div class="rh-kpi-value" style="color: {{HTML Score Saude Cor}};">{{HTML Score Saude Texto}}</div>
      <div class="rh-kpi-sub">{{HTML Score Saude Label}}</div>
    </div>
    <div class="rh-kpi" style="border-left-color: {{HTML Turnover Cor}};">
      <div class="rh-kpi-label">Turnover Mensal</div>
      <div class="rh-kpi-value" style="color: {{HTML Turnover Cor}};">{{HTML Turnover Texto}}</div>
      <div class="rh-kpi-sub">Meta: ≤ 2,0%</div>
    </div>
    <div class="rh-kpi" style="border-left-color: {{HTML Absenteismo Cor}};">
      <div class="rh-kpi-label">Absenteísmo</div>
      <div class="rh-kpi-value" style="color: {{HTML Absenteismo Cor}};">{{HTML Absenteismo Texto}}</div>
      <div class="rh-kpi-sub">Meta: ≤ 1,5%</div>
    </div>
    <div class="rh-kpi" style="border-left-color: {{HTML Conformidade Cor}};">
      <div class="rh-kpi-label">Conformidade Folha × Ponto</div>
      <div class="rh-kpi-value" style="color: {{HTML Conformidade Cor}};">{{HTML Conformidade Texto}}</div>
      <div class="rh-kpi-sub">Meta: ≥ 95%</div>
    </div>
  </div>
</div>
```

---

## Template 3 — Score Saúde RH com barra de progresso

**Medidas:** `HTML Score Saude Texto`, `HTML Score Saude Cor`, `HTML Score Saude Label`, `HTML Score Barra Width`

```html
<div style="font-family: 'Segoe UI', sans-serif; padding: 16px; background: #fff; border-radius: 10px; border: 1px solid #E0E0E0;">
  <div style="display: flex; justify-content: space-between; align-items: center;">
    <span style="font-size: 12px; color: #757575; font-weight: 600;">SCORE SAÚDE RH</span>
    <span style="font-size: 11px; padding: 3px 8px; border-radius: 4px; background: {{HTML Score Saude Cor}}; color: #fff;">
      {{HTML Score Saude Label}}
    </span>
  </div>
  <div style="font-size: 36px; font-weight: 800; color: {{HTML Score Saude Cor}}; margin: 8px 0;">
    {{HTML Score Saude Texto}}
  </div>
  <div style="background: #EEEEEE; border-radius: 6px; height: 10px; overflow: hidden;">
    <div style="width: {{HTML Score Barra Width}}; height: 100%; background: {{HTML Score Saude Cor}}; border-radius: 6px; transition: width 0.3s;"></div>
  </div>
  <div style="font-size: 10px; color: #9E9E9E; margin-top: 8px;">
    Composto: Turnover + Absenteísmo + Horas Extras
  </div>
</div>
```

---

## Template 4 — Semáforo estratégico (HTML puro + DAX)

**Medidas:** `HTML Turnover Texto`, `HTML Turnover Cor`, `HTML Turnover Status`, `HTML Horas Extras Texto`, `HTML Horas Extras Cor`, `HTML Folha vs Orcamento Texto`, `HTML Folha vs Orcamento Cor`

```html
<div style="font-family: 'Segoe UI', sans-serif;">
  <table style="width: 100%; border-collapse: collapse; font-size: 13px;">
    <tr style="background: #1F4E79; color: #fff;">
      <th style="padding: 10px; text-align: left;">Indicador</th>
      <th style="padding: 10px; text-align: center;">Valor</th>
      <th style="padding: 10px; text-align: left;">Status</th>
    </tr>
    <tr style="border-bottom: 1px solid #EEE;">
      <td style="padding: 10px;">Turnover</td>
      <td style="padding: 10px; text-align: center; font-weight: 700; color: {{HTML Turnover Cor}};">{{HTML Turnover Texto}}</td>
      <td style="padding: 10px; color: {{HTML Turnover Cor}};">{{HTML Turnover Status}}</td>
    </tr>
    <tr style="border-bottom: 1px solid #EEE;">
      <td style="padding: 10px;">Horas Extras</td>
      <td style="padding: 10px; text-align: center; font-weight: 700; color: {{HTML Horas Extras Cor}};">{{HTML Horas Extras Texto}}</td>
      <td style="padding: 10px; color: {{HTML Horas Extras Cor}};">Meta ≤ 5%</td>
    </tr>
    <tr>
      <td style="padding: 10px;">Folha vs Orçamento</td>
      <td style="padding: 10px; text-align: center; font-weight: 700; color: {{HTML Folha vs Orcamento Cor}};">{{HTML Folha vs Orcamento Texto}}</td>
      <td style="padding: 10px; color: {{HTML Folha vs Orcamento Cor}};">Controle financeiro</td>
    </tr>
  </table>
</div>
```

---

## Template 5 — Cabeçalho executivo da página

**Medidas:** `HTML Titulo Periodo`, `HTML Headcount Texto`, `HTML Headcount Variacao`, `HTML Data Atualizacao`, `HTML Cor Primaria`

```html
<div style="
  font-family: 'Segoe UI', sans-serif;
  background: {{HTML Cor Primaria}};
  color: #fff;
  padding: 16px 20px;
  border-radius: 0 0 12px 12px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 12px;
">
  <div>
    <div style="font-size: 20px; font-weight: 700;">RH Analytics — Visão Estratégica</div>
    <div style="font-size: 12px; opacity: 0.9;">{{HTML Titulo Periodo}}</div>
  </div>
  <div style="text-align: right;">
    <div style="font-size: 16px; font-weight: 600;">{{HTML Headcount Texto}}</div>
    <div style="font-size: 11px; opacity: 0.85;">{{HTML Headcount Variacao}}</div>
    <div style="font-size: 10px; opacity: 0.7; margin-top: 4px;">{{HTML Data Atualizacao}}</div>
  </div>
</div>
```

---

## Template 6 — Card comparativo Folha + Ponto (estratégico integrado)

**Medidas:** `HTML Custo Total Texto`, `HTML Custo Por FTE Texto`, `HTML Horas Extras Texto`, `HTML Horas Extras Cor`, `HTML Conformidade Texto`, `HTML Produtividade Texto`

```html
<div style="font-family: 'Segoe UI', sans-serif; display: flex; gap: 12px; flex-wrap: wrap;">
  <div style="flex: 1; min-width: 140px; background: #E3F2FD; border-radius: 10px; padding: 16px; text-align: center;">
    <div style="font-size: 10px; color: #1565C0; font-weight: 600;">FOLHA</div>
    <div style="font-size: 20px; font-weight: 700; color: #0D47A1;">{{HTML Custo Total Texto}}</div>
    <div style="font-size: 11px; color: #546E7A;">{{HTML Custo Por FTE Texto}} / FTE</div>
  </div>
  <div style="flex: 1; min-width: 140px; background: #FFF3E0; border-radius: 10px; padding: 16px; text-align: center;">
    <div style="font-size: 10px; color: #E65100; font-weight: 600;">PONTO</div>
    <div style="font-size: 20px; font-weight: 700; color: {{HTML Horas Extras Cor}};">{{HTML Horas Extras Texto}}</div>
    <div style="font-size: 11px; color: #546E7A;">Horas extras s/ jornada</div>
  </div>
  <div style="flex: 1; min-width: 140px; background: #E8F5E9; border-radius: 10px; padding: 16px; text-align: center;">
    <div style="font-size: 10px; color: #2E7D32; font-weight: 600;">INTEGRAÇÃO</div>
    <div style="font-size: 20px; font-weight: 700; color: #1B5E20;">{{HTML Conformidade Texto}}</div>
    <div style="font-size: 11px; color: #546E7A;">Produtividade {{HTML Produtividade Texto}}</div>
  </div>
</div>
```

---

## Medida DAX unificada — HTML completo gerado no DAX

Alternativa avançada: **uma única medida retorna HTML inteiro** (útil quando o visual aceita apenas um campo).

```dax
HTML Card Turnover Completo =
VAR Valor = FORMAT([% Turnover Mensal], "0.00%", "pt-BR")
VAR Cor =
    SWITCH(
        [Semaforo Turnover],
        "Verde", "#2E7D32",
        "Amarelo", "#F9A825",
        "#C62828"
    )
VAR Status =
    SWITCH([Semaforo Turnover], "Verde", "Dentro da meta", "Amarelo", "Atenção", "Crítico")
RETURN
    "<div style='font-family:Segoe UI,sans-serif;background:#FAFAFA;border-radius:8px;padding:16px;border-left:4px solid " & Cor & ";'>" &
    "<div style='font-size:10px;color:#757575;'>TURNOVER MENSAL</div>" &
    "<div style='font-size:28px;font-weight:700;color:" & Cor & ";'>" & Valor & "</div>" &
    "<div style='font-size:11px;color:" & Cor & ";'>" & Status & " (meta ≤ 2%)</div>" &
    "</div>"
```

> **Atenção:** medidas HTML completas exigem desativar sanitização no visual (config **Allow HTML** / **Style HTML**). Use apenas com conteúdo confiável.

---

## Configuração do visual HTML Content no Power BI

| Passo | Ação |
|-------|------|
| 1 | Instalar **HTML Content** pelo AppSource |
| 2 | Criar pasta de medidas `Medidas HTML` no modelo (organização) |
| 3 | Criar medidas com prefixo `HTML ` para fácil identificação |
| 4 | Arrastar medidas para **Values** do visual |
| 5 | Colar template HTML no campo **Content** |
| 6 | Ativar **Allow HTML rendering** se usar tags completas |
| 7 | Redimensionar visual e testar com slicers de competência |

---

## Organização recomendada no modelo

```
Tabelas de medidas (Display folders)
├── _Base           → Custo Total Empresa, Headcount Snapshot...
├── _KPI Negócio    → Score Saude RH, % Turnover...
└── _HTML Content   → HTML Custo Total Texto, HTML Turnover Cor...
```

---

## Boas práticas

| Prática | Motivo |
|---------|--------|
| Prefixo `HTML ` nas medidas | Separação clara de medidas para visual |
| Medidas de cor separadas do texto | Reutilizar cor em bordas, barras e badges |
| Não embutir lógica de negócio no HTML | Toda regra fica no DAX |
| FORMAT com locale `pt-BR` | Consistência monetária e percentual |
| Testar com slicer vazio | Validar comportamento de DIVIDE e BLANK |

---

## Referências

- Medidas base: [`formulas-dax.md`](formulas-dax.md)
- Exemplos DAX/M: [`exemplos-dax-m.md`](exemplos-dax-m.md)
- Wireframes: [`dashboards.md`](dashboards.md)
- Templates HTML copiáveis: [`../assets/html/`](../assets/html/)
