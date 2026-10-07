# Constitution — Zona Azul Digital

Convenções persistentes. Valem para toda tarefa e para todo arquivo gerado. Em caso de conflito entre este documento e qualquer exemplo, **vale o contrato do enunciado e este documento**. Só a API REST é escopo (sem back-office, sem UI).

## 1. Parâmetros da variante

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

## 3. Tempo
Todo instante é ISO-8601 com fuso -03:00 (entradas em outro fuso são convertidas).
Relógio injetável: uma única função agora(); nada mais lê o relógio do sistema.
minutos é inteiro: duração em segundos truncada para minutos, mínimo 0.

## 4. Erros
Todo erro é JSON {"erro": "<codigo>"}, sem campos extras, e nunca altera estado.
Usar apenas os códigos e status do enunciado; entrada inválida nunca gera 500.
Todo endpoint documenta seus status de erro.

## 5. Dados
Placa: ^[A-Z0-9]{7}$, sem normalização.
id inteiro sequencial a partir de 1, nunca reutilizado.
Campos ausentes são omitidos, não null.
Listagens: mais recentes primeiro (por entrada, desempate por id decrescente).

## 6. Contrato
O contrato do enunciado é a fonte de verdade; exemplos que o contradigam são ignorados.
Nomes de campos exatamente como no contrato.
Não criar endpoints, campos ou erros não especificados.

## 7. Limites dos .md

Especificar, não implementar: snippets com no máximo 20 linhas.
