# api/chatbot

Assistente da Kalman: responde, em português, sobre a empresa do usuário, os atos do DOU e a matriz de obrigações (obrigações, tarefas, linha do tempo dos atos), só com dados do Athena (`gold_db`).

```
Front ──Bearer (Cognito)──> HTTP API ──> Lambda chatbot-api ──IAM──> Runtime AgentCore (agent.py) ──> Athena, Bedrock + Guardrail
                                           │ lê o CNPJ e o papel na tabela do cadastro (só leitura)
                                           └ limita as mensagens por usuário (DynamoDB)
```

São duas stacks, em pastas separadas: **`chatbot-api`** (a Lambda de entrada, aqui em `api/chatbot/`, uma API como as outras) e **`chatbot-agent`** (o agente de IA, em [`ia/chatbot/`](#iachatbot)). O agente não é uma API: ninguém o chama a não ser a Lambda.

## API de entrada

`POST /chatbot/mensagens` com `{"prompt": "...", "session_id": "opcional"}`, header `Authorization: Bearer <access token do Cognito>` (o mesmo login do cadastro).

| Resposta | Quando |
|---|---|
| **200** `{"resposta": "resumo", "blocos": [...]}` | ok. `resposta` é um resumo curto em texto puro (nunca como HTML); `blocos` são os dados que a tela desenha (ver [Contrato da resposta](#contrato-da-resposta)) |
| **400** | JSON inválido, `prompt` vazio ou com mais de 2000 caracteres, `session_id` inválido |
| **401 / 404** | sem token / usuário sem organização (envie um CNPJ ou aceite um convite) |
| **429** + `Retry-After` | passou de 20 mensagens por hora ou 100 por dia (configurável no template) |
| **502 / 504** | o agente falhou (mensagem genérica; o motivo fica só no log) / demorou mais de ~26 s |

Quem garante a identidade é a Lambda: o `sub` vem do token, o CNPJ vem da tabela do cadastro (nunca do corpo), e a sessão do runtime é `sha256(sub:session_id)`, então ninguém entra na conversa de outro usuário. Dono e membros conversam (é leitura). O log tem uma linha por pedido (latência, nº de consultas, resultado, um identificador curto do usuário e o CNPJ), nunca a pergunta, a resposta nem o `sub`; a retenção é de 30 dias.

