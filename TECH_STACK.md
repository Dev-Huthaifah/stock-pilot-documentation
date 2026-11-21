# Technology Stack

## Confirmed
| Area | Technology |
|---|---|
| Backend | ASP.NET Core Web API |
| Language | C# |
| Target | .NET 10 (`net10.0`) |
| ORM | Entity Framework Core 10 |
| Database | SQL Server |
| Local DB | SQL Server Express LocalDB |
| API description | ASP.NET Core OpenAPI |
| Identity | ASP.NET Core Identity (`IdentityCore`, EF stores) |
| Authentication | JWT Bearer (`Microsoft.AspNetCore.Authentication.JwtBearer`) |
| Authorization | ASP.NET Core authorization policies; custom permission-evaluation service |
| Source control | Git / GitHub |

Known local environment: Windows 11; solution `StockPilot.slnx`; API project `src/StockPilot.Api/StockPilot.Api.csproj`; local path previously `C:\Projects\stock-pilot`; LocalDB instance `MSSQLLocalDB`; database `StockPilotDb`; API previously listened on `http://localhost:5130`; SDK reported as `10.0.403`.

Reported package versions (verify project files before changing): Microsoft.AspNetCore.OpenApi 10.0.13; Microsoft.EntityFrameworkCore.Design 10.0.8; Microsoft.EntityFrameworkCore.SqlServer 10.0.8; dotnet-ef 10.0.8.

## Authentication configuration
- JWT configuration keys: `Jwt:Secret` (Base64-encoded key, at least 32 bytes), `Jwt:Issuer`, `Jwt:Audience`, and `Jwt:ExpirationMinutes`.
- Platform Admin bootstrap keys: `PlatformAdmin:Email`, `PlatformAdmin:Password`, and `PlatformAdmin:FullName`.
- Development secrets are stored with .NET User Secrets, not committed to `appsettings.json` or Git.
- Identity password policy: minimum 10 characters, uppercase, lowercase, digit, and non-alphanumeric character. Lockout is enabled after 5 failed attempts for 15 minutes. Email confirmation is not currently required.

## Undecided
Frontend framework, UI library, authentication provider, AI provider/model, production database hosting, cloud provider, CI/CD, logging, and monitoring. Do not invent selections.
