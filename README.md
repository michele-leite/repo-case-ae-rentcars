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
Optamos por enveredar na abstração do Star Schema dividindo a inteligência em **Dimensões** estritas (`dim_users`, `dim_partners`) para fatiamento (slice-and-dice), operando contra as **Fatos** granulares e cronológicas (`fct_bookings`, `fct_sessions`). A premissa central aqui foi desacoplar entidades lógicas, isolando os cálculos intensos em um bloco central `Intermediate`, impedindo vazamento de complexidade para a camada analítica final.

### Estratégia de Materialização
- **Staging** (`view`): Por serem puramente cascatas de renomeação de colunas e castings leves, materializar em disco causaria custos inócuos de redundância em cloud computing.
- **Intermediate** (`table`): São efetuados dectecções de anomalias por JOINs multi-tabelas grossos e deduplicações pesadas por `row_number()`. Cachar essa carga como de forma espessa na memória tira o burden imposto às facts na ponta.
- **Marts_Core - Facts** (`incremental` com _delete+insert_): São logs que explodem geometricamente em volume de linhas a cada mês que se passa. Refazê-los do zero todo dia seria péssima arquitetura. A estratégia incremental engata apenas a "fatia temporal nova" do dia preservando o orçamento da infraestrutura sem impactar performance de merge nativo por conflito temporal.
- **Marts_Analytics** (`table`): Para leitura direta no painel do Business Intelligence. Exigem velocidade vertiginosa e estão frequentemente recalculando agregações do nível superior.

### Estratégia de Modelagem Incremental

Para otimizar o processamento e garantir a integridade dos dados nas tabelas `fct_booking` e `fct_session`, adotei a materialização **Incremental** com a estratégia `delete+insert` (padrão robusto para PostgreSQL).

#### 1. Lógica de Atualização e Lookback

Diferente de uma carga incremental simples que apenas anexa novos dados, nossa lógica utiliza uma **Janela de Lookback (Retrocesso)** de **7 a 30 dias**.

- **Por quê:** Reservas (*bookings*) e Sessões (*sessions*) são entidades "vivas". Uma reserva pode ser criada hoje, mas ter seu status alterado para "cancelado" ou "concluído" dias depois.
- **Funcionamento:** O modelo revisita os últimos dias de dados, garantindo que qualquer alteração de status na origem seja sincronizada com o Data Warehouse, mantendo a precisão das métricas financeiras e operacionais.

#### 2. Tratamento de Sessões Órfãs (Regra dos 1440 min)

Para a `fct_session`, implementei uma lógica de **fechamento forçado**:

- Sessões que não possuem um evento de término (`ended_at`) e que ultrapassam **1440 minutos (24 horas)** de duração são encerradas artificialmente.
- **Filtro Incremental Especial:** O modelo monitora registros onde `ended_at_final IS NULL`, garantindo que sessões abertas sejam reavaliadas a cada execução até que recebam um status final.

#### 3. Garantia de Unicidade e Prevenção de Fan-out

Como as tabelas de fatos realizam joins com dimensões e eventos (como buscas e cancelamentos), existe o risco de duplicação de linhas (*fan-out*).

- **Deduplicação Explícita:** Utilizamos a função de janela `row_number()` particionada pela `unique_key` (`booking_id` / `session_id`) no estágio final da transformação.
- Isso garante que, mesmo que ocorram inconsistências na origem ou nos joins, cada registro seja único na camada final, respeitando a integridade referencial do modelo.

#### 4. Otimização de Performance

Para evitar o escaneamento completo das tabelas (*Full Table Scan*) no PostgreSQL:

- Aplicamos filtros baseados na coluna de partição (`started_at` / `tsp_booked`) limitando a busca aos **últimos 10 dias**.
- Isso garante que a consulta seja executada apenas sobre os índices, reduzindo drasticamente o tempo de processamento e o consumo de recursos no Supabase.

> **Plano de Testes do Incremental:** Para validar cada um desses comportamentos das tabelas incrementais com queries reais, consulte o [plano_teste_incremental.md](extras/plano_teste_incremental.md).
> **Log de Execução do Incremental:** Para visualizar o log de execução de cada um dos modelos incrementais, consulte o [log_execucao_e_testes.md](extras/log_execucao_e_testes.md).

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
