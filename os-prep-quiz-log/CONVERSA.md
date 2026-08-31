# OS Prep Quiz — Registro da Conversa (28/08/2026)

> Este arquivo é um resumo fiel da conversa com o Claude Code sobre o build do app "OS Prep Quiz" no OutSystems. Para o status técnico detalhado do projeto (o que está pronto, o que falta, IDs, etc.), veja `OS-Prep-Quiz-STATUS.md` na mesma pasta.

## Como começou

Você pediu para listar seus apps no tenant OutSystems (`personal-zkgbsms6.outsystems.dev`). Depois de conectar o MCP do OutSystems e autenticar, listei os apps do tenant — o mais recente era `mecanicasteste` (descrito por você como "teste usar mcp, go jarvis").

## O pedido principal

Você pediu para transformar o `mecanicasteste` num app de quiz completo para estudar para a certificação OutSystems, baseado no repositório estático `github.com/Gabrielcafens/os-prep` (hoje um site simples de simulados/flashcards com 50 perguntas). Você queria recriar isso como um app ODC de verdade, com:

- Banco de perguntas maior, organizado por categoria: **Service Studio, Integration Studio, Lifecycle Management, Segurança, Performance/Escalabilidade**
- Modo simulado **cronometrado**
- Modo **flashcard**
- **Explicação** da resposta correta em cada questão
- **Pontuação** e **tracker de histórico** de tentativas salvo no banco de dados

## Decisões tomadas juntos (perguntei antes de começar)

1. **Sem login** — o app ficaria público, identificando o histórico por um `DeviceId` gerado no navegador (sem tela de autenticação).
2. **Renomear** o app para "OS Prep Quiz" (mantendo o módulo interno como `mecanicasteste`).
3. **~130-145 perguntas** — reaproveitar as 50 já existentes no repo (recategorizadas) e eu escrever ~95 novas para cobrir bem as 4 categorias que o repo original não tinha.

Escrevi um plano detalhado (modelo de dados, telas, lógica, sequência de execução) e você aprovou antes de eu começar a mexer no tenant.

## O que fiz, passo a passo

1. **Explorei o repositório os-prep** no GitHub e extraí as 50 perguntas originais (com tópicos, opções e explicações).
2. **Criei o modelo de dados** no módulo `mecanicasteste` via Mentor (a IA de edição do OutSystems): entidades `Categoria`, `ModoTentativa`, `Pergunta`, `Opcao`, `Tentativa`, `TentativaResposta`. Publiquei essa revisão.
3. **Criei o backend**: uma REST API (`QuizAdminAPI`) com um endpoint protegido por senha para popular o banco de perguntas, mais 4 Server Actions (iniciar tentativa, registrar resposta, finalizar tentativa, obter estatísticas). Tive que corrigir um erro de publicação (uma propriedade obsoleta da plataforma) e publiquei de novo.
4. **Escrevi 123 perguntas novas** (73 sobre Integration Studio, Lifecycle Management, Segurança extra e Performance, mais as 50 originais recategorizadas: 46 foram para Service Studio e 4 já eram de Segurança) com opções e explicações, e populei o banco via chamada HTTP direta ao endpoint que criei.
5. **Achei e corrigi um bug**: o endpoint de contagem de perguntas (`ContarPerguntas`) estava sempre retornando 1 em vez do total real — era um erro de leitura do resultado de uma Aggregate (`.Count` da lista em vez do valor agregado). Corrigido e publicado — confirmei as 123 perguntas certinhas no banco, distribuídas nas 5 categorias.
6. **Comecei a construir as telas** (Home, Histórico, mecanismo de identificação de dispositivo sem login) — mas o turno da IA foi interrompido pela cota de uso da plataforma antes de terminar (nada foi publicado quebrado, só ficou incompleto).

## Onde travamos

A plataforma OutSystems tem uma **cota de uso da IA (Mentor)** por tenant. Bati nesse limite **duas vezes** na mesma tarde:
- 1ª vez: resetaria em ~8h23min
- 2ª vez (mais tarde, tentando construir as telas): resetaria em **23h33min**, ou seja, só libera de novo perto das **20:59 de amanhã (29/08)**

Isso não é algo que eu controle — é um teto da própria OutSystems no seu tenant/plano.

## O que falta

- Telas: **ConfigurarSimulado**, **Simulado** (com timer), **ResultadoSimulado**, **Flashcards**, e finalizar **Home**/**Histórico**
- Renomear o app para "OS Prep Quiz" no Portal ODC (não dá pra fazer isso via Mentor/API)
- Testar o fluxo completo depois de tudo publicado

## Combinado

Você pediu para eu continuar automaticamente assim que o limite resetar. Agendei uma retomada automática (cron) para **29/08 às 21:02**, mas expliquei que esse agendamento só existe **dentro desta sessão do Claude Code** — se o PC for desligado, a sessão morre e o agendamento não dispara sozinho. Nesse caso, você só precisa abrir o Claude Code de novo amanhã e pedir para continuar o "OS Prep Quiz" — o arquivo `OS-Prep-Quiz-STATUS.md` tem tudo que eu preciso para retomar exatamente do ponto onde paramos, sem perder nada do que já foi feito e publicado no OutSystems.
