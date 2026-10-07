# Tests — Zona Azul Digital
 
Casos de borda por regra de negócio (entrada → esperado). Variante: tarifa 500,
fração 15 min, teto 8000, tolerância 15 min, porta 8003. `VALOR_FRACAO_CENTAVOS = 125`.
 
## Convenções de teste
- Relógio fixo em `T0 = 2026-10-05T14:30:00-03:00` (fixture; nunca `sleep`).
- "N min" = bilhete aberto com `entrada = T0 - N min` e encerrado em `T0`.
- Todo campo `_centavos` e `minutos` deve ser `int` (não `bool`, não `float`).
- Cada teste parte de estado limpo (repositório vazio, `id` volta a 1).
