# Constitution — Zona Azul Digital

Convenções persistentes. Valem para toda tarefa e para todo arquivo gerado. Em caso de conflito entre este documento e qualquer exemplo, **vale o contrato do enunciado e este documento**. Só a API REST é escopo (sem back-office, sem UI).

## 1. Parâmetros da variante (constantes nomeadas, nunca números mágicos)

|Constante|Valor|
|---|---|
|`TARIFA_HORA_CENTAVOS`|500|
|`FRACAO_MINUTOS`|15|
|`TETO_DIARIO_CENTAVOS`|8000|
|`TOLERANCIA_MINUTOS`|15|
|`PORTA_SERVICO`|8003|

Derivados (calculados em inteiros a partir das constantes):

- `FRACOES_POR_HORA = 60 // FRACAO_MINUTOS` = **4**
- `VALOR_FRACAO_CENTAVOS = TARIFA_HORA_CENTAVOS // FRACOES_POR_HORA` = **125**

Regras operacionais:

1. Definir constantes e derivados em **um único módulo de configuração**; nenhum outro arquivo repete esses valores como literais.
2. O serviço escuta em `0.0.0.0:8003`. Base URL: `http://localhost:8003`.
3. Toda resposta (sucesso ou erro) é `application/json`.