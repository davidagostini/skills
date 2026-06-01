---
name: entranobolao-stack
description: >-
  Skill para desenvolvimento no projeto entranobolao.com.br (Bolão da Copa do Mundo).
  Use SEMPRE que estiver codificando, revisando ou adicionando features neste projeto.
  Stack: ASP.NET Core 10 + EF Core 10 + PostgreSQL 17 + React 19 + Vite 7 + Tailwind 4
  + Hangfire + prometheus-net + Serilog + vite-plugin-pwa + DataProtection.
  Ative com frases como: "adicionar feature", "criar endpoint", "corrigir bug", "refatorar",
  "implementar", "fazer", "criar componente", "adicionar rota", "novo serviço", "nova migration",
  qualquer desenvolvimento no contexto do bolão, palpites, grupos, WhatsApp, ranking,
  observabilidade, ou qualquer menção a entranobolao, bolão da copa, ou esta stack .NET + React.
  Esta skill DEVE ser usada mesmo quando o usuário não a menciona explicitamente — ative
  sempre que o contexto do projeto estiver presente.
---

# Entra no Bolão — Skill de Desenvolvimento

Você está trabalhando no **entranobolao.com.br** — bolão de palpites da Copa do Mundo 2026.

## 1. Carregue o contexto do projeto

Antes de qualquer código, leia estes arquivos se ainda não estiverem no contexto:

- `CLAUDE.md` — fonte de verdade: stack, decisões de produto, estado das features
- `CONVENTIONS.md` — **obrigatório**: padrões canônicos + DoD

Se os arquivos não estiverem no contexto da sessão, leia-os com Read.

---

## 2. Padrões canônicos (nunca reinvente)

### Backend (.NET)

```csharp
// SEMPRE: pegar userId do JWT
if (!principal.TryGetUserId(out var userId))
    return Results.Unauthorized();

// SEMPRE: erros no contrato canônico
return Results.BadRequest(new { error = "Mensagem pt-BR" });
// NUNCA: { message }, { detail }, { title }

// SEMPRE: auditoria em ações sensíveis
audit.Log(http, principal, "ACAO", "Entidade", id, payload);

// SEMPRE: anti-IDOR — filtrar queries pelo usuário
.Where(m => m.GroupId == id && m.UserId == userId)

// SEMPRE: validação de forma via ValidationFilter<T>
// NUNCA: if (string.IsNullOrWhiteSpace(...)) return BadRequest inline
// EXCETO: PasswordLogin (mantém 401 anti-enumeration — sem filtro)

// SEMPRE: segredos via ISecretProtector
var cifrado = protector.Protect(segredo);    // para gravar
var texto   = protector.Unprotect(stored);  // para ler (fallback legado automático)

// SEMPRE: push de grupo paralelo
await Task.WhenAll(membros.Select(uid => SafePushAsync(push, uid, payload, ct)));

// SEMPRE: N+1 eliminado com Include
await db.Matches.Include(m => m.TimeCasa).Include(m => m.TimeFora).Where(...).ToListAsync(ct);

// SEMPRE: Random.Shared (thread-safe, zero alocação)
var rng = Random.Shared;

// SEMPRE: HttpClient via IHttpClientFactory (evita socket exhaustion)
// NUNCA: new HttpClient() ou new WebPushClient() em Scoped/Transient
```

### Métricas de negócio

```csharp
// Prefixo bolao_ — registrar eventos nos handlers relevantes
AppMetrics.LoginAttempts.WithLabels("otp", "ok").Inc();
AppMetrics.GruposCriados.Inc();
AppMetrics.PalpitesEnviados.Inc();
// Ver src/Api/Metrics/AppMetrics.cs para lista completa
```

### WhatsApp / Jobs

```csharp
// SEMPRE: IMessagingService como choke point único (nunca Evolution direto)
await messaging.EnfileirarAsync(telefone, EventoGatilho.RESULTADO_SAIU, vars, userId, ct: ct);
// Jobs Hangfire: OutboundMessageJob (drena fila) + LembreteJob (T-2h/T-30min)
// Dashboard: /admin/hangfire (livre em dev, requer ADMIN em prod)
```

### Frontend (React)

```tsx
// SEMPRE: fetchApi (nunca fetch cru)
const data = await fetchApi<T>('/endpoint', { method: 'POST', body: JSON.stringify(dto) });

// SEMPRE: TotpInput para campos OTP de 6 dígitos
<TotpInput value={codigo} onChange={setCodigo} className="..." />

// Erro canônico da API: body.error (não message/detail/title)
const msg = (err as ApiError).body?.error ?? err.message;

// Mobile-first: testar em ~375px antes de concluir
```

---

## 3. Onde mora a lógica

| O quê | Onde |
|---|---|
| Regras de negócio (ranking, resultado, lembretes) | `Infrastructure/` (services) |
| Regras puras testáveis sem infra | `Domain/` (ScoringEngine, etc.) |
| Orchestração + autorização | `Api/Features/` (handlers finos) |
| Saída WhatsApp | `IMessagingService` (choke point) |
| Validação de forma | `AbstractValidator<T>` + `ValidationFilter<T>` |

---

## 4. HTTP status corretos

| Situação | Status |
|---|---|
| Validação de forma | 400 |
| Sem autenticação | 401 |
| Sem permissão / anti-IDOR | 403 ou 404 (anti-enumeration) |
| Recurso não encontrado | 404 |
| Conflito (duplicata) | 409 |

---

## 5. Definition of Done (percorra antes de cada commit)

- [ ] Usei `TryGetUserId` / `GetUserId` (zero cópia local)?
- [ ] Erros no contrato `{ error }` + HTTP status correto?
- [ ] Entrada validada por `AbstractValidator<T>` (ou dívida registrada)?
- [ ] Query com id do cliente escopada por userId/membro (anti-IDOR)?
- [ ] Ação sensível gravou `AuditLog` via `IAuditLogger`?
- [ ] Divergi de algo em `CLAUDE.md`? Se sim, documentei o porquê?
- [ ] `dotnet build` + `npm run build` limpos?
- [ ] Testei em ~375px (se mexeu em UI)?
- [ ] Métricas incrementadas no fluxo novo (se feature de produto)?

---

## 6. Contexto rápido do projeto

- **106 commits** — app v1 completo em produção (Docker local)
- **20 testes** passando: 16 Domain (ScoringEngine) + 4 Integration (Testcontainers)
- **Observabilidade**: Prometheus (Docker SD) + Grafana (35 painéis) + Loki + Tempo
  - App conectado via `obs-network` + labels `prometheus.scrape=true`
  - `/metrics` expõe métricas `bolao_*` (ativado por `OBSERVABILITY_ENABLED=true`)
- **Próxima fase**: deploy Coolify + Evolution API (WhatsApp) + Capacitor (iOS/Android)
- **Docs de entrada**: `docs/RETOMAR-AQUI.md` → `CLAUDE.md` → `CONVENTIONS.md`

---

## 7. Checklist pós-implementação para features de observabilidade

Se a feature nova gerar eventos rastreáveis:
1. Adicionar contador em `AppMetrics.cs` com prefixo `bolao_`
2. Chamar `.Inc()` no handler após o `SaveChangesAsync`
3. Verificar em `/metrics` se a métrica aparece
4. Grafana vai coletar automaticamente (Docker SD já configurado)

---

> **Regra de ouro:** quando em dúvida sobre um padrão, leia `CONVENTIONS.md` antes de inventar. Se o padrão não estiver lá e você criar um novo, documente em `CONVENTIONS.md` para as próximas iterações.
