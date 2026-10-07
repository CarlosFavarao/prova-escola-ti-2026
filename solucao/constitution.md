> Leia `fatos.md` antes deste arquivo: ele é a Fonte da Verdade do projeto.

# Constitution

Regras inegociáveis de como trabalhar neste projeto. Valem para toda tarefa e toda geração de código, e não descrevem funcionalidades (isso é papel do `spec.md`).

## Regras operacionais

- **RO-01** Em conflito entre documentos, vale a Fonte da Verdade. Na ambiguidade, escolha a opção mais próxima do contrato e registre a decisão no README; não pare a geração.
- **RO-02** Nenhuma resposta usa status `400`. Erro de entrada é `422` com um `erro` da tabela de erros.
- **RO-03** É proibido criar código de erro, rota, campo ou status que não esteja na Fonte da Verdade.
- **RO-04** Dinheiro é sempre `integer` em centavos, do banco à resposta. Nunca `float`.
- **RO-05** Uma tarefa só é concluída quando `npm test` passa sem testes pulados.

## Regras de código

- **RO-06** Stack fixa: Node.js 20, Express 4 e SQLite com `better-sqlite3`. Um único processo em `0.0.0.0:8005`, sem serviços externos.
- **RO-07** Toda data-hora na resposta usa o fuso `-03:00`, sem milissegundos. O dia do relatório também é contado em `-03:00`.
- **RO-08** `minutos` arredonda para baixo; frações de cobrança arredondam para cima; o teto de `5000` é aplicado por último.
- **RO-09** As regras recebem o "agora" por um relógio injetável; em produção o padrão é o relógio do sistema. Testes injetam um relógio fixo ou usam `entrada` relativa ao agora; nunca esperam tempo real.

## Regras de entrega

- **RO-10** `Dockerfile` e `Containerfile` na raiz, idênticos, com `EXPOSE 8005` e `CMD`.
- **RO-11** `package.json` lista todas as dependências, inclusive as de teste. `README.md` traz os comandos para rodar localmente, testar e usar Docker e Podman.