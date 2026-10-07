> Leia `fatos.md` antes deste arquivo: ele é a Fonte da Verdade do projeto.

## Tarefas

- **TK-01 Projeto base.** Criar `package.json` (dependências e scripts `start`, `test`, `lint`), `src/server.js`, `src/app.js` e `src/db.js` com a tabela `bilhetes`. *Pronto quando:* `npm start` sobe o serviço em `0.0.0.0:8005` e o banco é criado sozinho.
- **TK-02 Cálculo de valor.** Criar `src/pricing.js` com `calcularValor(minutos)` aplicando fração, tolerância e teto. *Pronto quando:* passam testes unitários de `calcularValor` para a tabela de minutos do `fatos.md`: 0 → 0, 1 → 300, 29 → 300, 30 → 300, 31 → 600, 60 → 600, 61 → 900, 95 → 1200, 480 → 4800, 481 → 5000, 1440 → 5000.
- **TK-03 Abrir, encerrar e cancelar.** Implementar `POST /bilhetes`, `POST /bilhetes/{id}/encerramento` e `POST /bilhetes/{id}/cancelamento`, com todos os erros da tabela. *Pronto quando:* passam os testes CT-01 a CT-20 e CT-30 a CT-34.
- **TK-04 Listagens e relatório.** Implementar `GET /bilhetes/ativos`, `GET /bilhetes?placa=` e `GET /relatorios/diario?data=`. *Pronto quando:* passam os testes CT-21 a CT-29.
- **TK-05 Entrega.** Criar `Dockerfile` e `Containerfile` (idênticos, com `EXPOSE 8005` e `CMD`), `README.md` com os comandos de execução, testes, Docker e Podman, e `.gitignore`. *Pronto quando:* `npm test` passa com os 34 testes e a imagem sobe respondendo na porta 8005.