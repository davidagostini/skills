---
name: session-checkpoint
description: >-
  Persist everything that happened in the current chat into DURABLE project files so no context is lost
  when switching AI assistant, switching machine, or reformatting and restoring the repo. Use whenever the
  user wants to "salvar a sessão", "fazer checkpoint", "documentar o que fizemos", "handoff", "não quero
  perder o contexto", "vou trocar de máquina/IA", "antes de formatar", "registra tudo no projeto",
  "save the session", "write a handoff", "snapshot the work so another AI can continue". Captures the
  session goal, decisions + rationale, files created/changed, commands, current state (works / pending /
  open), findings, next steps, and how to resume from scratch — written into the REPOSITORY (portable,
  survives a reformat) and mirrored into the local auto-memory. Proactively offer this at the end of a
  meaningful work session even if the user doesn't explicitly ask.
---

# Session Checkpoint — registro durável e portável da sessão

Objetivo: garantir que **nada do que aconteceu no chat se perca** — para que outra IA, outra máquina, ou um restore após formatação consigam **retomar o projeto sem você ter que reexplicar nada**.

## Princípio central (entenda antes de agir)

Há dois lugares para guardar contexto, com durabilidade diferente:

- **Repositório (docs no projeto)** → viaja com o código (git/backup/zip). **Sobrevive a formatar a máquina e trocar de IA.** É a **fonte da verdade portável**. *Tudo que importa para retomar o projeto vai aqui.*
- **Auto-memória** (`~/.claude/.../memory`) → rápida e ótima para o Claude local lembrar entre sessões, **mas é local da máquina e some num reformat** (a menos que haja backup). É um **cache**, não o registro definitivo.

Por isso a regra é: **escreva o registro durável no repo primeiro**; depois **espelhe** o essencial na memória. Nunca confie só na memória para algo que precisa sobreviver a uma troca de máquina.

## Onde escrever (descubra a convenção do projeto)

1. Ache a pasta de documentação do repo: procure `docs/`, `documentacao*/`, `documentacaoprojeto/`, ou similar. Se não houver, crie `docs/`.
2. Dentro dela, use uma subpasta de sessões: `sessoes/` (ou `sessions/`). Crie se não existir.
3. Mantenha **dois artefatos** com papéis distintos:
   - **Log da sessão (histórico, append-only):** `…/sessoes/SESSAO-AAAA-MM-DD-<slug>.md` — um arquivo por sessão. Nunca sobrescreva os antigos; eles são a trilha histórica.
   - **Snapshot "comece aqui" (rolante, sempre atual):** um único arquivo no topo da doc, ex. `…/RETOMAR-AQUI.md` (ou se o projeto já tiver um handoff/portável tipo `AGENT_HANDOFF.md`, `.claude/MEMORIA-AGENTES.md`, `ESTADO-ATUAL.md`, **atualize esse** em vez de criar outro). Este reflete **o estado mais recente** e é a primeira coisa que uma IA/pessoa nova deve ler.
4. Se o projeto já tem um arquivo portável de handoff, **respeite-o e o mantenha**; não duplique convenções.

## Workflow

1. **Descubra o destino** (passos acima) e confirme se já existe um arquivo "comece aqui" / handoff para atualizar.
2. **Reconstrua a sessão fielmente.** Se for repositório git, rode `git status`/`git diff --stat` para enumerar o que mudou de fato (não confie só na memória). Caso contrário, reconstrua pela conversa. Liste arquivos criados/alterados com o **porquê**, não só o quê.
3. **Seja honesto sobre lacunas.** Se o contexto foi resumido/comprimido e você não tem certeza de algo, marque como "⚠️ a confirmar" em vez de inventar. Um registro confiável vale mais que um completo-porém-errado.
4. **Escreva o log da sessão** (arquivo novo, timestamp) no formato abaixo.
5. **Atualize o snapshot "comece aqui"** com o estado atual (sobrescreve a seção de estado; pode manter um índice dos logs de sessão).
6. **Espelhe o essencial na auto-memória** (índice `MEMORY.md` + arquivo(s) de memória relevantes), deixando claro que o repo é a fonte durável. Linke os dois mundos (a memória pode apontar para o doc de sessão).
7. **Torne a memória portável (recomendado).** A auto-memória vive em `~/.claude/...` e **morre num reformat**. Para o registro sobreviver de fato, **exporte uma cópia da pasta de memória para dentro do repo** (ex.: `…/sessoes/.memoria-snapshot/` ou `.claude/memory-export/`) — assim, ao restaurar o projeto em outra máquina, dá para repovoar a memória local. É uma cópia simples dos `.md` da memória; não grave segredos.
8. **Não grave segredos.** Tokens, senhas reais, chaves, `.env` com valores → nunca em doc versionado. Use placeholders e diga onde o segredo real fica.
9. **Relate** ao usuário, com caminhos clicáveis, o que foi salvo e onde — e como restaurar.
10. **Ofereça a consolidação da memória — com guarda.** Ao final, sugira rodar a faxina da auto-memória (`consolidate-memory`: mesclar duplicatas, corrigir fatos obsoletos, podar índice). **Não rode automaticamente:** a auto-memória é compartilhada por todos os chats do projeto na mesma máquina; consolidar enquanto outro chat grava pode sobrescrever o trabalho dele. Antes de consolidar, **confirme com o usuário que este é o único chat ativo do projeto**. Se não for, apenas registre a sugestão e siga.

## Formato do log de sessão

```
# Sessão AAAA-MM-DD — <título curto>

> **TL;DR (retomar em 30s):** <1–2 frases: o que foi feito e onde parou>

## Objetivo da sessão
<o que o usuário queria>

## Decisões tomadas (com o porquê)
- <decisão> — porque <razão> (alternativas descartadas, se relevante)

## O que foi feito
- `caminho/arquivo` — <criado/alterado>: <o que mudou e por quê>
- Comandos relevantes: `<cmd>` (o que faz / resultado)

## Estado atual
- ✅ Funciona: <…>
- ⏳ Pendente: <…>
- ⚠️ Em aberto / a investigar / a confirmar: <…>

## Achados / problemas (se houver)
- <achado> — severidade/impacto — onde (arquivo:linha) — correção sugerida

## Próximos passos (ordenados)
1. <…>

## Como retomar do zero (restore)
- Pré-requisitos (runtimes, serviços), como subir, credenciais de seed (placeholder, sem segredo real), comandos de build/test.
- Onde está o segredo real (ex.: "valores em `.env`, não versionado; pedir ao responsável").

## Contexto para outra IA/máquina
- Docs-chave a ler primeiro: <links>
- Convenções/regras do projeto que não estão óbvias no código.
- Links externos (tickets, dashboards, repos relacionados).
```

## Snapshot "comece aqui" (estrutura mínima)

Mantenha curto e sempre atual — é a porta de entrada:

```
# Comece aqui — estado atual do projeto (atualizado em AAAA-MM-DD)

> **Onde estamos:** <2–4 linhas>

## Como rodar / restaurar
<o mínimo para subir o projeto>

## Próximos passos imediatos
1. <…>

## Histórico de sessões
- [AAAA-MM-DD — <título>](sessoes/SESSAO-AAAA-MM-DD-<slug>.md) — <hook de 1 linha>
```

## Quando oferecer proativamente

Ao fim de uma sessão com trabalho real (decisões, arquivos mudados, problemas resolvidos), **ofereça** fazer o checkpoint — é exatamente quando o contexto é mais valioso e mais fácil de perder. Também é o momento natural para sugerir **consolidar/compactar a auto-memória** (mesclar duplicatas, corrigir fatos obsoletos, podar o índice), tarefa irmã mas distinta — não misture: o checkpoint preserva a sessão; a consolidação faz faxina na memória.

> Princípio de ouro: se amanhã o usuário abrir o projeto numa máquina nova com outra IA, **o `RETOMAR-AQUI.md` + os logs de sessão devem bastar** para continuar. Escreva pensando nesse leitor frio.
