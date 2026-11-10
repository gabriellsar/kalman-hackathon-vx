# Desenvolvimento

## Pré-requisitos

- [uv](https://docs.astral.sh/uv/) (Python) e Node.js (front).
- [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html) e credenciais AWS, **só para publicar** ou rodar os fluxos que usam a nuvem. Os testes não precisam de AWS.

## Testes (sem AWS)

```bash
make help          # alvos de todas as partes
make test          # testes de todas as partes
```

Cada parte tem os próprios testes e pode ser rodada separada: `make api-test`, `make matriz-test`, `make ia-test`, `make pipeline-test`. Eles usam AWS simulada; o SQL da matriz roda num DuckDB no lugar do Athena.

A avaliação do assistente e das propostas de IA usa o modelo de verdade e, portanto, a conta AWS:

| Comando | O que mede |
|---|---|
| `make ia-chatbot-eval MODEL_ID=...` | o agente do chatbot contra casos de referência (`ia/chatbot/evals/`) |
| `make ia-propostas-simular` | propostas do DOU com dados e modelos reais, sem gravar na AWS |
| `make ia-propostas-metricas` | cobertura e precisão da busca e da triagem contra a amostra rotulada à mão (`ia/propostas/rotulagem/`) |

## Front

```bash
cd front/web
npm install && npm run dev     # http://localhost:5173
```

Sem AWS, o painel abre com dados de exemplo em `http://localhost:5173/painel?demo=dash`. Detalhes em [`front/web/README.md`](../front/web/README.md).

## Publicar na AWS

Cada stack tem seu `samconfig.toml` e é publicada com `make <parte>-deploy`. As stacks dependem umas das outras: a **ordem de publicação** está em [`ia/propostas/README.md`](../ia/propostas/README.md#deploy-ordem). Todos os alvos usam `confirm_changeset`: confira o changeset antes de confirmar.

Cuidados:

- `sam deploy` publica **o código da branch de quem roda**. Antes de publicar uma stack que outra pessoa mexeu, traga para a sua branch o que já está publicado e confira depois que as rotas e os gatilhos continuam lá (por exemplo, o gatilho de e-mail do Cognito e as rotas de `/importacoes`).
- **CORS:** as APIs só aceitam as origens listadas no parâmetro `AllowedOrigins`. Ao hospedar o front em outro endereço, inclua-o no deploy de `cnpj-cadastro`, `matriz-api` e `chatbot-api`.

## Onde ler mais

| Parte | Documentação |
|---|---|
| Arquitetura | [arquitetura.md](arquitetura.md) |
| Cadastro, login, equipe e convites | [`api/cnpj-cadastro`](../api/cnpj-cadastro/README.md) |
| Matriz, tarefas e decisões (API) | [`api/matriz`](../api/matriz/README.md) |
| Porta de entrada do chat | [`api/chatbot`](../api/chatbot/README.md) |
| E-mail de aviso de mudanças | [`api/alertas`](../api/alertas/README.md) |
| Agente do chatbot e contrato da resposta | [`ia/chatbot`](../ia/chatbot/README.md) |
| Propostas do DOU com IA | [`ia/propostas`](../ia/propostas/README.md) |
| Agente do cadastro | [`ia/agente-cadastro`](../ia/agente-cadastro/README.md) |
| Lakehouse (ETLs) | [`pipeline`](../pipeline/README.md) · [matriz](../pipeline/matriz/README.md) · [DuckDB](../pipeline/DUCKDB.md) |
| Front | [`front/web`](../front/web/README.md) |
| Identidade visual | [`front/identidade-visual`](../front/identidade-visual/README.md) |
| Linha de base por regras para as propostas (Aurora) | [`ia/propostas/atos-relevantes-aurora`](../ia/propostas/atos-relevantes-aurora/README.md) |
