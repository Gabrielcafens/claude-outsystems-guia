# Claude + OutSystems

Guia direto de como usar o Claude Code integrado ao OutSystems — pra editar,
publicar e explorar aplicações direto do editor, sem sair do fluxo.

## O que é

O Claude Code tem um skill nativo (`outsystems`) que fala com o OutSystems via
MCP (Model Context Protocol). Na prática, isso quer dizer que dá pra pedir
pro Claude:

- Editar telas, lógica e dados de uma app OutSystems
- Publicar e fazer deploy
- Buscar elementos no tenant (módulos, entidades, ações, telas)
- Gerenciar bibliotecas externas (JS/CSS)

Tudo isso a partir de comandos em linguagem natural, sem precisar ficar
trocando de janela entre o Service Studio e o resto do trabalho.

**Fontes oficiais:**
- [OutSystems/outsystems-mcp](https://github.com/OutSystems/outsystems-mcp) — repositório oficial do skill/MCP que faz essa integração
- [OutSystems/docs-odc](https://github.com/OutSystems/docs-odc) — documentação oficial do ODC, incluindo os códigos de erro e limites do Mentor (ex.: `OS-AISA-42903`)

## Pré-requisitos

- Acesso a um ambiente OutSystems (Personal Area, dev, ou o tenant do
  trabalho — **ver seção de cuidados abaixo**)
- Claude Code instalado e autenticado
- Credenciais/token de acesso ao tenant configurados (o próprio skill guia
  a autenticação na primeira vez que é usado)

## Como usar

1. Abra o Claude Code na pasta do seu projeto (ou em qualquer pasta, já que
   a integração fala direto com o tenant, não com arquivos locais)
2. Peça a tarefa em português normal, por exemplo:
   - "cria uma tela de listagem de clientes na app X"
   - "adiciona uma validação obrigatória no campo CPF"
   - "publica a última versão da app Y"
   - "busca todas as entidades que têm o campo email"
3. O Claude identifica que é uma tarefa de OutSystems e usa o skill
   automaticamente — não precisa invocar nada manualmente

## Fluxo típico

```
Você: "adiciona um botão de exportar CSV na tela de relatórios"
   ↓
Claude entende o pedido, localiza a tela via MCP
   ↓
Claude edita a lógica/tela no Service Studio via API
   ↓
Claude publica (se você pedir) e reporta o resultado
```

## O que vale a pena testar primeiro

- Pedir pra **buscar** elementos do tenant antes de editar algo — ajuda a
  confirmar que o Claude está enxergando a app certa
- Pedir mudanças pequenas e específicas primeiro (um campo, uma validação)
  antes de pedir refactors grandes
- Sempre revisar o que foi publicado no Service Studio depois — o Claude
  edita rápido, mas quem valida o resultado final é você

## Cuidado com código de trabalho

Se o ambiente que você for testar for o **tenant da empresa** (não uma
Personal Area pessoal), trate esse conteúdo como você trataria qualquer
código de cliente/empregador:

- Não publique nada daqui em repositório público
- Não compartilhe capturas de tela de apps internas fora do ambiente de
  trabalho
- Se tiver dúvida se algo pode ser testado/documentado, pergunta antes

Este repositório é privado justamente por causa disso — é espaço de estudo,
não de divulgação.

## Comandos prontos pra copiar e testar

Ajuste o nome da app/tela pro seu caso real:

**Explorar antes de mexer**
```
busca todos os módulos que têm "Cliente" no nome
lista as entidades da app Financeiro
mostra as telas que usam a entidade Pedido
```

**Editar**
```
adiciona um campo "telefone" (texto) na entidade Cliente
cria uma validação obrigatória no campo CPF da tela de cadastro
adiciona um botão "Exportar CSV" na tela de relatórios
```

**Publicar**
```
publica a app X no ambiente de desenvolvimento
faz o deploy da última versão pra produção
```

**Biblioteca externa**
```
adiciona a biblioteca X (JS) como referência externa na app Y
```

## Glossário rápido

| Termo | O que é |
|---|---|
| **MCP** | Model Context Protocol — o "cano" que deixa o Claude conversar com o tenant OutSystems em tempo real |
| **Tenant** | O ambiente OutSystems (empresa ou pessoal) onde as apps vivem |
| **Personal Area** | Ambiente OutSystems gratuito pra estudo/teste pessoal, sem risco de mexer em nada de produção |
| **Service Studio** | O editor visual tradicional do OutSystems — o Claude edita por trás dele, via API |
| **Publicar** | Compilar e ativar a versão editada da app no ambiente escolhido |

## Perguntas frequentes

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
Testado e confirmado em **ODC** (OutSystems Developer Cloud), com um tenant
pessoal (`*.outsystems.dev`). Ainda não testado em **O11** (plataforma
clássica) — atenção: são conceitos e interfaces bem diferentes (Service
Center, Integration Studio, etc), então o skill pode não cobrir os mesmos
recursos lá.

## Limitações conhecidas (preencher conforme for testando)

- Na primeira conexão, o registro dinâmico de cliente OAuth (Dynamic Client
  Registration) pode falhar com **HTTP 404** mesmo com o tenant certo e
  ativo. Resolvido rodando `/mcp` no Claude Code pra reautenticar e/ou
  pedindo pro skill remover e re-registrar o servidor MCP.
- O skill às vezes esquece a configuração do tenant entre uma invocação e
  outra (perguntou o hostname de novo depois de já estar configurado) —
  só reenviar o hostname resolve.

## Próximos passos

- [x] Conectar num ambiente pessoal (Personal Area / tenant ODC pessoal)
- [x] Testar o fluxo de busca de elementos (`lista meus apps` — funcionou,
      retornou todas as apps do tenant com tipo, revisão e data)
- [ ] Documentar aqui os comandos que funcionaram bem, com exemplo real
      de edição/publicação (não só leitura)
- [ ] Testar em tenant O11 pra comparar comportamento
- [ ] Testar edição de tela/entidade de verdade e publicação

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
