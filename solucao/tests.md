> Leia `fatos.md` antes deste arquivo: ele é a Fonte da Verdade do projeto.

## Casos de teste

**Total de casos de teste: 34.** O código gerado deve ter pelo menos 34 testes
automatizados, um para cada CT, com o ID no nome do teste (ex.: `CT-03`).

Convenções:
- Cada teste começa com banco vazio (`:memory:`) e relógio falso fixo em `2026-10-05T12:00:00-03:00`. Esse é o único instante fixo dos testes; nenhum teste espera tempo real.
- "Entrada há N min" significa enviar `entrada` igual ao instante do relógio falso menos N minutos, com fuso `-03:00`. Toda `entrada` é calculada a partir do relógio, nunca escrita à mão.
- "Encerrar" é `POST /bilhetes/{id}/encerramento`. "Cancelar" é `POST /bilhetes/{id}/cancelamento`.
- `<hoje em AAAA-MM-DD>` é a data do relógio falso no fuso `-03:00` (`2026-10-05`). Na URL vai a data, nunca a palavra "hoje".

### RN-01 Fração (30 min, arredonda para cima)

| ID    | Preparação                              | Requisição | Esperado                                   |
| ----- | --------------------------------------- | ---------- | ------------------------------------------ |
| CT-01 | `POST /bilhetes` com entrada há 29 min  | Encerrar   | `200`, `minutos: 29`, `valor_centavos: 300` |
| CT-02 | `POST /bilhetes` com entrada há 30 min  | Encerrar   | `200`, `minutos: 30`, `valor_centavos: 300` |
| CT-03 | `POST /bilhetes` com entrada há 31 min  | Encerrar   | `200`, `minutos: 31`, `valor_centavos: 600` |
| CT-04 | `POST /bilhetes` com entrada há 60 min  | Encerrar   | `200`, `valor_centavos: 600`                |
| CT-05 | `POST /bilhetes` com entrada há 61 min  | Encerrar   | `200`, `valor_centavos: 900`                |

### RN-02 Teto (5000 centavos)

> [!WARNING]
> O teto vale por bilhete, mesmo que dure mais de um dia.

| ID    | Preparação                               | Requisição | Esperado                      |
| ----- | ---------------------------------------- | ---------- | ----------------------------- |
| CT-06 | `POST /bilhetes` com entrada há 480 min  | Encerrar   | `200`, `valor_centavos: 4800` |
| CT-07 | `POST /bilhetes` com entrada há 481 min  | Encerrar   | `200`, `valor_centavos: 5000` |
| CT-08 | `POST /bilhetes` com entrada há 1440 min | Encerrar   | `200`, `valor_centavos: 5000` |

### RN-03 Tolerância (0 min)

| ID    | Preparação                            | Requisição | Esperado                                  |
| ----- | ------------------------------------- | ---------- | ----------------------------------------- |
| CT-09 | `POST /bilhetes` sem `entrada`        | Encerrar   | `200`, `minutos: 0`, `valor_centavos: 0`   |
| CT-10 | `POST /bilhetes` com entrada há 1 min | Encerrar   | `200`, `minutos: 1`, `valor_centavos: 300` |

### RN-04 Valor inteiro

| ID    | Preparação                             | Requisição | Esperado                                                                  |
| ----- | -------------------------------------- | ---------- | ------------------------------------------------------------------------- |
| CT-11 | `POST /bilhetes` com entrada há 95 min | Encerrar   | `200`, `valor_centavos: 1200`, número inteiro; não existe o campo `valor` |

### RN-05 Placa e entrada válidas

| ID    | Preparação | Requisição                                              | Esperado                                                        |
| ----- | ---------- | ------------------------------------------------------- | --------------------------------------------------------------- |
| CT-12 | Nenhuma    | `POST /bilhetes` com `{"placa": "ABC1D23"}`             | `201` com `id`, `placa`, `entrada` em `-03:00`, `status: "aberto"` |
| CT-13 | Nenhuma    | `POST /bilhetes` com `{"placa": "ABC1D2"}`              | `422 {"erro": "placa_invalida"}`                                |
| CT-14 | Nenhuma    | `POST /bilhetes` com `{"placa": "ABC1D234"}`            | `422 {"erro": "placa_invalida"}`                                |
| CT-15 | Nenhuma    | `POST /bilhetes` com `{"placa": "abc1d23"}`             | `422 {"erro": "placa_invalida"}`                                |
| CT-16 | Nenhuma    | `POST /bilhetes` com `{}`                               | `422 {"erro": "placa_invalida"}`, nunca `400`                   |
| CT-17 | Nenhuma    | `POST /bilhetes` com `{"placa": "ABC1D23", "entrada": "ontem"}` | `422 {"erro": "entrada_invalida"}`                      |

### RN-06 Uma vaga por placa

| ID    | Preparação                            | Requisição                        | Esperado                            |
| ----- | ------------------------------------- | --------------------------------- | ----------------------------------- |
| CT-18 | Bilhete aberto para `ABC1D23`         | `POST /bilhetes` com a mesma placa | `409 {"erro": "bilhete_em_aberto"}` |
| CT-19 | Bilhete de `ABC1D23` aberto e encerrado | `POST /bilhetes` com a mesma placa | `201` com `id` novo               |
| CT-20 | Bilhete de `ABC1D23` aberto e cancelado | `POST /bilhetes` com a mesma placa | `201` com `id` novo               |

### RN-07 Ordenação e listagens

| ID    | Preparação                                                              | Requisição                   | Esperado                                         |
| ----- | ----------------------------------------------------------------------- | ---------------------------- | ------------------------------------------------ |
| CT-21 | Três placas abertas com entrada há 60, 10 e 30 min                      | `GET /bilhetes/ativos`       | `200`, ordem: 10, 30, 60 min                     |
| CT-22 | Três bilhetes: um aberto, um encerrado, um cancelado                    | `GET /bilhetes/ativos`       | `200`, só o aberto                               |
| CT-23 | Mesma placa: encerrado (há 120 min), cancelado (há 60 min), aberto (há 5 min) | `GET /bilhetes?placa=ABC1D23` | `200`, três itens, do mais recente ao mais antigo |
| CT-24 | Nenhuma                                                                 | `GET /bilhetes?placa=ZZZ9Z99` | `200`, `[]`                                      |

### RN-08 Relatório e tempo médio (0,5 para cima)

| ID    | Preparação                          | Requisição                                         | Esperado                                                                        |
| ----- | ----------------------------------- | -------------------------------------------------- | ------------------------------------------------------------------------------- |
| CT-25 | Encerrados hoje com 10 e 11 min     | `GET /relatorios/diario?data=<hoje em AAAA-MM-DD>` | `200`, `total_bilhetes: 2`, `faturamento_centavos: 600`, `tempo_medio_minutos: 11` |
| CT-26 | Encerrados hoje com 10, 10 e 11 min | `GET /relatorios/diario?data=<hoje em AAAA-MM-DD>` | `200`, `tempo_medio_minutos: 10`                                                |
| CT-27 | Encerrados hoje com 10, 11 e 11 min | `GET /relatorios/diario?data=<hoje em AAAA-MM-DD>` | `200`, `tempo_medio_minutos: 11`                                                |
| CT-28 | Nenhuma                             | `GET /relatorios/diario?data=<hoje em AAAA-MM-DD>` | `200`, `total_bilhetes: 0`, `faturamento_centavos: 0`, `tempo_medio_minutos: 0` |
| CT-29 | Nenhuma                             | `GET /relatorios/diario?data=05/10/2026`           | `422 {"erro": "data_invalida"}`                                                 |

### Encerramento e cancelamento

| ID    | Preparação                  | Requisição                           | Esperado                                                         |
| ----- | --------------------------- | ------------------------------------ | ---------------------------------------------------------------- |
| CT-30 | Nenhuma                     | `POST /bilhetes/123/encerramento` | `404 {"erro": "bilhete_nao_encontrado"}`                         |
| CT-31 | Bilhete já encerrado        | Encerrar de novo                     | `409 {"erro": "bilhete_ja_encerrado"}`                           |
| CT-32 | Bilhete aberto              | Cancelar                             | `200`, `status: "cancelado"`, sem `saida` nem `valor_centavos`   |
| CT-33 | Bilhete já encerrado        | Cancelar                             | `409 {"erro": "bilhete_nao_aberto"}`                             |
| CT-34 | Nenhuma                     | `POST /bilhetes/123/cancelamento` | `404 {"erro": "bilhete_nao_encontrado"}`                         |