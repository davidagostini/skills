---
name: pwa-webpush
description: >-
  Skill para implementar PWA instalável com Web Push (notificações em background) usando
  vite-plugin-pwa, Workbox, VAPID e service worker customizado. Use sempre que precisar
  configurar PWA, notificações push, service worker com cache offline, ou integrar Web Push
  num app React/Vite com backend .NET ou Node. Ativa em contextos como: "configurar PWA",
  "adicionar notificações push", "service worker", "vite-plugin-pwa", "VAPID keys",
  "offline-first", "instalar no celular", "Web Push", "manifest PWA", "ícones PWA",
  ou qualquer combinação de Vite + PWA + notificações. Esta skill cobre frontend (React/Vite)
  e backend (.NET) necessário para enviar as notificações.
---

# PWA + Web Push (VAPID) — Skill

Stack: **vite-plugin-pwa** (injectManifest) + **Workbox** + **VAPID** + **Service Worker TypeScript**

---

## 1. Setup vite-plugin-pwa

```bash
npm install vite-plugin-pwa workbox-precaching workbox-routing workbox-strategies -D
```

```typescript
// vite.config.ts
import { VitePWA } from 'vite-plugin-pwa'

export default defineConfig({
  plugins: [
    react(),
    VitePWA({
      strategies: 'injectManifest',   // usa seu SW customizado
      srcDir: 'src',
      filename: 'sw.ts',
      registerType: 'autoUpdate',     // nunca registrar manualmente no main.tsx
      injectManifest: {
        globIgnores: ['**/api/**', '**/*.map'],  // nunca cachear API calls
      },
      devOptions: { enabled: true, type: 'module' },  // testar push em dev
      manifest: {
        name: 'Nome do App',
        short_name: 'App',
        start_url: '/',
        display: 'standalone',
        background_color: '#ffffff',
        theme_color: '#000000',
        icons: [
          { src: '/logo-192.png', sizes: '192x192', type: 'image/png', purpose: 'any' },
          { src: '/logo-512.png', sizes: '512x512', type: 'image/png', purpose: 'any maskable' },
        ],
      },
    }),
  ],
})
```

---

## 2. Service Worker (src/sw.ts)

```typescript
import { cleanupOutdatedCaches, precacheAndRoute } from 'workbox-precaching'
declare const self: ServiceWorkerGlobalScope

precacheAndRoute(self.__WB_MANIFEST)   // Workbox injeta lista de assets em build
cleanupOutdatedCaches()

self.addEventListener('push', (event) => {
  let data: { title?: string; body?: string; icon?: string; url?: string } = {}
  try { data = event.data ? event.data.json() : {} }
  catch { data = { body: event.data?.text() ?? '' } }

  event.waitUntil(
    self.registration.showNotification(data.title ?? 'Notificação', {
      body: data.body ?? '',
      icon: data.icon ?? '/logo-192.png',
      badge: '/logo-192.png',
      data: { url: data.url ?? '/' },
    })
  )
})

self.addEventListener('notificationclick', (event) => {
  event.notification.close()
  const url = (event.notification.data as { url?: string } | null)?.url ?? '/'
  event.waitUntil(
    self.clients.matchAll({ type: 'window', includeUncontrolled: true }).then((clients) => {
      for (const client of clients) {
        if ('focus' in client) {
          void (client as WindowClient).navigate(url).catch(() => undefined)
          return client.focus()
        }
      }
      return self.clients.openWindow(url)
    })
  )
})
```

**tsconfig.app.json** — adicionar `WebWorker`:
```json
{ "compilerOptions": { "lib": ["ES2023", "DOM", "WebWorker"] } }
```

---

## 3. Ícones PNG (obrigatório para instalabilidade)

```javascript
// scripts/gen-icons.mjs — gera PNG sólido sem dependências externas
import { writeFileSync } from 'fs'
import { deflateSync } from 'zlib'

function crc32(buf) {
  let crc = 0xFFFFFFFF
  const t = Array.from({length:256},(_,n)=>{let c=n;for(let i=0;i<8;i++)c=(c&1)?(0xEDB88320^(c>>>1)):(c>>>1);return c>>>0})
  for (const b of buf) crc = t[(crc^b)&0xFF]^(crc>>>8)
  return ((crc^0xFFFFFFFF)>>>0)
}
function chunk(type, data) {
  const len=Buffer.alloc(4);len.writeUInt32BE(data.length)
  const td=Buffer.concat([Buffer.from(type,'ascii'),data])
  const crc=Buffer.alloc(4);crc.writeUInt32BE(crc32(td))
  return Buffer.concat([len,td,crc])
}
function solidPNG(w, h, r, g, b) {
  const sig=Buffer.from([137,80,78,71,13,10,26,10])
  const ihdr=Buffer.alloc(13);ihdr.writeUInt32BE(w,0);ihdr.writeUInt32BE(h,4);ihdr[8]=8;ihdr[9]=2
  const row=Buffer.alloc(1+w*3);row[0]=0
  for(let x=0;x<w;x++){row[1+x*3]=r;row[2+x*3]=g;row[3+x*3]=b}
  const raw=Buffer.concat(Array.from({length:h},()=>row))
  return Buffer.concat([sig,chunk('IHDR',ihdr),chunk('IDAT',deflateSync(raw,{level:6})),chunk('IEND',Buffer.alloc(0))])
}
// Substitua os valores RGB pela cor da sua marca
writeFileSync('public/logo-192.png', solidPNG(192, 192, 14, 138, 67))   // ex: #0E8A43
writeFileSync('public/logo-512.png', solidPNG(512, 512, 14, 138, 67))
console.log('Icones gerados!')
```

Para ícones com logo real: use `@vite-pwa/assets-generator` com o SVG da marca.

---

## 4. VAPID Keys

```bash
npx web-push generate-vapid-keys
# Public Key: BCW9...  (expor ao frontend via env)
# Private Key: REP6... (manter secreto no backend)
```

Variáveis de ambiente:
```
WebPush__Subject=mailto:admin@seudominio.com
WebPush__PublicKey=BCW9...
WebPush__PrivateKey=REP6...
```

---

## 5. Backend .NET — enviar notificações

```csharp
// NuGet: WebPush
public class WebPushSender(AppDbContext db, IHttpClientFactory httpFactory, IConfiguration config, ILogger<WebPushSender> logger)
{
    private readonly VapidDetails? _vapid = string.IsNullOrWhiteSpace(config["WebPush:PublicKey"]) ? null
        : new VapidDetails(config["WebPush:Subject"] ?? "mailto:admin@app.com",
                           config["WebPush:PublicKey"]!, config["WebPush:PrivateKey"]!);

    // IHttpClientFactory evita socket exhaustion — nunca new WebPushClient() sem ela
    private readonly WebPushClient _client = new(httpFactory.CreateClient("webpush"));

    public bool Habilitado => _vapid is not null;

    public async Task EnviarAsync(Guid userId, string titulo, string corpo, string? url = null, CancellationToken ct = default)
    {
        if (_vapid is null) return;
        var subs = await db.PushSubscriptions.Where(s => s.UserId == userId && s.Ativa).ToListAsync(ct);
        var payload = JsonSerializer.Serialize(new { title = titulo, body = corpo, url });
        var mudou = false;
        foreach (var s in subs) {
            try {
                await _client.SendNotificationAsync(new WebPush.PushSubscription(s.Endpoint, s.P256dh, s.Auth), payload, _vapid);
                s.UltimoSucessoEm = DateTime.UtcNow; mudou = true;
            }
            catch (WebPushException ex) when (ex.StatusCode is HttpStatusCode.NotFound or HttpStatusCode.Gone) {
                s.Ativa = false; mudou = true;  // assinatura expirada
            }
            catch (Exception ex) { logger.LogWarning(ex, "Falha push subscription {Id}", s.Id); }
        }
        if (mudou) await db.SaveChangesAsync(ct);
    }
}
```

---

## 6. Frontend — helper push.ts

```typescript
export function isPushSupported() {
  return 'serviceWorker' in navigator && 'PushManager' in window && 'Notification' in window
}

function urlBase64ToUint8Array(base64: string): Uint8Array<ArrayBuffer> {
  const padding = '='.repeat((4 - (base64.length % 4)) % 4)
  const b64 = (base64 + padding).replace(/-/g, '+').replace(/_/g, '/')
  const raw = atob(b64)
  const buffer = new ArrayBuffer(raw.length)
  const arr = new Uint8Array(buffer)
  for (let i = 0; i < raw.length; i++) arr[i] = raw.charCodeAt(i)
  return arr
}

export async function enablePush(): Promise<void> {
  if (!isPushSupported()) throw new Error('Push nao suportado neste browser.')
  const perm = await Notification.requestPermission()
  if (perm !== 'granted') throw new Error('Permissao negada.')
  const key = await fetchApi<{ publicKey: string } | undefined>('/push/vapid-key')
  if (!key?.publicKey) throw new Error('Push nao configurado no servidor.')
  const reg = await navigator.serviceWorker.ready
  const sub = await reg.pushManager.subscribe({
    userVisibleOnly: true,
    applicationServerKey: urlBase64ToUint8Array(key.publicKey),
  })
  const json = sub.toJSON()
  await fetchApi('/push/subscribe', {
    method: 'POST',
    body: JSON.stringify({ endpoint: sub.endpoint, p256dh: json.keys?.p256dh, auth: json.keys?.auth }),
  })
}
```

---

## 7. Limitações iOS

| Plataforma | Suporte |
|---|---|
| Android Chrome | Pleno |
| Desktop Chrome/Firefox/Edge | Pleno |
| iOS Safari 16.4+ **instalado** | Sim (Add to Home Screen obrigatorio) |
| iOS Safari sem instalar | Nao suporta push |
| iOS Chrome/Firefox | Nao (usam engine Safari) |

Para iOS: mostre banner pedindo para instalar antes de ativar push.
