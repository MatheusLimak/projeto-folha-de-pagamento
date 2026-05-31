# RH Analytics — Folha de Pagamento e Gestão de Ponto

Projeto de portfólio em **Power BI** focado em **Recursos Humanos**, com análises de **folha de pagamento**, **gestão de ponto** e **indicadores estratégicos de people analytics**.

> Este repositório contém a **documentação teórica** do projeto: modelo de dados, relacionamentos, medidas DAX e desenho de dashboards. Não inclui arquivo `.pbix` nem link publicado — ideal para demonstrar raciocínio analítico e modelagem dimensional em entrevistas e portfólio.

---

## Objetivo de negócio

Apoiar gestores de RH, DP e liderança na tomada de decisão sobre:

- **Custo de pessoal** por centro de custo, departamento e cargo
- **Absenteísmo e horas extras** com base no registro de ponto
- **Turnover e headcount** ao longo do tempo
- **Conformidade** entre ponto registrado e folha processada

---

## Escopo funcional

| Área | Perguntas respondidas |
|------|------------------------|
| **Headcount** | Quantos colaboradores ativos temos por área, unidade e tipo de contrato? |
| **Folha** | Qual o custo total, médio e por componente (salário, benefícios, encargos)? |
| **Ponto** | Qual a taxa de atraso, hora extra e banco de horas por equipe? |
| **Integração** | Há divergência entre horas apontadas e horas pagas na folha? |

---

## Arquitetura do modelo

Modelo **Star Schema** com uma tabela fato principal de folha, uma fato de ponto e dimensões compartilhadas.

```
                    ┌─────────────┐
                    │  DimData    │
                    └──────┬──────┘
                           │
    ┌──────────────┐       │       ┌──────────────┐
    │ DimColaborador◄───────┼───────►│  DimCargo    │
    └──────┬───────┘       │       └──────────────┘
           │               │
           │        ┌──────▼──────┐
           ├───────►│ FatoFolha   │
           │        └─────────────┘
           │               │
    ┌──────▼───────┐       │
    │  DimCentro   │       │
    │    Custo     │       │
    └──────┬───────┘       │
           │        ┌──────▼──────┐
           └───────►│  FatoPonto  │
                    └─────────────┘
                           │
                    ┌──────▼──────┐
                    │ DimCalendario │
                    │   Trabalho   │
                    └─────────────┘
```

Documentação detalhada: [`docs/modelo-dados.md`](docs/modelo-dados.md)

---

## Dashboards planejados

### 1. Visão Executiva RH
- Headcount ativo vs. mês anterior
- Custo total da folha (YTD)
- Turnover mensal
- Horas extras acumuladas

### 2. Folha de Pagamento
- Composição do custo (salário base, benefícios, encargos, descontos)
- Evolução mensal por centro de custo
- Top 10 rubricas por valor

### 3. Gestão de Ponto
- Taxa de presença e atrasos
- Horas extras por departamento
- Saldo de banco de horas
- Colaboradores com jornada irregular

### 4. Auditoria Folha × Ponto
- Divergência entre horas registradas e horas remuneradas
- Indicador de conformidade por unidade

---

## Medidas DAX (amostra)

| Medida | Descrição |
|--------|-----------|
| `Total Custo Folha` | Soma de proventos e encargos |
| `Custo Médio por Colaborador` | Custo total / headcount ativo |
| `% Turnover` | Desligamentos / média de headcount |
| `Taxa de Atraso` | Dias com atraso / dias úteis registrados |
| `Horas Extras %` | Horas extras / horas contratuais |
| `Conformidade Folha-Ponto` | % registros sem divergência |

Lista completa com fórmulas: [`docs/formulas-dax.md`](docs/formulas-dax.md)

---

## Estrutura de dados simulada

Origens fictícias que simulam integração entre sistemas de RH:

| Sistema | Tabela | Descrição |
|---------|--------|-----------|
| ERP / Folha | `FatoFolha` | Eventos financeiros por competência |
| REP / Ponto | `FatoPonto` | Marcações diárias e totais |
| HRIS | `DimColaborador` | Cadastro mestre de colaboradores |
| Organograma | `DimCentroCusto`, `DimCargo` | Estrutura organizacional |
| Calendário | `DimData`, `DimCalendarioTrabalho` | Datas e dias úteis |

Detalhamento de colunas: [`docs/estrutura-dados.md`](docs/estrutura-dados.md)

---

## Tecnologias e práticas

- **Power BI Desktop** — modelagem, DAX e visualização
- **Star Schema** — performance e clareza semântica
- **DimData** — tabela de datas desconectada para time intelligence
- **Medidas calculadas** — KPIs centralizados (sem colunas calculadas desnecessárias)
- **RLS (Row-Level Security)** — perfis Gestor RH, DP e Diretoria *(planejado)*

---

## Estrutura do repositório

```
.
├── README.md
├── docs/
│   ├── modelo-dados.md      # Relacionamentos e cardinalidade
│   ├── formulas-dax.md      # Medidas e colunas calculadas
│   ├── estrutura-dados.md   # Dicionário de dados simulado
│   └── dashboards.md        # Wireframes e filtros por página
└── assets/
    └── diagrama-modelo.png  # (placeholder — gerar a partir do diagrama acima)
```

---

## Próximos passos (evolução do portfólio)

- [ ] Implementar arquivo `.pbix` com dados fictícios em CSV
- [ ] Publicar no Power BI Service com RLS
- [ ] Adicionar screenshots dos dashboards em `assets/`
- [ ] Incluir pipeline ETL documentado (Power Query M)

---
