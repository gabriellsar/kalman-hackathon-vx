# api/matriz

API da **matriz de obrigações** (stack `matriz-api`): lê obrigações e tarefas da gold do lakehouse pelo Athena e registra as decisões do dono. Usa o mesmo login (Cognito) e a tabela do cadastro, só para leitura: o CNPJ e o papel vêm de lá, nunca do front.

| Rota | Body | Resposta |
|---|---|---|
| `GET /obrigacoes` | `?em=2026-11-05` | versão vigente de cada obrigação e propostas do LLM aguardando decisão. Com `em`, a matriz como estava naquele dia |
| `GET /obrigacoes/{id}` | | todas as versões (com o período em vigor) e o histórico |
| `GET /linha-do-tempo` | `?obrigacao_id=OBR-013&ato_id=dou-...` | o que cada ato mudou, do mais recente ao mais antigo: evento, ato (id, link, data de publicação, trecho citado) e cada campo com o valor antes e depois |
| `POST /obrigacoes/{id}/decisao` | `{"versao": 2, "decisao": "aprovar"}` | **200** com as versões e as tarefas recalculadas. **409** se a versão já foi decidida. Só o dono (**403**) |
| `GET /tarefas` | `?em_risco=true&obrigacao_id=OBR-013` | tarefas com vencimento, área, status e `em_risco` |
| `PATCH /tarefas/{id}` | `{"status": "concluida", "evidencia_url": "https://..."}` | **200** com a tarefa. Só o dono (**403**) |
| `POST /importacoes` | `{"nome": "matriz.csv", "conteudo": "<texto>"}` ou `{"nome": "matriz.xlsx", "conteudo_base64": "..."}` | **201** com `arquivo`, `enviado_em` e `processamento`. Grava em `landing/matriz/importada/<cnpj>/` e dispara o job da matriz na hora. Até 2 MB (**413**); CSV sem `obrigacao_id`/`titulo` ou extensão errada: **400**. Só o dono (**403**) |
| `GET /importacoes` | | arquivos enviados (mais novos primeiro) e o estado do último processamento (`RUNNING`, `SUCCEEDED`, `FAILED`…) |

A decisão é gravada em `bronze_db.matriz_decisoes` (de onde o job refaz a gold) e já aplicada na gold, para o front ver o resultado na hora. As regras de prazo e de versão são as mesmas do job (`pipeline/matriz/src/regras_matriz.py`, copiado em `api/matriz/src/`). Os dados vêm do ETL em `pipeline/matriz/`.

```bash
make matriz-test     # testes (o SQL roda num DuckDB no lugar do Athena)
make matriz-deploy   # sam deploy; resolve User Pool, tabela, bucket, workgroup e a máquina de estados da matriz pelos outputs das stacks
```

