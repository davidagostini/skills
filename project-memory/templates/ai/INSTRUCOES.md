# Instruções do Projeto para Agentes de IA

Fonte única da verdade para **qualquer** agente de IA (Claude, Codex, Cursor, Gemini, etc.).
`AGENTS.md` e `CLAUDE.md` na raiz apenas apontam para este arquivo — não duplique conteúdo neles.

## Antes de qualquer tarefa, leia obrigatoriamente:

- `ai/PADRAO-GERAL.md`
- `ai/MEMORIA.md`
- `ai/CONTEXTO.md`
- `ai/DECISOES.md`
- `ai/PENDENCIAS.md`

Siga todas as regras definidas nesses arquivos.

## Ao finalizar qualquer tarefa, atualize:

- `ai/HISTORICO.md` — adicione uma entrada datada (append-only)
- `ai/MEMORIA.md` — "Último estado conhecido" e o que mudou
- `ai/CONTEXTO.md` — o estado atual
- `ai/PENDENCIAS.md` — feche/abra pendências
- `ai/DECISOES.md` — registre decisões técnicas relevantes (append-only)

Use sempre datas absolutas (AAAA-MM-DD). Nunca apague histórico nem decisões antigas.
