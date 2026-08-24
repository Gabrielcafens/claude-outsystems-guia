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
Não testado ainda — anotar aqui assim que confirmar em qual versão você
está usando.

## Limitações conhecidas (preencher conforme for testando)

- [ ] *(ainda não testado o suficiente pra listar limitações reais)*

## Próximos passos

- [ ] Testar o fluxo de busca de elementos num ambiente pessoal (Personal
      Area) primeiro
- [ ] Documentar aqui os comandos que funcionaram bem, com exemplo real
- [ ] Anotar limitações encontradas (o que o skill não consegue fazer ainda)
