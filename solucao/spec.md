> Leia `fatos.md` antes deste arquivo: ele é a Fonte da Verdade do projeto.

# Spec

## Regras de negócio

| ID    | Regra              | Descrição                                                                              |
| ----- | ------------------ | -------------------------------------------------------------------------------------- |
| RN-01 | Fração             | Cobra por fração de 30 min, arredondando para cima; cada fração vale 300 centavos.     |
| RN-02 | Teto               | `valor_centavos` nunca passa de 5000 por bilhete.                                      |
| RN-03 | Tolerância         | Tolerância de 0 min; 0 minuto custa 0, a partir de 1 minuto cobra integral.            |
| RN-04 | Valor inteiro      | Valores sempre em centavos inteiros, nunca ponto flutuante.                            |
| RN-05 | Placa válida       | 7 caracteres alfanuméricos maiúsculos.                                                 |
| RN-06 | Uma vaga por placa | Placa com bilhete aberto não abre outro.                                               |
| RN-07 | Ordenação          | Listagens trazem os mais recentes primeiro.                                            |
| RN-08 | Tempo médio        | Só encerrados no dia, arredondando 0,5 para cima.                                      |

## Casos de uso

### UC1 — Abrir bilhete

**Descrição:** o operador abre um bilhete para uma placa.

**Pré-condição:** a placa não tem bilhete aberto.

**Regras aplicadas:** RN-05 (placa válida), RN-06 (uma vaga por placa).

**Fluxo principal:**

1. Cliente envia `POST /bilhetes` com `{"placa": "ABC1D23"}` e, opcionalmente, `entrada`.
2. Sistema valida os dados e cria o bilhete com status `"aberto"`.
3. Sistema responde `201` com `id`, `placa`, `entrada` e `status`.

**Fluxos de erro:**

- Placa ausente ou inválida → `422 placa_invalida`
- `entrada` em formato inválido → `422 entrada_invalida`
- Placa com bilhete aberto → `409 bilhete_em_aberto`

**Critérios de aceite:**

| ID     | Dado                                    | Quando | Então                                                   |
| ------ | --------------------------------------- | ------ | ------------------------------------------------------- |
| CA-1.1 | Placa `"ABC1D23"` sem bilhete aberto    | Abrir  | `201`, status `"aberto"`, `entrada` com fuso `-03:00`   |
| CA-1.2 | Placa `"abc1d23"` ou com 6 caracteres   | Abrir  | `422 {"erro": "placa_invalida"}`                        |
| CA-1.3 | Placa válida e `entrada` `"ontem"`      | Abrir  | `422 {"erro": "entrada_invalida"}`                      |

### UC2 — Encerrar bilhete

**Descrição:** o operador encerra um bilhete aberto e recebe o valor a cobrar.

**Pré-condição:** existe um bilhete com o `id` informado.

**Regras aplicadas:** RN-01 (fração), RN-02 (teto), RN-03 (tolerância), RN-04 (valor inteiro).

**Fluxo principal:**

1. Cliente envia `POST /bilhetes/{id}/encerramento`.
2. Sistema registra `saida`, calcula `minutos` e `valor_centavos`.
3. Sistema responde `200` com o bilhete e `"status": "encerrado"`.

**Fluxos de erro:**

- `id` inexistente → `404 bilhete_nao_encontrado`
- Bilhete já encerrado ou cancelado → `409 bilhete_ja_encerrado`

**Critérios de aceite:**

| ID     | Dado                                      | Quando           | Então                                         |
| ------ | ----------------------------------------- | ---------------- | --------------------------------------------- |
| CA-2.1 | Bilhete aberto com `entrada` há 30 min    | Encerrar         | `200`, `minutos: 30`, `valor_centavos: 300`   |
| CA-2.2 | Bilhete aberto com `entrada` há 31 min    | Encerrar         | `valor_centavos: 600`                         |
| CA-2.3 | Bilhete aberto com `entrada` há 481 min   | Encerrar         | `valor_centavos: 5000`                        |
| CA-2.4 | Bilhete já encerrado                      | Encerrar de novo | `409 {"erro": "bilhete_ja_encerrado"}`        |

### UC3 — Listar ativos

**Descrição:** o operador consulta os bilhetes abertos no momento.

**Pré-condição:** nenhuma.

**Regras aplicadas:** RN-07 (ordenação).

**Fluxo principal:**

1. Cliente envia `GET /bilhetes/ativos`.
2. Sistema busca os bilhetes com status `"aberto"`.
3. Sistema responde `200` com o array, mais recentes primeiro.

**Fluxos de erro:**

- Nenhum.

**Critérios de aceite:**

| ID     | Dado                                              | Quando        | Então                               |
| ------ | ------------------------------------------------- | ------------- | ----------------------------------- |
| CA-3.1 | Bilhetes abertos com `entrada` há 60 e há 10 min  | Listar ativos | `200`, o de 10 min vem primeiro     |
| CA-3.2 | Um bilhete encerrado e um cancelado               | Listar ativos | Nenhum dos dois aparece             |

### UC4 — Relatório diário

**Descrição:** o operador consulta o resumo de um dia.

**Pré-condição:** nenhuma.

**Regras aplicadas:** RN-04 (valor inteiro), RN-08 (tempo médio).

**Fluxo principal:**

1. Cliente envia `GET /relatorios/diario?data=AAAA-MM-DD`.
2. Sistema conta os bilhetes do dia, soma o faturamento e calcula o tempo médio dos encerrados.
3. Sistema responde `200` com `data`, `total_bilhetes`, `faturamento_centavos` e `tempo_medio_minutos`.

**Fluxos de erro:**

- `data` ausente ou fora de `AAAA-MM-DD` → `422 data_invalida`

**Critérios de aceite:**

| ID     | Dado                                    | Quando                     | Então                                                          |
| ------ | --------------------------------------- | -------------------------- | -------------------------------------------------------------- |
| CA-4.1 | Dois encerrados hoje com 10 e 11 min    | Pedir o relatório de hoje  | `200`, `tempo_medio_minutos: 11`, `faturamento_centavos: 600`  |
| CA-4.2 | Dia sem bilhetes                        | Pedir o relatório          | `200` com os três números iguais a 0                           |
| CA-4.3 | Data `"05/10/2026"`                     | Pedir o relatório          | `422 {"erro": "data_invalida"}`                                |

### UC5 — Cancelar bilhete

**Descrição:** o operador cancela um bilhete aberto, sem cobrança.

**Pré-condição:** existe um bilhete aberto com o `id` informado.

**Regras aplicadas:** nenhuma regra de valor (cancelamento não cobra).

**Fluxo principal:**

1. Cliente envia `POST /bilhetes/{id}/cancelamento`.
2. Sistema muda o status para `"cancelado"`, sem registrar `saida` nem valor.
3. Sistema responde `200` com `id`, `placa`, `entrada` e `"status": "cancelado"`.

**Fluxos de erro:**

- `id` inexistente → `404 bilhete_nao_encontrado`
- Bilhete encerrado ou já cancelado → `409 bilhete_nao_aberto`

**Critérios de aceite:**

| ID     | Dado              | Quando   | Então                                                            |
| ------ | ----------------- | -------- | ---------------------------------------------------------------- |
| CA-5.1 | Bilhete aberto    | Cancelar | `200`, status `"cancelado"`, sem `saida` nem `valor_centavos`    |
| CA-5.2 | Bilhete encerrado | Cancelar | `409 {"erro": "bilhete_nao_aberto"}`                             |

### UC6 — Histórico por placa

**Descrição:** o operador consulta todos os bilhetes de uma placa.

**Pré-condição:** nenhuma.

**Regras aplicadas:** RN-05 (placa válida), RN-07 (ordenação).

**Fluxo principal:**

1. Cliente envia `GET /bilhetes?placa=ABC1D23`.
2. Sistema busca os bilhetes da placa, de qualquer status.
3. Sistema responde `200` com o array, mais recentes primeiro.

**Fluxos de erro:**

- `placa` inválida na query → `422 placa_invalida`

**Critérios de aceite:**

| ID     | Dado                                                       | Quando                | Então                                        |
| ------ | ---------------------------------------------------------- | --------------------- | -------------------------------------------- |
| CA-6.1 | Placa com um bilhete encerrado, um cancelado e um aberto   | Consultar o histórico | `200` com os três, mais recente primeiro     |
| CA-6.2 | Placa válida que nunca estacionou                          | Consultar o histórico | `200` e `[]`                                 |

### UC7 — Tolerância gratuita

**Descrição:** os primeiros minutos de tolerância são grátis; nesta variante a tolerância é 0.

**Pré-condição:** existe um bilhete aberto.

**Regras aplicadas:** RN-03 (tolerância), RN-01 (fração).

**Fluxo principal:**

1. Cliente encerra o bilhete (UC2).
2. Sistema compara `minutos` com a tolerância (0).
3. Se `minutos` for 0, `valor_centavos` é 0; se for 1 ou mais, cobra integral desde o primeiro minuto.

**Fluxos de erro:**

- Os mesmos do UC2.

**Critérios de aceite:**

| ID     | Dado                                    | Quando                 | Então                                |
| ------ | --------------------------------------- | ---------------------- | ------------------------------------ |
| CA-7.1 | Bilhete aberto agora                    | Encerrar imediatamente | `minutos: 0`, `valor_centavos: 0`    |
| CA-7.2 | Bilhete aberto com `entrada` há 1 min   | Encerrar               | `valor_centavos: 300`                |

### UC8 — Uma vaga por placa

**Descrição:** uma placa só pode ter um bilhete aberto por vez.

**Pré-condição:** nenhuma.

**Regras aplicadas:** RN-06 (uma vaga por placa).

**Fluxo principal:**

1. Cliente envia `POST /bilhetes` para uma placa.
2. Sistema verifica se a placa já tem bilhete aberto.
3. Se não tiver, cria o bilhete (UC1).

**Fluxos de erro:**

- Placa com bilhete aberto → `409 bilhete_em_aberto`

**Critérios de aceite:**

| ID     | Dado                                             | Quando      | Então                                |
| ------ | ------------------------------------------------ | ----------- | ------------------------------------ |
| CA-8.1 | Placa com bilhete aberto                         | Abrir outro | `409 {"erro": "bilhete_em_aberto"}`  |
| CA-8.2 | Placa cujo bilhete foi encerrado ou cancelado    | Abrir outro | `201` com `id` novo                  |
