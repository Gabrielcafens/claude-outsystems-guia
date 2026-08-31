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
- **Mentor session_id (sessão em andamento):** `476dd242-edd4-440d-8c15-a8daf500250a`
- **Revisão publicada mais recente:** 9

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

## ⏳ O que FALTA (telas e lógica de tela)

Um turno do Mentor tentou construir isso mas foi **interrompido pelo rate limit antes de terminar** (não publicado, nada quebrado, só incompleto):
1. Client Variable `DeviceId` + Client Action `ObterDeviceId` (gera/lê um GUID no navegador, sem precisar de login)
2. Server Action `ObterHistorico` (lista tentativas finalizadas por DeviceId)
3. Tela **Home** — dashboard com estatísticas por categoria + botões "Iniciar Simulado" / "Modo Flashcard" / "Ver Histórico"
4. Tela **Historico** — lista de tentativas passadas

Ainda nem começado:
5. Tela **ConfigurarSimulado** — escolher categorias, quantidade de perguntas, tempo do simulado
6. Tela **Simulado** — execução com timer cronometrado, pergunta + opções, barra de progresso
7. Tela **ResultadoSimulado** — pontuação final, quebra por categoria, revisão das erradas com explicação
8. Tela **Flashcards** — modo de estudo com cartões (frente/verso) por categoria
9. Renomear o app para "OS Prep Quiz" no Portal ODC (manual, ou pedir pra eu tentar via mentor de novo)

## ⚠️ Bloqueio atual: limite de uso do Mentor (IA) da OutSystems

Bati o limite de uso do Mentor **duas vezes** hoje (28/08):
- 1ª vez (~12:46): resetou em 8h23min
- 2ª vez (~21:26): resetou em **23h33min** → ou seja, disponível de novo por volta de **29/08 ~20:59**

Isso é uma cota da própria plataforma OutSystems (erro `OS-AISA-42903`), não algo que eu controle por aqui. Pode valer a pena verificar no Portal ODC se o plano/trial do tenant tem algum limite diário de chamadas de IA documentado.

## Retomada automática já agendada
Um cron one-shot foi criado nesta sessão do Claude Code para **29/08 às 21:02**, que vai automaticamente: verificar/completar Home+Histórico, publicar, depois construir ConfigurarSimulado+Simulado+Resultado, depois Flashcards, e no final te mandar o link do app. **Esse agendamento só existe dentro desta sessão do Claude Code — se você fechar o terminal/app antes do horário, ele não dispara.** Nesse caso, é só abrir uma nova conversa e pedir para continuar o "OS Prep Quiz" — este arquivo tem tudo que eu preciso pra retomar do zero.
