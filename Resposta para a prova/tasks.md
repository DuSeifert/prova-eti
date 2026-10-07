# Tasks — Zona Azul Digital
 
Executar na ordem das dependências. Os detalhes de cada tarefa estão nos
documentos referenciados: regras em `spec.md`, decisões em `plan.md`,
convenções em `constitution.md` e casos de teste em `tests.md`.
Uma tarefa só termina quando seus testes passam.
 
### T1 — Base do projeto
- **Depende de:** —
- **Fazer:** `config.py`, `relogio.py`, `erros.py` e `requirements.txt`.
- **Referências:** constitution 1 a 5; plan D1, D2, D4, D5, D7.
- **Conclusão:** erro de qualquer rota sai como `{"erro": ...}` (tests F1, F2).
### T2 — Cálculo de cobrança
- **Depende de:** T1
- **Fazer:** `cobranca.py` com minutos, valor e tempo médio, mais testes unitários.
- **Referências:** spec "Regra de cobrança" e UC4 (4.4); plan D6; tests 1, 2, 3 e R1–R7.
- **Conclusão:** casos C1–C13, T1–T6, M1–M5 e R1–R7 passam.
### T3 — Repositório em memória
- **Depende de:** T1
- **Fazer:** `repositorio.py` com armazenamento, lock e contador de `id`.
- **Referências:** plan D3; constitution 6.
- **Conclusão:** `id` sequencial a partir de 1, sem reutilização; estado limpo por fixture.
### T4 — Abrir bilhete e conflito de placa
- **Depende de:** T1, T3
- **Fazer:** `POST /bilhetes` em `main.py`.
- **Referências:** spec UC1 e UC8; plan D8, D9; tests 4 e 10 (V1–V6).
- **Conclusão:** casos A1–A25 e V1–V6 passam.
### T5 — Encerrar bilhete com tolerância e teto
- **Depende de:** T2, T4
- **Fazer:** `POST /bilhetes/{id}/encerramento`.
- **Referências:** spec UC2 e UC7; plan D5, D6, D9; tests 1, 2, 3, 5 e U1–U2.
- **Conclusão:** casos E1–E9, C1–C13, T1–T6, M1–M5 e U1–U2 passam.
### T6 — Ativos, cancelamento e histórico
- **Depende de:** T4
- **Fazer:** `GET /bilhetes/ativos`, `POST /bilhetes/{id}/cancelamento`, `GET /bilhetes?placa=`.
- **Referências:** spec UC3, UC5 e UC6; plan D8, D9; tests 6, 8 e 9.
- **Conclusão:** casos L1–L6, K1–K7 e H1–H9 passam.
### T7 — Relatório diário
- **Depende de:** T2, T5
- **Fazer:** `GET /relatorios/diario`.
- **Referências:** spec UC4; plan D6, D8; tests 7.
- **Conclusão:** casos R1–R15 e D1–D7 passam.
### T8 — Docker
- **Depende de:** T1
- **Fazer:** `Dockerfile`, `.dockerignore` e `README.md`.
- **Referências:** constitution 1; spec N1, N6 a N9; plan D10.
- **Conclusão:** `docker build -t zona-azul .` conclui sem erro.
### T9 — Verificação final
- **Depende de:** T1 a T8
- **Fazer:** rodar o serviço e a suíte completa no container; corrigir o que falhar.
- **Referências:** spec N1 a N9; plan D10, D11; tests 11 e 12.
- **Conclusão:** `docker run --rm zona-azul python -m pytest` termina com código 0, e o serviço responde em `http://localhost:8003`.
 
