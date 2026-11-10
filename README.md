<p align="center">
  <img src="front/identidade-visual/marca/kalman-horizontal.svg" alt="Kalman" height="64">
</p>

<p align="center"><b>Saiba quais normas afetam a sua empresa.</b></p>

<p align="center">
O Kalman lê o Diário Oficial da União todo dia, confere cada ato com as obrigações da empresa,
propõe o que muda e transforma a mudança em tarefa com prazo e dono.
</p>

<p align="center">
  <a href="https://kalman-vx.lovable.app/painel?demo=dash">Painel de demonstração</a> ·
  <a href="docs/arquitetura.md">Arquitetura</a> ·
  <a href="docs/desenvolvimento.md">Desenvolvimento</a>
</p>

<p align="center"><sub>Projeto do Hackathon VX · Desafio 02, Torre de controle regulatória</sub></p>

https://github.com/user-attachments/assets/05cc3316-1dcc-4198-8d8a-55f09b3979f7

---

## O problema

Toda empresa regulada precisa descobrir, a tempo, qual norma nova muda o que ela tem de cumprir. O Diário Oficial publica cerca de **600 atos por dia** (595 em 02/11, segundo o IBPT), e a maioria não tem nada a ver com a empresa.

| | Quanto custa |
|---|---|
| **Acompanhar** | R$ 867 mil por ano para uma concessionária de energia manter 3 analistas de regulação (estimativa da equipe) |
| **Errar** | R$ 24,3 milhões numa única multa da ANEEL à CEEE Equatorial (2025, continuidade do serviço em 2022) |

As ferramentas que existem filtram por palavra-chave: avisam que a palavra apareceu, não que a obrigação mudou. Numa semana real do DOU, o Clipping (ferramenta de código aberto do governo) geraria **224 alertas, 217 deles inúteis**. Quem acompanha acaba lendo tudo, ou deixando passar o que importa.

## Como o Kalman resolve

| | |
|---|---|
| **1. Cadastro em minutos** | A empresa digita o CNPJ; os dados da Receita chegam sozinhos. Ela envia a matriz de obrigações que já usa e completa o perfil regulatório (setor, UFs, órgãos, outorgas). |
| **2. Leitura diária** | Todo dia o DOU é baixado, separado em atos e cruzado com o perfil e a matriz de cada empresa. |
| **3. Proposta, não palpite** | Quando um ato muda uma obrigação (prazo, responsável, regra nova), a IA propõe a nova versão mostrando o antes e o depois e o trecho da norma. |
| **4. Tarefa com dono** | Uma pessoa aprova ou rejeita. Aprovado, as tarefas são recalculadas, com prazo e área responsável. |

Além disso, um **painel executivo** mostra o que decidir, o que está em risco, os prazos da semana e a linha do tempo das mudanças, e a **IA do Kalman** responde perguntas sobre as normas da empresa citando a fonte.

### Resultado numa semana real do DOU

Funil: **1.931 atos lidos → 13 relevantes → 4 para decidir.**

No mesmo recorte de semana, comparado ao filtro por palavra-chave:

| | Palavra-chave (Clipping) | Kalman |
|---|---|---|
| Alertas na semana | 224 | **9** |
| Alertas inúteis | 217 | **3** |
| Alertas lidos para achar uma obrigação real | 32 | **1,5** |
| O que chega | "a palavra apareceu" | tarefa com prazo e dono |

A diferença vem de ler o **ato inteiro** e compará-lo com cada obrigação da empresa, em vez de procurar palavras.

## Por que dá para confiar

- **Uma pessoa decide.** A IA só propõe. Nenhuma obrigação muda sem aprovação, e cada versão guarda o ato, o dispositivo e o trecho citado.
- **A citação é conferida em código.** Se o trecho que a IA cita não existe no ato, a proposta não entra.
- **Histórico auditável.** Versões nunca são sobrescritas: dá para consultar a matriz como ela estava em qualquer dia, e cada etapa da IA registra eventos.
- **Com avaliação própria.** A busca e a triagem das propostas têm métricas de cobertura e precisão contra uma amostra de 352 atos do DOU rotulados à mão ([`ia/propostas`](ia/propostas/README.md)), e o assistente tem casos de avaliação próprios ([`ia/chatbot`](ia/chatbot/README.md)).
- **Dados isolados por empresa.** O usuário vem sempre do token de login; a API ignora qualquer identificador enviado pelo navegador.

## Estado do projeto

Funcionando: cadastro e equipe, matriz de obrigações com versões e tarefas, leitura diária do DOU, propostas com IA e decisão humana, painel executivo e IA do Kalman.

Em andamento ou fora do escopo do hackathon:
- **Aviso por e-mail:** o e-mail está montado e testado ([`api/alertas`](api/alertas/README.md)), mas o envio (agendamento e remetente próprio) não foi implementado.
- **Agente do cadastro:** a stack está pronta, mas ainda não está ligada ao front.
- **Classificação das empresas:** calibrada com duas empresas; precisa ser revista com mais cadastros.
- **Recall com clientes:** ainda não medido em campo.

## Veja rodando

O painel abre com dados de exemplo, sem login e sem AWS:

```bash
cd front/web
npm install && npm run dev
# http://localhost:5173/painel?demo=dash
```

Os testes de todas as partes rodam sem AWS:

```bash
make test
```

## Documentação técnica

| Para entender | Leia |
|---|---|
| Como as peças se encaixam, as stacks AWS e os princípios | [docs/arquitetura.md](docs/arquitetura.md) |
| Como rodar, testar e publicar | [docs/desenvolvimento.md](docs/desenvolvimento.md) |
| Cadastro, login, equipe e convites | [api/cnpj-cadastro](api/cnpj-cadastro/README.md) |
| Matriz de obrigações: API e lakehouse | [api/matriz](api/matriz/README.md) · [pipeline/matriz](pipeline/matriz/README.md) |
| Propostas do DOU com IA e como são avaliadas | [ia/propostas](ia/propostas/README.md) |
| IA do Kalman (assistente) e o contrato da resposta | [ia/chatbot](ia/chatbot/README.md) · [api/chatbot](api/chatbot/README.md) |
| Leitura do DOU e demais ETLs | [pipeline](pipeline/README.md) |
| Telas do front | [front/web](front/web/README.md) |
| Marca e design system | [front/identidade-visual](front/identidade-visual/README.md) |

## Licença

[MIT](LICENSE)
