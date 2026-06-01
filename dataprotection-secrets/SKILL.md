---
name: dataprotection-secrets
description: >-
  Skill para cifrar segredos em repouso usando ASP.NET Core DataProtection.
  Use sempre que precisar proteger dados sensíveis no banco de dados (tokens TOTP/MFA,
  API keys, webhooks secrets, tokens de integração), configurar key ring persistente
  que sobrevive a redeploy, ou cifrar campos específicos de entidades EF Core.
  Ativa em contextos como: "cifrar segredos", "criptografar campos", "DataProtection",
  "segredos em repouso", "key ring", "MFA segredo", "token cifrado", "chave no banco",
  "certificado DataProtection", "proteger API key no banco", "PFX key ring",
  ou qualquer situação onde dados sensíveis precisam ser armazenados cifrados.
  Cobre: IDataProtector, key ring persistente, cifra com certificado PFX, backward
  compatibility com texto puro legado, e padrão ISecretProtector reutilizável.
---

# DataProtection — Cifrar Segredos em Repouso (.NET)

Problema que resolve: guardar segredos sensíveis (token de API, segredo TOTP, webhook secret)
no banco de dados de forma que um dump do banco não revele os valores.

---

## 1. Registrar DataProtection no startup

```csharp
// Program.cs
var dataProtection = builder.Services.AddDataProtection()
    .SetApplicationName("NomeUnicoDoApp");   // importante: mesmo nome em todas as instâncias

// Persistir key ring em volume (sobrevive a redeploy)
var keysPath = builder.Configuration["DataProtection:KeysPath"];
if (!string.IsNullOrWhiteSpace(keysPath)) {
    Directory.CreateDirectory(keysPath);
    dataProtection.PersistKeysToFileSystem(new DirectoryInfo(keysPath));
}

// Hardening: cifrar o próprio key ring com certificado PFX
var certPath = builder.Configuration["DataProtection:CertPath"];
var certPass = builder.Configuration["DataProtection:CertPassword"];
if (!string.IsNullOrWhiteSpace(certPath) && File.Exists(certPath))
    dataProtection.ProtectKeysWithCertificate(new X509Certificate2(certPath, certPass));

// Registrar ISecretProtector como singleton
builder.Services.AddSingleton<ISecretProtector, DataProtectionSecretProtector>();
```

**Variáveis de ambiente:**
```
DataProtection__KeysPath=/keys          # volume Docker
DataProtection__CertPath=/certs/dp.pfx  # certificado PFX
DataProtection__CertPassword=senha
```

**docker-compose.yml:**
```yaml
volumes:
  dataprotection_keys:

services:
  api:
    environment:
      DataProtection__KeysPath: /keys
      DataProtection__CertPath: /certs/dp.pfx
      DataProtection__CertPassword: ${DP_CERT_PASSWORD}
    volumes:
      - dataprotection_keys:/keys
      - ./deploy/dp_dev.pfx:/certs/dp.pfx:ro
```

---

## 2. Gerar certificado dev (self-signed)

```bash
# Linux/Mac/WSL
MSYS_NO_PATHCONV=1 openssl req -x509 -newkey rsa:2048 \
  -keyout /tmp/dp.key -out /tmp/dp.crt -days 3650 -nodes \
  -subj "/CN=DataProtection-Dev"

MSYS_NO_PATHCONV=1 openssl pkcs12 -export \
  -out deploy/dp_dev.pfx \
  -inkey /tmp/dp.key -in /tmp/dp.crt \
  -passout pass:dp_dev_2026
```

**Em produção:** use cert gerenciado (Let's Encrypt, KMS, Azure Key Vault).

---

## 3. Abstração ISecretProtector

```csharp
// Domain/Abstractions/ISecretProtector.cs
public interface ISecretProtector
{
    // Cifra para gravar no banco (adiciona prefixo dp1:)
    string Protect(string plaintext);

    // Decifra — compatível com texto puro legado (sem prefixo)
    // Retorna string.Empty se key ring incompatível (fail-safe)
    string Unprotect(string stored);
}
```

```csharp
// Infrastructure/Security/DataProtectionSecretProtector.cs
public class DataProtectionSecretProtector : ISecretProtector
{
    private const string Marcador = "dp1:";
    private readonly IDataProtector _protector;
    private readonly ILogger<DataProtectionSecretProtector> _logger;

    public DataProtectionSecretProtector(IDataProtectionProvider provider,
        ILogger<DataProtectionSecretProtector> logger)
    {
        _protector = provider.CreateProtector("App.Secrets.v1");
        _logger = logger;
    }

    public string Protect(string plaintext) => Marcador + _protector.Protect(plaintext);

    public string Unprotect(string stored)
    {
        // Texto puro legado (sem prefixo) — passa direto para compatibilidade
        if (string.IsNullOrEmpty(stored) || !stored.StartsWith(Marcador, StringComparison.Ordinal))
            return stored;

        try { return _protector.Unprotect(stored[Marcador.Length..]); }
        catch (Exception ex)
        {
            // Key ring incompatível (redeploy sem volume) — fail-safe: nega acesso
            _logger.LogError(ex, "Falha ao decifrar segredo — key ring incompativel?");
            return string.Empty;
        }
    }
}
```

---

## 4. Usar nos handlers

```csharp
// Gravar segredo cifrado
user.MfaSegredo = protector.Protect(segredoTotp);
await db.SaveChangesAsync(ct);

// Ler e usar segredo
var segredo = protector.Unprotect(user.MfaSegredo!);
if (string.IsNullOrEmpty(segredo))
    return Results.BadRequest(new { error = "Segredo indisponivel. Re-configure o MFA." });

// Verificar TOTP
var ok = totp.Verificar(segredo, dto.Codigo);
```

---

## 5. Backward compatibility (migração)

O prefixo `dp1:` permite detectar se um valor já foi cifrado:

```csharp
// Verificar se precisa cifrar (migração de dados legados)
if (!stored.StartsWith("dp1:"))
{
    // Valor em texto puro legado — cifrar agora
    entity.Campo = protector.Protect(stored);
    await db.SaveChangesAsync(ct);
}
```

---

## 6. Verificar key ring cifrado

Após configurar o certificado, o arquivo XML no volume deve conter `<encryptedSecret>`:

```bash
# Linux
cat /keys/*.xml | grep "encryptedSecret"
# Se vazio: key ring não está cifrado (aviso "No XML encryptor configured")

# Docker
docker exec container-api cat /keys/*.xml | grep "encryptedSecret"
```

---

## 7. Rotacionar key ring (após configurar certificado)

```bash
# Apaga key ring atual — nova chave cifrada é gerada no próximo startup
docker exec container-api rm /keys/*.xml
docker restart container-api
```

**Atenção:** qualquer segredo cifrado com a chave antiga se torna indecifrável.
O Unprotect retorna `string.Empty` (fail-safe) — usuários perdem acesso aos segredos e
precisam re-configurar (ex: re-enrolar MFA). Documente essa janela de manutenção.

---

## 8. Testes

```csharp
// Usando DataProtectionProvider.Create (disponível em testes sem infra)
using Microsoft.AspNetCore.DataProtection;

var protector = new DataProtectionSecretProtector(
    DataProtectionProvider.Create("App-Test"),
    NullLogger<DataProtectionSecretProtector>.Instance);

// Roundtrip
var cifrado = protector.Protect("meu-segredo");
Assert.Equal("meu-segredo", protector.Unprotect(cifrado));

// Legado passthrough
Assert.Equal("texto-puro", protector.Unprotect("texto-puro"));

// Key ring incompatível → fail-safe
var cifradoOutroApp = new DataProtectionSecretProtector(
    DataProtectionProvider.Create("App-Outro"), logger).Protect("x");
Assert.Equal(string.Empty, protector.Unprotect(cifradoOutroApp));
```
