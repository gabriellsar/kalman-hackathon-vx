# front/web

Front do Kalman: landing, cadastro, matriz de obrigações e painel executivo. Vite, React 19, TypeScript, Amplify (login no Cognito) e React Router, na identidade visual **Laudo** (ver [`identidade-visual/`](../../identidade-visual/README.md)).

Detalhe de cada tela, regra e teste em [`TELAS.md`](TELAS.md).

## Rodar

```bash
make front-dev     # http://localhost:5173 (porta fixa: é a que está liberada no CORS das APIs)
make front-check   # typecheck
```

O `.env` é opcional: sem ele, `src/config.ts` usa os valores públicos das stacks publicadas (`cnpj-cadastro`, `matriz-api`, `chatbot-api`). Para apontar para outra conta, copie `.env.example` para `.env` e preencha as URLs e os IDs do Cognito (saem dos outputs das stacks: `make front-env`).

## Modo demonstração (sem AWS)

Percorre as telas com dados de exemplo (a empresa é a Light SESA, com dados públicos da Receita em `src/dados/light-opencnpj.json`). Nada é consultado na rede.

| URL | O que abre |
|---|---|
| `/entrar?demo` | entrar / criar conta (qualquer e-mail e senha; código: qualquer 6 dígitos) |
| `/painel?demo=dash` | painel executivo: Hoje, Obrigações, Prazos, Mudanças e IA do Kalman |
| `/painel?demo=dados`, `vazio`, `membro`, `convite` | etapas do cadastro e papéis de equipe |

O modo vale em `npm run dev`. Num build publicado, só com `VITE_DEMO=1`. O funcionamento está em `src/demo.ts`.

## Rotas

| Rota | Tela |
|---|---|
| `/` | Landing (`src/landing/`) |
| `/entrar`, `/criar-conta`, `/esqueci-senha` | Acesso. Criar conta pede CNPJ (conferido na hora: dígitos + OpenCNPJ), nome, e-mail e senha; o código de 6 dígitos chega por e-mail |
| `/painel` | Cadastro em 3 etapas (dados da empresa → matriz de obrigações → revisão) e, depois dele, o painel executivo (`src/dash/`) |
| `/privacidade`, `/termos` | Páginas legais |

## Código

- `src/App.tsx`: rotas, acesso, painel, conferência dos dados e equipe.
- `src/TelaMatriz.tsx` e `src/planilha.ts`: envio e conferência da planilha com as mesmas regras do job do lakehouse.
- `src/api.ts`, `src/matriz.ts`, `src/chatbot.ts`: clientes das três APIs.
- `src/dash/`: painel executivo.
- `src/identidade/`: tokens, componentes e marca.

## Integração com as APIs

- **Autenticação:** o login é o Cognito, via `aws-amplify/auth`. Toda chamada leva `Authorization: Bearer <accessToken>`. O front nunca envia `user_id`: a API o lê do token.
- **Configuração pública:** `UserPoolId`, `UserPoolClientId` e as URLs das APIs podem ir no código, porque o navegador as recebe de qualquer forma. Nunca coloque credenciais AWS (access key) no front.
- **Cadastro assíncrono:** depois do `POST /cadastro`, repita `GET /cadastro/me` até aparecer um `consultado_em` novo (o `aguardarConsulta` faz isso). Trate o 409 do POST (CNPJ já tem dono) e o 403 do PATCH (membros só leem).
- **CORS:** o API Gateway só aceita a origem exata (ou `*`), sem curinga de subdomínio. Ao publicar o front em outro endereço, inclua a URL no parâmetro `AllowedOrigins` das stacks:
  `sam deploy --parameter-overrides 'AllowedOrigins="http://localhost:5173,https://<seu-dominio>"'`

## Hospedagem

É um site estático (`npm run build` gera `dist/`). Em qualquer hospedagem estática, as rotas do React Router precisam de fallback para `index.html` (`/* /index.html 200`).

## Limite conhecido

`src/Matriz.tsx` e `src/matriz.ts` só diferem em maiúscula. No Windows e no macOS (sistema de arquivos sem distinção), `import "./Matriz"` cai no `matriz.ts` e a página abre em branco. As telas novas importam `./matriz` (o cliente). Renomeie um dos dois quando o `Matriz.tsx` for usado.
