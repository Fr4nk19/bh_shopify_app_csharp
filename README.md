# BH Shopify App — C# ASP.NET Core

App de Shopify en C# (.NET 8) para sincronización de inventario bidireccional con ERP,
construida con el patrón de **Inyección de Dependencias** de ASP.NET Core.

## Stack

| Capa | Tecnología |
|------|-----------|
| Web framework | ASP.NET Core 8 (Razor Pages + MVC Controllers) |
| ORM | Entity Framework Core 8 con SQL Server |
| HTTP client | `IHttpClientFactory` + Resilience (Polly) |
| Cron jobs | Quartz.NET 3 |
| Shopify Auth | ShopifySharp |
| Logging | Serilog |

## Patrón de Diseño: Inyección de Dependencias

Cada capa depende de **interfaces**, nunca de implementaciones concretas:

```
IErpService          ← ErpService           (HTTP a ERP C#)
IShopifyInventoryService ← ShopifyInventoryService (GraphQL a Shopify)
ISyncService         ← SyncService          (orquestador)
```

El registro se organiza en **extension methods** por responsabilidad:

```csharp
// Program.cs — composition root limpio
builder.Services
    .AddDatabase(config)       // EF Core + SQL Server
    .AddErpServices(config)    // IErpService + typed HttpClient
    .AddShopifyServices()      // IShopifyInventoryService + typed HttpClient
    .AddSyncServices()         // ISyncService
    .AddCronJobs();            // Quartz (InventorySyncJob, RetryQueueJob)
```

## Estructura del Proyecto

```
csharp/
└── src/BhShopifyApp/
    ├── Program.cs                          # Composition root (DI + pipeline)
    ├── Extensions/
    │   └── ServiceCollectionExtensions.cs  # Grupos de registro DI
    ├── Controllers/
    │   ├── AuthController.cs               # OAuth Shopify
    │   ├── WebhookController.cs            # Webhooks Shopify (HMAC)
    │   └── ErpWebhookController.cs         # Push ERP → App
    ├── Pages/App/
    │   ├── Index     (Dashboard)
    │   ├── Settings  (Config ERP)
    │   ├── Products  (Mapeo SKU)
    │   └── SyncLog   (Historial)
    ├── Services/
    │   ├── Erp/
    │   │   ├── IErpService.cs
    │   │   └── ErpService.cs
    │   ├── Shopify/
    │   │   ├── IShopifyInventoryService.cs
    │   │   └── ShopifyInventoryService.cs
    │   ├── Sync/
    │   │   ├── ISyncService.cs
    │   │   └── SyncService.cs
    │   └── Cron/
    │       ├── InventorySyncJob.cs
    │       └── RetryQueueJob.cs
    ├── Models/                             # Entidades EF Core
    ├── DTOs/                               # Records de transferencia
    └── Data/
        ├── AppDbContext.cs
        └── Migrations/
```

## Requisitos

- .NET 8 SDK
- SQL Server (Express, Developer Edition o Docker)
- Cuenta Shopify Partners

## Setup Local para Pruebas

### Opción A — SQL Server con Docker (recomendado, multiplataforma)

```bash
# 1. Levantar SQL Server en Docker
docker run -e "ACCEPT_EULA=Y" \
           -e "SA_PASSWORD=YourStrong@Passw0rd" \
           -p 1433:1433 \
           --name sqlserver \
           -d mcr.microsoft.com/mssql/server:2022-latest

# 2. Verificar que está corriendo
docker ps
```

Cadena de conexión para Docker (usuario `sa`):
```
Server=localhost,1433;Database=BhShopifyApp;User Id=sa;Password=YourStrong@Passw0rd;TrustServerCertificate=true;
```

### Opción B — SQL Server Express / Developer Edition (Windows)

Descarga gratuita: https://www.microsoft.com/en-us/sql-server/sql-server-downloads

Con Windows Authentication (sin usuario/contraseña):
```
Server=localhost;Database=BhShopifyApp;Trusted_Connection=true;TrustServerCertificate=true;
```

Con SQL Authentication:
```
Server=localhost;Database=BhShopifyApp;User Id=sa;Password=TuPassword;TrustServerCertificate=true;
```

---

### Instalación y ejecución

```bash
# 1. Restaurar dependencias
cd csharp
dotnet restore

# 2. Configurar User Secrets (desarrollo local)
cd src/BhShopifyApp
dotnet user-secrets set "Shopify:ApiKey"     "your_key"
dotnet user-secrets set "Shopify:ApiSecret"  "your_secret"
dotnet user-secrets set "Sync:WebhookSecret" "your_secret"

# Si usas Docker/SQL Auth, sobreescribe la cadena de conexión:
dotnet user-secrets set "ConnectionStrings:DefaultConnection" \
  "Server=localhost,1433;Database=BhShopifyApp;User Id=sa;Password=YourStrong@Passw0rd;TrustServerCertificate=true;"

# 3. Crear la base de datos y ejecutar migraciones
dotnet ef database update

# 4. Iniciar la app
dotnet run
# → https://localhost:5001
```

> **Nota:** Si no tienes `dotnet-ef` instalado:
> ```bash
> dotnet tool install --global dotnet-ef
> ```

---

## Endpoints del ERP Esperados

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/api/inventory` | Listar todo el inventario |
| GET | `/api/inventory/{sku}` | Stock de un SKU |
| PUT | `/api/inventory/{sku}` | Actualizar stock |
| GET | `/api/health` | Health check (opcional) |

## Endpoint Push ERP → App

```
POST /api/erp-webhook
Headers:
  X-Webhook-Secret: {Sync:WebhookSecret}
  X-Shop-Domain:    mi-tienda.myshopify.com
Body:
  { "sku": "PROD-001", "quantity": 50 }
  // batch:
  { "items": [{ "sku": "PROD-001", "quantity": 50 }] }
```

## Flujo de Autenticación

```
Browser → GET /auth/install?shop=xxx.myshopify.com
        → Redirect a Shopify OAuth
        → Shopify → GET /auth/callback?code=...
        → App intercambia code por access_token
        → Guarda en ShopSessions
        → Redirige a /app embebido en Shopify Admin
```

## Producción

```bash
dotnet publish -c Release -o ./publish
# Configurar variables de entorno en el servidor
# Ejecutar: dotnet BhShopifyApp.dll
```
