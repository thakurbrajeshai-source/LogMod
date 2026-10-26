# Insurance Agent Authentication API

This is a new .NET 10 backend alongside the existing AuthKit service. It does
not add or modify frontend code.

## Project layout

| Project | Responsibility |
| --- | --- |
| `LoginScree/InsuranceAuth/src/InsuranceAuth.Domain/` | Authentication domain records, results, and errors |
| `LoginScree/InsuranceAuth/src/InsuranceAuth.Application/` | Use cases, repository/client contracts, and FluentValidation rules |
| `LoginScree/InsuranceAuth/src/InsuranceAuth.Infrastructure/` | SQL Server stored-procedure repositories, Argon2id, external HTTP clients, and dev fakes |
| `LoginScree/InsuranceAuth/src/InsuranceAuth.Api/` | ASP.NET Core controllers, bearer auth, ProblemDetails envelope middleware, rate limiting, and Swagger |
| `LoginScree/InsuranceAuth/tests/InsuranceAuth.Tests/` | Focused use-case tests with fake repositories and external clients |

All SQL lives in `LoginScree/InsuranceAuth/sql/insurance-auth/`. The data layer uses
`Microsoft.Data.SqlClient`, explicit typed parameters, stored procedures, and
async I/O. Password, OTP, and token values are not logged.

## Requirements and database setup

- .NET 10 SDK
- SQL Server 2019+ (LocalDB works for local development)
- `sqlcmd`

Create the database, then run the scripts in order from the repository root:

```powershell
sqlcmd -S "(localdb)\MSSQLLocalDB" -E -Q "IF DB_ID(N'InsuranceAuth') IS NULL CREATE DATABASE [InsuranceAuth];"
Get-ChildItem .\LoginScree\InsuranceAuth\sql\insurance-auth\*.sql | Sort-Object Name | ForEach-Object {
  sqlcmd -S "(localdb)\MSSQLLocalDB" -E -d InsuranceAuth -b -i $_.FullName
  if ($LASTEXITCODE -ne 0) { throw "Failed: $($_.Name)" }
}
```

The numbered migrations are re-runnable. `004_Seed.sql` inserts only default
policy metadata; it does not insert an account with a fabricated password
hash. In Development, the opt-in API seeder creates the demo user and Argon2id
hash using the configured password.

The registration workflow is added by `005_Registration.sql`; apply it after
the initial four scripts. It creates pending-registration storage and the
AgentCode sequence required by the registration API.

## Configure and run

Set secrets through environment variables or a secret provider; do not put
production secrets in `appsettings.json`.

```powershell
$env:ConnectionStrings__AuthDatabase = "Server=(localdb)\MSSQLLocalDB;Database=InsuranceAuth;Integrated Security=true;TrustServerCertificate=true"
$env:DevSeed__Enabled = "true"
$env:DevSeed__Password = "Provide-A-Local-Strong-Password1"
dotnet run --project .\LoginScree\InsuranceAuth\src\InsuranceAuth.Api
```

The included launch profile sets `ASPNETCORE_ENVIRONMENT=Development` and
listens on `http://localhost:5080`. If launching without that profile, explicitly
set `$env:ASPNETCORE_ENVIRONMENT = "Development"` before running; otherwise
the API correctly requires production external-service configuration and JWT
authority settings.

Open Swagger at `http://localhost:5080/swagger`. Development uses local fake
CAPTCHA, OTP, and token services:

- CAPTCHA id/text: `dev-captcha` / `ABCD`
- OTP: `000000`
- The demo account is `AGENT001`, `agent@example.test`, or `IRDA-DEMO-0001`.
- The demo password is the value of `DevSeed__Password`.

Development access tokens have the form `dev.<user-guid>.<session-guid>` and
are accepted only by the development authentication handler. Do not deploy
with `ASPNETCORE_ENVIRONMENT=Development`.

Production settings are configured as follows (double underscores represent
configuration section separators):

```text
ConnectionStrings__AuthDatabase
Authentication__Authority
Authentication__Audience
ExternalApis__Captcha__BaseUrl
ExternalApis__Captcha__ApiKey
ExternalApis__Captcha__TimeoutSeconds
ExternalApis__Otp__BaseUrl
ExternalApis__Otp__ApiKey
ExternalApis__Otp__TimeoutSeconds
ExternalApis__Token__BaseUrl
ExternalApis__Token__ApiKey
ExternalApis__Token__TimeoutSeconds
```

Each production `ExternalApis__{Captcha|Otp|Token}__BaseUrl` must be an
absolute `http://` or `https://` service URL. Startup reports the exact
configuration key if one is empty or invalid; production never falls back to
the development fakes.

The service validates JWT bearer tokens using the configured authority and
audience. Access and refresh token issuance is delegated to the configured
token service. Refresh is intentionally not implemented here.

## Assumed external HTTP contracts

The task did not include actual CAPTCHA, OTP, or token API contracts. The
current typed clients therefore use configurable relative paths and these
explicit provisional JSON contracts; replace/adjust the DTOs and paths in
`LoginScree/InsuranceAuth/src/InsuranceAuth.Infrastructure/ExternalClients.cs` when the service owners provide the real
contracts. Each service may send an `X-API-Key` header. Non-success HTTP
statuses are raised as errors; transient network/5xx responses are retried
three times by Polly.

CAPTCHA:

```http
POST {Captcha.BaseUrl}/{ValidatePath}
X-API-Key: ...
Content-Type: application/json
{"captchaId":"...","captchaText":"..."}
```

```json
{"valid":true}
```

OTP send and verify:

```http
POST {Otp.BaseUrl}/{SendPath}
{"destinationType":"PrimaryMobile","destinationValue":"+91...","purpose":"PasswordReset"}
```

```json
{"otpReferenceId":"..."}
```

```http
POST {Otp.BaseUrl}/{VerifyPath}
{"otpReferenceId":"...","otp":"..."}
```

```json
{"verified":true}
```

Token issue:

```http
POST {Token.BaseUrl}/{IssuePath}
{"user":{"userId":"GUID","agentCode":"...","fullName":"...","primaryEmail":"...","irdaLicenseNumber":"..."},"sessionId":"GUID"}
```

```json
{"accessToken":"...","refreshToken":"...","expiresUtc":"2030-01-01T00:00:00Z","tokenId":"..."}
```

## API examples

Login (identifier may be agent code, primary email, or IRDA license number):

```http
POST /api/auth/login
Content-Type: application/json
{"loginId":"AGENT001","password":"<configured-development-password>"}
```

Successful responses use `{ success, message, data, errors, errorCode }`.
For a wrong password or unknown login ID the API returns the same generic
`INVALID_CREDENTIALS` response. On the fourth consecutive failed password
attempt, it returns HTTP 423 with `errorCode: ACCOUNT_LOCKED` and
`data.lockoutEndUtc`.

Reset flow, in required order:

```http
POST /api/auth/reset/identify
{"loginId":"agent@example.test","captchaId":"dev-captcha","captchaText":"ABCD"}
```

The response contains `resetId` and masked destination choices. Unknown IDs
receive HTTP 200 and the same generic message, without reset destinations.

```http
POST /api/auth/reset/send-otp
{"resetId":"<resetId>","destinationType":"PrimaryMobile"}
```

```http
POST /api/auth/reset/verify-otp
{"resetId":"<resetId>","otp":"000000"}
```

```http
POST /api/auth/reset/new-password
{"resetToken":"<one-time-token>","newPassword":"New-Strong-Password1"}
```

Agent registration collects the fields shown on the registration screen.
AgentCode is generated by SQL as `AG` followed by an eight-digit sequence.
IRDA license number is left null because the screen does not collect it.
The password entered in the form is stored as an Argon2id hash; registration
does not generate or send a temporary password. The new account remains
inactive until both its mobile and email OTPs are verified.

```http
POST /api/auth/register
Content-Type: application/json

{
  "fullName": "Aarav Sharma",
  "mobileNumber": "+919876540021",
  "email": "aarav.sharma@example.com",
  "alternateMobile": "",
  "alternateEmail": "",
  "password": "StrongPassword1!"
}
```

Returns HTTP 202 with `registrationId`, generated `agentCode`, and an expiry.
The API sends OTPs to both provided primary contact details using the existing
OTP service with purpose `AgentRegistration`.

If delivery fails, the error response includes the registration ID and
destination type so the client can retry:

```http
POST /api/auth/register/send-otp
Content-Type: application/json

{
  "registrationId": "<registrationId>",
  "destinationType": "PrimaryEmail"
}
```

After entering both codes:

```http
POST /api/auth/register/verify
Content-Type: application/json

{
  "registrationId": "<registrationId>",
  "mobileOtp": "123456",
  "emailOtp": "654321"
}
```

The account becomes active only after both OTPs pass verification. OTP values
are never stored. A pending registration expires after 24 hours by default;
an expired registration can be started again using the same email address.
Registration OTP attempt count resets when a new OTP is sent after the
cooldown.

Authenticated operations:

```http
POST /api/auth/logout
Authorization: Bearer <accessToken>

POST /api/auth/logout-all
Authorization: Bearer <accessToken>

GET /api/users/me
Authorization: Bearer <accessToken>
```

## Tests

```powershell
dotnet test .\LoginScree\InsuranceAuth\tests\InsuranceAuth.Tests\InsuranceAuth.Tests.csproj
dotnet build .\LoginScree\InsuranceAuth\src\InsuranceAuth.Api\InsuranceAuth.Api.csproj
```

The focused tests cover registration creation and two-contact verification,
password screen-policy enforcement, login success, generic wrong-password handling,
lockout on the fourth failure, reset request throttling, the OTP attempt cap,
password history, and reset-step ordering. SQL Server stored-procedure
integration requires a configured database and the scripts above.
