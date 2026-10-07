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
