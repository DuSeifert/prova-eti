# Plan — Zona Azul Digital
 
Decisões técnicas com justificativa. Regras de negócio estão em `spec.md`;
convenções transversais em `constitution.md`.
 
## D1. Stack: Python 3.11+, FastAPI, Uvicorn, pytest (+ httpx)
- **Por quê:** FastAPI entrega uma API REST JSON com pouco código; pytest com
  `TestClient` (requer `httpx`) testa a API sem subir servidor.
- Python 3.11+ porque `datetime.fromisoformat` aceita ISO-8601 com fuso e o sufixo `Z`.
- Fixar as versões instaladas em `requirements.txt` (`fastapi`, `uvicorn`, `pytest`, `httpx`).
## D2. Estrutura de arquivos
```text
app/
  config.py      # constantes e derivados (única fonte)
  relogio.py     # agora() injetável
  erros.py       # ErroApi e handlers
  cobranca.py    # funções puras: minutos, valor, média
  repositorio.py # armazenamento em memória
  main.py        # app FastAPI e rotas
tests/
  conftest.py    # fixtures: cliente, estado limpo, relógio
  test_*.py
requirements.txt
```
- **Por quê:** `cobranca.py` com funções puras permite testar fração, teto e média
  sem HTTP; `config.py` e `relogio.py` garantem as regras de fonte única da constitution.

## D3. Persistência: memória do processo
- Estrutura: dict `id → bilhete` e dict `placa → id` do bilhete aberto.
- **Por quê:** o enunciado não exige durabilidade; a suíte de correção sobe o
  serviço com estado limpo; evita dependência de banco e arquivos.
- Um `threading.Lock` protege toda operação de leitura-modificação-escrita.
  **Por quê:** endpoints síncronos do FastAPI rodam em threadpool, e o UC8
  (uma vaga por placa) não pode ter condição de corrida.
- Contador de `id` começa em 1 e nunca é reutilizado.
## D4. Relógio injetável
- `relogio.agora()` retorna `datetime` com fuso `-03:00`; é a única leitura de
  tempo do sistema.
- Os testes substituem `agora` via `monkeypatch`/fixture.
- **Por quê:** testar `saida` sem `sleep` e de forma determinística.
## D5. Tempo e `minutos`
- Parse de `entrada` com `datetime.fromisoformat`; **rejeitar** se `tzinfo` for
  `None` ou se o valor não for string. Converter para `-03:00` e descartar microssegundos.
- Serializar com `isoformat()` (ex.: sufixo `-03:00`).
- `minutos = max(0, int((saida - entrada).total_seconds()) // 60)`.
- **Por quê (truncar):** a suíte abre com `entrada` = agora − N min e encerra
  milissegundos depois; truncar devolve exatamente N, enquanto arredondar para cima daria N+1.
## D6. Cobrança e relatório (módulo `cobranca.py`)
- Funções puras e inteiras: `calcular_minutos`, `calcular_valor_centavos`,
  `tempo_medio_arredondado`. Seguem a regra de cobrança do `spec.md`.
- Média: `(2*soma + n) // (2*n)`; com `n = 0` retorna 0 antes de dividir.
- **Por quê (inteiros):** ponto flutuante acumula erro (0.1 + 0.2 ≠ 0.3) e a
  suíte testa essa classe de bug; centavos inteiros a eliminam.
- Nenhum arredondamento de meio-par (half-even): o contrato manda 0,5 para cima.
## D7. Erros no formato do contrato
- Criar `ErroApi(status, codigo)` e um handler que responde `{"erro": codigo}`.
- Registrar também handlers para `RequestValidationError` e `HTTPException`
  (inclusive 404/405 de rotas) que respondam no mesmo formato, para que nunca
  apareça `{"detail": ...}`.
- Handler genérico para `Exception` responde 500 apenas para falhas inesperadas.
- **Por quê:** o padrão do FastAPI é `{"detail": ...}` com 422 automático,
  o que quebraria o contrato.
## D8. Validação manual (sem modelos Pydantic de entrada)
- `POST /bilhetes`: ler o corpo com `await request.json()` dentro de `try`;
  JSON malformado, corpo ausente ou que não seja objeto → `placa_invalida`.
  Validar na ordem: placa → entrada → conflito (spec UC1).
- `placa`: `isinstance(str)` e regex `^[A-Z0-9]{7}$` (usar `fullmatch`).
- `GET /bilhetes`: `placa` como query opcional; ausente ou inválida → `placa_invalida`.
- `GET /relatorios/diario`: `data` como query opcional; exigir regex
  `^\d{4}-\d{2}-\d{2}$` e depois `datetime.strptime` para rejeitar datas
  inexistentes (ex.: 30 de fevereiro); falha → `data_invalida`.
- **Por quê:** a validação automática do Pydantic gera 422 com outro corpo e
  coage tipos (ex.: aceita número como placa), divergindo do contrato.
## D9. Rotas
- Declarar `GET /bilhetes/ativos` **antes** de qualquer rota com `{id}`.
- `{id}` recebido como `str`: se não for inteiro positivo ou não existir →
  404 `bilhete_nao_encontrado`.
- **Por quê:** evita conflito `ativos` × `{id}` e impede o 422 automático para id não numérico.
- Rotas: `POST /bilhetes`, `GET /bilhetes`, `GET /bilhetes/ativos`,
  `POST /bilhetes/{id}/encerramento`, `POST /bilhetes/{id}/cancelamento`,
  `GET /relatorios/diario`. Endpoints `def` síncronos, protegidos pelo lock (D3).
## D10. Execução
- Comando: `uvicorn app.main:app --host 0.0.0.0 --port 8003`.
- Porta lida de `config.PORTA_SERVICO`; instruir também um bloco
  `if __name__ == "__main__"` que sobe o servidor com essa porta.
- Incluir `README` curto com instalar (`pip install -r requirements.txt`),
  subir e testar (`pytest`).
## D11. Estratégia de testes
- pytest + `TestClient`, uma fixture que limpa o repositório e o contador de id a cada teste.
- Dois níveis: testes unitários de `cobranca.py` e testes de contrato HTTP por UC.
- Tempo controlado pela fixture de relógio; nunca `sleep`.
- Casos concretos vêm de `tests.md`; cada critério do `spec.md` tem ao menos um teste.
- Os testes de contrato verificam o tipo inteiro dos campos em centavos.
## D12. Decisões em casos omissos
- Corpo extra no encerramento e no cancelamento é ignorado.
- Campos desconhecidos no body do `POST /bilhetes` são ignorados.
- `entrada` no futuro é aceita; se o encerramento ocorrer antes dela, `minutos = 0`.
