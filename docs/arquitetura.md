# Arquitetura

O Kalman roda inteiro em serviços gerenciados da AWS (`us-east-1`), sem servidor ligado. Cada parte é uma stack SAM/CloudFormation independente.

<img width="3660" height="2180" alt="Arquitetura do Kalman na AWS" src="https://github.com/user-attachments/assets/8ba2b53e-448b-4445-92a7-37fc11d92169" />

## Fluxo de ponta a ponta

1. **Cadastro.** A empresa entra com o CNPJ; os dados da Receita chegam sozinhos (OpenCNPJ) e ela completa o perfil regulatório. A empresa envia a matriz de obrigações que já usa.
2. **Leitura diária.** O ETL do DOU baixa o Diário Oficial, separa em atos e grava no lakehouse.
3. **Propostas.** Quando o ETL termina, uma máquina de estados acha os atos que importam para cada empresa e propõe novas versões das obrigações, sempre citando o trecho do ato.
4. **Decisão.** O ETL da matriz valida as propostas e as grava como `proposta`. Uma pessoa aprova ou rejeita pela API; aprovada, a obrigação vira vigente e as tarefas são recalculadas.
5. **Uso.** O painel mostra o que decidir, o que está em risco e a linha do tempo das mudanças. A IA do Kalman responde sobre as normas da empresa, com a fonte.

## Stacks

| Stack | Pasta | Serviços | Papel |
|---|---|---|---|
| `cnpj-cadastro` | [`api/cnpj-cadastro`](../api/cnpj-cadastro) | Cognito, API Gateway HTTP, 5 Lambdas, SQS, DynamoDB | Login, cadastro pelo CNPJ, perfil regulatório, equipe e convites, e-mails do Cognito com a marca |
| `matriz-api` | [`api/matriz`](../api/matriz) | API Gateway HTTP, Lambda, DynamoDB, Athena | Matriz de obrigações, tarefas, linha do tempo, importação da planilha e decisões sobre as propostas |
| `chatbot-api` | [`api/chatbot`](../api/chatbot) | API Gateway HTTP, Lambda, DynamoDB, CloudWatch Logs | Porta de entrada do chat: sessão, histórico, limite de mensagens e chamada ao agente |
| IA do Kalman | [`ia/chatbot`](../ia/chatbot) | Bedrock AgentCore Runtime, Bedrock Guardrail | Agente com ferramentas sobre a matriz, as tarefas e os atos do DOU; respostas em blocos com fonte |
| Agente do cadastro | [`ia/agente-cadastro`](../ia/agente-cadastro) | AgentCore Runtime + Memory, ECR | Ajuda a preencher o perfil regulatório (stack pronta, ainda não ligada ao front) |
| Propostas | [`ia/propostas`](../ia/propostas) | Step Functions, 4 Lambdas, S3 Vectors, Glue | Classifica os atos do dia e propõe novas versões de obrigações |
| `lakehouse-pipeline` | [`pipeline/cadastro`](../pipeline/cadastro) | S3, Glue Job (DuckDB + PyIceberg), Athena, Step Functions, SNS | Dono do bucket e dos databases `landing/bronze/silver/gold`; perfis das empresas |
| `lakehouse-dou` | [`pipeline/dou`](../pipeline/dou) | Glue Job, Step Functions (EventBridge Scheduler), SNS | Baixa e estrutura o DOU todo dia |
| `lakehouse-matriz` | [`pipeline/matriz`](../pipeline/matriz) | Glue Job, Step Functions, SNS | Matriz e tarefas em Iceberg (`gold_db.matriz_*`) |
| `kb` | [`pipeline/kb`](../pipeline/kb) | Bedrock Knowledge Base, S3 Vectors, Glue Job | Base de conhecimento das normas (Titan Embeddings v2) |

## Princípios

- **Uma pessoa decide.** A IA propõe; nenhuma obrigação muda sem aprovação, e cada versão guarda o ato, o dispositivo e o trecho citado. O trecho citado é conferido em código: tem de existir no ato.
- **Isolamento por empresa.** O `user_id` vem sempre do token do Cognito; a API ignora qualquer identificador enviado pelo front.
- **Reprodutível e auditável.** A matriz é recalculada por replay (importações, propostas, decisões) e nunca sobrescreve versões; o resultado de rodar de novo é o mesmo. Cada etapa da IA grava eventos de auditoria.
- **Custo baixo, sem servidor ligado.** Lambda, Step Functions, Glue Python Shell (1 DPU) e Athena: paga-se só o que roda.

## Onde está cada coisa

```
api/                       APIs serverless (cnpj-cadastro, matriz, chatbot) e o modelo do e-mail de alertas
ia/                        agentes no Bedrock AgentCore (chatbot, cadastro) e o fluxo de propostas
pipeline/                  ETLs do lakehouse (cadastro, dou, matriz, kb) com DuckDB + Iceberg
front/web/                 landing, cadastro, matriz e painel executivo (React 19 + Vite)
front/identidade-visual/   marca, tokens e componentes (versão "Laudo")
docs/                      esta documentação
```

Detalhes técnicos de cada parte estão no README da própria pasta. Como rodar e testar: [desenvolvimento.md](desenvolvimento.md).
