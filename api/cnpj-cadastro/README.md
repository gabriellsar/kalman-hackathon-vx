# api/cnpj-cadastro

Cadastro de usuários pessoa jurídica: login com **Amazon Cognito**, consulta do CNPJ no [OpenCNPJ](https://api.opencnpj.org) e dados salvos no DynamoDB.

```
Front ──login──> Cognito (e-mail + senha) ──> access token
Front ──token──> API Gateway (valida o token) ──> Lambdas ──> DynamoDB
                                                    └─ POST: EventBridge ──> consulta OpenCNPJ
```

- **Cognito** guarda e-mail e senha. A senha nunca passa pelo nosso código nem pelo DynamoDB.
- **DynamoDB** (tabela única) guarda a organização de cada CNPJ (dados da empresa e formulário), a qual organização cada usuário pertence e os convites pendentes.
- O `user_id` **sempre** vem do token. A API ignora qualquer `user_id` enviado pelo front.

## Pré-requisitos

- [uv](https://docs.astral.sh/uv/)
- [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html) (só para build e deploy)

## Comandos

Da raiz do repositório (cada parte tem seu próprio `Makefile`, e a raiz só delega):

```bash
make help          # lista os alvos
make api-test      # uv sync + pytest (local: DynamoDB com moto, OpenCNPJ com servidor local)
make api-smoke-login  # login + rotas recusam chamadas sem token (stack publicada, usa suas credenciais AWS)
make api-smoke     # fluxo completo: CNPJ, formulário, isolamento, convite, membro, troca de CNPJ (stack publicada)
make api-seed-aurora  # cria a empresa de teste Aurora (KALMANTESTE100) com perfil completo; os dados ficam
make api-build     # sam build
make api-deploy    # sam build + sam deploy (samconfig.toml: stack cnpj-cadastro, us-east-1)
make api-clean     # remove .venv e caches
make test          # testes de todas as partes
```

O ambiente Python (`pyproject.toml`, `uv.lock`, `.venv`) fica em `api/` e é compartilhado por todas as lambdas. Cada lambda tem seu `template.yaml` e seu `Makefile` de build/deploy. Ou direto na pasta, sem o prefixo: `cd api && make test`.

Os smoke tests criam usuários de teste direto no Cognito (sem enviar e-mail) e apagam esses usuários e os itens deles no final.

## Rotas

Todas exigem o header `Authorization: Bearer <access token do Cognito>`. Sem token, a API responde **401**.

| Rota | Body | Resposta |
|---|---|---|
| `POST /cadastro` | `{"cnpj": "00.000.000/0001-91"}` | **202** na hora: cria a organização do CNPJ com o usuário como dono, e a consulta roda em segundo plano (~3s). **409** se o CNPJ já tiver dono. **400** se o CNPJ for inválido. |
| `GET /cadastro/me` | | **200** com `papel`, `cnpj`, `status`, `empresa` (sem sócios) e `dados` da organização. **404** se o usuário ainda não participa de uma. |
| `PATCH /cadastro/me` | `{"dados": {"setor": "energia_eletrica", "ufs_atuacao": ["SP"], "email_alertas": null}}` | **200** com o cadastro atualizado. Só o dono (**403** para membros). Faz merge campo a campo (uma lista é trocada inteira), e `null` apaga o campo. **400** com `campos` se um campo não for do perfil ou tiver o tipo errado. **404** se não houver organização. |

`status` pode ser `processando` (logo depois do POST), `ativa`, `inativa` ou `nao_encontrado`.

`dados` guarda o **perfil regulatório**, que o filtro da matriz de obrigações usa junto com os dados da Receita em `empresa` (CNAEs, natureza jurídica, porte, UF). Os campos e as regras de cada um estão em `src/perfil.py`:

| Campo | Tipo |
|---|---|
| `nome_fantasia`, `email_alertas`, `faixa_funcionarios`, `setor` | texto |
| `ufs_atuacao` (siglas), `municipios_atuacao` (código IBGE), `orgaos_fiscalizadores`, `temas_interesse`, `temas_fora_escopo`, `certificacoes` | lista de textos |
| `unidades` (`id`, `nome`, `tipo`, `tags`) | lista de objetos |
| `outorgas` (`id`, `tipo`, `numero`, `orgao`, `objeto`, `unidade_id`, `validade` AAAA-MM-DD, `antecedencia_renovacao_dias`) | lista de objetos |
| `areas` (`id`, `nome`, `responsavel`, `email`) | lista de objetos |

Em cada lista de objetos o `id` é obrigatório e único.

**Empresa de teste.** Com `FixturesEnabled=true` (só na stack do hackathon, ligado no `samconfig.toml`), CNPJs que começam com `KALMAN` vêm de `src/fixtures/` em vez do OpenCNPJ. `KALMANTESTE100` é a Aurora Energia S.A., distribuidora ativa em SP. `make api-seed-aurora` cria o usuário, envia o CNPJ e grava o perfil de `tests/aurora_dados.json`.

## Organizações: dono e membros

Cada CNPJ é uma **organização** com um único dono: quem enviou o CNPJ primeiro. O dono convida outras pessoas pelo e-mail. Os membros veem os mesmos dados da empresa e do formulário, mas não editam. Cada usuário participa de uma única organização, como dono ou como membro.

| Rota | Quem | Body | Resposta |
|---|---|---|---|
| `GET /cadastro/membros` | dono e membros | | **200** com `dono`, `membros` e, só para o dono, `convites` pendentes |
| `POST /cadastro/membros` | dono | `{"email": "pessoa@empresa.com"}` | **201**. Convite válido por 7 dias. **409** se a pessoa já faz parte ou já tem convite de outra organização |
| `DELETE /cadastro/membros/{id}` | dono, ou o próprio membro | | `id` = `user_id` remove o membro (ou é o membro saindo); `id` = e-mail cancela o convite |
| `GET /cadastro/convite` | convidado | | **200** com o convite pendente para o e-mail do usuário. **404** se não houver |
| `POST /cadastro/convite` | convidado | | **200**: aceita e vira membro. **409** se já participa de uma organização |
| `DELETE /cadastro/convite` | convidado | | **200**: recusa |

O convite não envia e-mail. O dono avisa a pessoa, ela cria a conta com o e-mail convidado e vê o convite ao entrar. O e-mail vem do Cognito, que só confirma a conta depois que a pessoa digita o código recebido. Por isso só quem tem acesso ao e-mail aceita o convite.

O `PATCH /cadastro/me` responde **403** para membros. O dono só troca de CNPJ enquanto a organização não tem membros, convites pendentes nem unidades, outorgas ou áreas no perfil (senão **409**): o perfil vai para o lake da matriz de obrigações chaveado pelo CNPJ. O CNPJ anterior fica livre e o formulário vai junto. Um membro não cadastra CNPJ: precisa sair da organização antes.

Tudo fica na tabela única, com a chave `user_id` (detalhes em `src/dados.py`):

| `user_id` (chave) | Campos | Para quê |
|---|---|---|
| `<sub do Cognito>` | `cnpj`, `papel` (`dono` ou `membro`) | a organização do usuário |
| `CNPJ#<cnpj>` | `dono`, `dono_email`, `status`, `empresa`, `dados`, `membros`, `convites` | a organização: um item por CNPJ, um dono |
| `CONVITE#<email>` | `cnpj`, `convidado_por`, `expira_em` (TTL) | convite pendente, um por e-mail |

O DynamoDB só garante unicidade pela chave. Por isso, criar a organização, aceitar um convite e remover um membro são transações: ou grava tudo, ou nada. Se duas pessoas enviarem o mesmo CNPJ ao mesmo tempo, só uma vira dona; a outra recebe 409. A Lambda da consulta só grava o resultado se quem disparou o evento ainda é o dono do CNPJ.

Ninguém prova que é dono do CNPJ: quem enviar primeiro fica com ele. Se precisar disso, valide a titularidade (por exemplo, com certificado e-CNPJ) antes de criar a organização.

## Como o front usa (Amplify v6)

Os valores vêm dos outputs da stack: `aws cloudformation describe-stacks --stack-name cnpj-cadastro --query "Stacks[0].Outputs"`.

```js
import { Amplify } from "aws-amplify";
import { signUp, confirmSignUp, signIn, fetchAuthSession } from "aws-amplify/auth";

Amplify.configure({ Auth: { Cognito: { userPoolId: "<UserPoolId>", userPoolClientId: "<UserPoolClientId>" } } });

await signUp({ username: email, password });                // Cognito envia um código por e-mail
await confirmSignUp({ username: email, confirmationCode });
await signIn({ username: email, password });

const { tokens } = await fetchAuthSession();                 // renova o token sozinho quando expira
const auth = { Authorization: `Bearer ${tokens.accessToken}`, "Content-Type": "application/json" };

await fetch(API_URL, { method: "POST", headers: auth, body: JSON.stringify({ cnpj }) });
const cadastro = await (await fetch(`${API_URL}/me`, { headers: auth })).json();   // repita enquanto status for "processando"
await fetch(`${API_URL}/me`, { method: "PATCH", headers: auth, body: JSON.stringify({ dados: { setor } }) });
```

O e-mail padrão do Cognito tem limite baixo de envios por dia. Antes de produção, configure o Amazon SES no User Pool.

## E-mails de código (gatilho CustomMessage)

`src/email_cognito.py` é chamado pelo Cognito antes de cada e-mail com código e devolve assunto e HTML na identidade do
Kalman: um texto para cadastro e reenvio, outro para "esqueci a senha". Outros tipos (AdminCreateUser, MFA) ficam com a
mensagem padrão. O envio continua pelo Cognito, com o remetente padrão (`no-reply@verificationemail.com`). Se a Lambda
falhar, o cadastro falha junto: o HTML não depende de nada externo e tem teste (`tests/test_email_cognito.py`).

⚠️ Um `sam deploy` feito de uma branch sem este gatilho tira o `LambdaConfig` do User Pool, e o e-mail volta ao padrão em
inglês.

