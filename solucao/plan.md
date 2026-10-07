> Leia `fatos.md` antes deste arquivo: ele é a Fonte da Verdade do projeto.

## Estrutura de arquivos

- `src/server.js`: abre o banco e sobe o servidor.
- `src/app.js`: exporta `createApp({ db, clock })` com todas as rotas. A rota `/bilhetes/ativos` é registrada antes das rotas com `{id}`.
- `src/db.js`: cria a tabela `bilhetes` se não existir.
- `src/pricing.js`: função `calcularValor(minutos)`.
- `test/`: testes automatizados.
- Raiz: `package.json`, `Dockerfile`, `Containerfile`, `README.md`, `.gitignore`.

## Tabela `bilhetes`

- `id`: inteiro, autoincrementado, começa em 1.
- `placa`: texto.
- `entrada_ms`: inteiro.
- `saida_ms`, `minutos`, `valor_centavos`: inteiros, nulos enquanto o bilhete não é encerrado.
- `status`: `aberto`, `encerrado` ou `cancelado`.

## Dependências

- Produção: `express` (versão 4) e `better-sqlite3`.
- Desenvolvimento: `supertest` e `eslint`. Os testes rodam com `node --test`.
- A lista não é fechada: se os testes precisarem de outra biblioteca, ela entra no `package.json`.
- Scripts: `start` (`node src/server.js`), `test` (`node --test`), `lint` (`eslint .`).

## Dockerfile e Containerfile

Os dois arquivos ficam na raiz, com este mesmo conteúdo aqui:

```dockerfile
FROM node:20-slim
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev
COPY src ./src
ENV PORT=8005
EXPOSE 8005
CMD ["node", "src/server.js"]
```

## README

O `README.md` do código gerado tem estas seções, cada uma com o comando:

- Rodar localmente: `npm install` e `npm start`.
- Rodar os testes: `npm test`.
- Docker: `docker build -t zona-azul .` e `docker run -p 8005:8005 zona-azul`.
- Podman: `podman build -t zona-azul .` e `podman run -p 8005:8005 zona-azul`.
- O Banco deve ficar na Raiz do projeto