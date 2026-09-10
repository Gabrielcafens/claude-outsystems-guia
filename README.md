# Claude + OutSystems

Guia de estudo e teste prático de como integrar o Claude Code ao OutSystems
— tanto pelo caminho oficial (MCP, só em ODC) quanto por uma técnica
alternativa (Block como ponte pra código de IA) que funciona em qualquer
versão da plataforma. Cada seção abaixo é um passo: leia, teste, marque
como feito.

## Índice

1. [O que é a integração oficial (MCP)](#1-o-que-é-a-integração-oficial-mcp)
2. [Pré-requisitos](#2-pré-requisitos)
3. [Passo a passo: usando o MCP oficial](#3-passo-a-passo-usando-o-mcp-oficial)
4. [Cuidado com código de trabalho](#4-cuidado-com-código-de-trabalho)
5. [Limitação do MCP oficial: só funciona em ODC](#5-limitação-do-mcp-oficial-só-funciona-em-odc)
6. [Estudo: a técnica do artigo do António Carvalho](#6-estudo-a-técnica-do-artigo-do-antónio-carvalho)
7. [Passo a passo: replicando o POC (HelloAiBlock)](#7-passo-a-passo-replicando-o-poc-helloaiblock)
8. [O que deu errado e como foi corrigido](#8-o-que-deu-errado-e-como-foi-corrigido)
9. [Glossário](#9-glossário)
10. [Perguntas frequentes](#10-perguntas-frequentes)
11. [Limitações conhecidas](#11-limitações-conhecidas)
12. [Próximos passos](#12-próximos-passos)

---

## 1. O que é a integração oficial (MCP)

O Claude Code tem um skill nativo (`outsystems`) que fala com o OutSystems
via **MCP** (Model Context Protocol) — um protocolo padrão que permite a um
agente de IA se conectar a um software externo através de uma interface
comum. Pense nele como um "cano": de um lado o Claude, do outro o tenant
OutSystems.

Na prática, com o MCP dá pra pedir pro Claude:

- Editar telas, lógica e dados de uma app OutSystems
- Publicar e fazer deploy
- Buscar elementos no tenant (módulos, entidades, ações, telas)
- Gerenciar bibliotecas externas (JS/CSS)

Tudo isso a partir de comandos em linguagem natural, sem precisar ficar
trocando de janela entre o Service Studio e o resto do trabalho.

**Fontes oficiais:**
- [OutSystems/outsystems-mcp](https://github.com/OutSystems/outsystems-mcp) — repositório oficial do skill/MCP que faz essa integração
- [OutSystems/docs-odc](https://github.com/OutSystems/docs-odc) — documentação oficial do ODC, incluindo os códigos de erro e limites do Mentor (ex.: `OS-AISA-42903`)

## 2. Pré-requisitos

- Acesso a um ambiente OutSystems (Personal Area, dev, ou o tenant do
  trabalho — **ver seção 4 antes de usar o do trabalho**)
- Claude Code instalado e autenticado
- Credenciais/token de acesso ao tenant configurados (o próprio skill guia
  a autenticação na primeira vez que é usado)

## 3. Passo a passo: usando o MCP oficial

1. Abra o Claude Code na pasta do seu projeto (ou em qualquer pasta, já que
   a integração fala direto com o tenant, não com arquivos locais).
2. Peça a tarefa em português normal — o Claude identifica sozinho que é
   uma tarefa de OutSystems e usa o skill automaticamente, sem precisar
   invocar nada manualmente.
3. Comece **buscando** antes de editar, pra confirmar que o Claude está
   enxergando a app certa (ver comandos prontos abaixo).
4. Peça mudanças pequenas e específicas primeiro (um campo, uma validação)
   antes de pedir refactors grandes.
5. Revise sempre o que foi publicado no Service Studio depois — o Claude
   edita rápido, mas quem valida o resultado final é você.

**Fluxo típico:**

```
Você: "adiciona um botão de exportar CSV na tela de relatórios"
   ↓
Claude entende o pedido, localiza a tela via MCP
   ↓
Claude edita a lógica/tela no Service Studio via API
   ↓
Claude publica (se você pedir) e reporta o resultado
```

**Comandos prontos pra copiar e testar** (ajuste o nome da app/tela pro seu
caso real):

<details>
<summary><strong>Explorar antes de mexer</strong></summary>

```
busca todos os módulos que têm "Cliente" no nome
lista as entidades da app Financeiro
mostra as telas que usam a entidade Pedido
```
</details>

<details>
<summary><strong>Editar</strong></summary>

```
adiciona um campo "telefone" (texto) na entidade Cliente
cria uma validação obrigatória no campo CPF da tela de cadastro
adiciona um botão "Exportar CSV" na tela de relatórios
```
</details>

<details>
<summary><strong>Publicar</strong></summary>

```
publica a app X no ambiente de desenvolvimento
faz o deploy da última versão pra produção
```
</details>

<details>
<summary><strong>Biblioteca externa</strong></summary>

```
adiciona a biblioteca X (JS) como referência externa na app Y
```
</details>

## 4. Cuidado com código de trabalho

Se o ambiente que você for testar for o **tenant da empresa** (não uma
Personal Area pessoal), trate esse conteúdo como você trataria qualquer
código de cliente/empregador:

- Não publique nada daqui em repositório público
- Não compartilhe capturas de tela de apps internas fora do ambiente de
  trabalho
- Se tiver dúvida se algo pode ser testado/documentado, pergunta antes

Este repositório é privado justamente por causa disso — é espaço de estudo,
não de divulgação.

## 5. Limitação do MCP oficial: só funciona em ODC

O MCP oficial da OutSystems hoje só funciona com **ODC**. Pra **O11**
(Traditional ou Reactive) não existe integração nativa via MCP ainda — e é
justamente aí que entra o material das seções seguintes.

---

## 6. Estudo: a técnica do artigo do António Carvalho

**Fonte:** [*AI-Powered Frontend Development in OutSystems O11 and ODC*](https://antonio-carvalho.medium.com/ai-powered-frontend-development-in-outsystems-o11-and-odc-4d408fdb4ade),
por António Carvalho — Frontend Lead que trabalha majoritariamente com O11,
autor de vários artigos sobre frontend em OutSystems (publica em [ITNEXT](https://itnext.io/)
e no blog [osfrontendtips.com](https://www.osfrontendtips.com/)).

**Problema que o artigo resolve:** a maioria das apps críticas de clientes
pagantes roda em O11, não em ODC — mas o MCP oficial (seção 5) só cobre
ODC. O autor precisava usar IA em projetos O11 maduros mesmo sem o MCP.

**A técnica: Block como ponte pra código de IA**

A ideia central é usar o **Block** do OutSystems como uma casca fina —
só a ponte — enquanto todo o comportamento real vive em **TypeScript/
JavaScript/CSS externo**, gerado ou editado por um agente de IA fora do
Service Studio.

> **Analogia do iceberg:** o Block visível no Service Studio é só a ponta.
> A maior parte do valor é o código high-code que fica "abaixo da linha
> d'água" — pode crescer, evoluir e ser versionado normalmente em Git
> (branches, PRs, code review) por anos, sem depender do Service Studio.

Peças do mecanismo:

| Peça | Papel |
|---|---|
| **Input Parameters** | dados que entram no Block vindos do OutSystems |
| **Container vazio** | o "buraco" no HTML do Block onde o high-code vai desenhar |
| **`OnReady`** | cria a instância da classe JS/TS, passando o `runtimeId` do Container |
| **`OnParametersChanged`** | repassa mudanças de Input Parameters pra instância JS |
| **`OnDestroy`** | chama `destroy()` na instância, pra cleanup (listeners etc.) |
| **Local Variable `Instance` (Object)** | guarda a instância JS entre os handlers |
| **Events** | forma do high-code "falar de volta" com o OutSystems |
| **Placeholders** | equivalente a Slots de Web Component — conteúdo nativo OutSystems dentro do Block |

**Onde funciona:**
- **O11 Reactive** e **ODC**: mesma arquitetura, Block com handlers de JS node.
- **O11 Traditional**: precisa de ajuste — a inicialização vira uma
  **Expression**, executada na renderização do Block, mas o conceito de
  fundo (Block fino + JS externo) é o mesmo.

**Vantagem chave:** o código por trás do Block pode **evoluir sem
republicar o Block** — só atualizar o JS/CSS externo e dar refresh no
módulo de UI top-level, desde que não quebre o contrato (Input
Parameters/Events) do Block.

**Produtividade citada pelo autor:** ~80–90% do frontend dele hoje é
escrito por agentes de IA, sobrando tempo pra arquitetura e validação.

**Outros artigos do mesmo autor, relacionados:**
- [How do I use an AI-powered IDE in OutSystems](https://medium.com/itnext/how-do-i-use-an-ai-powered-ide-in-outsystems-00fce740a08b) (Cursor)
- [How I develop CSS and JavaScript 2x faster in OutSystems](https://itnext.io/how-i-develop-css-and-javascript-2x-faster-in-outsystems-b8b9ebc8675d)
- Web Component in Action, using OutSystems
- Stop using Grid and Gutter: start using Flex in OutSystems

## 7. Passo a passo: replicando o POC (HelloAiBlock)

Testamos essa técnica na prática, num app ODC de sandbox (`AiSandbox`),
usando Claude Code com a skill `outsystems` pra fazer todo o trabalho no
Service Studio. Resultado: **funcionou de ponta a ponta**, incluindo
comunicação bidirecional. Roteiro pra repetir:

**Passo 1 — Block base com Container vazio**

Peça pro Claude criar um Block com um Input Parameter de texto, um
Container vazio (ponto de conexão) e os três handlers de ciclo de vida
chamando uma classe JS ainda inexistente:

```
$parameters.Instance = new AiSandbox.HelloAiBlock({
  runtimeId: $parameters.RuntimeId,
  nome: $parameters.Nome
})
```

> Nesse ponto o Block publica, mas ainda quebra em runtime — o objeto
> `AiSandbox.HelloAiBlock` não existe. É esperado: falta o script.

**Passo 2 — Script externo controlando o DOM**

Peça a criação de um Script Resource (JS puro — o Resource do ODC não
aceita `interface`/`private`/tipos do TypeScript, então uma versão TS serve
só de referência de design) que define a classe, com um `render()` que
escreve HTML de verdade (não só texto) dentro do `runtimeId` recebido.

Associe o Script aos `RequiredScripts` do Block, pra carregar antes do
`OnReady`. Publique e confirme via DevTools (aba Elements) que o HTML
apareceu dentro do Container — isso prova que o high-code está controlando
o DOM.

**Passo 3 — Evento bidirecional (high-code → OutSystems)**

Adicione um Event no Block (ex: `OnGreetClicked`, com output `Timestamp`) e
uma Screen Action que reage a ele mostrando uma mensagem. No script,
dispare o evento a partir de um clique de botão renderizado pelo próprio
JS. Veja a seção 8 — esse passo tem uma pegadinha que só aparece na
prática.

**Passo 4 — validar via DevTools**

Sempre confirme visualmente:
- **Elements**: o HTML dentro do Container bate com o que o script deveria
  gerar (pega problemas de cache ou de implementação simplificada demais).
- **Console**: sem erros vermelhos ao carregar ou interagir.

Transcript completo da sessão real (prompts exatos enviados, respostas do
agente, o erro que apareceu e a correção aplicada) está em
[poc-hello-ai-block-transcript.md](poc-hello-ai-block-transcript.md) — vale
ler antes de repetir o teste, pra não cair nos mesmos obstáculos.

## 8. O que deu errado e como foi corrigido

Duas coisas não saíram certas na primeira tentativa — documentar isso é
mais útil do que só documentar o que funcionou de primeira:

**1. `render()` simplificado demais**
Ao pedir o script pro agente, ele implementou uma versão que só escrevia
texto puro no lugar do HTML estilizado pedido. Só foi pego revisando o
DOM real no DevTools — o Block "funcionava" (sem erro), mas não fazia o
que devia. **Lição:** sempre confira o resultado renderizado, não só se
publicou sem erro.

**2. `$actions` não resolve Events dentro de um JS node**
Tentativa direta:
```js
events: { OnGreetClicked: $actions.OnGreetClicked }
```
**Falhou** com `Invalid JavaScript — Unknown 'OnGreetClicked' action`.
Motivo: dentro de um JS node do ODC, `$actions` só resolve **Client
Actions**, nunca um Block Event diretamente.

**Correção que funcionou** — Client Action ponte:
1. Criar uma Client Action pública no Block, ex. `TriggerGreetClicked(Timestamp: Text)`.
2. O único fluxo dessa Client Action é: disparar o Event `OnGreetClicked` com o `Timestamp` recebido.
3. No JS node, referenciar `$actions.TriggerGreetClicked` (isso sim resolve, por ser uma Client Action de verdade).
4. O script continua chamando `this.events.OnGreetClicked(timestamp)` normalmente — a indireção fica só do lado do Block, o TS nem percebe.

Esse detalhe **não está no artigo original** — é uma particularidade de
como o compilador do ODC valida `$actions` dentro de JS nodes, descoberta
só ao testar na prática.

## 9. Glossário

| Termo | O que é |
|---|---|
| **MCP** | Model Context Protocol — o "cano" que deixa o Claude conversar com o tenant OutSystems em tempo real |
| **Tenant** | O ambiente OutSystems (empresa ou pessoal) onde as apps vivem |
| **Personal Area** | Ambiente OutSystems gratuito pra estudo/teste pessoal, sem risco de mexer em nada de produção |
| **Service Studio** | O editor visual tradicional do OutSystems — o Claude edita por trás dele, via API |
| **Publicar** | Compilar e ativar a versão editada da app no ambiente escolhido |
| **Block** | Componente de UI do OutSystems — na técnica do artigo, vira a "casca" fina que aponta pro código externo |
| **Iceberg (analogia)** | O Block é a ponta visível; o JS/CSS externo é o volume real, abaixo da "linha d'água" do Service Studio |
| **Client Action** | Ação client-side do OutSystems — é o que `$actions` consegue resolver dentro de um JS node |
| **Placeholder (Block)** | Slot onde se pode inserir conteúdo nativo OutSystems dentro de um Block controlado por high-code |

## 10. Perguntas frequentes

**Preciso saber programar em OutSystems pra usar isso?**
Ajuda entender a lógica (entidades, telas, ações), mas o Claude cuida da
parte técnica de "como fazer" — você só precisa saber pedir o que quer.

**O Claude publica sozinho sem eu mandar?**
Não. Publicar/fazer deploy é uma ação que você pede explicitamente. Editar
também deveria ser sempre revisado antes de publicar em ambiente sério.

**Dá pra desfazer uma edição?**
O OutSystems tem histórico de versões no Service Studio — sempre dá pra
voltar pra uma versão anterior por lá, independente de como a edição foi
feita.

**Funciona em qualquer versão do OutSystems (O11, ODC)?**
O MCP oficial só cobre ODC — testado e confirmado com um tenant pessoal
(`*.outsystems.dev`). Ainda não testado em **O11** (plataforma clássica) —
atenção: são conceitos e interfaces bem diferentes (Service Center,
Integration Studio, etc), então o skill pode não cobrir os mesmos recursos
lá. Pra O11, a alternativa testada é a técnica de Block + high-code
descrita nas seções 6–8.

**A técnica do Block funciona igual em O11 e ODC?**
Na prática testamos só em ODC (seção 7). O artigo descreve O11 Reactive
como equivalente e O11 Traditional com o ajuste da Expression — mas isso
ainda não foi validado por nós, só pelo autor original.

## 11. Limitações conhecidas

- Na primeira conexão, o registro dinâmico de cliente OAuth (Dynamic Client
  Registration) pode falhar com **HTTP 404** mesmo com o tenant certo e
  ativo. Resolvido rodando `/mcp` no Claude Code pra reautenticar e/ou
  pedindo pro skill remover e re-registrar o servidor MCP.
- O skill às vezes esquece a configuração do tenant entre uma invocação e
  outra (perguntou o hostname de novo depois de já estar configurado) —
  só reenviar o hostname resolve.
- [x] MCP oficial não funciona em O11 (Traditional/Reactive), só ODC
- [x] `$actions` dentro de um JS node não resolve Block Events diretamente —
      precisa de uma Client Action ponte (seção 8)
- [ ] Placeholders (Slots) ainda não testados na prática
- [ ] Comportamento em O11 Traditional/Reactive (Expression em vez de
      handler) ainda não testado na prática, só descrito no artigo

## 12. Próximos passos

- [x] Conectar num ambiente pessoal (Personal Area / tenant ODC pessoal)
- [x] Testar o fluxo de busca de elementos (`lista meus apps` — funcionou,
      retornou todas as apps do tenant com tipo, revisão e data)
- [x] Testar o fluxo de busca de elementos num ambiente pessoal (Personal
      Area) — feito via app AiSandbox
- [x] Documentar os comandos que funcionaram, com exemplo real (seção 7)
- [x] Anotar limitações encontradas (seção 11)
- [ ] Testar em tenant O11 pra comparar comportamento
- [ ] Testar edição de tela/entidade de verdade e publicação
- [ ] Testar Placeholders (Slots) num Block
- [ ] Testar o mesmo padrão em O11 Traditional/Reactive de verdade

## Projeto em andamento (caso de teste real da integração)

Este é o caso de teste real usado pra validar a integração de ponta a ponta:
transformar o app de estudo [os-prep](https://github.com/Gabrielcafens/os-prep)
(simulados/flashcards pra certificação OutSystems, hoje um site estático
simples) num app OutSystems real (ODC), construído a partir de um app de
teste (`mecanicasteste`) no tenant pessoal — banco de perguntas por
categoria, modo cronometrado, flashcards, pontuação e histórico de
tentativas.

### Log de progresso — 2026-08-28

**Publicado com sucesso (revisão 8, ambiente Development):**
- Modelo de dados completo: entidades `Categoria`, `ModoTentativa`,
  `Pergunta`, `Opcao`, `Tentativa`, `TentativaResposta`
- Backend: REST API `QuizAdminAPI` + Server Actions (`SeedPerguntas`,
  `ContarPerguntas`, `ListarReferencia`, `IniciarTentativa`,
  `RegistrarResposta`, `FinalizarTentativa`, `ObterEstatisticasPorCategoria`)
- Banco de 123 perguntas com explicações escritas (46 recategorizadas do
  `os-prep` original + 4 novas de Segurança + 73 novas cobrindo
  Integration Studio, Lifecycle Management, Segurança e
  Performance/Escalabilidade)

**Bug encontrado (em aberto):** `ContarPerguntas` retorna `Total: 1` em vez
de 123 — suspeita de erro num `ForEach`/agregação na Server Action de seed.
Não afeta o que já foi publicado, só bloqueia a confirmação de que as 123
perguntas foram salvas corretamente.

**Erro de build corrigido durante o processo:** o Mentor usou uma
propriedade obsoleta (`ServerActionPublicPropertyApp`, removida da
plataforma) numa Server Action — identificado e corrigido automaticamente
numa nova sessão do Mentor.

**Bloqueio externo:** o **Mentor** (IA nativa do OutSystems ODC que o skill
usa por trás pra gerar telas/lógica — cota separada da conta Claude/Anthropic)
atingiu o limite de uso do tenant pessoal. Erro: `"Mentor is currently
unavailable due to reached usage limits"` (código `OS-AISA-42903`), reset
em ~8h. Dados já publicados não são afetados; só trava a criação das 6
telas restantes (Home, Configurar Simulado, Simulado cronometrado,
Resultado, Flashcards, Histórico) até o limite resetar.

**Lição:** tenants pessoais/trial do ODC têm cota de Mentor limitada —
pra projetos maiores/contínuos, considerar tenant corporativo ou espaçar
os pedidos de geração de tela ao longo de várias sessões.

> Log completo e atualizado do projeto (conversa, status técnico, dicas
> de uso do Mentor) fica em [`os-prep-quiz-log/`](os-prep-quiz-log/) —
> os arquivos abaixo eram o resumo de 28/08, veja a pasta pra estado atual.

### Log de progresso — 2026-08-30 e 2026-08-31

**Confirmado: o tenant é plano gratuito/trial.** Isso explica a cota de
Mentor apertada.

**Publicado com sucesso (revisão 12, 31/08 00:41 UTC):** telas **Home**
e **Histórico**, identificação de dispositivo sem login (`DeviceId` via
`localStorage`), e a Server Action `ObterHistorico`. 0 erros de validação.

**Descoberta importante:** rodar o turno do Mentor com o Claude em
**Sonnet + esforço baixo** produziu um turno com `internal_retry_count: 14`
mesmo terminando limpo. A documentação oficial do skill recomenda o
**tier de modelo mais forte (Opus)** justamente pra turnos multi-tela —
confirmado na prática: trocar para Opus foi a mudança feita antes do
turno seguinte.

**3º estouro de cota do Mentor** aconteceu de novo no meio de um turno
que pedia 3 telas de uma vez (ConfigurarSimulado + Simulado +
ResultadoSimulado) — nada publicado quebrado, só o turno incompleto foi
perdido. **Decisão tomada:** mudar a estratégia pra **uma tela por
turno, publicando entre cada uma** — rende menos por dia, mas um
estouro de cota custa no máximo uma tela, não o build inteiro. Detalhes
completos e os prompts de cada turno estão em
[`os-prep-quiz-log/STATUS.md`](os-prep-quiz-log/STATUS.md).

### Log de progresso — 2026-08-31 (noite) a 2026-09-01

**A estratégia de uma tela por turno funcionou na prática.** Turno 1
(ConfigurarSimulado) publicado limpo na revisão 13; Turno 2 (Simulado,
com timer regressivo, feedback imediato e explicação) publicado limpo
na revisão 14 — e o `internal_retry_count` caiu de 14 para 5 nesse
turno, sugerindo que o escopo menor por turno também reduz o atrito
interno do Mentor, não só o risco de perda. O Turno 3 (ResultadoSimulado)
foi cortado no meio pelo **4º estouro de cota** (`OS-AISA-42903`) — mas
dessa vez o estouro custou só 1 tela em vez de 3, confirmando o valor da
mudança de estratégia.

**Fluxo real testável agora:**
`https://personal-zkgbsms6-dev.outsystems.app/mecanicasteste` — Home →
Iniciar Simulado → configurar categoria/quantidade/tempo → responder
com timer e feedback (falta só a tela de resultado no fim, que hoje
volta direto pra Home).

### Log de progresso — 2026-09-01 (noite): Fase 1 completa + virada pra simulado real

**Fase 1 fechada, revisão 16.** Os turnos 3 (ResultadoSimulado) e 4
(Flashcards) publicaram limpos, sem estourar cota — quatro turnos numa
noite só, zero estouros, confirmando de vez a estratégia de uma tela
por turno. As 6 telas planejadas ficaram no ar: Home, Histórico,
ConfigurarSimulado, Simulado, ResultadoSimulado, Flashcards. Único
pendente: o Mentor **não consegue renomear o app via API** (confirmado
2x) — o nome no Portal ODC precisa ser trocado manualmente, mesmo com
todo o texto visível dentro do app já dizendo "OS Prep Quiz". Gabriel
fez isso manualmente: app renomeado, **URL mudou** para
`https://personal-zkgbsms6-dev.outsystems.app/OSPrepQuiz` (revisão 17).

**Guinada de escopo — de "quiz de estudo" pra "simulado real de
certificação".** Gabriel esclareceu o objetivo verdadeiro: prep pra
certificação **O11 Associate Reactive Developer** (2 reprovações a
54%, meta 70%), com um relatório de desempenho real listando pontos
fracos por assunto. Isso disparou uma segunda fase de turnos:

- **Rev 18** — Simulado vira modo prova de verdade: nada de feedback
  durante o exame (removido painel "certo/errado" + explicação em
  tempo real).
- **Rev 19** — ResultadoSimulado vira gabarito comentado completo:
  todas as questões (não só as erradas), com explicação inclusive nos
  acertos, pra pegar quem acertou no chute.
- **Rev 20** — Formato ajustado pro exame real: **50 questões, 90
  minutos** como padrão (a prova O11 é 50 questões; tempo exato o
  Gabriel não lembrava, 90min foi estimativa de treino). Adicionada
  validação pra não quebrar se uma categoria tiver menos perguntas
  ativas que o pedido.
- **Rev 21** — **As 5 categorias genéricas (Service Studio, Integration
  Studio...) foram substituídas pelas 7 categorias reais do relatório
  de desempenho do Gabriel**: Client/Server Actions, Eventos de Blocos,
  Fluxos Lógicos, Usando Blocos, Entidades, Validações de Formulário,
  Aggregates. As 123 perguntas antigas foram desativadas (não
  apagadas) por não servirem mais pro objetivo real.

**Estado atual: zero perguntas ativas, e é esperado.** A troca de
taxonomia esvaziou o banco visível — o simulado avisa e não inicia até
alguém escrever o banco novo. Esse é o próximo trabalho: perguntas
reais focadas nos pontos fracos verdadeiros do Gabriel, priorizando os
quatro tópicos zerados no relatório. A boa notícia é que popular o
banco é via endpoint REST direto (não consome cota do Mentor).

**Aula de tutoria em paralelo:** essa mesma sessão do Claude também deu
uma aula sobre os quatro tópicos zerados (Client vs Server Action,
nós de fluxo lógico exclusivos de cada um, Blocos e Eventos de Bloco),
confirmando que o par "app de simulado real + conversa de tutoria" é o
combo certo — simulado treina formato/memória, conversa resolve o
buraco conceitual de verdade.

**Detalhe operacional novo:** o token de autenticação MCP expirou de um
dia para o outro, exigindo re-rodar `/mcp` antes de retomar — vale
sempre checar a autenticação primeiro ao reabrir uma sessão depois de
~24h.

### Achado à parte: bug real corrigido em outro projeto

Enquanto o Mentor estava bloqueado, uma varredura rápida encontrou e
corrigiu um bug de UX real no site estático
[RangodoTuba](https://github.com/Gabrielcafens/RangodoTuba): o menu
mobile não fechava sozinho depois de clicar num link de navegação —
precisava de um segundo toque manual. Corrigido e mergeado em
[PR #2](https://github.com/Gabrielcafens/RangodoTuba/pull/2).
