# Skills — Instalação e Atualização (Claude Code & Codex)

Repositório: `https://github.com/davidagostini/skills.git`

> **Substitua** `<SEU-USUARIO>` pelo seu usuário do Windows em todos os comandos.
> Os comandos abaixo são para o **Prompt de Comando (cmd.exe)**. Se usar PowerShell, troque `%USERPROFILE%` por `$HOME`.

As skills são apenas pastas com um `SKILL.md`. O Claude Code e o Codex descobrem automaticamente — não precisa de upload. A diferença entre as ferramentas é só a **pasta** onde elas ficam.

| Ferramenta | Pasta GLOBAL (todos os projetos) | Pasta LOCAL (só aquele projeto) |
|---|---|---|
| Claude Code | `%USERPROFILE%\.claude\skills\` | `<projeto>\.claude\skills\` |
| Codex CLI | `%USERPROFILE%\.codex\skills\` | `<projeto>\.codex\skills\` |

---

## CLAUDE CODE

### Global (recomendado para sincronizar entre máquinas)

Clona o repositório direto na pasta de skills do Claude:

```cmd
git clone https://github.com/davidagostini/skills.git %USERPROFILE%\.claude\skills
```

### Local (uma skill só para um projeto específico)

```cmd
cd C:\caminho\do\seu-projeto
git clone https://github.com/davidagostini/skills.git .claude\skills
```

> Se o projeto já é um repositório Git e você não quer aninhar repositórios, em vez de clonar copie só as pastas das skills para dentro de `.claude\skills\`.

### Verificar se carregou

Abra o Claude Code na pasta do projeto e rode:

```
/skills
```

---

## CODEX CLI

### Global

```cmd
git clone https://github.com/davidagostini/skills.git %USERPROFILE%\.codex\skills
```

### Local (por projeto)

```cmd
cd C:\caminho\do\seu-projeto
git clone https://github.com/davidagostini/skills.git .codex\skills
```

### Verificar / usar

```
/skills
```

Ou digite `$` seguido do nome da skill para chamá-la explicitamente. Ela também ativa sozinha quando o pedido bate com a descrição.

> **Precedência no Codex** (da maior para a menor): `.codex/skills/` no diretório atual → `.codex/skills/` na raiz do repo → `~/.codex/skills/` (usuário) → sistema.

---

## COMO ATUALIZAR AS SKILLS

Você edita as skills em qualquer máquina, sobe pro GitHub, e puxa nas outras.

### 1. Onde você editou (subir as mudanças)

```cmd
cd C:\projetos_ia\skills
git add .
git commit -m "atualiza skills"
git push
```

### 2. Nas outras máquinas (puxar as mudanças)

Vá até a pasta onde você clonou e rode `git pull`.

**Claude Code (global):**
```cmd
cd %USERPROFILE%\.claude\skills
git pull
```

**Codex (global):**
```cmd
cd %USERPROFILE%\.codex\skills
git pull
```

**Versão local (dentro de um projeto):**
```cmd
cd C:\caminho\do\seu-projeto\.claude\skills
git pull
```

### 3. Recarregar

- **Claude Code:** as mudanças entram em uma nova sessão.
- **Codex:** detecta as mudanças automaticamente; se não aparecer, **reinicie o Codex**.

---

## DICA: manter as duas ferramentas sincronizadas na mesma máquina

Se você usa **Claude e Codex no mesmo PC** e não quer clonar duas vezes, clone uma vez num lugar central e aponte as duas pastas para lá com um *link simbólico* (rode o cmd **como Administrador**):

```cmd
git clone https://github.com/davidagostini/skills.git C:\projetos_ia\skills-central

mklink /D %USERPROFILE%\.claude\skills C:\projetos_ia\skills-central
mklink /D %USERPROFILE%\.codex\skills  C:\projetos_ia\skills-central
```

Assim, um único `git pull` em `C:\projetos_ia\skills-central` atualiza as skills das duas ferramentas de uma vez.

---

## RESUMO RÁPIDO

| Quero... | Comando |
|---|---|
| Instalar global no Claude | `git clone <repo> %USERPROFILE%\.claude\skills` |
| Instalar global no Codex | `git clone <repo> %USERPROFILE%\.codex\skills` |
| Instalar local num projeto | `git clone <repo> .claude\skills` (ou `.codex\skills`) |
| Atualizar (onde editei) | `git add . && git commit -m "..." && git push` |
| Atualizar (outras máquinas) | `cd <pasta-skills>` + `git pull` |
| Ver skills carregadas | `/skills` |
