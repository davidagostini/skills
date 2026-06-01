---
name: project-memory
description: >-
  Cria e mantém uma memória de projeto PORTÁVEL e independente de ferramenta (Claude, Codex, Cursor,
  Gemini e outros) baseada numa pasta `/ai/` versionada no Git mais um `AGENTS.md`/`CLAUDE.md` de
  bootstrap. Use quando o usuário quiser "preparar o projeto para IA", "criar a estrutura /ai",
  "padronizar contexto entre IAs", "fazer o projeto funcionar igual no Claude e no Codex", "iniciar a
  memória do projeto", "init da memória", "estrutura AGENTS.md", "registrar contexto/decisões/pendências
  do projeto", "set up project memory", "bootstrap AGENTS.md", "make the repo agent-portable". Tem dois
  modos: INIT (gera a estrutura no repo) e CHECKPOINT (atualiza a pasta /ai ao fim de uma tarefa).
  Para um handoff narrativo/por sessão use a skill session-checkpoint; esta foca na estrutura /ai durável.
---

# Project Memory — contexto portável entre IAs (`/ai/` + AGENTS.md)

Objetivo: deixar o repositório **autoexplicável para qualquer agente de IA**, de modo que Claude, Codex, Cursor, Gemini etc. **leiam o mesmo contexto e continuem o trabalho um do outro** sem reexplicação.

## Princípio central (entenda antes de agir)

A portabilidade **não vem de uma skill** — skills são um mecanismo só do Claude. Ela vem de uma **convenção de arquivos versionados no Git**:

- **`ai/INSTRUCOES.md`** → **fonte única da verdade** do contrato: o que ler antes e o que atualizar depois de cada tarefa.
- **`AGENTS.md`** (raiz) → quase-padrão de mercado lido por Codex, Cursor, Zed e outros. É um **ponteiro fino** para `ai/INSTRUCOES.md` (sem duplicar conteúdo).
- **`CLAUDE.md`** (raiz) → ponteiro fino idêntico, para o Claude Code.
- **Pasta `/ai/`** → a memória durável (objetivo, stack, contexto atual, decisões, pendências, histórico).

Esta skill **gera e mantém** essa convenção. A camada portável é o que sobrevive à troca de ferramenta; a skill é só o automatizador no lado Claude.

> Relação com outras skills: **session-checkpoint** registra um *handoff narrativo por sessão* (o que aconteceu no chat). **project-memory** mantém a *estrutura durável `/ai/`* do projeto. As duas se complementam — o CHECKPOINT desta skill atualiza `/ai/`, e a session-checkpoint pode referenciar esses arquivos. Não duplique conteúdo entre elas.

## Modo INIT — criar a estrutura

Use quando o repo ainda não tem `/ai/` (ou o usuário pede para inicializar).

1. **Verifique o que já existe.** Procure `AGENTS.md`, `CLAUDE.md` e a pasta `ai/` na raiz. Se já existirem, **não sobrescreva** — leia, e só preencha lacunas/atualize. Confirme com o usuário antes de alterar arquivos existentes.
2. **Inspecione o projeto** para preencher placeholders de verdade (não deixe tudo "A definir"):
   - Stack/linguagem: olhe `package.json`, `pyproject.toml`, `go.mod`, `pom.xml`, `*.csproj`, etc.
   - Objetivo: README, descrição do repo, ou pergunte ao usuário em uma linha.
   - Estrutura de pastas relevante.
3. **Copie os templates** de `templates/` deste diretório para a raiz do projeto:
   - `templates/AGENTS.md` → `AGENTS.md` (ponteiro fino)
   - `templates/CLAUDE.md` → `CLAUDE.md` (ponteiro fino)
   - `templates/ai/*.md` → `ai/*.md` (inclui `ai/INSTRUCOES.md`, a fonte única da verdade)
4. **Preencha** `ai/MEMORIA.md` (objetivo, stack, padrões) e `ai/CONTEXTO.md` com o que você descobriu. Deixe `DECISOES/PENDENCIAS/HISTORICO` prontos para uso.
5. **Não comite** a menos que o usuário peça. Liste os arquivos criados e o que ficou preenchido vs. pendente.

## Modo CHECKPOINT — atualizar ao fim de uma tarefa

Use ao terminar um bloco de trabalho (ou quando o usuário pedir para "registrar"/"atualizar a memória").

1. Se for repo git, rode `git status` / `git diff --stat` para enumerar o que **de fato** mudou — não confie só na conversa.
2. Atualize, em ordem:
   - **`ai/HISTORICO.md`** — *append-only*. Adicione uma entrada datada: o que foi feito, arquivos tocados, testes rodados, resultado. Nunca apague entradas antigas.
   - **`ai/MEMORIA.md`** — atualize "Último estado conhecido", e stack/padrões se mudaram.
   - **`ai/CONTEXTO.md`** — reescreva para refletir o estado **atual** (tarefa em andamento, áreas em foco).
   - **`ai/PENDENCIAS.md`** — feche o que foi resolvido, adicione o que surgiu.
   - **`ai/DECISOES.md`** — registre decisões técnicas relevantes (ADR leve: contexto → decisão → motivo). Só decisões que importam, não toda mudança.
3. Use **datas absolutas** (ex.: `2026-06-01`), nunca "hoje"/"ontem".
4. Encerre informando, conforme `ai/PADRAO-GERAL.md`: o que foi feito, arquivos alterados, testes, pendências, riscos, próximo passo.

## Paridade com outras ferramentas (Codex/Cursor/Gemini)

O usuário não precisa desta skill nessas ferramentas — o `AGENTS.md`/`CLAUDE.md` já aponta qualquer agente para `ai/INSTRUCOES.md`, que manda ler e atualizar `/ai/`. Se o usuário pedir comandos equivalentes, oriente: basta abrir o projeto na outra ferramenta e pedir "siga o AGENTS.md" — o contrato é o mesmo. Como `AGENTS.md` e `CLAUDE.md` são só ponteiros, **não há conteúdo a sincronizar**: edite apenas `ai/INSTRUCOES.md`.

## Regras

- Trate `/ai/` como **fonte da verdade portável**; não invente convenções paralelas se o repo já tiver uma (ex.: `docs/`).
- Caminhos nos templates são **relativos** (`ai/...`), para funcionar em qualquer SO/checkout.
- Nunca apague histórico; `HISTORICO.md` e `DECISOES.md` são append-only.
- Não comitar nem fazer push sem o usuário pedir.
