# End-to-End Analytics Engineering Case | Rentcars

Este projeto é uma implementação completa de um **Data Warehouse moderno orientado a produto**, construído com foco em:

- Escalabilidade
- Governança de dados
- Confiabilidade analítica
- Alinhamento com stakeholders

A solução cobre todo o ciclo de dados: ingestão → modelagem → qualidade → métricas → storytelling.

---

## Contexto do Problema

A plataforma de aluguel de veículos gera dados distribuídos entre:

- Sessões de navegação
- Buscas
- Reservas
- Cancelamentos
- Parceiros

O desafio é transformar esse ecossistema em um modelo confiável para responder perguntas críticas de negócio, como:

- Onde estamos perdendo conversão?
- Quais parceiros geram mais valor?
- Existe comportamento suspeito (bots)?
- Qual o impacto real dos cancelamentos?

---

## Diagrama de Arquitetura do Modelo de Dados

O fluxo foi delineado aderindo estritamente aos pilares robustos do dbt modular (Medallion + Kimball Architecture).

```text
       [Raw Postgree Database] 
                 |
                 v
+---------------------------------+
|          STAGING LAYER          | 
|  (Clean, Type-cast, Rename)     |
+---------------------------------+
                 |
                 v
+---------------------------------+
|       INTERMEDIATE LAYER        |
| (Business Rules, Anomaly Flags, |
|      Window Func Dedupl.)       |
+---------------------------------+
                 |
        +--------+--------+
        v                 v
+---------------+ +---------------+
| CORE FACTS    | | DIMENSIONS    |
| - fct_bookings| | - dim_users   |
| - fct_sessions| | - dim_partners|
|               | | - dim_dates   |
+---------------+ +---------------+
                 |
                 v
+---------------------------------+
|         ANALYTICS MARTS         |
| Aggregações finas e Dashboards  |
| - mart_funnel                   |
+---------------------------------+
```

---

## Justificativas: Modelagem e Materialização (Decisões Técnicas)

### Modelagem Dimensional
Optei por enveredar na abstração do Star Schema dividindo a inteligência em **Dimensões** estritas (`dim_users`, `dim_partners`) para fatiamento (slice-and-dice), operando contra as **Fatos** granulares e cronológicas (`fct_bookings`, `fct_sessions`). A premissa central aqui foi desacoplar entidades lógicas, isolando os cálculos intensos em um bloco central `Intermediate`, impedindo vazamento de complexidade para a camada analítica final.

### Estratégia de Materialização
- **Staging** (`view`): Por serem puramente cascatas de renomeação de colunas e castings leves, materializar em disco causaria custos inócuos de redundância em cloud computing.
- **Intermediate** (`table`): São efetuados dectecções de anomalias por JOINs multi-tabelas grossos e deduplicações pesadas por `row_number()`. Cachar essa carga como de forma espessa na memória tira o burden imposto às facts na ponta.
- **Marts_Core - Facts** (`incremental` com _delete+insert_): São logs que explodem geometricamente em volume de linhas a cada mês que se passa. Refazê-los do zero todo dia seria péssima arquitetura. A estratégia incremental engata apenas a "fatia temporal nova" do dia preservando o orçamento da infraestrutura sem impactar performance de merge nativo por conflito temporal.
- **Marts_Analytics** (`table`): Para leitura direta no painel do Business Intelligence. Exigem velocidade vertiginosa e estão frequentemente recalculando agregações do nível superior.

---

## Camada Analítica

### mart_funnel

Principais métricas:

- Conversão
- Receita
- Ticket médio
- Engajamento

---

## Dashboard & Insights

- Evolução de receita
- Funil de conversão
- Cancelamentos

Foco em insights acionáveis.

---

## Analytics

- Conversão por país/device
- Top parceiros
- LTV
- Detecção de bots
- Outliers

---

## Governança

- SLA de dados
- PII
- Catálogo
- Data contract

---

## Como rodar

```bash
pip install dbt-core dbt-postgres
dbt deps
dbt run
dbt test
```

---

## Roadmap

- SCD2
- CI/CD
- Observabilidade
- Testes avançados

---

## Diferenciais

✔ Arquitetura escalável  
✔ Data quality embutido  
✔ Foco em negócio  
✔ Pensamento de produto  

---

## TL;DR

Projeto que demonstra capacidade real de  Analytics Engineer.


---

## 🔗 Data Lineage (dbt)

![dbt lineage](./docs/dbt_lineage.png)

Este grafo representa o fluxo completo de dados no projeto:

- Camada RAW → STAGING → INTERMEDIATE → MARTS
- Separação clara entre regras de negócio e consumo analítico
- Dependências explícitas entre fatos e dimensões

Destaque para:
- Centralização de regras na camada `int_`
- Construção incremental das facts (`fct_sessions`, `fct_bookings`)
- Camada final otimizada para consumo (`mart_funnel`)
