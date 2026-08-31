# Como economizar a cota do Mentor (OutSystems ODC)

> Fontes oficiais confirmadas: o repositório do skill
> [OutSystems/outsystems-mcp](https://github.com/OutSystems/outsystems-mcp)
> (o próprio conector que o Claude usa) e a documentação de erros
> [OutSystems/docs-odc](https://github.com/OutSystems/docs-odc). Isso é
> diferente da versão anterior deste arquivo, que era só hipótese —
> agora é confirmado pela OutSystems.

## O erro que batemos (OS-AISA-42903)

> "Mentor is currently unavailable due to reached usage limits. Limit
> resets in `<reset_time>`."

**Causa oficial:** a organização atingiu o **limite diário de uso do
Mentor** (por tenant/organização, não por usuário). **Ação recomendada
oficial:** esperar o horário de reset indicado na mensagem de erro. A
OutSystems não documenta publicamente o número exato da cota diária —
só confirma que existe e que reseta uma vez por dia.

Existem também dois erros de **taxa** (diferentes do limite diário):
- `OS-AISA-42901` — excesso de chamadas ao provedor de IA (esperar
  alguns minutos)
- `OS-AISA-42902` — excesso de chamadas **por minuto** no tenant — a doc
  oficial recomenda **coordenar o uso se várias pessoas usam o Mentor ao
  mesmo tempo** no mesmo tenant.

## O que a documentação oficial do skill recomenda pra não desperdiçar cota

1. **Nunca começar uma sessão nova (`app_key`) depois de uma falha —
   retomar a MESMA sessão.** Um turno que falhou ou foi cancelado ainda
   devolve `mentor_session_id` + um `mentor_session_token` novo. Retomar
   com esses dados continua de onde parou; começar do zero **gasta um
   novo "slot" de sessão por tenant** e descarta edições não publicadas.
2. **Usar `fresh_context: true` ao retomar**, em vez de sessão nova,
   quando: a conversa bate no limite de tamanho (`OS-AISA-40001`), o
   Mentor começa a "alucinar" entidades/ações que não existem, ou você
   muda de tarefa dentro do mesmo app. Isso reinicia só o contexto da
   conversa, mantendo a sessão, o OML já editado no servidor, e qualquer
   edição ainda não publicada — ou seja, **não gasta outro slot de
   sessão**.
3. **Publicar o modelo de dados ANTES de pedir as telas.** Assim que um
   turno cria as entidades, publicar essa sessão e só depois pedir as
   telas em turnos separados. Enquanto não publica, tudo vive só na
   sessão do servidor — se a sessão expirar/for coletada, o trabalho não
   publicado se perde e o turno seguinte teria que refazer do zero.
   Publicar cedo garante que uma falha no meio do caminho custe **um
   turno**, não o build inteiro. *(Foi exatamente essa estratégia que
   usamos aqui — publicamos o modelo de dados e o backend antes de tentar
   as telas — o que evitou perder o trabalho já feito quando bateu o
   limite.)*
4. **Evitar pooling/polling agressivo do andamento de um turno do Mentor
   — cada poll custa um turno inteiro de modelo.** A recomendação oficial
   é esperar pelo menos **~30 segundos** entre verificações de status de
   um turno do Mentor (não vale pra outros tipos de status, tipo publish/
   deploy, que podem ser checados mais rápido, a cada 5-15s). Isso é
   consumo "escondido": ficar checando com pressa literalmente queima
   cota sem produzir nada.
5. **Pedidos de várias telas ou modelo de dados completo de uma vez são
   os mais "caros"** (a doc chama de "the most reasoning-heavy thing this
   transport asks for") — e a própria OutSystems recomenda rodar esse
   tipo de turno no **tier de modelo mais forte disponível**, porque um
   modelo mais fraco tenta, falha, tenta de novo, e acaba gastando *mais*
   tokens/turnos do que um modelo forte que resolve de primeira.

## Resumo prático pra próxima sessão

- Se der erro/timeout num turno do Mentor: **retomar a mesma sessão**
  (ou usar `fresh_context: true`), nunca começar uma sessão nova por
  padrão.
- **Publicar cada marco pronto** (modelo de dados, depois backend, depois
  cada leva de telas) antes de seguir pro próximo — já fizemos isso certo
  aqui.
- **Não ficar pedindo status a cada poucos segundos** durante um turno do
  Mentor — esperar pelo menos uns 30s entre checagens.
- Pedidos grandes (múltiplas telas de uma vez) são mais propensos a
  gastar cota com retrabalho — preferir pedir uma tela por vez com
  descrição clara, como já estava planejado pra retomada de amanhã.

## Atualização — 28 a 30/08: confirmado na prática

**O plano é gratuito/trial** (`personal-zkgbsms6.outsystems.dev`), o que
explica a cota de Mentor apertada — não é bug nem limitação geral da
OutSystems, é o nível esperado pra um tenant pessoal sem custo.

**Contagem real de estouros:** 3 vezes em 3 dias (28/08 duas vezes,
30/08 uma), **sempre em turnos que pediam múltiplas telas de uma vez**.
Isso bate exatamente com o ponto 5 acima (turnos multi-tela são os mais
caros) — na prática, não é só teoria.

**O tier do modelo por trás do Claude Code também importa, não só o
Mentor em si.** Rodar o turno com `Sonnet` + esforço "Baixo" produziu um
turno com `internal_retry_count: 14` mesmo terminando "limpo" (0 erros).
Trocar para `Opus` antes do próximo turno multi-tela é a recomendação
oficial do skill, e a diferença de retries observada bate com o aviso
oficial ("mid tiers are marginal... expensive on a hard multi-screen
build").

**Decisão tomada a partir disso:** parar de pedir várias telas por
turno. Passar a construir **uma tela por vez, publicando entre cada
uma** — rende menos por dia, mas um estouro de cota no meio custa no
máximo uma tela, não o build inteiro. Ver `STATUS.md` para o plano de
turnos detalhado por essa nova regra.
