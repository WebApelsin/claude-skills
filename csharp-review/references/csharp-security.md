# C# / .NET Security Review

The C#-specific security patterns to check whenever code touches untrusted input, authentication,
authorization, data access, serialization, secrets, or logging. Each entry is an `❌ / ✅` pair.
Most of these are 🔴 when present on a path reachable by untrusted input.

## Contents

- [Injection (SQL & commands)](#injection-sql--commands)
- [Insecure deserialization](#insecure-deserialization)
- [Over-posting / mass assignment](#over-posting--mass-assignment)
- [Authentication vs authorization](#authentication-vs-authorization)
- [Secrets & configuration](#secrets--configuration)
- [Sensitive data in logs](#sensitive-data-in-logs)
- [Cryptography](#cryptography)
- [Transport, cookies & CORS](#transport-cookies--cors)
- [Input validation & SSRF/path](#input-validation--ssrf--path)

---

## Injection (SQL & commands)

### SQL built by string concatenation / interpolation

```csharp
// ❌ Classic SQL injection — user input concatenated into SQL
var users = await _db.Users
    .FromSqlRaw($"SELECT * FROM Users WHERE Name = '{name}'")
    .ToListAsync(ct);

// ✅ Parameterize. FromSqlInterpolated parameterizes interpolated holes safely
var users = await _db.Users
    .FromSqlInterpolated($"SELECT * FROM Users WHERE Name = {name}")
    .ToListAsync(ct);
// or FromSqlRaw with explicit SqlParameter values
```

Note: `FromSqlInterpolated` is safe because it turns the holes into parameters; `FromSqlRaw` with
a pre-built interpolated string is **not**. Flag any dynamic SQL where user input reaches the
query text rather than a parameter. Same principle for Dapper (`@param`, never string-built) and
ADO.NET (`SqlCommand.Parameters`).

### Command / LDAP / path injection

User input flowing into `Process.Start`, shell arguments, LDAP filters, or file paths needs the
same scrutiny — validate against an allow-list and never concatenate raw input into a command.

---

## Insecure deserialization

### `BinaryFormatter` and friends

`BinaryFormatter`, `SoapFormatter`, `NetDataContractSerializer`, and `LosFormatter` are
remote-code-execution vectors on untrusted data and are obsolete/removed in modern .NET. Any use
on untrusted input is 🔴.

```csharp
// ❌ RCE risk; obsolete
var obj = new BinaryFormatter().Deserialize(stream);

// ✅ System.Text.Json with known types
var obj = JsonSerializer.Deserialize<MyDto>(json);
```

### Polymorphic JSON / type-name handling

Deserializers that embed CLR type names let an attacker instantiate arbitrary types.

```csharp
// ❌ Newtonsoft with TypeNameHandling.All/Auto on untrusted input
var o = JsonConvert.DeserializeObject(json,
    new JsonSerializerSettings { TypeNameHandling = TypeNameHandling.All });

// ✅ Deserialize to concrete known types; if polymorphism is needed, use a
// vetted, constrained type discriminator (System.Text.Json [JsonPolymorphic]/[JsonDerivedType])
```

---

## Over-posting / mass assignment

Binding request bodies straight onto EF entities lets a client set fields they shouldn't (e.g.
`IsAdmin`, `Balance`, `Id`).

```csharp
// ❌ The whole entity is bound from the request — client can set IsAdmin
[HttpPost] public async Task<IActionResult> Create(User user) { _db.Add(user); … }

// ✅ Bind a DTO that exposes only the writable fields, then map deliberately
[HttpPost] public async Task<IActionResult> Create(CreateUserDto dto)
{
    var user = new User { Name = dto.Name, Email = dto.Email }; // IsAdmin not settable here
    _db.Add(user); …
}
```

Use input DTOs as the model-binding surface; never bind directly onto persistence entities.

---

## Authentication vs authorization

### Missing authorization (authenticated ≠ allowed)

The most common real-world hole: the user is logged in, but nothing checks they own the resource
they're acting on (IDOR).

```csharp
// ❌ Any authenticated user can fetch anyone's order by guessing the id
[Authorize] [HttpGet("orders/{id}")]
public async Task<IActionResult> Get(int id) =>
    Ok(await _db.Orders.FindAsync(id));

// ✅ Scope to the caller / check ownership (resource-based authorization)
[Authorize] [HttpGet("orders/{id}")]
public async Task<IActionResult> Get(int id)
{
    var order = await _db.Orders.FirstOrDefaultAsync(
        o => o.Id == id && o.UserId == _currentUser.Id, ct);
    return order is null ? NotFound() : Ok(order);
}
```

Also flag: `[AllowAnonymous]` on endpoints that handle sensitive actions; role checks done in the
UI but not the API; authorization logic duplicated and drifting across endpoints (centralize via
policies / `IAuthorizationHandler`).

---

## Secrets & configuration

### Hard-coded secrets

```csharp
// ❌ Secret in source — leaks via VCS history forever
var key = "sk-live-abc123…";
```

Flag connection strings, API keys, tokens, and passwords in source, `appsettings.json` committed
to the repo, or test files. Secrets belong in user-secrets (dev), environment variables, or a
vault (Azure Key Vault / AWS Secrets Manager) bound through `IConfiguration`. Recommend rotating
any secret that reached source control — it's compromised.

---

## Sensitive data in logs

Logging request bodies, tokens, full exception detail, or PII (emails, names, payment data,
location) writes sensitive data to sinks with weaker access controls and broader retention. This
is both a privacy and a compliance issue.

```csharp
// ❌ Logs PII and possibly secrets
_logger.LogInformation("Login: {Body}", JsonSerializer.Serialize(request));

// ✅ Log identifiers and outcomes, not the payload
_logger.LogInformation("Login attempt for user {UserId}: {Result}", userId, result);
```

Watch for: tokens/passwords in logs, structured-logging templates that capture whole DTOs, and
exception logging that includes user input verbatim. (If the project has PII rules, hold logging
to them.)

---

## Cryptography

- **Password hashing:** never MD5/SHA-1/SHA-256 raw. Use a slow, salted KDF — ASP.NET Core
  Identity's hasher, or `Rfc2898DeriveBytes`/Argon2/bcrypt. Flag any fast-hash of passwords as 🔴.
- **Randomness for security:** `System.Random` is predictable. Use
  `RandomNumberGenerator.GetBytes` / `GetInt32` for tokens, salts, and anything security-relevant.
- **Roll-your-own crypto / hard-coded IVs / ECB mode:** flag. Use vetted, authenticated
  algorithms (AES-GCM) and platform APIs.

```csharp
// ❌ Predictable token
var token = new Random().Next().ToString();

// ✅ Cryptographically secure
var token = Convert.ToHexString(RandomNumberGenerator.GetBytes(32));
```

---

## Transport, cookies & CORS

- **CORS too open:** `AllowAnyOrigin()` combined with credentials, or reflecting the `Origin`
  header, exposes authenticated endpoints to any site. Allow a specific origin list.
- **Cookies:** auth cookies should be `HttpOnly`, `Secure`, and an appropriate `SameSite`.
- **Anti-forgery:** cookie-authenticated state-changing endpoints need anti-forgery protection
  (`[ValidateAntiForgeryToken]`); pure token-auth APIs generally don't, but verify the auth model.
- **HTTPS:** flag disabled cert validation
  (`ServerCertificateCustomValidationCallback => true`) — it defeats TLS.

---

## Input validation & SSRF / path

- **Validate at the boundary:** length, range, format, allow-lists on all external input. Don't
  rely on client-side validation.
- **SSRF:** if the server fetches a URL the user supplied, an attacker can pivot to internal
  services / cloud metadata endpoints. Restrict to an allow-list of hosts/schemes.
- **Path traversal:** user-controlled file names can escape the intended directory
  (`../../etc/...`). Canonicalize and confirm the resolved path stays within the allowed root;
  never concatenate user input straight into a path.

```csharp
// ❌ Path traversal — userFile can be "../../appsettings.json"
var path = Path.Combine(root, userFile);

// ✅ Resolve and verify containment
var full = Path.GetFullPath(Path.Combine(root, userFile));
if (!full.StartsWith(Path.GetFullPath(root), StringComparison.Ordinal))
    return BadRequest();
```
