# Data Dictionary — Case Técnico Analytics Engineer
**Rentcars | Processo Seletivo**
Gerado em: 2025-03-31 | Período dos dados: 01/10/2024 – 31/03/2025

---

## Visão Geral dos Datasets

| Tabela | Descrição | Linhas | Colunas |
|---|---|---|---|
| `raw_partners` | Cadastro de locadoras parceiras | 22 | 9 |
| `raw_sessions` | Sessões de navegação no site | 120.350 | 11 |
| `raw_searches` | Buscas realizadas pelos usuários | 80.240 | 11 |
| `raw_bookings` | Reservas concluídas | 18.124 | 14 |
| `raw_cancellations` | Cancelamentos de reservas | 4.410 | 8 |

> ⚠️ **Atenção:** Os dados contêm problemas de qualidade propositais (duplicatas, nulos, outliers e inconsistências). Parte do desafio é identificá-los, documentá-los e tratá-los adequadamente.

---

## Diagrama de Relacionamento (ERD simplificado)

```
raw_sessions ──────────────────────────────────────────────────────────────────────┐
    │ session_id                                                                    │
    ▼                                                                               │
raw_searches                                                                        │
    │ partner_id_clicked ──────────► raw_partners                                  │
                                          ▲                                        │
raw_bookings ◄────────────────────────────┘                                        │
    │ session_id ◄──────────────────────────────────────────────────────────────────┘
    │ booking_id
    ▼
raw_cancellations
```

---

## raw_partners

**Descrição:** Tabela de cadastro das locadoras parceiras da plataforma. Contém informações sobre nome, país de operação, tier comercial, status de ativação e taxa de comissão.

**Granularidade:** 1 linha por parceiro

| Coluna | Tipo esperado | Nullable | Descrição | Valores possíveis / Exemplo | Problemas conhecidos |
|---|---|---|---|---|---|
| `partner_id` | VARCHAR | NÃO | Identificador único do parceiro | `PRT0001`, `PRT0020` | Registros duplicados (2 linhas extras propositais) |
| `partner_name` | VARCHAR | NÃO | Nome comercial da locadora | `Localiza`, `Hertz` | — |
| `country` | VARCHAR | SIM (3%) | País de operação principal (ISO 3166-1 alpha-2) | `BR`, `AR`, `US` | — |
| `tier` | VARCHAR | SIM (14%) | Categoria comercial do parceiro | `gold`, `silver`, `bronze` | Nulos representam parceiros não classificados |
| `status` | VARCHAR | NÃO | Status de ativação na plataforma | `active`, `inactive`, `suspended` | ⚠️ Inconsistência de capitalização: `ACTIVE`, `Active` coexistem |
| `commission_rate` | FLOAT | NÃO | Taxa de comissão acordada (em decimal) | `0.1200` = 12% | — |
| `created_at` | TIMESTAMP | NÃO | Data de cadastro do parceiro | `2019-04-29 01:23:31` | — |
| `updated_at` | TIMESTAMP | SIM (9%) | Última atualização do registro | `2024-10-01 07:59:25` | Nulo indica que o registro nunca foi atualizado |
| `contact_email` | VARCHAR | SIM (9%) | E-mail de contato operacional | `ops@localiza.com` | — |

**Regras de negócio:**
- Somente parceiros com `status = active` devem receber reservas
- `commission_rate` deve estar entre 0.05 e 0.30
- `partner_id` deve ser único

---

## raw_sessions

**Descrição:** Registro de todas as sessões de navegação iniciadas no site da Rentcars. Uma sessão agrupa toda a atividade de um usuário (ou visitante anônimo) durante uma visita contínua.

**Granularidade:** 1 linha por sessão

| Coluna | Tipo esperado | Nullable | Descrição | Valores possíveis / Exemplo | Problemas conhecidos |
|---|---|---|---|---|---|
| `session_id` | VARCHAR (UUID) | NÃO | Identificador único da sessão | `ab8e3cd2-479d-4bd0-...` | ⚠️ ~350 linhas duplicadas |
| `user_id` | VARCHAR (UUID) | SIM (12%) | Identificador do usuário autenticado | `7b5c2164-9c17-...` | Nulo = sessão anônima (usuário não logado) |
| `started_at` | TIMESTAMP | NÃO | Timestamp de início da sessão | `2024-10-16 18:21:39` | — |
| `ended_at` | TIMESTAMP | SIM (4%) | Timestamp de encerramento da sessão | `2024-10-16 18:45:02` | ⚠️ ~0.5% das sessões têm duração > 24h (outlier) |
| `channel` | VARCHAR | SIM (6%) | Canal de aquisição da sessão | `google_cpc`, `direct`, `affiliate` | — |
| `device` | VARCHAR | NÃO | Tipo de dispositivo utilizado | `desktop`, `mobile`, `tablet` | ⚠️ Inconsistência de capitalização: `Mobile`, `DESKTOP` |
| `country` | VARCHAR | NÃO | País de origem da sessão (ISO 3166-1) | `BR`, `AR`, `MX` | ⚠️ Inconsistência de case: `br`, `Br` coexistem com `BR` |
| `page_views` | INTEGER | SIM (3%) | Número de páginas visualizadas na sessão | `1` a `25` | — |
| `utm_source` | VARCHAR | SIM (29%) | Fonte UTM da campanha | `google`, `facebook`, `email` | Alto índice de nulo = tráfego sem rastreamento |
| `utm_campaign` | VARCHAR | SIM (20%) | Nome da campanha UTM | `camp_1` a `camp_50` | — |
| `is_bot` | BOOLEAN | NÃO | Indica se a sessão foi classificada como bot | `True`, `False` | ~3% das sessões são bots; devem ser excluídas das análises |

**Regras de negócio:**
- `ended_at` deve ser sempre posterior a `started_at`
- Sessões com `is_bot = True` devem ser filtradas em modelos analíticos
- Uma sessão com `user_id` nulo representa um visitante anônimo

---

## raw_searches

**Descrição:** Registro de todas as buscas realizadas na plataforma. Cada linha representa uma busca por disponibilidade de veículos feita por um usuário dentro de uma sessão.

**Granularidade:** 1 linha por busca

| Coluna | Tipo esperado | Nullable | Descrição | Valores possíveis / Exemplo | Problemas conhecidos |
|---|---|---|---|---|---|
| `search_id` | VARCHAR (UUID) | NÃO | Identificador único da busca | `4a82ac01-4ead-...` | ⚠️ ~240 linhas duplicadas |
| `session_id` | VARCHAR (UUID) | NÃO | Sessão em que a busca foi realizada | `6f083ccd-f590-...` | ⚠️ 1 session_id com 65 buscas em < 4 min (suspeito de bot/fraude) |
| `searched_at` | TIMESTAMP | NÃO | Timestamp da busca | `2025-02-09 09:47:31` | — |
| `pickup_location` | VARCHAR | NÃO | Local de retirada do veículo | `São Paulo`, `Curitiba` | ⚠️ Inconsistência de case: `São paulo`, `rio de janeiro`, `FLORIANÓPOLIS` |
| `dropoff_location` | VARCHAR | SIM (25%) | Local de devolução do veículo | `Rio de Janeiro` | Nulo = devolução no mesmo local de retirada |
| `pickup_date` | DATE | NÃO | Data de retirada desejada | `2025-05-02` | — |
| `dropoff_date` | DATE | SIM (4%) | Data de devolução desejada | `2025-05-09` | ⚠️ ~1% dos registros têm `dropoff_date` anterior a `pickup_date` |
| `car_category` | VARCHAR | SIM (14%) | Categoria de veículo filtrada | `economy`, `suv`, `luxury`, `pickup`, `minivan` | Nulo = busca sem filtro de categoria |
| `num_results` | INTEGER | SIM (4%) | Número de ofertas retornadas | `0` a `40` | — |
| `partner_id_clicked` | VARCHAR | SIM (35%) | Parceiro cujo resultado o usuário clicou | `PRT0001` | Nulo = usuário não clicou em nenhum resultado |
| `price_shown` | FLOAT | SIM (6%) | Menor preço exibido no resultado da busca (R$) | `499.60` | — |

**Regras de negócio:**
- `dropoff_date` deve ser igual ou posterior a `pickup_date`
- `partner_id_clicked` deve existir em `raw_partners`
- Buscas oriundas de sessões marcadas como `is_bot = True` devem ser descartadas

---

## raw_bookings

**Descrição:** Registro de todas as reservas finalizadas na plataforma, incluindo reservas confirmadas, pendentes, canceladas e concluídas. É a tabela central para análise de receita e conversão.

**Granularidade:** 1 linha por reserva

| Coluna | Tipo esperado | Nullable | Descrição | Valores possíveis / Exemplo | Problemas conhecidos |
|---|---|---|---|---|---|
| `booking_id` | VARCHAR | NÃO | Identificador único da reserva | `BKG0000001` | ⚠️ ~120 linhas duplicadas |
| `session_id` | VARCHAR (UUID) | NÃO | Sessão em que a reserva foi realizada | `d65dd874-aa5f-...` | — |
| `user_id` | VARCHAR (UUID) | SIM (5%) | Identificador do usuário | `8b95538c-f950-...` | ⚠️ 1 user_id com 4 reservas no mesmo dia (suspeito de fraude) |
| `partner_id` | VARCHAR | NÃO | Parceiro com quem a reserva foi feita | `PRT0007` | Deve existir em `raw_partners` |
| `booked_at` | TIMESTAMP | NÃO | Timestamp em que a reserva foi realizada | `2025-03-29 00:07:01` | — |
| `pickup_date` | DATE | NÃO | Data de retirada do veículo | `2025-05-02` | — |
| `dropoff_date` | DATE | SIM (2%) | Data de devolução do veículo | `2025-05-09` | — |
| `pickup_location` | VARCHAR | SIM (3%) | Local de retirada | `São Paulo`, `Recife` | ⚠️ Inconsistência de case (herdada de raw_searches) |
| `car_category` | VARCHAR | SIM (14%) | Categoria do veículo reservado | `economy`, `suv`, `luxury` | — |
| `daily_rate` | FLOAT | SIM (4%) | Diária cobrada ao cliente (em `currency`) | `326.46` | — |
| `total_amount` | FLOAT | SIM (3%) | Valor total da reserva (em `currency`) | `1235.50` | ⚠️ ~0.8% com valores negativos; ~0.5% zerados; ~0.3% acima de R$15.000 |
| `currency` | VARCHAR | NÃO | Moeda da transação (ISO 4217) | `BRL`, `USD`, `ARS`, `CLP`, `COP` | — |
| `status` | VARCHAR | NÃO | Status atual da reserva | `confirmed`, `pending`, `cancelled`, `no_show`, `completed` | ⚠️ Inconsistência de capitalização: `CONFIRMED`, `Confirmed` |
| `payment_method` | VARCHAR | SIM (25%) | Forma de pagamento utilizada | `credit_card`, `pix`, `boleto`, `debit_card` | Alto índice de nulo |

**Regras de negócio:**
- `total_amount` deve ser > 0 para reservas com `status = confirmed` ou `completed`
- `dropoff_date` deve ser posterior a `pickup_date`
- `partner_id` deve existir e estar ativo em `raw_partners`
- Reservas duplicadas devem ser deduplicadas antes de qualquer agregação financeira

---

## raw_cancellations

**Descrição:** Registro de todos os eventos de cancelamento de reservas. Uma reserva pode ter no máximo um cancelamento associado. A tabela contém o motivo, responsável, valor de reembolso e timing em relação à data de retirada.

**Granularidade:** 1 linha por cancelamento

| Coluna | Tipo esperado | Nullable | Descrição | Valores possíveis / Exemplo | Problemas conhecidos |
|---|---|---|---|---|---|
| `cancellation_id` | VARCHAR (UUID) | NÃO | Identificador único do cancelamento | `1319ffd6-4413-...` | ⚠️ ~80 linhas duplicadas |
| `booking_id` | VARCHAR | NÃO | Reserva associada ao cancelamento | `BKG0003506` | Deve existir em `raw_bookings` |
| `cancelled_at` | TIMESTAMP | NÃO | Timestamp do cancelamento | `2024-12-17 22:31:19` | — |
| `reason` | VARCHAR | SIM (17%) | Motivo declarado do cancelamento | `cliente_desistiu`, `voo_cancelado`, `preco_alto`, `parceiro_sem_veiculo`, `fraude_detectada`, `duplicidade`, `erro_usuario`, `outro` | Nulo = motivo não informado |
| `cancelled_by` | VARCHAR | SIM (21%) | Quem iniciou o cancelamento | `customer`, `partner`, `system`, `admin` | — |
| `refund_amount` | FLOAT | SIM (10%) | Valor reembolsado ao cliente (R$) | `651.24` | — |
| `refund_status` | VARCHAR | SIM (31%) | Status do reembolso | `processed`, `pending`, `denied` | Alto índice de nulo; pode indicar cancelamentos sem reembolso aplicável |
| `days_before_pickup` | INTEGER | SIM (5%) | Dias entre o cancelamento e a data de retirada | `0` a `90` | ⚠️ Valores negativos (~2 ocorrências) indicam cancelamento após a data de retirada |

**Regras de negócio:**
- `booking_id` deve existir em `raw_bookings`
- `days_before_pickup` negativo indica cancelamento tardio (após início do período de locação)
- `refund_amount` não deve ser superior ao `total_amount` da reserva associada

---

## Problemas de Qualidade — Resumo para o Candidato

A tabela abaixo consolida todos os problemas propositalmente injetados nos dados. Parte do desafio é detectar, documentar e tratar cada um deles nos modelos dbt.

| # | Tabela | Tipo de Problema | Descrição | Impacto Potencial |
|---|---|---|---|---|
| 1 | Todas | Duplicatas | Registros idênticos inseridos propositalmente | Dupla contagem em métricas de receita e volume |
| 2 | `raw_sessions` | Inconsistência de case | `device`: `desktop` / `DESKTOP` / `Mobile` | Segmentação incorreta por dispositivo |
| 3 | `raw_sessions` | Inconsistência de case | `country`: `BR` / `br` / `Br` | Agregação errada por país |
| 4 | `raw_sessions` | Outlier temporal | Sessões com duração > 24h (~0.5%) | Distorção em métricas de engajamento |
| 5 | `raw_searches` | Bot / fraude | 1 session_id com 65 buscas em < 4 minutos | Inflação artificial do volume de buscas |
| 6 | `raw_searches` | Anomalia lógica | `dropoff_date` < `pickup_date` (~1%) | Erros em cálculo de diárias e disponibilidade |
| 7 | `raw_searches` | Inconsistência de case | Destinations: `São paulo`, `FLORIANÓPOLIS` | Contagem errada por destino |
| 8 | `raw_bookings` | Outlier de valor | `total_amount` negativo (~0.8%) | Distorção em KPIs de receita |
| 9 | `raw_bookings` | Outlier de valor | `total_amount` = 0 (~0.5%) | Pode mascarar falhas de processamento |
| 10 | `raw_bookings` | Outlier de valor | `total_amount` > R$15.000 (~0.3%) | Pode ser fraude ou erro de sistema |
| 11 | `raw_bookings` | Fraude potencial | 1 user_id com 4 reservas no mesmo dia | Risco de chargeback / exposição financeira |
| 12 | `raw_bookings` | Inconsistência de case | `status`: `confirmed` / `CONFIRMED` / `Confirmed` | Filtros de status quebram sem normalização |
| 13 | `raw_partners` | Duplicatas | 2 registros de parceiros duplicados | Joins inflados com `raw_bookings` |
| 14 | `raw_partners` | Inconsistência de case | `status`: `active` / `ACTIVE` / `Active` | Filtro de parceiros ativos falha |
| 15 | `raw_cancellations` | Outlier temporal | `days_before_pickup` negativo | Cancelamento após início da locação |

---

## Glossário de Negócio

| Termo | Definição |
|---|---|
| **Sessão** | Visita contínua de um usuário ao site; encerrada após 30 min de inatividade |
| **Busca** | Consulta de disponibilidade feita pelo usuário com datas e localidade |
| **Reserva (booking)** | Confirmação de aluguel de veículo; gera receita quando `status = confirmed` ou `completed` |
| **Parceiro** | Locadora que disponibiliza veículos na plataforma Rentcars |
| **Taxa de Conversão** | Proporção de sessões que resultam em ao menos uma reserva confirmada |
| **Ticket Médio** | Valor médio de `total_amount` para reservas confirmadas ou concluídas |
| **Taxa de Cancelamento** | % de reservas canceladas sobre o total de reservas confirmadas |
| **Tier** | Nível de parceria comercial: `gold` > `silver` > `bronze` |
| **Canal de Aquisição** | Origem da sessão: `google_cpc`, `direct`, `affiliate`, `metasearch`, etc. |
| **Daily Rate** | Valor cobrado por diária de locação, em moeda local |

---

*Dúvidas sobre os dados? Registre suas suposições no README.md do seu projeto — isso faz parte da avaliação.*
