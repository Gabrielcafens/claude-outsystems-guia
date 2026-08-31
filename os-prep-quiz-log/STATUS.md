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
- **Mentor session_id (sessão em andamento):** `aa6e1653-9a41-407c-b066-381a8ef2d987` (iniciada 30/08 ~21:26; a antiga `476dd242-edd4-440d-8c15-a8daf500250a` foi abandonada — token não foi salvo e a sessão já tinha passado do GC de 30min)
- **Revisão publicada mais recente:** **12** (publicada 31/08 00:41 UTC — deployment `2b5e93ee-75a3-486a-b0ea-df60cb9a20d4`)

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

## ⚠️ Bloqueio atual (30/08 ~21:55): cota do Mentor estourou de novo

O run `0f06284b-12f8-42d8-9ec8-35874c9d981f` (ConfigurarSimulado + Simulado + ResultadoSimulado) foi **interrompido no meio** por `OS-AISA-42903` — "Mentor is currently unavailable due to reached usage limits. **Limit resets in 23h 05min**" → disponível de novo em **31/08 por volta das 21:00** (horário local).

Estado do turno interrompido:
- `validation.error_count: 1` → **Invalid Client Action Flow** (o fluxo do simulado ficou pela metade)
- `internal_retry_count: 11`
- **Nada foi publicado** — e não deveria ser: o turno tem erro de validação e carrega `turn_error`, então o `publish_start` recusaria com `mentor_run_not_clean` de qualquer forma.
- A revisão **12 continua intacta e no ar**. Nada quebrou.

**As edições parciais do simulado serão perdidas** — a sessão do Mentor sofre GC depois de 30 min ociosa, e o reset é só daqui a 23h. Isso não é problema: o trabalho estava incompleto e com erro. Na retomada, começar uma **sessão nova** com `app_key` (re-baixa a revisão 12, que é limpa). **Nunca criar um app novo.**

## 🔑 ESTRATÉGIA DE BUILD (decidida por Gabriel em 30/08) — UMA TELA POR TURNO

**Regra:** construir e publicar **uma tela por vez**, nunca várias juntas. Turno do Mentor → validar → publicar → só então começar a próxima tela.

**Por quê:** a cota do Mentor (`OS-AISA-42903`) já estourou três vezes (28/08 duas vezes, 30/08 uma), sempre no meio de turnos multi-tela, e cada estouro perdeu o turno inteiro. Publicando a cada tela, um estouro custa no máximo uma tela em vez de três. Rende menos por turno, e é esse o trade-off que Gabriel escolheu conscientemente.

**Ordem:** ConfigurarSimulado → Simulado → ResultadoSimulado → Flashcards → renomear o app.

### Prompts por turno (um de cada vez, publicando entre eles)

Em todos: reaproveitar as Server Actions existentes `IniciarTentativa`, `RegistrarResposta`, `FinalizarTentativa` e `ObterEstatisticasPorCategoria` — mandar o Mentor **inspecionar as assinaturas antes** de ligar as telas, não reescrever a lógica. Usar `IsMandatory=True` nos inputs obrigatórios e **não** colocar asterisco literal no texto do label (a plataforma pinta o dela). Manter a UI consistente com Home e Historico.

**Turno 1 — ConfigurarSimulado**
Escolher Categoria (com opção "todas", já que `CategoriaId` é nullable → `NullIdentifier`), quantidade de perguntas (10/20/30, default 10) e tempo em minutos (10/20/30, default 20) convertido para `TempoLimiteSegundos`. Botão "Começar Simulado" chama `ObterDeviceId` + `IniciarTentativa` e navega para a tela Simulado passando o `TentativaId` — como Simulado ainda não existe neste turno, deixar a navegação como placeholder e religar no Turno 2. Link "Voltar" para Home. **Também religar o botão "Iniciar Simulado" da Home**, que hoje aponta para a própria Home. → publicar

**Turno 2 — Simulado**
Input `TentativaId`. Uma pergunta por vez com suas `Opcao`; timer regressivo mm:ss client-side que ao zerar chama `FinalizarTentativa`; indicador de progresso (atual/total); ao escolher uma opção chama `RegistrarResposta` e mostra feedback imediato (certo/errado + `Explicacao`), com botão "Próxima". Depois da última, `FinalizarTentativa`. Navegação para ResultadoSimulado fica placeholder até o Turno 3. **Religar a navegação de ConfigurarSimulado para cá.** → publicar

**Turno 3 — ResultadoSimulado**
Input `TentativaId`. Mostra `Pontuacao`, `Acertos`/`TotalPerguntas`, tempo gasto, quebra por Categoria, e a lista das erradas com a opção correta e a `Explicacao`. Botões "Novo Simulado" (→ ConfigurarSimulado) e "Voltar ao Início" (→ Home). Se precisar, criar uma Server Action/Structure extra que devolva as respostas de uma Tentativa com opção escolhida + correta + explicação. **Religar as navegações do Turno 2 para cá.** → publicar

**Turno 4 — Flashcards**
Modo de estudo com cartões frente/verso por Categoria, usando `Pergunta`/`Opcao`/`Explicacao`. **Religar o botão "Modo Flashcard" da Home**, hoje placeholder. → publicar

**Turno 5 — Renomear o app para "OS Prep Quiz"**
Tentar via Mentor. Uma tentativa anterior falhou; se falhar de novo, **não insistir** — avisar Gabriel para fazer manualmente no Portal ODC.

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
