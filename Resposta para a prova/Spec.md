# Spec — Zona Azul Digital
 
Requisitos com critérios de aceite mensuráveis. Convenções gerais (centavos,
fuso, erros, formato) estão em `constitution.md`. Base URL: `http://localhost:8003`.
 
## Modelo de bilhete
| Campo | Presente quando |
| --- | --- |
| `id`, `placa`, `entrada`, `status` | sempre |
| `saida`, `minutos`, `valor_centavos` | somente `status = encerrado` |
 
`status` ∈ {`aberto`, `encerrado`, `cancelado`}. Transições permitidas:
`aberto → encerrado` e `aberto → cancelado`.
 
## Regra de cobrança (usada no UC2 e no UC7)
Aplicar nesta ordem:
1. `minutos <= TOLERANCIA_MINUTOS` → `valor_centavos = 0`.
2. Senão, `fracoes = ceil(minutos / FRACAO_MINUTOS)` contando desde o primeiro minuto (tolerância não é descontada).
3. `valor = fracoes * VALOR_FRACAO_CENTAVOS`.
4. `valor_centavos = min(valor, TETO_DIARIO_CENTAVOS)`.

## UC1 — Abrir bilhete
`POST /bilhetes` · body `{"placa": "...", "entrada": "..."}` (`entrada` opcional)
 
Ordem de validação: placa → entrada → bilhete aberto da placa.
 
| # | Critério de aceite | Resultado |
| --- | --- | --- |
| 1.1 | Placa válida, sem `entrada` | 201 `{id, placa, entrada, status:"aberto"}`; `entrada` = `agora()` em `-03:00` |
| 1.2 | Placa válida com `entrada` ISO-8601 com fuso | 201; `entrada` devolvida igual ao instante enviado, em `-03:00` |
| 1.3 | `id` do primeiro bilhete | `1`; cada novo bilhete tem `id` anterior + 1 |
| 1.4 | Placa ausente, não string, minúscula ou ≠ 7 caracteres | 422 `placa_invalida` |
| 1.5 | `entrada` sem fuso ou fora de ISO-8601 | 422 `entrada_invalida` |
| 1.6 | Body ausente ou JSON malformado | 422 `placa_invalida` |
| 1.7 | Placa e `entrada` ambas inválidas | 422 `placa_invalida` |
| 1.8 | Placa com bilhete `aberto` | 409 `bilhete_em_aberto` (nenhum bilhete criado) |

## UC2 — Encerrar bilhete
`POST /bilhetes/{id}/encerramento` · sem body
 
| # | Critério de aceite | Resultado |
| --- | --- | --- |
| 2.1 | Bilhete aberto | 200 `{id, placa, entrada, saida, minutos, valor_centavos}` (sem `status`) |
| 2.2 | `saida` | `agora()` em `-03:00` |
| 2.3 | `minutos` | inteiro: `floor((saida - entrada) em segundos / 60)`, mínimo 0 |
| 2.4 | `valor_centavos` | inteiro, calculado pela regra de cobrança, nunca > 8000 |
| 2.5 | Bilhete passa a `status = encerrado` | refletido em UC3, UC4 e UC6 |
| 2.6 | `id` inexistente ou não numérico | 404 `bilhete_nao_encontrado` |
| 2.7 | Bilhete `encerrado` ou `cancelado` | 409 `bilhete_ja_encerrado`; estado inalterado |
| 2.8 | Resposta | nenhum campo numérico com ponto decimal |
 
## UC3 — Listar ativos
`GET /bilhetes/ativos`
 
| # | Critério de aceite | Resultado |
| --- | --- | --- |
| 3.1 | Retorno | 200 array apenas de bilhetes `aberto`, cada item `{id, placa, entrada, status}` |
| 3.2 | Ordem | `entrada` decrescente; empate por `id` decrescente |
| 3.3 | Nenhum aberto | 200 `[]` |
| 3.4 | Rota | `/bilhetes/ativos` nunca é interpretada como `{id}` |
 
## UC4 — Relatório diário
`GET /relatorios/diario?data=AAAA-MM-DD`
 
Um bilhete pertence ao dia se está `encerrado` e a data local (`-03:00`) da sua `saida` é igual a `data`.
 
| # | Critério de aceite | Resultado |
| --- | --- | --- |
| 4.1 | Retorno | 200 `{data, total_bilhetes, faturamento_centavos, tempo_medio_minutos}` |
| 4.2 | `total_bilhetes` | quantidade de bilhetes do dia |
| 4.3 | `faturamento_centavos` | soma de `valor_centavos` dos bilhetes do dia (inteiro) |
| 4.4 | `tempo_medio_minutos` | média de `minutos` dos bilhetes do dia, arredondada com 0,5 para cima, em inteiros: `(2*soma + n) // (2*n)` |
| 4.5 | Dia sem bilhetes | `total_bilhetes: 0`, `faturamento_centavos: 0`, `tempo_medio_minutos: 0` |
| 4.6 | Cancelados e abertos | nunca entram em nenhum campo |
| 4.7 | `data` ausente, fora de `AAAA-MM-DD` ou data de calendário inexistente | 422 `data_invalida` |
 
## UC5 — Cancelar bilhete
`POST /bilhetes/{id}/cancelamento` · sem body
 
| # | Critério de aceite | Resultado |
| --- | --- | --- |
| 5.1 | Bilhete aberto | 200 `{id, placa, entrada, status:"cancelado"}` |
| 5.2 | Resposta e dados | sem `saida`, `minutos` nem `valor_centavos` (campos omitidos) |
| 5.3 | `id` inexistente ou não numérico | 404 `bilhete_nao_encontrado` |
| 5.4 | Bilhete `encerrado` ou `cancelado` | 409 `bilhete_nao_aberto`; estado inalterado |
 
## UC6 — Histórico por placa
`GET /bilhetes?placa=...`
 
| # | Critério de aceite | Resultado |
| --- | --- | --- |
| 6.1 | Placa válida com histórico | 200 array com bilhetes de qualquer status |
| 6.2 | Campos por item | `{id, placa, entrada, status}`; encerrados incluem também `saida`, `minutos`, `valor_centavos` |
| 6.3 | Ordem | `entrada` decrescente; empate por `id` decrescente |
| 6.4 | Placa válida sem histórico | 200 `[]` |
| 6.5 | Placa ausente ou inválida | 422 `placa_invalida` |
| 6.6 | Bilhetes de outras placas | nunca aparecem |
 
## UC7 — Tolerância gratuita
Aplicada no encerramento, pela regra de cobrança. Valores esperados com a variante (tolerância 15):
 
| # | Critério de aceite | Resultado |
| --- | --- | --- |
| 7.1 | `minutos` ≤ 15 (incluindo 0) | `valor_centavos = 0` |
| 7.2 | `minutos` = 16 | `valor_centavos` ≠ 0, cobrado desde o primeiro minuto |
| 7.3 | Tolerância | nunca é subtraída do tempo cobrado |
| 7.4 | Bilhete com valor 0 | continua `encerrado` e entra no relatório do dia |
 
## UC8 — Uma vaga por placa
 
| # | Critério de aceite | Resultado |
| --- | --- | --- |
| 8.1 | Abrir com placa que tem bilhete aberto | 409 `bilhete_em_aberto` |
| 8.2 | Abrir após encerrar o bilhete anterior da placa | 201 com novo `id` |
| 8.3 | Abrir após cancelar o bilhete anterior da placa | 201 com novo `id` |
| 8.4 | Placas diferentes | independentes; podem ter bilhetes abertos ao mesmo tempo |

## Requisitos não funcionais
| # | Critério |
| --- | --- |
| N1 | Serviço escuta na porta 8003 e responde após a subida sem passos manuais |
| N2 | Todas as respostas são JSON, inclusive erros |
| N3 | Nenhuma entrada inválida resulta em status 500 |
| N4 | Nenhum erro altera o estado do sistema |
| N5 | Nenhum campo `valor`, nem número com ponto decimal, em qualquer resposta |
 
