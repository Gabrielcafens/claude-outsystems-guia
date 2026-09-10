# Transcript do POC — HelloAiBlock (ODC, app AiSandbox)

Log da sessão de teste da técnica descrita em
[*AI-Powered Frontend Development in OutSystems O11 and ODC*](https://antonio-carvalho.medium.com/ai-powered-frontend-development-in-outsystems-o11-and-odc-4d408fdb4ade)
(António Carvalho), aplicada num app ODC de sandbox via Claude Code + skill
`outsystems`. Documentado em 2026-09-10.

Setup: dois terminais Claude Code em paralelo — este aqui (Mentor, orientando)
e outro conectado ao tenant ODC (executando via subagente em background, com
a skill `outsystems`).

---

## 1. Criação do Block base

**Prompt enviado ao terminal conectado ao ODC:**

```
Cria um Block chamado "HelloAiBlock" numa tela de teste (ou módulo novo de sandbox), com:
- Um Input Parameter de texto chamado "Nome"
- Um Container vazio dentro do Block (vai ser o ponto de conexão)
- No handler OnReady do Block, um JavaScript node que chama:
  $parameters.Instance = new AiSandbox.HelloAiBlock({
    runtimeId: $parameters.RuntimeId,
    nome: $parameters.Nome
  })
- No OnParametersChanged: $parameters.Instance.onParametersChanged({ nome: $parameters.Nome })
- No OnDestroy: $parameters.Instance.destroy()
```

**Resultado do subagente:**

> App: AiSandbox (novo Web App — não existia sandbox anterior)
> Block: HelloAiBlock
> - Input Parameter Nome (Text)
> - Container vazio JsMountPoint (com Name, para dar o RuntimeId)
> - Local Variable Instance (Object), partilhada entre handlers
>
> Handlers:
> - OnReady → JS node CreateInstance (inputs RuntimeId ligado a JsMountPoint.Id e Nome; output Instance), seguido de um Assign
> - OnParametersChanged → `$parameters.Instance.onParametersChanged({ nome: $parameters.Nome })`
> - OnDestroy → `$parameters.Instance.destroy()`
>
> Publish: sucesso (revision 2), sem erros.
> URL: https://personal-zkgbsms6-dev.outsystems.app/AiSandbox
>
> Nota prática: o objeto `AiSandbox.HelloAiBlock` ainda não existe em runtime
> — falta o script JS que o define. Sem isso o OnReady vai rebentar.

---

## 2. Script TypeScript/JS externo

**Prompt enviado:**

```
Cria um Script Resource JavaScript no módulo AiSandbox (nível de Layout ou
Common) com o conteúdo abaixo (classe HelloAiBlock com constructor, render()
via innerHTML, onParametersChanged, destroy) e associa esse Resource à
tela/layout onde o HelloAiBlock é usado.
```

Versão TypeScript original enviada como referência (o autor da doc — Claude
"Mentor" — escreveu com tipos; o agente do ODC teve que transpilar pra JS
puro porque o Resource não aceita `interface`/`private`/anotações de tipo):

```ts
interface IHelloAiBlockConfig {
  runtimeId: string;
  nome: string;
}

class HelloAiBlock {
  private runtimeId: string;
  private widgetElement: HTMLElement | null;

  constructor(config: IHelloAiBlockConfig) {
    this.runtimeId = config.runtimeId;
    this.widgetElement = document.getElementById(this.runtimeId);
    this.render(config.nome);
  }

  private render(nome: string) {
    if (this.widgetElement) {
      this.widgetElement.innerHTML = `
        <div style="padding:12px;border:1px solid #ccc;border-radius:8px;font-family:sans-serif">
          <strong>Olá, ${nome}!</strong><br/>
          <span>Este bloco foi renderizado via TypeScript, fora do Service Studio.</span>
        </div>
      `;
    }
  }

  onParametersChanged(payload: { nome: string }) {
    this.render(payload.nome);
  }

  destroy() {}
}

(window as any).AiSandbox = (window as any).AiSandbox || {};
(window as any).AiSandbox.HelloAiBlock = HelloAiBlock;
```

**Resultado do subagente:**

> Script: HelloAiBlock.js criado como recurso JS no módulo AiSandbox
> Ligação: adicionado aos RequiredScripts do HelloAiBlock (flow Common), por
> isso carrega antes do OnReady.
> Publish: sucesso, revision 3, sem erros de validação.

**Primeiro teste real** (após criar a tela `TestHelloAi` e verificar via
DevTools): o texto "Olá, Gabriel!" apareceu, mas **sem** a `div` estilizada
— só texto puro. O agente havia simplificado o `render()` na implementação.

**Correção enviada:**

```
Abre o Resource HelloAiBlock.js e substitui o método render() pra usar
[o HTML completo com div estilizada], mantendo o resto da classe igual.
Publica de novo e confirma a revision.
```

**Resultado:** revision 5, publicado. Verificado via DevTools — a `div` com
`padding:12px;border:1px solid #ccc;border-radius:8px` apareceu corretamente
dentro de `div#b1-JsMountPoint`, contendo `<strong>Olá, Gabriel!</strong>` e
o texto explicativo. **Confirmado: high-code controlando o DOM de verdade.**

---

## 3. Evento bidirecional (high-code → OutSystems)

**Prompt enviado:**

```
No Block HelloAiBlock, adiciona um Event chamado "OnGreetClicked" que emite
um parâmetro de saída "Timestamp" (Text).

No handler OnReady, atualiza o JS node CreateInstance pra também passar os
Events como parte do config:

$parameters.Instance = new AiSandbox.HelloAiBlock({
  runtimeId: $parameters.RuntimeId,
  nome: $parameters.Nome,
  events: {
    OnGreetClicked: $actions.OnGreetClicked
  }
})

Cria uma Screen Action ligada ao OnGreetClicked que mostra uma mensagem
(Message/Toast) com o timestamp recebido. Publica de novo.
```

**Primeira tentativa — falhou:**

> O que falhou: o validador recusou o JS node com `Invalid JavaScript —
> Unknown 'OnGreetClicked' action used in 'CreateInstance'`. Dentro de um JS
> node o ODC só reconhece `$parameters`; `$actions` não resolve Block
> Events, porque um Event não é uma client action.

**Diagnóstico e solução proposta pelo agente:**

> A saída limpa é uma indireção de um salto: criar no block uma Client
> Action pública `TriggerGreetClicked(Timestamp)` cujo fluxo é só "trigger
> do Event OnGreetClicked", e no JS node passar
> `$actions.TriggerGreetClicked` — isso `$actions` já resolve, porque é uma
> client action. O TypeScript continua chamando
> `events.OnGreetClicked(timestamp)` exatamente como planejado; só muda o
> que está do lado direito no config.

Opção escolhida: **Client Action ponte** (recomendada pelo próprio agente).

**Script TS atualizado** enviado como referência (também transpilado depois
para JS puro, sem tipos, com `if (btn)` no lugar de `btn?.` e `function`
com `self` no lugar de arrow functions, por segurança de compatibilidade
com o nível de ES do runtime):

```ts
interface IHelloAiBlockConfig {
  runtimeId: string;
  nome: string;
  events: {
    OnGreetClicked: (timestamp: string) => void;
  };
}

class HelloAiBlock {
  private runtimeId: string;
  private widgetElement: HTMLElement | null;
  private nome: string;
  private events: IHelloAiBlockConfig["events"];

  constructor(config: IHelloAiBlockConfig) {
    this.runtimeId = config.runtimeId;
    this.nome = config.nome;
    this.events = config.events;
    this.widgetElement = document.getElementById(this.runtimeId);
    this.render();
  }

  private render() {
    if (this.widgetElement) {
      this.widgetElement.innerHTML = `
        <div style="padding:12px;border:1px solid #ccc;border-radius:8px;font-family:sans-serif">
          <strong>Olá, ${this.nome}!</strong><br/>
          <span>Este bloco foi renderizado via TypeScript, fora do Service Studio.</span><br/>
          <button id="${this.runtimeId}-btn" style="margin-top:8px">Disparar evento</button>
        </div>
      `;
      const btn = document.getElementById(`${this.runtimeId}-btn`);
      btn?.addEventListener("click", () => {
        this.events.OnGreetClicked(new Date().toISOString());
      });
    }
  }

  onParametersChanged(payload: { nome: string }) {
    this.nome = payload.nome;
    this.render();
  }

  destroy() {
    if (this.widgetElement) this.widgetElement.innerHTML = "";
  }
}

(window as any).AiSandbox = (window as any).AiSandbox || {};
(window as any).AiSandbox.HelloAiBlock = HelloAiBlock;
```

**Resultado final:**

> Publicado — revision 6, 0 erros e 0 avisos na validação.
>
> O que ficou no block HelloAiBlock:
> - Event `OnGreetClicked` com output `Timestamp` (Text)
> - Client Action pública `TriggerGreetClicked(Timestamp)` que dispara esse Event
> - JS node `CreateInstance` com `events: { OnGreetClicked: $actions.TriggerGreetClicked }`
> - `HelloAiBlock.js` substituído pela versão canónica (botão + listener)

---

## 4. Teste final ao vivo

Verificado via DevTools (Elements) e visualmente no browser:

- HTML renderizado dentro de `div#b1-JsMountPoint`: a caixa estilizada com
  "Olá, Gabriel!", texto explicativo, e o botão "Disparar evento".
- Ao clicar no botão: apareceu um toast azul no canto superior:
  **"Evento recebido do high-code! Timestamp: 2026-09-10T15:57:17.550Z"**
- Confirmado via DevTools que o toast é um elemento real
  (`div#feedbackMessageContainer` → `.feedback-message.feedback-message-info`),
  não um mock — veio da Screen Action do OutSystems disparada pelo Event do
  Block.

**Conclusão:** o ciclo completo funcionou —

```
clique no botão (DOM, controlado por TS)
  → this.events.OnGreetClicked(timestamp)   [high-code]
  → Client Action ponte TriggerGreetClicked  [Block]
  → Event OnGreetClicked                     [Block]
  → Screen Action no OutSystems              [nativo]
  → Toast exibido na tela
```

Isso valida, na prática e no próprio tenant ODC do Gabriel, os dois
mecanismos centrais do artigo: (1) o Block como ponte fina renderizando
DOM via script externo, e (2) comunicação bidirecional entre high-code e
OutSystems via Events — com o ajuste da Client Action ponte, que não estava
explícito no texto original e foi descoberto durante o teste.
