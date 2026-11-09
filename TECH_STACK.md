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
| Source control | Git / GitHub |

Known local environment: Windows 11; solution `StockPilot.slnx`; API project `src/StockPilot.Api/StockPilot.Api.csproj`; local path previously `C:\Projects\stock-pilot`; LocalDB instance `MSSQLLocalDB`; database `StockPilotDb`; API previously listened on `http://localhost:5130`; SDK reported as `10.0.403`.

Reported package versions (verify project files before changing): Microsoft.AspNetCore.OpenApi 10.0.13; Microsoft.EntityFrameworkCore.Design 10.0.8; Microsoft.EntityFrameworkCore.SqlServer 10.0.8; dotnet-ef 10.0.8.

## Undecided
Frontend framework, UI library, authentication provider, AI provider/model, production database hosting, cloud provider, CI/CD, logging, and monitoring. Do not invent selections.
