---
layout: post
title: "Usando Claude Code para programar em Lotus Notes e XPages com Team Development"
date: 2026-02-27
categories: [domino, xpages, claude-code, team-development, git]
---

Se voce desenvolve em HCL Domino e ainda nao experimentou usar IA para auxiliar na programacao de LotusScript, Java, XPages e formulas, este post vai mostrar como integramos o **Claude Code** ao fluxo de desenvolvimento do Domino usando **Team Development** e **Git**.

O resultado: um assistente de IA que entende o contexto completo da sua aplicacao Domino — forms, views, agents, XPages, script libraries — e pode sugerir, corrigir e criar codigo diretamente nos arquivos do projeto.

## Pre-requisitos

- HCL Domino Designer (com suporte a Team Development)
- Git instalado na maquina
- Conta no GitHub (ou GitLab, Bitbucket)
- Claude Code instalado (`npm install -g @anthropic-ai/claude-code`)

## O fluxo completo

```
NSF (Domino Designer)
    |
    v
[1] Team Development: Export NSF → ODP (On-Disk Project)
    |
    v
[2] Criar repositorio Git no diretorio do ODP
    |
    v
[3] Commit + Push para o GitHub
    |
    v
[4] Abrir Claude Code no diretorio do projeto
    |
    v
[5] Programar com assistencia de IA (LotusScript, Java, XPages, formulas)
    |
    v
[6] Commit + Push das alteracoes
    |
    v
[7] Sincronizar ODP de volta para o Domino Designer
    |
    v
[8] Clean + Build (para XPages)
    |
    v
[9] Testar a aplicacao
```

## Passo 1: Exportar o NSF para ODP via Team Development

No Domino Designer:

1. Abra o banco de dados NSF que deseja versionar
2. Va em **File → Application → Properties**
3. Na aba **Design**, marque **"Enable source control integration"** (se ainda nao estiver habilitado). Ao clicar OK o Designer vai criar o ODP em um diretorio local
4. O Designer pede para escolher o diretorio destino. Escolha um local como:

```
C:\projetos\minha-aplicacao\ntf\
```

Apos a exportacao, o diretorio tera a estrutura tipica de um ODP:

```
ntf/
├── AppProperties/
├── Code/
│   ├── Agents/
│   ├── ScriptLibraries/
│   └── Java/
├── CustomControls/
├── Forms/
├── Pages/
├── Resources/
│   ├── Files/
│   ├── Images/
│   └── StyleSheets/
├── SharedElements/
│   └── Subforms/
├── Views/
├── XPages/
└── META-INF/
    └── MANIFEST.MF
```

Cada elemento de design vira um ou mais arquivos no disco:
- **Forms** → arquivos `.form`
- **Views** → arquivos `.view`
- **Agents** → arquivos `.lsa` (LotusScript), `.ja` (Java), `.fa` (Formula)
- **Script Libraries** → arquivos `.lss` (LotusScript), `.javalib` (Java)
- **XPages** → arquivos `.xsp`
- **Custom Controls** → arquivos `.xsp`

## Passo 2: Criar o repositorio Git

No terminal, dentro do diretorio do projeto:

```bash
cd C:\projetos\minha-aplicacao
git init
```

Crie um `.gitignore` para evitar arquivos desnecessarios:

```
# .gitignore para projetos Domino ODP
*.nsf
*.ntf
*.ns5
*.ns6
*.ns7
*.ns8
build/
local.properties
plugin.xml
.project
.classpath
.settings/
```

Faca o primeiro commit:

```bash
git add .
git commit -m "Initial ODP export from Domino Designer"
```

Crie o repositorio no GitHub e faca o push:

```bash
git remote add origin https://github.com/seu-usuario/minha-aplicacao.git
git push -u origin main
```

## Passo 3: Abrir Claude Code no projeto

Com o repositorio pronto, abra o Claude Code no diretorio do projeto:

```bash
cd C:\projetos\minha-aplicacao
claude
```

O Claude Code vai indexar todos os arquivos do ODP e entender a estrutura da aplicacao Domino. Ele consegue ler e modificar:

- **LotusScript** em agents (`.lsa`) e script libraries (`.lss`)
- **Java** em agents (`.ja`) e Java libraries (`.javalib`)
- **XPages XML** em `.xsp` files
- **JavaScript (SSJS/CSJS)** embutido nas XPages
- **Formulas** em views, forms e agents
- **Propriedades** de forms e views (DXL/XML)

### Exemplo: pedir ajuda com um agent LotusScript

```
> Analise o agent "EnviaEmail" em Code/Agents/EnviaEmail.lsa e sugira
  melhorias para tratamento de erro e logging
```

### Exemplo: criar um Custom Control XPage

```
> Crie um Custom Control chamado ccDatePicker.xsp que use o componente
  xe:djDateTextBox com validacao de data no formato dd/MM/yyyy
```

### Exemplo: otimizar uma view formula

```
> A view "VendasPorMes" em Views/VendasPorMes.view esta lenta.
  Analise a formula de selecao e as colunas e sugira otimizacoes
```

### Exemplo: migrar LotusScript para Java

```
> Converta o agent LotusScript "ProcessaImportacao.lsa" para um
  agent Java equivalente, mantendo a mesma logica de negocio
```

## Dica: crie um CLAUDE.md no projeto

O arquivo `CLAUDE.md` na raiz do projeto fornece contexto permanente ao Claude Code. Para projetos Domino, inclua:

```markdown
# Projeto: Minha Aplicacao

## Tipo
Aplicacao HCL Domino/Notes com XPages

## Estrutura
- ntf/ - On-Disk Project (ODP) exportado do Domino Designer
- Forms: entidades principais de dados
- Views: indices e consultas
- Agents: logica de negocio automatizada
- XPages: interface web
- Custom Controls: componentes reutilizaveis

## Convencoes
- LotusScript segue o padrao de nomenclatura existente
- Agents agendados usam prefixo "Sched_"
- Views internas usam prefixo "_intra"
- XPages usam o framework de controles customizados do projeto

## Servidor
- Versao: HCL Domino 14.0
- XPages runtime ativo

## Observacoes
- Nao modificar o MANIFEST.MF manualmente
- Forms e views sao XML (DXL) - editar com cuidado
- Testar sempre via Clean + Build antes de deploy
```

## Passo 4: Programar e fazer commit

Apos o Claude Code fazer as alteracoes nos arquivos do ODP, revise e faca o commit:

```bash
git add .
git commit -m "Adicionar tratamento de erro no agent EnviaEmail"
git push
```

## Passo 5: Sincronizar de volta para o Domino Designer

Agora vem a parte que fecha o ciclo. As alteracoes feitas pelo Claude Code nos arquivos do ODP precisam voltar para o NSF no Domino Designer:

1. Abra o **Domino Designer**
2. Abra o banco de dados NSF vinculado ao ODP
3. O Designer deve detectar que os arquivos no disco mudaram
4. Se nao detectar automaticamente, clique com o botao direito no projeto e selecione **Team Development → Sync with On-Disk Project**
5. O Designer vai importar as alteracoes do ODP para o NSF

## Passo 6: Clean e Build (para XPages)

Se voce modificou XPages ou Custom Controls, e necessario fazer Clean e Build:

1. No Domino Designer, va em **Project → Clean**
2. Selecione o projeto e clique **OK**
3. Aguarde o clean completar
4. Va em **Project → Build All** (ou o build automatico ja faz isso)

O Clean remove os artefatos compilados anteriores. O Build recompila todas as XPages, Custom Controls e Java code. Sem esse passo, voce pode ver versoes antigas do codigo rodando.

**Importante**: se voce alterou apenas LotusScript ou formulas (sem XPages), o Clean/Build nao e necessario — as mudancas entram em vigor imediatamente apos a sincronizacao.

## Passo 7: Testar

- **XPages**: abra o navegador e acesse a URL da XPage (ex: `http://domino-server/app.nsf/pagina.xsp`)
- **Notes Client**: abra o banco pelo Notes client para testar forms, views e agents
- **Agents agendados**: verifique o log do servidor ou execute manualmente pelo Designer

## Fluxo no dia a dia

Apos a configuracao inicial, o ciclo de desenvolvimento fica assim:

```
1. Abrir Claude Code no diretorio do projeto
2. Descrever o que precisa (correcao, nova feature, refatoracao)
3. Claude Code edita os arquivos do ODP
4. Revisar as alteracoes (git diff)
5. Commit + Push
6. No Designer: Sync with On-Disk Project
7. Clean + Build (se XPages)
8. Testar
9. Voltar ao passo 1
```

## O que funciona bem com Claude Code + Domino

| Tarefa | Eficacia |
|---|---|
| Analisar e corrigir LotusScript | Excelente — entende a linguagem e o contexto |
| Criar/modificar XPages XML | Muito bom — gera XHTML valido com componentes XSP |
| Otimizar formulas de view | Bom — sugere melhores abordagens para selecao e colunas |
| Criar agents Java | Excelente — gera codigo Java completo com imports |
| Refatorar Script Libraries | Muito bom — reorganiza e modulariza o codigo |
| Escrever SSJS para XPages | Bom — conhece a API do XPages (SSJS runtime) |
| Editar propriedades de forms/views (DXL) | Com cautela — DXL e sensivel a formatacao |

## Cuidados importantes

- **Sempre faca backup do NSF** antes de sincronizar alteracoes extensas
- **Nao edite o MANIFEST.MF** manualmente — ele e gerenciado pelo Designer
- **Revise o DXL** de forms e views com atencao — um atributo errado pode corromper o elemento
- **Teste em ambiente de desenvolvimento** antes de levar para producao
- **O Clean apaga e reconstroi** — pode levar alguns minutos em projetos grandes

---

O Team Development do Domino Designer, combinado com Git e Claude Code, transforma o desenvolvimento Notes/XPages em uma experiencia moderna sem abandonar a plataforma. O codigo fica versionado, o historico e rastreavel, e voce tem um assistente de IA que entende LotusScript, Java, XPages e formulas do Domino.
