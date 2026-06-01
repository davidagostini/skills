---
name: aspnetcore-react-stack
description: >-
  Skill para projetos com stack ASP.NET Core 10 + EF Core + PostgreSQL + React 19 + Vite +
  Tailwind 4 + Hangfire + prometheus-net + Serilog + vite-plugin-pwa + DataProtection.
  Use SEMPRE que estiver desenvolvendo nesta stack — criar endpoint, corrigir bug, novo componente,
  refatorar, nova migration, novo job Hangfire, configurar observabilidade, ou qualquer
  desenvolvimento em projetos .NET + React com este conjunto de bibliotecas.
  Ative automaticamente quando o contexto usar .NET minimal APIs, EF Core, React com Vite,
  Hangfire, ou qualquer combinação destes. Carrega padrões corretos para evitar armadilhas comuns:
  N+1 queries, socket exhaustion, anti-IDOR, JWT handling, FluentValidation, DataProtection,
  Hangfire jobs, prometheus-net métricas e vite-plugin-pwa com service worker customizado.
---

# ASP.NET Core + React — Stack Skill

Você está trabalhando numa stack **.NET 10 + React 19 + PostgreSQL 17**.

> **Projeto específico:** leia `CLAUDE.md` e `CONVENTIONS.md` para regras e padrões do projeto atual.

---

## 1. Padrões canônicos .NET (Minimal APIs + EF Core)

### Auth + Claims
```csharp
// Pegar userId do JWT — nunca reinvente localmente
if (!principal.TryGetUserId(out var userId))
    return Results.Unauthorized();
```

### Contrato de erro
```csharp
// SEMPRE: { error } no idioma do projeto
return Results.BadRequest(new { error = "Mensagem amigável." });
// NUNCA: { message }, { detail }, { title } em código novo
// Status: 400 validação · 401 sem auth · 403 sem permissão · 404 não encontrado · 409 conflito
```

### Anti-IDOR — obrigatório em toda query com id externo
```csharp
// Sempre escopar pelo userId/membro antes de ler ou gravar
.Where(m => m.ResourceId == id && m.UserId == userId)
```

### EF Core — eliminar N+1
```csharp
// Carregar navegações na query inicial, não em loop
await db.Matches
    .Include(m => m.TimeCasa)
    .Include(m => m.TimeFora)
    .Where(...)
    .ToListAsync(ct);
// Depois usar propriedade inline: m.TimeCasa?.Nome ?? m.RotuloCasa ?? "fallback"
```

### FluentValidation
```csharp
// Validação de forma via AbstractValidator<T> + ValidationFilter<T>
// O filtro roda antes do handler — handler não faz validação inline
// Exceção: endpoints com anti-enumeration ficam fora do filtro
public class MeuDtoValidator : AbstractValidator<MeuDto>
{
    public MeuDtoValidator() =>
        RuleFor(x => x.Campo).NotEmpty().WithMessage("Campo obrigatório.");
}
// Registrar: builder.Services.AddValidatorsFromAssembly(typeof(Program).Assembly)
// Aplicar: .AddEndpointFilter<ValidationFilter<MeuDto>>()
```

### DataProtection — segredos em repouso
```csharp
// Nunca gravar tokens/chaves TOTP em texto puro
var cifrado = protector.Protect(plaintext);    // gravar no banco
var texto   = protector.Unprotect(stored);    // ler (fallback automático para legado)
```

### HttpClient — evitar socket exhaustion
```csharp
// NUNCA: new HttpClient() em Scoped/Transient
// SEMPRE: IHttpClientFactory
_client = new MeuClient(httpClientFactory.CreateClient("nome"));
// Registrar: builder.Services.AddHttpClient("nome")
```

### Thread-safety
```csharp
var rng = Random.Shared;  // não: new Random() por método
```

---

## 2. Hangfire — background jobs

```csharp
[DisableConcurrentExecution(timeoutInSeconds: 120)]
public class MeuJob(ILogger<MeuJob> logger)
{
    [JobDisplayName("Descrição no dashboard")]
    public async Task ExecutarAsync(CancellationToken ct = default)
    {
        try { /* lógica */ }
        catch (Exception ex) { logger.LogError(ex, "Erro no job"); throw; }
    }
}

// Registrar no startup (idempotente)
var jobs = scope.ServiceProvider.GetRequiredService<IRecurringJobManager>();
jobs.AddOrUpdate<MeuJob>("meu-job", j => j.ExecutarAsync(default), Cron.Minutely());

// Dashboard: /admin/hangfire — proteger com IDashboardAuthorizationFilter em produção
```

---

## 3. prometheus-net — métricas

```csharp
// Contadores estáticos: thread-safe, zero overhead quando OBSERVABILITY_ENABLED=false
private static readonly Counter _total = Metrics.CreateCounter(
    "app_eventos_total", "Descrição",
    new CounterConfiguration { LabelNames = ["tipo"] });
_total.WithLabels("login").Inc();

// Ativar no pipeline (gated por config)
if (config.GetValue<bool>("OBSERVABILITY_ENABLED")) {
    app.UseHttpMetrics();
    app.MapMetrics("/metrics");
}

// Convenção: prefixo_do_projeto_nome_unidade
// Ex: myapp_orders_total, myapp_payment_duration_seconds
```

---

## 4. Serilog — logging estruturado

```csharp
// Startup
builder.Host.UseSerilog((ctx, services, cfg) => cfg
    .ReadFrom.Configuration(ctx.Configuration)
    .Enrich.FromLogContext()
    .Enrich.WithProperty("app", "nome-app")
    .WriteTo.Console(
        formatter: ctx.HostingEnvironment.IsDevelopment()
            ? new MessageTemplateTextFormatter("[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj}{NewLine}{Exception}")
            : new CompactJsonFormatter())); // JSON CLEF para Loki/Promtail em produção

// Suprimir rotas de infra nos logs
app.UseSerilogRequestLogging(options => {
    options.GetLevel = (ctx, _, _) =>
        ctx.Request.Path.StartsWithSegments("/health") || ctx.Request.Path == "/metrics"
            ? LogEventLevel.Verbose : LogEventLevel.Information;
});
```

---

## 5. vite-plugin-pwa — PWA com SW customizado

```typescript
// vite.config.ts
VitePWA({
  strategies: 'injectManifest',   // seu src/sw.ts
  srcDir: 'src', filename: 'sw.ts',
  registerType: 'autoUpdate',     // não registrar manualmente no main.tsx
  manifest: { name: '...', theme_color: '#...', icons: [...] }
})

// src/sw.ts — Workbox precache + push handler
import { cleanupOutdatedCaches, precacheAndRoute } from 'workbox-precaching';
declare const self: ServiceWorkerGlobalScope;
precacheAndRoute(self.__WB_MANIFEST);
cleanupOutdatedCaches();
self.addEventListener('push', (e) => { /* showNotification */ });
self.addEventListener('notificationclick', (e) => { /* navigate */ });
```

---

## 6. React 19 + Vite — padrões

```tsx
// fetchApi wrapper — nunca fetch cru em componentes
const data = await fetchApi<T>('/api/endpoint', { method: 'POST', body: JSON.stringify(dto) });

// Erro da API
const msg = (err as ApiError).body?.error ?? err.message;

// Push para N usuários — paralelizar, não serializar
await Task.WhenAll(users.Select(id => SafePushAsync(sender, id, payload, ct)));

// Mobile-first: testar em ~375px antes de concluir qualquer UI
```

---

## 7. Onde mora a lógica

| O quê | Onde |
|---|---|
| Regras puras (sem infra) | `Domain/` |
| Regras de negócio + persistência | `Infrastructure/` |
| Orchestração + HTTP | `Api/Features/` (handlers finos) |
| Canal externo (email, WhatsApp, push) | Abstração única injetável |
| Validação de forma | `AbstractValidator<T>` + `ValidationFilter<T>` |

---

## 8. Definition of Done

- [ ] Auth via helper (`TryGetUserId`) sem cópia local?
- [ ] Erros no contrato `{ error }` + HTTP status correto?
- [ ] Queries com id externo escopadas por userId (anti-IDOR)?
- [ ] N+1 eliminado com `Include` na query inicial?
- [ ] Ação sensível auditada via logger centralizado?
- [ ] `dotnet build` e `npm run build` limpos?
- [ ] Testei em ~375px (se UI foi alterada)?
- [ ] Divergi de padrão documentado? Se sim, documentei o porquê?
