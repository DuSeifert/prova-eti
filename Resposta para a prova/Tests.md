# Tests — Zona Azul Digital
 
Casos de borda por regra de negócio (entrada → esperado). Variante: tarifa 500,
fração 15 min, teto 8000, tolerância 15 min, porta 8003. `VALOR_FRACAO_CENTAVOS = 125`.
 
## Convenções de teste
- Relógio fixo em `T0 = 2026-10-05T14:30:00-03:00` (fixture; nunca `sleep`).
- "N min" = bilhete aberto com `entrada = T0 - N min` e encerrado em `T0`.
- Todo campo `_centavos` e `minutos` deve ser `int` (não `bool`, não `float`).
- Cada teste parte de estado limpo (repositório vazio, `id` volta a 1).

## 1. Cobrança — tolerância, fração e +1 minuto
 
| # | Minutos | `valor_centavos` | Regra testada |
| --- | --- | --- | --- |
| C1 | 0 | 0 | tolerância |
| C2 | 1 | 0 | tolerância |
| C3 | 14 | 0 | tolerância |
| C4 | 15 | 0 | tolerância exata (≤ 15 é grátis) |
| C5 | 16 | 250 | passou 1 min da tolerância: cobra 2 frações desde o início |
| C6 | 17 | 250 | dentro da 2ª fração |
| C7 | 30 | 250 | fração exata cobra 1 fração (2 frações no total) |
| C8 | 31 | 375 | +1 min cobra a fração seguinte (3) |
| C9 | 45 | 375 | fração exata |
| C10 | 46 | 500 | +1 min |
| C11 | 60 | 500 | hora cheia = tarifa |
| C12 | 61 | 625 | +1 min depois da hora |
| C13 | 120 | 1000 | duas horas |
 
## 2. Teto diário
 
> [!WARNING]
> `valor_centavos` nunca supera 8000. 64 frações = 8000 (960 min).
 
| # | Minutos | `valor_centavos` | Observação |
| --- | --- | --- | --- |
| T1 | 945 | 7875 | 63 frações, abaixo do teto |
| T2 | 946 | 8000 | 64 frações, exatamente o teto |
| T3 | 960 | 8000 | 64 frações |
| T4 | 961 | 8000 | 65 frações = 8125, limitado pelo teto |
| T5 | 1440 | 8000 | 24 h |
| T6 | 10000 | 8000 | muito acima do teto |
 
## 3. Cálculo de `minutos` (truncamento)
 
| # | Duração (entrada → saída) | `minutos` | `valor_centavos` |
| --- | --- | --- | --- |
| M1 | 31 min 59 s | 31 | 375 |
| M2 | 15 min 59 s | 15 | 0 |
| M3 | 16 min 00 s | 16 | 250 |
| M4 | 0 min 30 s | 0 | 0 |
| M5 | `entrada` no futuro (saída anterior à entrada) | 0 | 0 |
 
## 4. UC1 — Abrir bilhete
 
| # | Body | Esperado |
| --- | --- | --- |
| A1 | `{"placa":"ABC1D23"}` | 201; `id` 1; `status` `aberto`; `entrada` = T0 em `-03:00` |
| A2 | segundo bilhete com outra placa | 201; `id` 2 |
| A3 | `{"placa":"ABC1D23","entrada":"2026-10-05T14:30:00-03:00"}` | 201; `entrada` igual à enviada |
| A4 | `{"placa":"ABC1D23","entrada":"2026-10-05T17:30:00Z"}` | 201; `entrada` = `2026-10-05T14:30:00-03:00` |
| A5 | `{"placa":"1234567"}` (7 dígitos) | 201 |
| A6 | `{}` | 422 `placa_invalida` |
| A7 | `{"placa":null}` | 422 `placa_invalida` |
| A8 | `{"placa":1234567}` (número) | 422 `placa_invalida` |
| A9 | `{"placa":["ABC1D23"]}` | 422 `placa_invalida` |
| A10 | `{"placa":"abc1d23"}` | 422 `placa_invalida` |
| A11 | `{"placa":"ABC1D2"}` (6) | 422 `placa_invalida` |
| A12 | `{"placa":"ABC1D234"}` (8) | 422 `placa_invalida` |
| A13 | `{"placa":"ABC-D23"}` | 422 `placa_invalida` |
| A14 | `{"placa":"ABC 123"}` | 422 `placa_invalida` |
| A15 | `{"placa":""}` | 422 `placa_invalida` |
| A16 | corpo vazio | 422 `placa_invalida` |
| A17 | corpo `{` (JSON malformado) | 422 `placa_invalida` |
| A18 | corpo `[]` | 422 `placa_invalida` |
| A19 | placa válida + `"entrada":"ontem"` | 422 `entrada_invalida` |
| A20 | placa válida + `"entrada":"2026-10-05"` (só data) | 422 `entrada_invalida` |
| A21 | placa válida + `"entrada":"2026-10-05T14:30:00"` (sem fuso) | 422 `entrada_invalida` |
| A22 | placa válida + `"entrada":"2026-13-05T14:30:00-03:00"` | 422 `entrada_invalida` |
| A23 | placa válida + `"entrada":123` | 422 `entrada_invalida` |
| A24 | placa inválida **e** `entrada` inválida | 422 `placa_invalida` (placa vem primeiro) |
| A25 | após qualquer 422 | nenhum bilhete criado; próximo `id` continua sequencial |
 
## 5. UC2 — Encerrar bilhete
 
| # | Cenário | Esperado |
| --- | --- | --- |
| E1 | encerrar bilhete de 95 min | 200; chaves exatas `id, placa, entrada, saida, minutos, valor_centavos` (sem `status`); `minutos` 95; `valor_centavos` 875 |
| E2 | `saida` | = T0 em `-03:00` |
| E3 | `valor_centavos` e `minutos` | tipo `int`; JSON sem ponto decimal |
| E4 | encerrar `id` inexistente (999) | 404 `bilhete_nao_encontrado` |
| E5 | encerrar `id` = `abc` | 404 `bilhete_nao_encontrado` |
| E6 | encerrar `id` = `0` e `-1` | 404 `bilhete_nao_encontrado` |
| E7 | encerrar duas vezes o mesmo bilhete | 2ª: 409 `bilhete_ja_encerrado`; dados do 1º inalterados |
| E8 | encerrar bilhete cancelado | 409 `bilhete_ja_encerrado` |
| E9 | resposta nunca contém a chave `valor` | verdadeiro |
 
## 6. UC3 — Listar ativos
 
| # | Cenário | Esperado |
| --- | --- | --- |
| L1 | sem bilhetes | 200 `[]` |
| L2 | 3 abertos com `entrada` distintas | 200; ordenados por `entrada` decrescente |
| L3 | 2 abertos com a mesma `entrada` | maior `id` primeiro |
| L4 | 1 aberto, 1 encerrado, 1 cancelado | só o aberto aparece |
| L5 | item da lista | chaves `id, placa, entrada, status`; `status` `aberto` |
| L6 | `GET /bilhetes/ativos` | resposta de lista, nunca 404 `bilhete_nao_encontrado` |
 
## 7. UC4 — Relatório diário
 
Dia de referência: `2026-10-05`. Dia = data `-03:00` da `saida`.
 
### Tempo médio (arredondamento 0,5 para cima)
 
| # | Minutos dos encerrados do dia | Média real | `tempo_medio_minutos` |
| --- | --- | --- | --- |
| R1 | (nenhum) | — | 0 |
| R2 | 47 | 47 | 47 |
| R3 | 10, 15 | 12,5 | 13 |
| R4 | 10, 11 | 10,5 | 11 |
| R5 | 10, 10, 11 | 10,33 | 10 |
| R6 | 10, 11, 11 | 10,67 | 11 |
| R7 | 0, 1 | 0,5 | 1 |
 
### Totais e filtros
 
| # | Cenário | Esperado |
| --- | --- | --- |
| R8 | encerrados de 16 min (250) e 31 min (375) no dia | `total_bilhetes` 2; `faturamento_centavos` 625; `tempo_medio_minutos` 24 |
| R9 | dia sem encerramentos | `{"data":"2026-10-05","total_bilhetes":0,"faturamento_centavos":0,"tempo_medio_minutos":0}` |
| R10 | bilhete aberto e bilhete cancelado no dia | não entram em nenhum campo |
| R11 | bilhete encerrado com valor 0 (10 min) | conta em `total_bilhetes` e na média |
| R12 | `saida` = `2026-10-05T23:59:00-03:00` | entra no dia 05 |
| R13 | `saida` = `2026-10-06T00:00:00-03:00` | entra no dia 06, não no 05 |
| R14 | encerrados de dias diferentes | cada relatório conta só o seu dia |
| R15 | campos numéricos | `int`; `data` ecoa o parâmetro recebido |
 
### Parâmetro `data` inválido
 
| # | `data` | Esperado |
| --- | --- | --- |
| D1 | ausente | 422 `data_invalida` |
| D2 | `2026-13-01` | 422 `data_invalida` |
| D3 | `2026-02-30` | 422 `data_invalida` |
| D4 | `05/10/2026` | 422 `data_invalida` |
| D5 | `2026-1-5` | 422 `data_invalida` |
| D6 | `2026-10-05T10:00` | 422 `data_invalida` |
| D7 | vazio (`data=`) | 422 `data_invalida` |
 
## 8. UC5 — Cancelar bilhete
 
| # | Cenário | Esperado |
| --- | --- | --- |
| K1 | cancelar bilhete aberto | 200; `{id, placa, entrada, status:"cancelado"}` |
| K2 | resposta e bilhete cancelado | sem `saida`, `minutos`, `valor_centavos` (chaves ausentes, não `null`) |
| K3 | cancelar `id` inexistente | 404 `bilhete_nao_encontrado` |
| K4 | cancelar `id` = `abc` | 404 `bilhete_nao_encontrado` |
| K5 | cancelar bilhete encerrado | 409 `bilhete_nao_aberto` |
| K6 | cancelar duas vezes | 2ª: 409 `bilhete_nao_aberto` |
| K7 | bilhete cancelado | some de `/bilhetes/ativos` |
 
## 9. UC6 — Histórico por placa
 
| # | Cenário | Esperado |
| --- | --- | --- |
| H1 | placa com bilhetes aberto, encerrado e cancelado | 200; os 3, qualquer status |
| H2 | ordem | `entrada` decrescente; empate por `id` decrescente |
| H3 | item encerrado | inclui `saida`, `minutos`, `valor_centavos` |
| H4 | item aberto ou cancelado | só `id, placa, entrada, status` |
| H5 | placa válida que nunca estacionou | 200 `[]` |
| H6 | bilhetes de outras placas | não aparecem |
| H7 | `placa` ausente | 422 `placa_invalida` |
| H8 | `placa=abc1d23` (minúscula) | 422 `placa_invalida` |
| H9 | `GET /bilhetes` e `GET /bilhetes/ativos` | respostas distintas (rota não confundida) |
 
## 10. UC7 e UC8 — Tolerância e uma vaga por placa
 
Tolerância: casos C1–C5 (grátis até 15) e C5 (16 min cobra 250) cobrem o UC7;
M2 cobre 15 min 59 s = grátis. Adicional:
 
| # | Cenário | Esperado |
| --- | --- | --- |
| U1 | bilhete de 15 min | 200; `valor_centavos` 0; `status` encerrado |
| U2 | bilhete de 16 min | `valor_centavos` 250 (não 125) |
 
| # | Cenário | Esperado |
| --- | --- | --- |
| V1 | abrir placa com bilhete aberto | 409 `bilhete_em_aberto`; nenhum bilhete novo |
| V2 | abrir, encerrar, abrir de novo a mesma placa | 2º `POST`: 201, novo `id` |
| V3 | abrir, cancelar, abrir de novo a mesma placa | 2º `POST`: 201, novo `id` |
| V4 | duas placas diferentes abertas | ambas 201 |
| V5 | conflito com `entrada` explícita diferente | 409 `bilhete_em_aberto` |
| V6 | placa inválida **e** com bilhete aberto | 422 `placa_invalida` (validação antes do conflito) |
 
## 11. Formato dos erros (transversal)
 
| # | Cenário | Esperado |
| --- | --- | --- |
| F1 | qualquer resposta de erro | corpo exatamente `{"erro": "<codigo>"}`; sem chave `detail` |
| F2 | rota inexistente (`GET /xyz`) | corpo `{"erro": ...}`; nunca `detail` |
| F3 | todas as entradas inválidas dos casos A6–A23, D1–D7 | nenhuma resposta 500 |
| F4 | `Content-Type` de sucesso e de erro | `application/json` |
| F5 | todo erro | estado inalterado (verificar `GET /bilhetes/ativos` antes e depois) |
 
## 12. Rastreabilidade (regra → casos)
 
| Regra de negócio | Casos |
| --- | --- |
| Tolerância (UC7) | C1–C5, M2, U1, U2 |
| Fração exata / +1 minuto | C7–C12 |
| Teto | T1–T6 |
| Truncamento de minutos | M1–M5 |
| Tempo médio (0,5 para cima) | R1–R7 |
| Relatório do dia | R8–R15, D1–D7 |
| Centavos inteiros | E3, E9, R15 |
| Conflito de placa (UC8) | V1–V6 |
| Cancelamento | K1–K7 |
| Erros 404/409/422 | A6–A25, E4–E8, K3–K6, H7–H8, F1–F5 |
