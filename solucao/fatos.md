## Fonte da Verdade

> O que está escrito aqui é LEI: nenhum outro trecho pode contrariá-la.

### Constantes da variante

| Constante              | Valor |
| ---------------------- | ----- |
| `TARIFA_HORA_CENTAVOS` | 600   |
| `FRACAO_MINUTOS`       | 30    |
| `TETO_DIARIO_CENTAVOS` | 5000  |
| `TOLERANCIA_MINUTOS`   | 0     |
| `PORTA_SERVICO`        | 8005  |

Valor da fração = `TARIFA_HORA_CENTAVOS` ÷ (60 ÷ `FRACAO_MINUTOS`) = 600 ÷ 2 = **300 centavos**.

### Rotas da API REST

Base URL: `http://localhost:8005`

#### UC1 — Abrir bilhete

`POST /bilhetes`

Body (placa com 7 caracteres alfanuméricos, maiúsculos):

```json
{"placa": "ABC1D23"}
```

Resposta `201`:

```json
{"id": 1, "placa": "ABC1D23", "entrada": "<ISO-8601 com fuso -03:00>", "status": "aberto"}
```

O body aceita `entrada` opcional (ISO-8601 com fuso): quando presente, o bilhete abre naquele instante em vez de "agora". É o gancho de testabilidade da correção — sem ele, testar fração/teto exigiria esperar tempo real. Formato inválido → `422 {"erro": "entrada_invalida"}`.

#### UC2 — Encerrar bilhete

`POST /bilhetes/{id}/encerramento`

Resposta `200`:

```json
{"id": 1, "placa": "ABC1D23", "entrada": "...", "saida": "...",
 "minutos": 95, "valor_centavos": 1200}
```

Regras de valor:

- Cobra-se por fração de `FRACAO_MINUTOS` minutos, arredondando para cima (fração exata cobra 1 fração; 1 minuto a mais já cobra a fração seguinte).
- Hora cheia = `TARIFA_HORA_CENTAVOS`; valor da fração = tarifa ÷ (60 ÷ `FRACAO_MINUTOS`).
- Aplica-se o teto diário: `valor_centavos` nunca supera `TETO_DIARIO_CENTAVOS`.
- Valor sempre em centavos, inteiro — a API nunca retorna ponto flutuante.

#### UC3 — Listar ativos

`GET /bilhetes/ativos`

Resposta `200` com array dos bilhetes abertos, mais recentes primeiro.

#### UC4 — Relatório diário

`GET /relatorios/diario?data=AAAA-MM-DD`

Resposta `200`:

```json
{"data": "2026-10-05", "total_bilhetes": 12,
 "faturamento_centavos": 8400, "tempo_medio_minutos": 47}
```

`tempo_medio_minutos` considera apenas bilhetes encerrados no dia, arredondando 0,5 para cima.

#### UC5 — Cancelar bilhete

`POST /bilhetes/{id}/cancelamento`

Resposta `200` com `status: "cancelado"`. Só bilhetes abertos podem ser cancelados — sem cobrança (não gera `saida` nem `valor_centavos`).

#### UC6 — Histórico por placa

`GET /bilhetes?placa=ABC1D23`

Resposta `200` com array de todos os bilhetes da placa (qualquer status), mais recentes primeiro. Placa que nunca estacionou → array vazio.

#### UC7 — Tolerância gratuita

Os primeiros `TOLERANCIA_MINUTOS` de um bilhete são grátis: duração ≤ tolerância → `valor_centavos: 0`. Passou da tolerância (mesmo por 1 minuto) → cobra integral desde o primeiro minuto — a tolerância **não** é descontada.

#### UC8 — Uma vaga por placa

`POST /bilhetes` para placa que já tem bilhete aberto → `409 {"erro": "bilhete_em_aberto"}`. Após encerrar ou cancelar, a placa volta a poder abrir.

### Erros

| Situação                           | Status | Body                                 |
| ---------------------------------- | ------ | ------------------------------------ |
| Placa ausente ou inválida          | 422    | `{"erro": "placa_invalida"}`         |
| `entrada` fora de ISO-8601         | 422    | `{"erro": "entrada_invalida"}`       |
| `data` fora de AAAA-MM-DD          | 422    | `{"erro": "data_invalida"}`          |
| Bilhete inexistente                | 404    | `{"erro": "bilhete_nao_encontrado"}` |
| Encerrar bilhete já encerrado      | 409    | `{"erro": "bilhete_ja_encerrado"}`   |
| Cancelar bilhete não aberto        | 409    | `{"erro": "bilhete_nao_aberto"}`     |
| Abrir bilhete com placa já ocupada | 409    | `{"erro": "bilhete_em_aberto"}`      |

Não invente nenhum código de erro que não exista nesta lista.

**Execução:** Node.js 20 + Express 4 + SQLite (`better-sqlite3`, arquivo criado
na inicialização). Um único processo escutando em `0.0.0.0:8005`. Sem serviços externos.

| Tema | Decisão |
| --- | --- |
| Minutos | `minutos = floor((saida − entrada) ÷ 60 s)`, mínimo 0. Nunca arredondar para cima |
| Valor | `fracoes = ceil(minutos ÷ 30)`; `valor_centavos = min(fracoes × 300, 5000)` |
| Tolerância 0 | 0 minuto → 0; a partir de 1 minuto cobra normalmente |
| Datas na saída | Sempre `AAAA-MM-DDTHH:MM:SS-03:00`, sem milissegundos |
| `entrada` na entrada | Aceita qualquer fuso (`Z`, `±HH:MM`) e converte; sem fuso ou só data → 422 `entrada_invalida`; no futuro é aceita |
| Placa | Válida só se casar com `^[A-Z0-9]{7}$`; minúscula é inválida (não normalizar) |
| Status | `"aberto"`, `"encerrado"`, `"cancelado"`. A resposta do UC2 inclui `"status": "encerrado"` |
| Cancelado | Sem as chaves `saida`, `minutos`, `valor_centavos` (ausentes, não `null`) |
| Listagens | Cada item no formato do próprio status; ordem por `entrada` decrescente, empate por `id` decrescente |
| Encerrar cancelado | 409 `bilhete_ja_encerrado` |
| `{id}` não numérico | 404 `bilhete_nao_encontrado` |
| Body vazio ou JSON malformado | 422 `placa_invalida`. **Nunca responder 400** |
| Ordem de validação | Placa → `entrada` → conflito 409 |
| `GET /bilhetes` sem `placa` | 200 com todos; `placa` inválida na query → 422 `placa_invalida` |
| Relatório | Dia no fuso −03:00. `total_bilhetes` = bilhetes com `entrada` no dia (qualquer status); `faturamento_centavos` e `tempo_medio_minutos` = encerrados com `saida` no dia. Sem encerrados → 0. `data` ausente ou impossível → 422 `data_invalida` |
| Tempo médio | `floor((2 × soma + n) ÷ (2 × n))`; ex.: 10 e 11 → 11; 10, 10 e 11 → 10 |

> [!WARNING]
> O teto vale por bilhete, mesmo que dure vários dias: `valor_centavos` nunca passa de 5000.

| minutos | 0 | 1 | 29 | 30 | 31 | 60 | 61 | 95 | 480 | 481 | 1440 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `valor_centavos` | 0 | 300 | 300 | 300 | 600 | 600 | 900 | 1200 | 4800 | 5000 | 5000 |