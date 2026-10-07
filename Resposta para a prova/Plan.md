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
