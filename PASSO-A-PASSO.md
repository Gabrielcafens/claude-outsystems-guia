# Passo a passo — Claude Code + OutSystems, do zero

Guia único, sequencial, pra montar todo o ambiente e repetir os testes
documentados neste repositório: desde instalar/conectar o Claude Code até
publicar o primeiro Block com código gerado por IA no OutSystems. Cada
seção assume que a anterior já foi feita.

## Índice

1. [Instalar e autenticar o Claude Code](#1-instalar-e-autenticar-o-claude-code)
2. [Ter um ambiente OutSystems pra testar](#2-ter-um-ambiente-outsystems-pra-testar)
3. [Conectar o Claude Code ao OutSystems via MCP](#3-conectar-o-claude-code-ao-outsystems-via-mcp)
4. [Primeiro teste: buscar informação do tenant](#4-primeiro-teste-buscar-informação-do-tenant)
5. [Extra: técnica Block + high-code (pra O11, ou por controle fino em ODC)](#5-extra-técnica-block--high-code-pra-o11-ou-por-controle-fino-em-odc)
6. [Extra: reskin com classes shadcn/ui](#6-extra-reskin-com-classes-shadcnui)
7. [Links de referência, todos juntos](#7-links-de-referência-todos-juntos)

---

## 1. Instalar e autenticar o Claude Code

1. Instale o Claude Code (CLI ou extensão, conforme o ambiente que você
   usa — terminal, VS Code, JetBrains).
2. Autentique com sua conta Anthropic/Claude na primeira execução.
3. Confirme que está funcionando abrindo um terminal na pasta de um
   projeto qualquer e mandando uma mensagem simples.

> Se já usa Claude Code no dia a dia, esse passo já está feito — o que
> muda a partir daqui é habilitar o **skill do OutSystems**.

## 2. Ter um ambiente OutSystems pra testar

Duas opções, na ordem de preferência pra estudo/teste pessoal:

- **ODC (OutSystems Developer Cloud) — Personal Area:** ambiente gratuito
  de nuvem, tenant pessoal em `*.outsystems.dev` ou `*.outsystems.app`.
  É onde o MCP oficial funciona (seção 3). Crie uma conta gratuita direto
  no site da OutSystems se ainda não tiver.
- **O11 (plataforma clássica):** se você já tem acesso via trabalho/estudo.
  O MCP oficial **não cobre O11** ainda — pra essa versão, use a técnica
  alternativa da seção 5 (Block + high-code).

**Cuidado:** se o ambiente disponível for o **tenant da empresa**, trate
tudo como código de cliente — não publique nada em repositório público,
não documente apps internas fora do ambiente de trabalho. Prefira sempre
uma Personal Area pessoal pra estudo e teste livre.

## 3. Conectar o Claude Code ao OutSystems via MCP

O Claude Code tem um skill nativo `outsystems` que fala com o tenant ODC
via **MCP** (Model Context Protocol).

1. No terminal do Claude Code, rode:
   ```
   /mcp
   ```
   Isso inicia (ou reautentica) a conexão MCP.
2. Na primeira vez, o skill vai pedir o **hostname do seu tenant** (ex:
   `personal-xxxxxxxx-dev.outsystems.app`) e conduzir a autenticação OAuth
   no navegador.
3. Confirme que autenticou com sucesso (mensagem de "Log in successful" no
   navegador, e o terminal libera o próximo comando).

**Problema comum:** na primeira conexão, o registro dinâmico de cliente
OAuth pode falhar com **HTTP 404** mesmo com o tenant certo. Solução:
rodar `/mcp` de novo pra reautenticar, ou pedir pro skill remover e
re-registrar o servidor MCP.

**Outro problema comum:** o skill às vezes "esquece" a configuração do
tenant entre uma invocação e outra (pergunta o hostname de novo do nada)
— só reenviar o hostname resolve.

**Fontes oficiais do skill/MCP:**
- [OutSystems/outsystems-mcp](https://github.com/OutSystems/outsystems-mcp)
- [OutSystems/docs-odc](https://github.com/OutSystems/docs-odc)

## 4. Primeiro teste: buscar informação do tenant

Sempre comece **buscando**, nunca editando de cara — confirma que o Claude
está enxergando o tenant certo antes de qualquer mudança:

```
lista meus apps
busca todos os módulos que têm "Cliente" no nome
mostra as telas que usam a entidade Pedido
```

Se isso retornar dados reais do seu tenant (nome das apps, tipo, revisão,
data), a conexão MCP está funcionando de ponta a ponta.

Depois disso, comandos de edição e publicação seguem o mesmo padrão de
linguagem natural — ver a seção 3 do [README.md](README.md#3-passo-a-passo-usando-o-mcp-oficial)
pra mais exemplos prontos.

## 5. Extra: técnica Block + high-code (pra O11, ou por controle fino em ODC)

Essa parte não depende do MCP — funciona em qualquer versão do OutSystems
(O11 Traditional, O11 Reactive, ODC), porque não fala com o tenant por
fora: é um padrão de arquitetura dentro da própria app. Baseada no artigo
[*AI-Powered Frontend Development in OutSystems O11 and ODC*](https://antonio-carvalho.medium.com/ai-powered-frontend-development-in-outsystems-o11-and-odc-4d408fdb4ade)
de António Carvalho — ver a explicação completa da técnica na seção 6 do
[README.md](README.md#6-estudo-a-técnica-do-artigo-do-antónio-carvalho).

Roteiro testado e validado (detalhes e prompts exatos usados estão no
[poc-hello-ai-block-transcript.md](poc-hello-ai-block-transcript.md)):

**5.1 — Criar o Block base**

Peça ao Claude Code (com o skill `outsystems` ativo, se for ODC — ou
direto no Service Studio se for O11):

```
Cria um Block chamado "MeuComponente" com:
- Um Input Parameter de texto
- Um Container vazio dentro do Block (ponto de conexão)
- No OnReady, um JS node que cria a instância:
  $parameters.Instance = new MeuApp.MeuComponente({
    runtimeId: $parameters.RuntimeId,
    valor: $parameters.MeuParametro
  })
- No OnParametersChanged: $parameters.Instance.onParametersChanged({...})
- No OnDestroy: $parameters.Instance.destroy()
```

**5.2 — Criar o script externo (o high-code de verdade)**

```
Cria um Script Resource JavaScript (sem tipos TypeScript — o Resource não
aceita interface/private) que define a classe MeuComponente, com um
render() que escreve HTML de verdade dentro do runtimeId recebido, e
associa esse Resource ao Layout ou aos RequiredScripts do Block.
```

Publique e confirme via **DevTools (F12) → Elements** que o HTML apareceu
dentro do Container — isso prova que o script está controlando o DOM.

**5.3 — Evento bidirecional (high-code → OutSystems), se precisar**

Se o componente precisa "avisar" o OutSystems de algo (um clique, uma
mudança de estado), **não tente `$actions.NomeDoEvent` direto num JS
node** — isso falha, porque `$actions` só resolve Client Actions, nunca
Events do Block diretamente.

Faça a ponte:
```
1. Cria uma Client Action pública no Block, ex: TriggerMeuEvento(Param)
2. O único fluxo dela é disparar o Event correspondente do Block
3. No JS node, usa $actions.TriggerMeuEvento (isso sim resolve)
4. O script continua chamando this.events.MeuEvento(valor) normalmente
```

**5.4 — Validar**

Sempre confira via DevTools (Elements + Console), não só se publicou sem
erro — um `render()` simplificado demais ainda "funciona" sem erro, mas
não faz o que devia.

## 6. Extra: reskin com classes shadcn/ui

Objetivo: aplicar o **visual** do [shadcn/ui](https://ui.shadcn.com/)
(cores, cantos arredondados, sombras) em widgets **nativos** do OutSystems
— sem Block, sem JavaScript, só CSS. Mais simples que a seção 5 porque não
tem comportamento pra replicar, só aparência.

> **Não peça o CSS de memória.** Na primeira tentativa, o Mentor escreveu um
> CSS no formato antigo do shadcn (preset "zinc", cores em HSL). O shadcn
> atual usa Tailwind v4 e cores em `oklch`, e os valores são outros. Os
> valores abaixo foram convertidos do código-fonte oficial
> (`registry/new-york-v4/ui/button.tsx`, tema "neutral").

**6.1 — Colar o CSS na folha de estilo do tema**

No ODC Studio: app → **Interface → Themes →** o tema do app → folha de
estilo. Cole isto e publique:

```css
:root {
  --shadcn-radius: 0.625rem;
  --shadcn-radius-md: calc(var(--shadcn-radius) * 0.8);
  --shadcn-primary: oklch(0.205 0 0);
  --shadcn-primary-foreground: oklch(0.985 0 0);
  --shadcn-ring: oklch(0.708 0 0);
  --shadcn-focus-ring: 0 0 0 3px color-mix(in oklab, var(--shadcn-ring) 50%, transparent);
}

.shadcn-btn {
  box-sizing: border-box;
  margin: 0;
  border: 0;
  font-family: inherit;
  text-decoration: none;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  height: 2.25rem;
  padding: 0.5rem 1rem;
  border-radius: var(--shadcn-radius-md);
  font-size: 0.875rem;
  line-height: calc(1.25 / 0.875);
  font-weight: 500;
  white-space: nowrap;
  background-color: var(--shadcn-primary);
  color: var(--shadcn-primary-foreground);
  outline-style: none;
  transition: all 150ms cubic-bezier(0.4, 0, 0.2, 1);
}
/* o OutSystems UI pinta e sublinha links no hover; aqui a cor volta */
.shadcn-btn:is(:hover, :focus, :visited, :active) {
  color: var(--shadcn-primary-foreground);
  text-decoration: none;
}
@media (hover: hover) {
  .shadcn-btn:hover { background-color: color-mix(in oklab, var(--shadcn-primary) 90%, transparent); }
}
.shadcn-btn:focus-visible { box-shadow: var(--shadcn-focus-ring); }
.shadcn-btn:disabled { pointer-events: none; opacity: 0.5; }
```

Isso é só a variante padrão do Button. Tudo foi testado no ODC: o
compactador do tema mantém `oklch()`, `color-mix()` e `:is()`.

> **Atenção:** `transition: colors 0.15s ease` é sintaxe do Tailwind
> compilado, **não é CSS válido**. Em CSS puro é `transition-property` com
> as propriedades reais, ou `all`.

**6.2 — Aplicar no widget nativo**

No Button, na propriedade **Style Classes**, **troque** `btn` por
`shadcn-btn`. Com as duas classes o shadcn ainda ganha no visual padrão,
mas as regras do `.btn` voltam no hover, no foco e no estado desabilitado.

Publique e compare com a [documentação do shadcn/ui](https://ui.shadcn.com/docs/components/button).

**Por que isso funciona sem Block/JS:** o widget Button do OutSystems já
renderiza como uma tag `<button>` de verdade — só precisava das classes
CSS certas por cima. Pra componentes mais simples (Badge, Card, Alert)
o mesmo raciocínio se aplica. Pra componentes com **comportamento**
próprio do Radix UI (Dialog, Select, Combobox), essa técnica sozinha não
basta — aí entra a técnica da seção 5.

**Arquivo grande vai à mão.** Um CSS de ~100 KB mandado pelo Mentor
derrubou o turno. Colar direto no ODC Studio é instantâneo e não gasta
cota.

## 7. Links de referência, todos juntos

**Artigo-base da técnica Block + high-code:**
- [AI-Powered Frontend Development in OutSystems O11 and ODC](https://antonio-carvalho.medium.com/ai-powered-frontend-development-in-outsystems-o11-and-odc-4d408fdb4ade) — António Carvalho
- [How do I use an AI-powered IDE in OutSystems](https://medium.com/itnext/how-do-i-use-an-ai-powered-ide-in-outsystems-00fce740a08b) (Cursor, mesmo autor)
- [How I develop CSS and JavaScript 2x faster in OutSystems](https://itnext.io/how-i-develop-css-and-javascript-2x-faster-in-outsystems-b8b9ebc8675d) (mesmo autor)
- Blog do autor: [osfrontendtips.com](https://www.osfrontendtips.com/)

**MCP oficial da OutSystems:**
- [OutSystems/outsystems-mcp](https://github.com/OutSystems/outsystems-mcp)
- [OutSystems/docs-odc](https://github.com/OutSystems/docs-odc)

**shadcn/ui:**
- [ui.shadcn.com](https://ui.shadcn.com/) — documentação e componentes

**Documentação/logs deste repositório:**
- [README.md](README.md) — guia de estudo completo (MCP + técnica + POC)
- [poc-hello-ai-block-transcript.md](poc-hello-ai-block-transcript.md) — transcript real do POC HelloAiBlock
- [os-prep-quiz-log/](os-prep-quiz-log/) — log do projeto real usado como caso de teste do MCP
