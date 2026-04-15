# Plano de Teste: Validação da Estratégia Incremental

Este plano visa garantir que o modelo captura atualizações de status em registros passados (idempotência) e não gera duplicatas ao reprocessar janelas de tempo sobrepostas.

---

## Cenário 1: Atualização de Registro Existente (Update)

**Objetivo:** Verificar se uma mudança na origem (ex: reserva alterada ou sessão fechada) é refletida na tabela final.

### Passos

**Passo 1** — Identificar um registro já carregado na tabela de fatos dentro do período de interesse:

```sql
SELECT
    booking_id,
    booking_status,
    tsp_booked
FROM public_marts_core.fct_bookings
WHERE tsp_booked BETWEEN '2025-03-01' AND '2025-03-15'
LIMIT 1;
```

**Passo 2** — Confirmar o estado atual do registro na camada intermediária (antes da alteração):

```sql
SELECT *
FROM public_intermediate.int_valid_bookings
WHERE booking_id = 'BKG0000029';
```

**Passo 3** — Alterar um atributo do registro diretamente na tabela de origem (ex: mudar status de `'pending'` para `'confirmed'`):

```sql
UPDATE public_intermediate.int_valid_bookings
SET booking_status = 'confirmed'
WHERE booking_id = 'BKG0000029';
```

**Passo 4** — Executar o modelo em modo incremental:

```bash
dbt run --select fct_bookings
```

**Resultado Esperado:** A tabela de fatos deve exibir o novo status sem criar uma nova linha.

---

## Cenário 2: Prevenção de Duplicidade (Unique Key)

**Objetivo:** Garantir que a estratégia `delete+insert` está removendo a versão antiga antes de inserir a nova.

### Passos

**Passo 1** — Após a execução do Cenário 1, consultar o registro para verificar unicidade:

```sql
SELECT
    booking_id,
    booking_status,
    COUNT(*) AS ocorrencias
FROM public_marts_core.fct_bookings
WHERE booking_id = 'BKG0000029'
GROUP BY 1, 2;
```

**Resultado Esperado:** O `count` deve ser obrigatoriamente `1`.

**Passo 2** — Verificar que o total de registros únicos permanece estável (sem crescimento por duplicidade):

```sql
SELECT COUNT(DISTINCT booking_id)
FROM public_marts_core.fct_bookings;
-- Esperado: ~14336 (mesma contagem antes e depois do incremental)
```

---

## Cenário 3: Janela de Lookback (Retrocesso)

**Objetivo:** Validar se o filtro incremental alcança registros alterados no passado.

### Passos

**Passo 1** — Registrar a contagem base de registros únicos antes do teste:

```sql
SELECT COUNT(DISTINCT booking_id)
FROM public_marts_core.fct_bookings;
-- Anotar o valor para comparar após a execução
```

**Passo 2** — Escolher um registro com `tsp_booked` de aproximadamente 15 dias atrás e simular uma alteração na origem (repetir o `UPDATE` do Cenário 1 com o `booking_id` identificado).

**Passo 3** — Rodar o incremental garantindo que o filtro de lookback de 30 dias está ativo no modelo:

```bash
dbt run --select fct_bookings
```

**Resultado Esperado:** O registro deve ser atualizado, provando que a "janela deslizante" de 30 dias é eficaz para capturar atrasos de sincronia ou cancelamentos tardios. A contagem total de `booking_id` únicos deve permanecer igual à do Passo 1.

---

## Cenário 4: Sessões em Tempo Real (Handover de Nulos)

**Objetivo:** Validar o fechamento de sessões que entraram como `NULL` (abertas).

### Passos

**Passo 1** — Localizar uma sessão ainda aberta (`tsp_ended IS NULL`) na tabela de fatos:

```sql
SELECT
    session_id,
    tsp_started,
    tsp_ended,
    tsp_ended_final
FROM public_marts_core.fct_sessions
WHERE tsp_ended IS NULL
  AND tsp_started >= '2025-03-01'
LIMIT 1;
```

**Passo 2** — Confirmar o estado atual do registro na camada intermediária (antes da alteração):

```sql
SELECT
    session_id,
    tsp_started,
    tsp_ended
FROM public_intermediate.int_valid_sessions
WHERE session_id = '01430460-690a-4d26-8883-ec7878ed212b';
```

**Passo 3** — Inserir o horário de término na tabela intermediária:

```sql
UPDATE public_intermediate.int_valid_sessions
SET tsp_ended = tsp_started + INTERVAL '30 minutes'
WHERE session_id = '01430460-690a-4d26-8883-ec7878ed212b';
```

**Passo 4** — Executar o modelo incremental de sessões:

```bash
dbt run --select fct_sessions
```

**Passo 5** — Validar o resultado na tabela de fatos — unicidade e preenchimento do campo:

```sql
SELECT
    session_id,
    tsp_started,
    tsp_ended,
    tsp_ended_final,
    COUNT(*) AS ocorrencias
FROM public_marts_core.fct_sessions
WHERE session_id = '01430460-690a-4d26-8883-ec7878ed212b'
GROUP BY 1, 2, 3, 4;
```

**Resultado Esperado:** O campo `tsp_ended_final` deve estar preenchido na tabela de fatos, `ocorrencias = 1`, demonstrando que a lógica de reavaliar nulos funciona corretamente.
