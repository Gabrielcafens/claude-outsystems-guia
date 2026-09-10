# OS Prep Quiz — Status do Build (OutSystems ODC)

> Retomar amanhã: abrir Claude Code nesta mesma pasta/sessão e pedir para continuar. A retomada automática também está agendada para **29/08 ~21:02** (só funciona se esta sessão do Claude Code continuar aberta até lá — se fechar, é só colar este arquivo ou pedir "continua o OS Prep Quiz" que eu retomo do ponto exato).

## Objetivo
Transformar o app `mecanicasteste` (hoje um teste vazio) em um app completo de estudo para certificação OutSystems, baseado no repo estático `github.com/Gabrielcafens/os-prep`, com:
- Banco de perguntas organizado em 5 categorias: Service Studio, Integration Studio, Lifecycle Management, Segurança, Performance/Escalabilidade
- Modo simulado cronometrado
- Modo flashcard
- Explicação da resposta correta em cada questão
- Pontuação e histórico de tentativas salvo no banco (sem login — identificado por DeviceId gerado no client)

Plano completo salvo em: `C:\Users\gabriel.cafe\.claude\plans\gleaming-riding-swan.md`

## IDs importantes
- **App (assetKey):** `164f7265-9696-4a8b-8c55-8fe4f7a7d519` (módulo interno continua `mecanicasteste`; nome de exibição "OS Prep Quiz" ainda precisa ser trocado manualmente no Portal ODC — Mentor não consegue mudar isso via API)
- **Tenant:** `personal-zkgbsms6.outsystems.dev`
- **Ambiente (Development):** `b2a33b20-dc33-4d31-9890-6b3aded35fac`
- **URL runtime:** `https://personal-zkgbsms6-dev.outsystems.app/mecanicasteste`
- **Mentor session_id:** nenhuma aberta — a última (`89c94845-e03b-4e2d-a258-d993826b6850`) foi encerrada em 01/09 ao concluir o build
- **Revisão publicada mais recente:** **16** (01/09 — Flashcards, publish key `98a57d30-3ea6-4949-a75e-32b732f33f02`)

> **Nota de retomada (30/08):** o Mentor voltou a funcionar. Run `0c386cff-031e-4108-a9eb-07fb94035ab1` concluído com sucesso e **publicado na revisão 12** — itens 1–4 da lista "O que FALTA" estão prontos e no ar (DeviceId, ObterHistorico, telas Home e Historico), 0 erros de validação.

## ✅ O que já está PRONTO e PUBLICADO (revisão 9)

### Modelo de dados
- `Categoria` (static entity) — 5 registros: `ServiceStudio`(Id 1), `IntegrationStudio`(Id 2), `LifecycleManagement`(Id 3), `Seguranca`(Id 4), `Performance`(Id 5)
- `ModoTentativa` (static entity) — `Simulado`(Id 1), `Flashcard`(Id 2)
- `Pergunta` (CategoriaId, Enunciado, Explicacao, Ativa)
- `Opcao` (PerguntaId, Texto, Correta, Ordem) — cascade delete de Pergunta
- `Tentativa` (DeviceId, ModoTentativaId, CategoriaId nullable, DataInicio, DataFim, TotalPerguntas, Acertos, Pontuacao, TempoLimiteSegundos, Finalizada)
- `TentativaResposta` (TentativaId, PerguntaId, OpcaoEscolhidaId nullable, Correta, TempoRespostaSegundos) — cascade delete de Tentativa

### Backend (REST API `QuizAdminAPI` + Server Actions)
- `POST SeedPerguntas` (protegida por secret — valor guardado fora deste repo, não versionar) — usada para popular o banco
- `GET ContarPerguntas` — conta total de perguntas (bug de contagem já corrigido)
- `GET ListarReferencia` — lista Ids/Labels de Categoria e ModoTentativa
- `GET ListarPerguntasDebug` — lista todas as perguntas (debug, pode remover depois)
- `IniciarTentativa` (Server Action) — seleciona perguntas aleatórias por categoria
- `RegistrarResposta` (Server Action) — grava resposta e retorna feedback imediato + explicação
- `FinalizarTentativa` (Server Action) — calcula pontuação final
- `ObterEstatisticasPorCategoria` (Server Action) — agrega histórico por categoria

### Conteúdo — 123 perguntas reais, com opções e explicações, já confirmadas no banco:
| Categoria | Qtd |
|---|---|
| Service Studio | 46 (reaproveitadas/recategorizadas do repo os-prep) |
| Segurança | 22 |
| Integration Studio | 20 |
| Lifecycle Management | 18 |
| Performance/Escalabilidade | 17 |
| **Total** | **123** |

Payload completo do seed salvo em (não apagar até finalizar o projeto):
`C:\Users\GABRIE~1.CAF\AppData\Local\Temp\claude\C--Users-gabriel-cafe\d9e0d7e3-31b0-4fbb-9654-2cb85f4b7b28\scratchpad\seed_payload.json` (e os fontes `questions_new.json`, `questions_old_mapped.json`, `seed_all.json` na mesma pasta — **esses arquivos são temporários e podem ser limpos pelo SO**; se sumirem, o conteúdo das perguntas já está fixo no banco de dados do OutSystems mesmo assim, então não é crítico).

### Telas e identificação de dispositivo (publicado na revisão 12, em 30/08)
- Client Variable `DeviceId` + Client Action `ObterDeviceId` — nó JavaScript que lê/grava `localStorage` na chave `os_quiz_device_id`, gerando um UUID v4 na primeira visita. Sem login.
- Structure `HistoricoItem` (TentativaId, DataInicio, DataFim, CategoriaLabel, ModoLabel, Pontuacao, Acertos, TotalPerguntas)
- Server Action `ObterHistorico` — join Tentativa + Categoria + ModoTentativa, filtra `Finalizada = True` por DeviceId, ordena por DataInicio desc
- Tela **Home** — `OnInitialize` chama ObterDeviceId; Data Action `GetEstatisticas` alimenta tabela Categoria / Tentativas / Média (%); botões "Iniciar Simulado" e "Modo Flashcard" (ainda placeholders apontando pra Home) e "Ver Histórico"
- Tela **Historico** — tabela Data / Modo / Categoria / Pontuação / Acertos-Total + link "← Voltar"

## 🟢 Autorização em vigor (dada em 30/08)

Gabriel autorizou publicar automaticamente cada etapa assim que o turno do Mentor terminar e validar sem erros, **sem perguntar de novo**, até acabar todas as telas restantes e renomear o app. Interromper e chamar ele só se: (a) der erro que precise de decisão, (b) bater o limite do Mentor de novo, ou (c) terminar tudo.

## 🎯 FASE 2 (01–02/09): virar simulado real da prova O11

Contexto novo que o Gabriel deu: ele está estudando para a **certificação O11 Associate Reactive Developer**, já reprovou duas vezes com 54%, precisa de 70%. **Nada sobre reprovação pode aparecer na aplicação** — só desempenho por assunto. A prova real tem **50 questões**; tempo exato ele não lembra (usando 90 min como padrão de treino); assumindo **múltipla escolha simples** por enquanto.

Pontos do relatório dele: **0%** em Client e Server Actions, Eventos de Blocos, Fluxos Lógicos, Usando Blocos · **33%** em Entidades e Entidades Estáticas e Validações de Formulários · **66%** em Aggregates · bom em Bootstrap, Botões, Ciclo de Vida de Telas, Client Variables, Dependências Modulares, Relações entre Dados e Segurança por Roles.

### ✅ Fase 2 concluída — revisões 17 a 21 (sessão do Mentor `aff67572-1f5b-4020-b770-5d73a5a2de81` encerrada)

| Rev | Mudança |
|---|---|
| **17** | App renomeado para "OS Prep Quiz" pelo Gabriel no Portal ODC (o Mentor não consegue). ⚠️ **A URL mudou** para `https://personal-zkgbsms6-dev.outsystems.app/OSPrepQuiz` — a antiga `/mecanicasteste` morreu |
| **18** | Simulado em **modo prova**: `RegistrarResposta` continua gravando, mas nada é revelado durante o exame. Painel de feedback, botão "Proxima" e as variáveis `RespostaCorreta`/`ExplicacaoCorreta`/`OpcaoCorretaTexto` removidos |
| **19** | ResultadoSimulado vira **gabarito completo**: todas as questões (não só as erradas), cada uma com opção escolhida, correta, marcador visual e `Explicacao` — **inclusive nos acertos**, para revelar acertos por sorte. Linhas compactas e expansíveis para aguentar 50 questões |
| **20** | **50 questões e 90 minutos como padrão** (10/20/30 continuam para treino curto). Proteção nova: se a Categoria escolhida tiver menos perguntas ativas que o pedido, o app avisa e roda com o que existe em vez de falhar ou encurtar em silêncio |
| **21** | **Taxonomia nova**: as 7 categorias dos objetivos reais da prova substituem as 5 antigas. As 123 perguntas antigas foram **desativadas, não apagadas** |

### As 7 categorias novas
`ClientServerActions` · `EventosBlocos` · `FluxosLogicos` · `UsandoBlocos` · `Entidades` · `ValidacoesFormularios` · `Aggregates`

### ⚠️ ESTADO ATUAL: ZERO perguntas ativas
Isso é esperado, não um bug. As 123 antigas foram desativadas com a troca de taxonomia e o banco novo ainda não foi escrito. **Um simulado agora não inicia** (a proteção da rev 20 avisa em vez de quebrar).

**Próximo trabalho: escrever o banco de perguntas.** Rota: endpoint REST `SeedPerguntas` que já existe (secret `seed-os-prep-2026`) — **não consome cota do Mentor**, é só um POST. O que falta é o conteúdo: as perguntas em si, com 4 alternativas, a correta e a explicação. Peso: primeiro os quatro 0% (Client e Server Actions, Eventos de Blocos, Fluxos Lógicos, Usando Blocos), depois os 33% (Entidades e Entidades Estáticas, Validações de Formulários), por último Aggregates (66%).

### Ainda não testado
Nenhuma das revisões 17–21 foi clicada. Todas passaram na validação do OutSystems com 0 erros, mas isso é análise estática. Falta rodar o fluxo real.

## ✅ FASE 1 COMPLETA (01/09) — revisão 16

Todas as telas planejadas estão construídas e publicadas. Sessão do Mentor `89c94845-e03b-4e2d-a258-d993826b6850` encerrada.

**Seis telas no ar:** Home, Historico, ConfigurarSimulado, Simulado, ResultadoSimulado, Flashcards.
**Revisões desta noite:** 13 (ConfigurarSimulado), 14 (Simulado), 15 (ResultadoSimulado), 16 (Flashcards).

**Pendência única, manual:** renomear a aplicação para "OS Prep Quiz" no Portal ODC (o Mentor não consegue — ver Turno 5 abaixo).

**Ainda não testado de verdade:** todos os turnos passaram na validação do OutSystems com 0 erros, mas validação é análise estática, não prova de que o app funciona. Falta clicar o fluxo completo (configurar → responder com o timer rodando → deixar terminar → conferir resultado e revisão das erradas) e o baralho de flashcards.

### Histórico de estouros de cota do Mentor
Quatro no total: 28/08 (duas vezes), 30/08, 31/08 — sempre no meio de um turno. **A estratégia de uma tela por turno funcionou:** em 30/08 o estouro custou três telas de uma vez; em 31/08 custou só uma. Nas noites de 31/08→01/09 os quatro turnos passaram sem estourar. Outro padrão: o token OAuth da OutSystems expirou de um dia para o outro nas duas retomadas, e nesse estado os tools `authenticate`/`complete_authentication` não ficam expostos — a saída é o usuário rodar `/mcp` → outsystems → Authenticate.

## 🔑 ESTRATÉGIA DE BUILD (decidida por Gabriel em 30/08) — UMA TELA POR TURNO

**Regra:** construir e publicar **uma tela por vez**, nunca várias juntas. Turno do Mentor → validar → publicar → só então começar a próxima tela.

**Por quê:** a cota do Mentor (`OS-AISA-42903`) já estourou três vezes (28/08 duas vezes, 30/08 uma), sempre no meio de turnos multi-tela, e cada estouro perdeu o turno inteiro. Publicando a cada tela, um estouro custa no máximo uma tela em vez de três. Rende menos por turno, e é esse o trade-off que Gabriel escolheu conscientemente.

**Ordem:** ConfigurarSimulado → Simulado → ResultadoSimulado → Flashcards → renomear o app.

### Prompts por turno (um de cada vez, publicando entre eles)

Em todos: reaproveitar as Server Actions existentes `IniciarTentativa`, `RegistrarResposta`, `FinalizarTentativa` e `ObterEstatisticasPorCategoria` — mandar o Mentor **inspecionar as assinaturas antes** de ligar as telas, não reescrever a lógica. Usar `IsMandatory=True` nos inputs obrigatórios e **não** colocar asterisco literal no texto do label (a plataforma pinta o dela). Manter a UI consistente com Home e Historico.

**Turno 1 — ConfigurarSimulado** ✅ **PUBLICADO na revisão 13** (01/09 00:17 UTC, deployment `537ef6d5-9aeb-49de-a3cf-195f497d4a7b`). Aggregate `GetAllCategorias` + dropdown de Categoria com opção vazia para "todas"; RadioGroups de quantidade (10/20/30) e tempo (10/20/30 min); botão chama `ObterDeviceId` + `IniciarTentativa`; botão "Iniciar Simulado" da Home religado. 0 erros de validação.
Escolher Categoria (com opção "todas", já que `CategoriaId` é nullable → `NullIdentifier`), quantidade de perguntas (10/20/30, default 10) e tempo em minutos (10/20/30, default 20) convertido para `TempoLimiteSegundos`. Botão "Começar Simulado" chama `ObterDeviceId` + `IniciarTentativa` e navega para a tela Simulado passando o `TentativaId` — como Simulado ainda não existe neste turno, deixar a navegação como placeholder e religar no Turno 2. Link "Voltar" para Home. **Também religar o botão "Iniciar Simulado" da Home**, que hoje aponta para a própria Home. → publicar

**Turno 2 — Simulado** ✅ **PUBLICADO na revisão 14** (01/09 00:28 UTC, deployment `de68d64c-067e-48d4-a9e7-ea39598ec339`). Input `TentativaId`; aggregates `GetTentativa` + `GetPerguntasDaTentativa` (uma pergunta por vez) + `GetOpcoesDaPergunta`; timer mm:ss via `setInterval` → action `TimerTick` → `FinalizarTentativa` ao zerar; barra "Pergunta X de N"; botões de opção desabilitam após responder; painel de feedback com certo/errado + opção correta + `Explicacao`; botão "Proxima". ConfigurarSimulado religado para navegar para cá. 0 erros, `internal_retry_count` 5 (contra 13 no turno anterior — escopo menor converge mais rápido).
Input `TentativaId`. Uma pergunta por vez com suas `Opcao`; timer regressivo mm:ss client-side que ao zerar chama `FinalizarTentativa`; indicador de progresso (atual/total); ao escolher uma opção chama `RegistrarResposta` e mostra feedback imediato (certo/errado + `Explicacao`), com botão "Próxima". Depois da última, `FinalizarTentativa`. Navegação para ResultadoSimulado fica placeholder até o Turno 3. **Religar a navegação de ConfigurarSimulado para cá.** → publicar

**Turno 3 — ResultadoSimulado** ✅ **PUBLICADO na revisão 15** (01/09, publish key `7cec048a-47f6-44c4-9c7e-a2c23a954106`). Structure `DetalheRespostaItem` + Server Action `ObterDetalhesRespostas` (SQL avançado unindo TentativaResposta/Pergunta/Categoria/Opcao numa chamada); tela com card de resumo (Pontuacao %, Acertos/TotalPerguntas, tempo decorrido), quebra por Categoria e lista das erradas com opção escolhida, correta e `Explicacao`; botões "Novo Simulado" e "Voltar ao Inicio". Simulado religado para navegar para cá ao fim do tempo ou da última pergunta. 0 erros.
Input `TentativaId`. Mostra `Pontuacao`, `Acertos`/`TotalPerguntas`, tempo gasto, quebra por Categoria, e a lista das erradas com a opção correta e a `Explicacao`. Botões "Novo Simulado" (→ ConfigurarSimulado) e "Voltar ao Início" (→ Home). Se precisar, criar uma Server Action/Structure extra que devolva as respostas de uma Tentativa com opção escolhida + correta + explicação. **Religar as navegações do Turno 2 para cá.** → publicar

**Turno 4 — Flashcards** ✅ **PUBLICADO na revisão 16** (01/09, publish key `98a57d30-3ea6-4949-a75e-32b732f33f02`). Modo de estudo sem pontuação: escolha de Categoria com opção "Todas as categorias" que recarrega o baralho; carta com frente (`Enunciado`) e verso (opção correta + `Explicacao`); botão para virar; navegação Anterior/Próxima com contador "Carta X de N"; mensagem própria quando a categoria não tem perguntas; link Voltar. Botão "Modo Flashcard" da Home religado. 0 erros.
Modo de estudo com cartões frente/verso por Categoria, usando `Pergunta`/`Opcao`/`Explicacao`. **Religar o botão "Modo Flashcard" da Home**, hoje placeholder. → publicar

**Turno 5 — Renomear o app para "OS Prep Quiz"** ⚠️ **NÃO É POSSÍVEL VIA MENTOR** (confirmado 01/09, segunda tentativa). O Mentor respondeu que o nome da aplicação precisa ser alterado direto no **Portal ODC, nas configurações da aplicação** — ele não tem acesso a isso. Nada foi publicado neste turno (`changeApplied: false`).

Ponto importante: o Mentor verificou que **todo o texto visível dentro do app já diz "OS Prep Quiz"**. O que continua como `mecanicasteste` é só o nome da aplicação no Portal e na URL. **Ação manual pendente para o Gabriel** (leva segundos no Portal).

## ⏳ O que FALTA (telas e lógica de tela)

Ainda nem começado:
5. Tela **ConfigurarSimulado** — escolher categorias, quantidade de perguntas, tempo do simulado
6. Tela **Simulado** — execução com timer cronometrado, pergunta + opções, barra de progresso
7. Tela **ResultadoSimulado** — pontuação final, quebra por categoria, revisão das erradas com explicação
8. Tela **Flashcards** — modo de estudo com cartões (frente/verso) por categoria
9. Renomear o app para "OS Prep Quiz" no Portal ODC (manual, ou pedir pra eu tentar via mentor de novo)

## Cota do Mentor (histórico)

Em 28/08 a cota do Mentor estourou duas vezes (erro `OS-AISA-42903`), com resets de ~8h e ~23h. **Em 30/08 o Mentor voltou a funcionar normalmente** e o turno das telas Home/Historico rodou até o fim sem esbarrar no limite. Se voltar a estourar, o erro traz a janela de reset no próprio texto.

## Retomada automática já agendada
Um cron one-shot foi criado nesta sessão do Claude Code para **29/08 às 21:02**, que vai automaticamente: verificar/completar Home+Histórico, publicar, depois construir ConfigurarSimulado+Simulado+Resultado, depois Flashcards, e no final te mandar o link do app. **Esse agendamento só existe dentro desta sessão do Claude Code — se você fechar o terminal/app antes do horário, ele não dispara.** Nesse caso, é só abrir uma nova conversa e pedir para continuar o "OS Prep Quiz" — este arquivo tem tudo que eu preciso pra retomar do zero.
