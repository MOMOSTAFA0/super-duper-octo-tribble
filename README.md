<img width="720" height="480" alt="image" src="https://github.com/user-attachments/assets/352ca134-3c8c-435d-ab30-df64f080db83" />

Introduction
The Invisible Orchestrator
Every .NET application, whether a simple console app or a complex microservices architecture, relies on a fundamental question: “How long should this object live?”

Dependency Injection (DI) in .NET isn’t just about avoiding the new keyword. It's about lifecycle management. When you register a service, you're not just telling the container "here's a class." You're telling it when to create it, when to destroy it, and who should share it.

Understanding these lifetimes — Transient, Scoped, and Singleton — is the difference between an application that scales gracefully and one that mysteriously crashes under load or leaks memory across requests.

But there’s a fourth, invisible concept that every developer must understand: The Captive Dependency Trap. This silent bug has brought down production systems, yet it’s entirely preventable once you understand the mechanics.

Let’s explore how .NET 10 handles each of these lifetimes under the hood, with detailed code examples, legacy comparisons, and visual diagrams.

👉 Read Full Story here

Part 1: Transient — The Ephemeral Worker
Concept: “Every injection = new instance”

Transient services are created every single time they’re requested. If a Transient service is injected into three different classes within the same request, you get three completely separate instances. If a single class requests the same Transient service multiple times through property injection or multiple constructors, you also get separate instances.

This is the simplest lifetime to understand but the most dangerous regarding performance if misused. Each new instance means:

Memory allocation
Constructor execution
Potential dependency resolution chains
Design Pattern Connection: Transient services embody the Prototype Pattern. Just as the Prototype pattern creates new objects by cloning, the DI container acts as a factory that constantly produces fresh copies. This ensures perfect isolation — no state is ever shared between consumers.

SOLID Connection: Transient services strongly support the Single Responsibility Principle and Interface Segregation Principle. Because each consumer gets their own instance, services can focus on specific tasks without worrying about thread safety or shared state contamination. This makes them ideal for:

Stateless calculation services
Formatting and transformation logic
Services that must not share state
High-throughput scenarios where isolation is critical
Legacy Implementation (.NET Core 3.1 — .NET 6)
In older .NET versions, Transient services were created using reflection-based activation. Every resolution required expensive reflection chain:

// Legacy Program.cs (Pre-.NET 8)
using Microsoft.Extensions.DependencyInjection;
using System.Diagnostics;

var services = new ServiceCollection();
// Legacy Transient Registration
services.AddTransient<IOperationLogger, FileOperationLogger>();
services.AddTransient<IDataProcessor, ComplexDataProcessor>();
services.AddTransient<INotificationService, EmailNotificationService>();
services.AddTransient<IAuditTrail, DatabaseAuditTrail>();
// Building the provider
var serviceProvider = services.BuildServiceProvider();
// Legacy service implementation
public class FileOperationLogger : IOperationLogger
{
    private readonly string _logPath;
    private readonly Guid _instanceId = Guid.NewGuid();
    
    public FileOperationLogger()
    {
        // Simulating expensive setup
        _logPath = Path.Combine(Path.GetTempPath(), $"logs_{DateTime.Now:yyyyMMdd}.txt");
        File.AppendAllText(_logPath, $"[{DateTime.UtcNow}] Logger {_instanceId} created{Environment.NewLine}");
    }
    
    public void Log(string message)
    {
        File.AppendAllText(_logPath, $"[{DateTime.UtcNow}] {_instanceId}: {message}{Environment.NewLine}");
    }
}

public class LegacyOrderService
{
    private readonly IOperationLogger _logger;
    private readonly IDataProcessor _processor;
    private readonly IAuditTrail _audit;
    
    public LegacyOrderService(
        IOperationLogger logger,  // New instance each time
        IDataProcessor processor, // Different new instance
        IAuditTrail audit)        // Another new instance
    {
        _logger = logger;
        _processor = processor;
        _audit = audit;
        
        _logger.Log($"OrderService constructed with {_processor.GetType().Name}");
    }
    
    public async Task ProcessOrder(Order order)
    {
        var stopwatch = Stopwatch.StartNew();
        
        _logger.Log($"Processing order {order.Id}");
        var result = await _processor.Process(order);
        await _audit.Record("OrderProcessed", order.Id);
        
        stopwatch.Stop();
        _logger.Log($"Order {order.Id} processed in {stopwatch.ElapsedMilliseconds}ms");
        
        // PROBLEM: In legacy, this method would create yet another instance
        // if we resolved from the provider directly
        // LINE 58-59: Using root service provider directly without scope
        // This can lead to memory leaks if service implements IDisposable
        using var scope = serviceProvider.CreateScope();
        var anotherLogger = scope.ServiceProvider.GetService<IOperationLogger>();
        anotherLogger.Log("Step completed - but this is a NEW instance!");
    }
}

❌ What’s Wrong in Legacy Code:

<img width="828" height="391" alt="image" src="https://github.com/user-attachments/assets/30ef212f-876a-42ff-a782-b2ee5ef9420e" />

.NET 10 Implementation
.NET 10 completely transforms Transient resolution through Compiled Expression Trees and Dynamic Method Generation. The syntax remains clean, but the engine underneath is radically different:

// .NET 10 Program.cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.DependencyInjection.Extensions;
using System.Runtime.CompilerServices;

var builder = WebApplication.CreateBuilder(args);
// Modern Transient Registration - Syntax unchanged, but behavior optimized
builder.Services.AddTransient<IOperationLogger, FileOperationLogger>();
builder.Services.AddTransient<IDataProcessor, ComplexDataProcessor>();
builder.Services.AddTransient<INotificationService, EmailNotificationService>();
builder.Services.AddTransient<IAuditTrail, DatabaseAuditTrail>();
// New in .NET 10: Keyed Transient Services
builder.Services.AddKeyedTransient<IPaymentProcessor, CreditCardProcessor>("credit");
builder.Services.AddKeyedTransient<IPaymentProcessor, PayPalProcessor>("paypal");
builder.Services.AddKeyedTransient<IPaymentProcessor, CryptoProcessor>("crypto");
// .NET 10: Factory-based Transients with compiled caching
builder.Services.AddTransient<ISpecializedService>(sp =>
{
    // This factory is compiled once and cached as an expression tree
    var logger = sp.GetRequiredService<IOperationLogger>();
    var config = sp.GetRequiredService<IConfiguration>();
    var connectionString = config.GetConnectionString("Analytics");
    
    return new SpecializedService(logger, connectionString, DateTime.UtcNow);
});
var app = builder.Build();


// Modern service implementation
public class FileOperationLogger : IOperationLogger
{
    private readonly string _logPath;
    private readonly Guid _instanceId = Guid.NewGuid();
    private readonly IMetricsCollector _metrics; // Injected
    
    public FileOperationLogger(IMetricsCollector metrics) // Constructor injection works
    {
        _metrics = metrics;
        _logPath = Path.Combine(Path.GetTempPath(), $"logs_{DateTime.Now:yyyyMMdd}.txt");
        
        // FIXED: Using async file I/O, not blocking constructor
        // LINE 90-91: Async file write doesn't block constructor
        _ = File.AppendAllTextAsync(_logPath, $"[{DateTime.UtcNow}] Logger {_instanceId} created{Environment.NewLine}");
    }
    
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public void Log(string message)
    {
        _metrics.IncrementCounter("logs.written");
        // FIXED: Async file write with fire-and-forget (properly handled)
        File.AppendAllTextAsync(_logPath, $"[{DateTime.UtcNow}] {_instanceId}: {message}{Environment.NewLine}");
    }
}
app.MapPost("/process-order", async (
    Order order,
    IOperationLogger logger,        // .NET 10: Ultra-fast resolution via compiled delegates
    IDataProcessor processor,       // Constructor cached at build time
    IAuditTrail audit,              // All Transient, all separate instances
    [FromKeyedServices("credit")] IPaymentProcessor payment,
    HttpContext context) =>
{
    // FIXED: Using Activity for distributed tracing instead of manual stopwatch
    // LINE 109-110: Built-in diagnostics with automatic correlation
    using var activity = ActivitySource.StartActivity("ProcessOrder");
    activity?.SetTag("order.id", order.Id);
    
    logger.Log($"Processing order {order.Id}");
    
    // All three services are separate instances
    var result = await processor.Process(order);
    await payment.ProcessPayment(order.Total);
    await audit.Record("OrderProcessed", order.Id, result);
    
    activity?.SetTag("processing.completed", true);
    logger.Log($"Order {order.Id} processed");
    
    // FIXED: Using HttpContext.RequestServices instead of root provider
    // LINE 122-123: Context-aware resolution, properly scoped
    var anotherLogger = context.RequestServices.GetRequiredService<IOperationLogger>();
    // This is a DIFFERENT instance from 'logger' above
    anotherLogger.Log("Manual resolution completed");
    
    return Results.Ok(new 
    { 
        OrderId = order.Id, 
        LoggerInstance = logger.GetHashCode(),
        ManualLoggerInstance = anotherLogger.GetHashCode() // Different!
    });
});


// .NET 10 Performance Test
public class Net10PerformanceTest
{
    public static async Task TestTransientPerformance()
    {
        var builder = WebApplication.CreateBuilder();
        builder.Services.AddTransient<HeavyService>();
        
        var app = builder.Build();
        var provider = app.Services;
        
        var stopwatch = Stopwatch.StartNew();
        
        // .NET 10: First resolution compiles the factory
        var first = provider.GetService<HeavyService>();
        var firstTime = stopwatch.ElapsedMilliseconds;
        
        stopwatch.Restart();
        for (int i = 0; i < 10000; i++)
        {
            var service = provider.GetService<HeavyService>();
        }
        
        stopwatch.Stop();
        Console.WriteLine($"First resolution: {firstTime}ms");
        Console.WriteLine($"10,000 resolutions: {stopwatch.ElapsedMilliseconds}ms");
        // .NET 10 output: First: ~15-20ms, 10,000: ~80-120ms
    }
}


// .NET 10: HeavyService with improved instantiation
public class HeavyService
{
    private readonly byte[] _buffer = GC.AllocateUninitializedArray<byte>(1024); // Pooled allocation
    private readonly Guid _id = Guid.NewGuid();
    private readonly ILogger<HeavyService>? _logger; // Optional dependency
    
    public HeavyService(ILogger<HeavyService>? logger = null) // Optional parameters work
    {
        _logger = logger;
        // .NET 10: Constructor is compiled, not reflected
    }
}

✅ What’s Fixed in .NET 10:

<img width="828" height="567" alt="image" src="https://github.com/user-attachments/assets/d77e14be-93b2-4d51-a4a8-ff662c1f0905" />

What Changed in .NET and Why It Matters
<img width="828" height="376" alt="image" src="https://github.com/user-attachments/assets/7374b601-f95d-4f93-a949-1f290764a92a" />

Transient Lifetime Flow
<img width="828" height="1457" alt="image" src="https://github.com/user-attachments/assets/4dbcb494-2d1e-48c0-8d4f-b5a0002c858e" />

Part 2: Scoped — The Request Context
Concept: “One per request. Shared within it.”

Scoped services are created once per scope — in web applications, this typically aligns with an HTTP request. Every class that participates in handling that specific request receives the exact same instance of the Scoped service.

This is the workhorse of web applications and the default lifetime for Entity Framework Core’s DbContext. Understanding scopes is crucial for:

Transaction management
Request-scoped caching
Unit of Work patterns
Tenant isolation in multi-tenant apps
Design Pattern Connection: Scoped services implement the Unit of Work pattern. By sharing the same instance across multiple operations, you can track changes and commit them as a single transaction. This is why DbContext is Scoped by default—it tracks entity changes throughout the request and saves them together with SaveChangesAsync().

The Repository Pattern also relies heavily on scoped services to ensure that multiple repositories working with the same DbContext share the same entity tracking cache.

SOLID Connection: Scoped services support the Open/Closed Principle by allowing cross-cutting concerns (like logging, auditing, or transaction management) to be injected and shared without modifying the classes themselves. They enable Dependency Inversion by ensuring consistent state across abstractions.

Real-World Example: In an e-commerce checkout flow:

OrderService (Scoped) needs the same DbContext as InventoryService (Scoped)
PaymentProcessor (Scoped) needs to share the transaction with OrderService
If any service fails, the entire unit of work can roll back
Legacy Implementation (.NET Core 3.1 — .NET 6)
In earlier .NET versions, scopes were managed through IServiceScope and IServiceScopeFactory, but scope disposal wasn't always predictable, especially in error scenarios or background tasks.

// Legacy Implementation (.NET 5 style)
public class LegacyRequestPipeline
{
    private readonly IServiceProvider _rootProvider;
    private readonly ILogger<LegacyRequestPipeline> _logger;
    
    public LegacyRequestPipeline(IServiceProvider rootProvider, ILogger<LegacyRequestPipeline> logger)
    {
        _rootProvider = rootProvider;
        _logger = logger;
    }
    
    public async Task ProcessRequest(HttpContext context)
    {
        // Legacy: Manual scope creation required
        // LINE 192: Manual scope creation - easy to forget
        using var scope = _rootProvider.CreateScope();
        var scopedProvider = scope.ServiceProvider;
        
        // These both get the SAME instance within this scope
        var repository = scopedProvider.GetService<IProductRepository>();
        var cache = scopedProvider.GetService<IRequestCache>();
        var dbContext = scopedProvider.GetService<AppDbContext>();
        
        try
        {
            _logger.LogInformation("Processing request with scope {ScopeId}", scope.GetHashCode());
            
            // All using same DbContext instance
            var products = await repository.GetProductsAsync();
            cache.Set("last-access", DateTime.UtcNow);
            
            // Update something
            var product = products.First();
            product.LastViewed = DateTime.UtcNow;
            
            // Save all changes together
            await dbContext.SaveChangesAsync();
            
            // PROBLEM: If something threw here, disposal might not happen correctly
            // LINE 214: Exception that could skip disposal
            throw new InvalidOperationException("Database timeout simulated");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing request");
            
            // PROBLEM: Had to manually ensure scope disposal
            // LINE 220-221: Easy to forget and cause connection leaks
            // If we don't dispose, DbContext connection stays open!
            // No automatic cleanup on exception path
        }
        // The 'using' statement helps, but if we didn't have it...
    }
}

// Legacy Registration
services.AddScoped<IProductRepository, EfProductRepository>();
services.AddScoped<IRequestCache, MemoryRequestCache>();
services.AddScoped<AppDbContext>(); // EF Core context - lives here
services.AddScoped<ICurrentUserService, CurrentUserService>();
// Legacy service that needs request context
public class CurrentUserService : ICurrentUserService
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    private string? _cachedUserId;
    
    public CurrentUserService(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }
    
    public string GetUserId()
    {
        // Legacy: This might be called multiple times, but it's cached
        if (_cachedUserId == null)
        {
            // PROBLEM: But if scope is disposed, HttpContext might be gone!
            // LINE 248-251: No null check for HttpContext
            _cachedUserId = _httpContextAccessor.HttpContext?.User?
                .FindFirst(ClaimTypes.NameIdentifier)?.Value ?? "anonymous";
        }
        return _cachedUserId;
    }
}

❌ What’s Wrong in Legacy Code:

<img width="828" height="390" alt="image" src="https://github.com/user-attachments/assets/fc748f5a-4f6e-49d9-84e6-dea1dfdf942b" />

.NET 10 Implementation
.NET 10 introduces AsyncLocal-aware scope management, automatic scope disposal even in complex failure scenarios, and seamless integration with the HTTP pipeline.

// .NET 10 Modern Implementation
using Microsoft.Extensions.DependencyInjection;
using Microsoft.AspNetCore.Http;
using System.Diagnostics;
using System.Runtime.CompilerServices;

var builder = WebApplication.CreateBuilder(args);
// Modern Scoped Registrations
builder.Services.AddScoped<IProductRepository, EfProductRepository>();
builder.Services.AddScoped<IRequestCache, DistributedRequestCache>();
builder.Services.AddScoped<AppDbContext>();
builder.Services.AddScoped<ICurrentUserService, CurrentUserService>();
builder.Services.AddScoped<ITenantService, TenantService>();
// .NET 10: Scoped service with compiled factory
builder.Services.AddScoped<ITenantService>(sp =>
{
    // This factory runs once per request in .NET 10
    // It's compiled and cached for performance
    var httpContext = sp.GetService<IHttpContextAccessor>()?.HttpContext;
    var tenantId = httpContext?.Request.Headers["X-Tenant-ID"].FirstOrDefault() 
                   ?? httpContext?.Request.Host.Host.Split('.')[0] 
                   ?? "default";
    
    var connectionString = sp.GetRequiredService<IConfiguration>()
        .GetConnectionString($"Tenant_{tenantId}");
    
    return new TenantService(tenantId, connectionString);
});

// .NET 10: Enhanced scope validation
builder.Services.ValidateScopes = true; // Validates at build time
builder.Services.ValidateOnBuild = true; // Check during app startup
var app = builder.Build();
// FIXED: .NET 10 automatically creates scope per request - no manual work needed!
app.MapGet("/products", async (
    IProductRepository repo,        // Scoped - same instance for this request
    IRequestCache cache,            // Scoped - same instance
    AppDbContext dbContext,         // Scoped - same EF Core instance
    ICurrentUserService currentUser, // Scoped - same user context
    HttpContext httpContext) =>
{
    // FIXED: All injected services above are the SAME INSTANCE throughout this request
    // LINE 309-310: Using HttpContext.RequestServices instead of root provider
    var scopedServices = httpContext.RequestServices;
    var sameRepo = scopedServices.GetRequiredService<IProductRepository>();
    
    // This is the same instance as 'repo' above
    Debug.Assert(ReferenceEquals(repo, sameRepo));
    
    // FIXED: Built-in activity tracing
    using var activity = ActivitySource.StartActivity("GetProducts");
    activity?.SetTag("user.id", currentUser.GetUserId());
    
    // Check cache first
    // FIXED: Using TryGet pattern instead of null check
    if (cache.TryGet<List<Product>>("products", out var cached))
    {
        activity?.SetTag("cache.hit", true);
        return cached;
    }
    
    activity?.SetTag("cache.hit", false);
    
    // Query database via repository
    var products = await repo.GetAllAsync();
    
    // Cache using the same cache instance
    await cache.SetAsync("products", products, TimeSpan.FromMinutes(5));
    
    return products;
});

// .NET 10: Modern background service with safe scoping
builder.Services.AddHostedService<LegacyCleanupService>();
public class LegacyCleanupService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<LegacyCleanupService> _logger;
    
    public LegacyCleanupService(
        IServiceScopeFactory scopeFactory,
        ILogger<LegacyCleanupService> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Cleanup service started");
        
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                // FIXED: Creating scopes is more efficient in .NET 10
                // LINE 355-356: Scopes are pooled and reused when possible
                using var scope = _scopeFactory.CreateScope();
                var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();
                var metrics = scope.ServiceProvider.GetRequiredService<IMetricsCollector>();
                
                _logger.LogDebug("Starting cleanup cycle with scope {ScopeId}", scope.GetHashCode());
                
                var stopwatch = Stopwatch.StartNew();
                
                // Clean up old records
                var deletedCount = await dbContext.Database.ExecuteSqlRawAsync(
                    "DELETE FROM AuditLogs WHERE CreatedAt < DATEADD(day, -30, GETUTCDATE())",
                    stoppingToken);
                
                stopwatch.Stop();
                
                metrics.RecordCleanup(deletedCount, stopwatch.ElapsedMilliseconds);
                
                _logger.LogInformation("Cleaned up {Count} records in {Elapsed}ms", 
                    deletedCount, stopwatch.ElapsedMilliseconds);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error during cleanup cycle");
                // FIXED: Scope is still disposed properly due to 'using'
                // LINE 381: No need for manual disposal - 'using' guarantees it
            }
            
            // Wait before next cycle
            await Task.Delay(TimeSpan.FromHours(24), stoppingToken);
        }
    }
}

// .NET 10: CurrentUserService with better async context
public class CurrentUserService : ICurrentUserService
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    private readonly AsyncLocal<string?> _userId = new();
    
    public CurrentUserService(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }
    
    public string GetUserId()
    {
        // FIXED: AsyncLocal maintains context across async boundaries
        // LINE 404-409: Safe null handling with AsyncLocal
        if (_userId.Value == null)
        {
            _userId.Value = _httpContextAccessor.HttpContext?.User?
                .FindFirst(ClaimTypes.NameIdentifier)?.Value 
                ?? _httpContextAccessor.HttpContext?.Session?.GetString("UserId")
                ?? "anonymous";
        }
        
        return _userId.Value;
    }
}
✅ What’s Fixed in .NET 10:

<img width="828" height="415" alt="image" src="https://github.com/user-attachments/assets/9f75f551-2b18-4f9a-ab7a-3d3d4e30d885" />

What Changed in .NET and Why It Matters
<img width="828" height="279" alt="image" src="https://github.com/user-attachments/assets/b5fb51af-73cc-404a-90b8-ffcdd7875e61" />

Scoped Lifetime Flow
<img width="828" height="437" alt="image" src="https://github.com/user-attachments/assets/dc946b3a-a28e-4ac2-9a42-94dca3216ae5" />

Part 3: Singleton — The Application Monarch
Concept: “One for the entire app lifetime. Created once, never disposed until shutdown.”

Singleton services are instantiated the first time they’re requested and live for the duration of the application. Every subsequent request, in any scope or thread, receives that same original instance.

This is the most powerful but also the most dangerous lifetime when misused. Properly used Singletons provide:

Caching: Share computed data across all requests
Configuration: Global settings that don’t change
Logging sinks: Write to same file from anywhere
Metrics aggregation: Count requests across the app
Connection pooling: Reuse expensive connections
Design Pattern Connection: This is the classic Singleton Pattern but with container-managed lifecycle. Unlike the traditional pattern (private constructor, static Instance property with double-check locking), the DI container handles:

Thread safety during creation
Lazy initialization
Disposal ordering
Dependency resolution
The Facade Pattern often uses Singletons to provide a unified interface to complex subsystems (like a global cache facade).

SOLID Connection: Singletons support the Single Responsibility Principle when used for truly global concerns:

Caching service: One responsibility: store and retrieve cached items
Configuration service: One responsibility: provide configuration values
Metrics collector: One responsibility: aggregate metrics
However, they violate Dependency Inversion if they depend on Scoped services (leading to the Captive Dependency Trap we’ll explore next).

Legacy Implementation (.NET Framework — .NET Core 3.1)
Before DI containers, Singletons required careful thread-safe implementation. Even with early DI containers, there were issues with disposal order, memory leaks from event handlers, and deadlock scenarios.

// Legacy Pre-DI Singleton (The Old Way - .NET Framework style)
public class LegacyConfigurationManager
{
    private static LegacyConfigurationManager _instance;
    private static readonly object _padlock = new object();
    private static bool _initialized = false;
    
    private readonly Dictionary<string, string> _settings;
    private readonly object _settingsLock = new object();
    
    private LegacyConfigurationManager()
    {
        // Load configuration once - expensive operation
        _settings = LoadFromFile();
        _initialized = true;
    }
    
    public static LegacyConfigurationManager Instance
    {
        get
        {
            // PROBLEM: Double-check locking pattern is error-prone
            // LINE 483-496: Complex threading code that's easy to get wrong
            if (_instance == null)
            {
                lock (_padlock)
                {
                    if (_instance == null)
                    {
                        _instance = new LegacyConfigurationManager();
                    }
                }
            }
            return _instance;
        }
    }
    
    private Dictionary<string, string> LoadFromFile()
    {
        // PROBLEM: Simulate expensive I/O - blocks constructor
        // LINE 504: Thread.Sleep in constructor is terrible for performance
        Thread.Sleep(2000); // 2 seconds load time
        
        return new Dictionary<string, string>
        {
            ["DatabaseConnection"] = "Server=localhost;Database=MyDb;Trusted_Connection=true;",
            ["ApiKey"] = "legacy-key-123",
            ["Environment"] = "Production",
            ["CacheTimeout"] = "300"
        };
    }
    
    public string GetSetting(string key)
    {
        // PROBLEM: Lock on every read - poor performance
        // LINE 518: Contention on every access
        lock (_settingsLock)
        {
            return _settings.GetValueOrDefault(key, "");
        }
    }
    
    public void UpdateSetting(string key, string value)
    {
        lock (_settingsLock)
        {
            _settings[key] = value;
            SaveToFile(); // Expensive!
        }
    }
    
    private void SaveToFile()
    {
        // Thread-safe file write
        lock (_padlock)
        {
            File.WriteAllText("config.json", JsonSerializer.Serialize(_settings));
        }
    }
}
// Usage throughout legacy app
var connString = LegacyConfigurationManager.Instance.GetSetting("DatabaseConnection");
LegacyConfigurationManager.Instance.UpdateSetting("LastAccess", DateTime.UtcNow.ToString());
// PROBLEM: Testing is impossible - static state persists across tests
// PROBLEM: No dependency inversion - hardcoded dependency

Early DI Container Singleton (.NET Core 2.0):


// Early .NET Core singleton registration
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // Simple registration
        services.AddSingleton<ICacheService, MemoryCacheService>();
        
        // Factory-based
        services.AddSingleton<IConfigurationService>(provider =>
        {
            var config = provider.GetRequiredService<IConfiguration>();
            return new FileConfigurationService(config["ConfigPath"]);
        });
        
        // Instance-based (eager)
        var metrics = new MetricsService();
        services.AddSingleton<IMetricsService>(metrics);
    }
}

public class MemoryCacheService : ICacheService
{
    private readonly ConcurrentDictionary<string, object> _cache = new();
    private readonly ILogger<MemoryCacheService> _logger;
    
    public MemoryCacheService(ILogger<MemoryCacheService> logger) // Constructor injection works
    {
        _logger = logger;
        _logger.LogInformation("Cache service created - this happens once");
    }
    
    public void Set(string key, object value)
    {
        _cache[key] = value;
    }
    
    public object? Get(string key)
    {
        return _cache.TryGetValue(key, out var value) ? value : null;
    }
}
❌ What’s Wrong in Legacy Code:

<img width="828" height="350" alt="image" src="https://github.com/user-attachments/assets/eb0e7a93-334d-4732-aca9-8da2ce33fd52" />

.NET 10 Implementation
.NET 10 Singletons are fully managed, thread-safe, and optimized for minimal overhead. The container handles lazy initialization with lock-free reads, and new features allow for singleton disposal tracking and health checks.

// .NET 10 Modern Singleton Implementation
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Options;
using System.Runtime.CompilerServices;
using System.Threading.Locks;

var builder = WebApplication.CreateBuilder(args);
// Modern Singleton Registrations
builder.Services.AddSingleton<ICacheService, HybridCacheService>();
builder.Services.AddSingleton<IMetricsCollector, OpenTelemetryMetricsCollector>();
builder.Services.AddSingleton<IConfigurationService, DynamicConfigurationService>();

// .NET 10: Singleton with complex initialization
builder.Services.AddSingleton<IConnectionPool>(sp =>
{
    // FIXED: Factory runs only once, but compiled for performance
    // LINE 579-588: Clean factory pattern with proper DI
    var config = sp.GetRequiredService<IConfiguration>();
    var logger = sp.GetRequiredService<ILogger<ConnectionPool>>();
    var metrics = sp.GetRequiredService<IMetricsCollector>();
    
    var connectionString = config.GetConnectionString("DefaultConnection");
    var maxPoolSize = config.GetValue<int>("ConnectionPool:MaxSize", 100);
    
    logger.LogInformation("Creating connection pool with max size {MaxPoolSize}", maxPoolSize);
    metrics.RegisterGauge("connection.pool.size", () => maxPoolSize);
    
    return new ConnectionPool(connectionString, maxPoolSize, logger);
});

// .NET 10: Singleton with disposal tracking
builder.Services.AddSingleton<IDatabaseHealthChecker, DatabaseHealthChecker>();
// .NET 10: Options pattern integration
builder.Services.AddSingleton<IFeatureFlags, FeatureFlags>();
builder.Services.AddOptions<FeatureFlagsOptions>()
    .BindConfiguration("FeatureFlags")
    .ValidateOnStart();
var app = builder.Build();

// Modern Singleton implementation with lock-free patterns
public class HybridCacheService : ICacheService
{
    private readonly ConcurrentDictionary<string, CacheEntry> _cache = new();
    private readonly ILogger<HybridCacheService> _logger;
    private readonly IMetricsCollector _metrics;
    private readonly Timer _cleanupTimer;
    private int _hitCount;
    private int _missCount;
    
    public HybridCacheService(
        ILogger<HybridCacheService> logger,
        IMetricsCollector metrics)
    {
        _logger = logger;
        _metrics = metrics;
        
        logger.LogInformation("Cache service initializing - singleton instance {Id}", 
            RuntimeHelpers.GetHashCode(this));
        
        // Start background cleanup
        _cleanupTimer = new Timer(CleanupExpiredEntries, null, 
            TimeSpan.FromMinutes(1), TimeSpan.FromMinutes(1));
    }
    
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public bool TryGet<T>(string key, out T? value)
    {
        // FIXED: Lock-free reads with ConcurrentDictionary
        // LINE 628-639: No locks, highly scalable
        if (_cache.TryGetValue(key, out var entry) && !entry.IsExpired)
        {
            Interlocked.Increment(ref _hitCount);
            _metrics.IncrementCounter("cache.hit");
            
            value = (T?)entry.Value;
            return true;
        }
        
        Interlocked.Increment(ref _missCount);
        _metrics.IncrementCounter("cache.miss");
        
        value = default;
        return false;
    }
    
    public void Set<T>(string key, T value, TimeSpan ttl)
    {
        var entry = new CacheEntry
        {
            Value = value,
            Expiry = DateTime.UtcNow.Add(ttl)
        };
        
        _cache[key] = entry;
        _metrics.IncrementCounter("cache.set");
        
        _logger.LogDebug("Cached {Key} for {TtlSeconds}s", key, ttl.TotalSeconds);
    }
    
    private void CleanupExpiredEntries(object? state)
    {
        var expiredKeys = _cache
            .Where(kvp => kvp.Value.IsExpired)
            .Select(kvp => kvp.Key)
            .ToList();
        
        foreach (var key in expiredKeys)
        {
            _cache.TryRemove(key, out _);
        }
        
        if (expiredKeys.Any())
        {
            _logger.LogInformation("Cleaned up {Count} expired cache entries", expiredKeys.Count);
        }
        
        // Report metrics
        _metrics.RecordGauge("cache.size", _cache.Count);
        _metrics.RecordGauge("cache.hit.ratio", GetHitRatio());
    }
    
    private double GetHitRatio()
    {
        var hits = Volatile.Read(ref _hitCount);
        var misses = Volatile.Read(ref _missCount);
        var total = hits + misses;
        
        return total == 0 ? 0 : (double)hits / total;
    }
    
    private class CacheEntry
    {
        public object? Value { get; set; }
        public DateTime Expiry { get; set; }
        public bool IsExpired => DateTime.UtcNow > Expiry;
    }
    
    // .NET 10: Proper disposal
    public void Dispose()
    {
        _logger.LogInformation("Shutting down cache service - disposing timer");
        _cleanupTimer?.Dispose();
        
        // Report final metrics
        _metrics.RecordGauge("cache.final.size", _cache.Count);
        _metrics.RecordGauge("cache.final.hit.ratio", GetHitRatio());
    }
}

// .NET 10: Configuration service with dynamic reload
public class DynamicConfigurationService : IConfigurationService
{
    private readonly IConfiguration _configuration;
    private readonly IOptionsMonitor<AppSettings> _options;
    private readonly ILogger<DynamicConfigurationService> _logger;
    private readonly ConcurrentDictionary<string, object> _computedValues = new();
    
    public DynamicConfigurationService(
        IConfiguration configuration,
        IOptionsMonitor<AppSettings> options,
        ILogger<DynamicConfigurationService> logger)
    {
        _configuration = configuration;
        _options = options;
        _logger = logger;
        
        // FIXED: Watch for changes without polling
        // LINE 719-722: Reactive configuration updates
        _options.OnChange(settings =>
        {
            logger.LogInformation("Configuration changed - clearing computed values");
            _computedValues.Clear();
        });
    }
    
    public T GetValue<T>(string key, T defaultValue = default!)
    {
        // Try computed cache first
        if (_computedValues.TryGetValue(key, out var cached))
            return (T)cached;
        
        // Get from configuration
        var value = _configuration.GetValue<T>(key, defaultValue);
        
        // Cache computed value
        _computedValues[key] = value!;
        
        return value;
    }
}

// Using Singletons in endpoints
app.MapGet("/cache-test", async (ICacheService cache) =>
{
    // First request creates the cache singleton
    var cacheKey = "last-access";
    
    if (cache.TryGet<string>(cacheKey, out var lastAccess))
    {
        return $"Previous access: {lastAccess}";
    }
    
    var now = DateTime.UtcNow.ToString("O");
    cache.Set(cacheKey, now, TimeSpan.FromSeconds(30));
    
    return $"First access recorded at {now}";
});
app.MapGet("/metrics", (IMetricsCollector metrics) =>
{
    // Same metrics collector accumulates across all requests
    metrics.IncrementRequestCount();
    return metrics.GetSnapshot();
});
app.MapGet("/config-test", (IConfigurationService config) =>
{
    var environment = config.GetValue<string>("Environment", "Development");
    var timeout = config.GetValue<int>("RequestTimeout", 30);
    
    return new
    {
        Environment = environment,
        RequestTimeout = timeout,
        ServerTime = DateTime.UtcNow
    };
});

// .NET 10: Health check for singletons
public class SingletonHealthCheck : IHealthCheck
{
    private readonly ICacheService _cache;
    private readonly IMetricsCollector _metrics;
    private readonly DateTime _startTime = DateTime.UtcNow;
    
    public SingletonHealthCheck(ICacheService cache, IMetricsCollector metrics)
    {
        _cache = cache;
        _metrics = metrics;
    }
    
    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        var uptime = DateTime.UtcNow - _startTime;
        var cacheHitRatio = _metrics.GetHitRatio();
        
        var data = new Dictionary<string, object>
        {
            ["uptime"] = uptime.ToString(),
            ["cache.hit_ratio"] = cacheHitRatio,
            ["singleton.age_hours"] = uptime.TotalHours
        };
        
        if (cacheHitRatio < 0.5)
        {
            return Task.FromResult(HealthCheckResult.Degraded("Cache hit ratio below 50%", data));
        }
        
        return Task.FromResult(HealthCheckResult.Healthy("Singleton services healthy", data));
    }
}
✅ What’s Fixed in .NET 10:

<img width="828" height="480" alt="image" src="https://github.com/user-attachments/assets/96d552c9-c88a-4623-af33-e7b0541137d2" />
What Changed in .NET and Why It Matters
<img width="828" height="418" alt="image" src="https://github.com/user-attachments/assets/54a8d2ad-55b3-4059-b0b3-0e71d0c6f862" />

Singleton Lifetime Flow
<img width="828" height="213" alt="image" src="https://github.com/user-attachments/assets/184cf2a0-ba45-46af-b0e2-cf462bd859d4" />



Part 4: The Captive Dependency Trap — The Silent Application Killer
Concept: This isn’t a lifetime type — it’s a bug pattern that occurs when a longer-lived service holds a reference to a shorter-lived service. The shorter-lived service becomes “captive” — it lives as long as the longer-lived service, defeating its intended lifetime.

The rule is absolute: Never inject a shorter-lived dependency into a longer-lived service.

Singleton → Scoped = ❌ DEADLY
Singleton → Transient = ❌ PROBLEMATIC (Transient becomes effectively Singleton)
Scoped → Transient = ✅ Safe (Transient is created and disposed within the scope)
Why It’s a Trap: When a Singleton holds a Scoped service, that Scoped service is “captured.” It’s created once with the Singleton and never released. It becomes a de facto Singleton, breaking:

Request Isolation — Data from Request 1 leaks to Request 2
User A sees User B’s data
Tenant isolation violated
Session state corrupted
2. Unit of Work — Transactions never complete

SaveChangesAsync never called
Changes lost or partially applied
Long-running transactions lock database
3. Resource Cleanup — Connections stay open forever

DbContext connections never returned to pool
Connection pool exhausts → app crashes
File handles leaked
4. Memory Leaks — Objects that should be short-lived live forever

Request-scoped caches grow unbounded
Event handlers prevent garbage collection
Memory usage grows until OOM crash
5. Stale Data — Captured services don’t refres

Entity Framework change tracker out of date
Stale cached data served to users
Business rules applied to wrong state
Design Pattern Connection: This violates the Dependency Inversion Principle because the high-level Singleton is forcing a specific lifetime on its dependency. The Singleton should depend on abstractions, not on concrete lifetimes.

It also breaks the Proxy pattern if you’re trying to use lazy loading or interception — the proxy captures the target and can’t release it.

Real-World Post-Mortem Example:

// The bug that took down an e-commerce site on Black Friday
public class ShoppingCartService // REGISTERED AS SINGLETON (WRONG!)
{
    private readonly AppDbContext _dbContext; // Should be Scoped - CAPTURED!
    private readonly List<CartItem> _items = new();
    
    public ShoppingCartService(AppDbContext dbContext)
    {
        _dbContext = dbContext; // Captured forever!
    }
    
    public async Task AddItem(int productId, int quantity)
    {
        // This DbContext was created ONCE and never refreshed
        var product = await _dbContext.Products.FindAsync(productId);
        
        // User A adds item - works
        // User B adds item - uses SAME DbContext, sees User A's tracked entities!
        // Data corruption begins...
    }
}
// Registration
services.AddSingleton<ShoppingCartService>(); // Singleton
services.AddScoped<AppDbContext>(); // Scoped - CONFLICT!
// Result:
// 10:00 AM - First request, DbContext created
// 10:05 AM - 100 users, all sharing the same DbContext
// 10:10 AM - Connection pool exhausted
// 10:15 AM - Site down, database connection limit reached
// 11:00 AM - Rollback to previous version, lost sales
Legacy Problem Demonstration
// LEGACY - THE SILENT BUG (Don't do this!)
public class LegacyUserService
{
    private readonly AppDbContext _dbContext; // This is SCOPED!
    private readonly ILogger<LegacyUserService> _logger;
    private readonly ICacheService _cache; // This is Singleton - OK
    
    // This service is registered as SINGLETON
    public LegacyUserService(
        AppDbContext dbContext,  // SCOPED - PROBLEM!
        ILogger<LegacyUserService> logger,
        ICacheService cache)     // Singleton - OK
    {
        // LINE 871: Captive dependency - Scoped injected into Singleton
        _dbContext = dbContext; // CAPTURED FOREVER
        _logger = logger;
        _cache = cache;
        
        _logger.LogWarning("UserService created - this should ONLY happen once, but DbContext is captured!");
    }
    
    public async Task<User> GetUserAsync(int id)
    {
        // PROBLEM: This DbContext was created ONCE and never refreshed
        // LINE 882-886: Using captured DbContext that's potentially stale/disposed
        _logger.LogInformation("Getting user {Id} with DbContext {ContextId}", 
            id, _dbContext.ContextId);
        
        // If this throws "Cannot access a disposed object", you have the trap!
        return await _dbContext.Users.FindAsync(id);
    }
    
    public async Task UpdateUserAsync(User user)
    {
        // PROBLEM: This DbContext is tracking entities from multiple requests!
        // LINE 894-896: Changes from User A might be saved when User B calls SaveChanges
        _dbContext.Users.Update(user);
        await _dbContext.SaveChangesAsync(); // Saves changes from ALL users!
    }
}
// Registration that causes the trap
services.AddSingleton<LegacyUserService>(); // Singleton
services.AddScoped<AppDbContext>(); // Scoped - CONFLICT!
// In a controller - the bug manifests subtly
public class UserController : ControllerBase
{
    private readonly LegacyUserService _userService; // Gets the Singleton
    
    public UserController(LegacyUserService userService)
    {
        _userService = userService;
    }
    
    [HttpGet("user/{id}")]
    public async Task<IActionResult> GetUser(int id)
    {
        // Request 1: Works fine
        // Request 2: Uses SAME DbContext - maybe disposed, maybe wrong data
        // Request 3: Connection pool exhausted
        // Request 4: "DbContext disposed" exception
        // Request 5: App crash
        
        var user = await _userService.GetUserAsync(id);
        return Ok(user);
    }
    
    [HttpPut("user/{id}")]
    public async Task<IActionResult> UpdateUser(int id, UserUpdateModel model)
    {
        // User A updates their profile
        // User B updates their profile using SAME DbContext
        // User A's changes are saved when User B saves!
        // Data corruption!
        
        var user = await _userService.GetUserAsync(id);
        user.Name = model.Name;
        await _userService.UpdateUserAsync(user);
        
        return Ok();
    }
}
❌ What’s Wrong in Legacy Code:

<img width="828" height="371" alt="image" src="https://github.com/user-attachments/assets/cbfbe434-fe9c-488d-af4e-76a59ca2da23" />

Symptoms of the Trap:

<img width="828" height="394" alt="image" src="https://github.com/user-attachments/assets/5d03b767-4848-4be3-bb1f-f88b4a69e7c9" />

.NET 10 Solution
.NET 10 introduces compile-time validation, runtime analysis, and built-in analyzers to catch captive dependencies before they cause production issues.

// .NET 10 - SAFE IMPLEMENTATION
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.DependencyInjection.Diagnostics;
using System.Diagnostics.CodeAnalysis;

var builder = WebApplication.CreateBuilder(args);
// .NET 10: Enable aggressive validation
builder.Services.ValidateScopes = true;              // Validates at build time
builder.Services.ValidateOnBuild = true;             // Check during app startup
builder.Services.ValidateCaptiveDependencies = true; // .NET 10: Specific captive dependency check

// .NET 10: The compiler will WARN about captive dependencies
// This registration would trigger a warning/error in .NET 10:
// warning NETSDK1204: 'UserService' depends on scoped service 'AppDbContext' but is registered as singleton
// INCORRECT - This will cause build warnings/errors
// builder.Services.AddSingleton<UserService>(); // Would warn if UserService depends on Scoped
// CORRECT PATTERN 1: Use IServiceScopeFactory (Recommended)
builder.Services.AddSingleton<ICorrectUserService, CorrectUserService>();
builder.Services.AddScoped<AppDbContext>();
public interface ICorrectUserService
{
    Task<User> GetUserAsync(int id);
    Task<User> UpdateUserAsync(int id, UserUpdateModel model);
}

public class CorrectUserService : ICorrectUserService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<CorrectUserService> _logger;
    private readonly ICacheService _cache; // Singleton dependency is fine
    
    public CorrectUserService(
        IServiceScopeFactory scopeFactory,  // FIXED: Safe for Singleton
        ILogger<CorrectUserService> logger, // Logger is usually Singleton
        ICacheService cache)                 // Singleton - OK
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
        _cache = cache;
    }
    
    public async Task<User> GetUserAsync(int id)
    {
        // Try cache first (Singleton cache is fine)
        var cacheKey = $"user_{id}";
        if (_cache.TryGet<User>(cacheKey, out var cached))
        {
            _logger.LogDebug("Cache hit for user {Id}", id);
            return cached!;
        }
        
        _logger.LogInformation("Cache miss for user {Id}, creating fresh scope", id);
        
        // FIXED: Create a fresh scope for each database operation
        // LINE 997-1000: Fresh scope per operation
        using var scope = _scopeFactory.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        
        // This DbContext is fresh, scoped to this operation
        // It will be disposed when the scope is disposed
        var user = await dbContext.Users
            .Include(u => u.Orders)
            .FirstOrDefaultAsync(u => u.Id == id);
        
        if (user != null)
        {
            // Cache the result (Singleton cache)
            _cache.Set(cacheKey, user, TimeSpan.FromMinutes(5));
        }
        
        return user!;
    }
    
    public async Task<User> UpdateUserAsync(int id, UserUpdateModel model)
    {
        _logger.LogInformation("Updating user {Id} in fresh scope", id);
        
        // FIXED: Each operation gets its own scope and DbContext
        // LINE 1018-1020: Fresh scope per update
        using var scope = _scopeFactory.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        
        var user = await dbContext.Users.FindAsync(id);
        if (user == null)
            throw new NotFoundException($"User {id} not found");
        
        // Update properties
        user.Name = model.Name;
        user.Email = model.Email;
        user.UpdatedAt = DateTime.UtcNow;
        
        await dbContext.SaveChangesAsync();
        
        // Invalidate cache
        _cache.Remove($"user_{id}");
        
        return user;
    }
}

// CORRECT PATTERN 2: Use IHttpContextAccessor for web scenarios
builder.Services.AddSingleton<ITenantService, TenantService>();
builder.Services.AddHttpContextAccessor();
public interface ITenantService
{
    string GetCurrentTenant();
    Task<TenantConfig> GetConfigAsync();
}

public class TenantService : ITenantService
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    private readonly ICacheService _cache;
    private readonly IConfiguration _configuration;
    
    public TenantService(
        IHttpContextAccessor httpContextAccessor, // FIXED: Singleton-safe
        ICacheService cache,
        IConfiguration configuration)
    {
        _httpContextAccessor = httpContextAccessor;
        _cache = cache;
        _configuration = configuration;
    }
    
    public string GetCurrentTenant()
    {
        // FIXED: Access request-scoped data without capturing request services
        // LINE 1061-1064: Safe per-call resolution
        return _httpContextAccessor.HttpContext?.Request.Headers["X-Tenant"].FirstOrDefault()
               ?? _httpContextAccessor.HttpContext?.Request.Host.Host.Split('.')[0]
               ?? "default";
    }
    
    public async Task<TenantConfig> GetConfigAsync()
    {
        var tenantId = GetCurrentTenant();
        var cacheKey = $"tenant_config_{tenantId}";
        
        // Cache tenant config (Singleton cache is fine)
        if (_cache.TryGet<TenantConfig>(cacheKey, out var cached))
            return cached!;
        
        // Load from configuration or database
        var config = new TenantConfig
        {
            TenantId = tenantId,
            ConnectionString = _configuration.GetConnectionString($"Tenant_{tenantId}"),
            Features = _configuration.GetSection($"Features_{tenantId}").Get<FeatureFlags>()
        };
        
        _cache.Set(cacheKey, config, TimeSpan.FromMinutes(10));
        return config;
    }
}

// CORRECT PATTERN 3: Use IOptionsSnapshot for configuration
builder.Services.AddSingleton<IFeatureFlagService, FeatureFlagService>();
builder.Services.AddScoped<IOptionsSnapshot<FeatureFlags>>(); // Scoped, refreshes per request
public interface IFeatureFlagService
{
    bool IsEnabled(string feature);
    Task<bool> IsEnabledAsync(string feature);
}

public class FeatureFlagService : IFeatureFlagService
{
    private readonly IOptionsMonitor<FeatureFlags> _optionsMonitor; // FIXED: Singleton-safe
    private readonly ILogger<FeatureFlagService> _logger;
    
    public FeatureFlagService(
        IOptionsMonitor<FeatureFlags> optionsMonitor, // IOptionsMonitor is Singleton-safe
        ILogger<FeatureFlagService> logger)
    {
        _optionsMonitor = optionsMonitor;
        _logger = logger;
    }
    
    public bool IsEnabled(string feature)
    {
        // FIXED: Gets current value without capturing scope
        // LINE 1105-1106: IOptionsMonitor updates when configuration changes
        return _optionsMonitor.CurrentValue.Flags.Contains(feature);
    }
    
    public async Task<bool> IsEnabledAsync(string feature)
    {
        // Simulate async check
        await Task.CompletedTask;
        return IsEnabled(feature);
    }
}

// CORRECT PATTERN 4: Hybrid approach with factory method
builder.Services.AddSingleton<IReportGenerator>(sp =>
{
    var scopeFactory = sp.GetRequiredService<IServiceScopeFactory>();
    var logger = sp.GetRequiredService<ILogger<ReportGenerator>>();
    var cache = sp.GetRequiredService<ICacheService>();
    
    // Create with dependencies that are safe for Singleton
    return new ReportGenerator(scopeFactory, logger, cache);
});

public class ReportGenerator : IReportGenerator
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<ReportGenerator> _logger;
    private readonly ICacheService _cache;
    
    public ReportGenerator(
        IServiceScopeFactory scopeFactory,
        ILogger<ReportGenerator> logger,
        ICacheService cache)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
        _cache = cache;
    }
    
    public async Task<byte[]> GenerateReportAsync(ReportRequest request)
    {
        _logger.LogInformation("Generating report {ReportId} in fresh scope", request.ReportId);
        
        using var scope = _scopeFactory.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        var tenantService = scope.ServiceProvider.GetRequiredService<ITenantService>();
        
        // All scoped services are fresh per operation
        var data = await dbContext.SalesData
            .Where(s => s.Date >= request.StartDate && s.Date <= request.EndDate)
            .ToListAsync();
        
        var report = await ReportBuilder.Create(data, tenantService.GetCurrentTenant());
        
        // Cache the report if needed
        if (request.CacheResults)
        {
            _cache.Set($"report_{request.ReportId}", report, TimeSpan.FromHours(1));
        }
        
        return report;
    }
}

// .NET 10: Built-in diagnostic helper
if (builder.Environment.IsDevelopment())
{
    // FIXED: Validate service graph for captive dependencies
    // LINE 1168-1175: Automatic validation at startup
    var serviceProvider = builder.Services.BuildServiceProvider();
    var validator = serviceProvider.GetRequiredService<ICaptiveDependencyValidator>();
    
    var results = validator.Validate();
    foreach (var issue in results.Issues)
    {
        Console.WriteLine($"⚠️ Captive Dependency: {issue.Message}");
        Console.WriteLine($"   Service: {issue.ServiceType}");
        Console.WriteLine($"   Dependency: {issue.DependencyType}");
        Console.WriteLine($"   Lifetime: {issue.ServiceLifetime} -> {issue.DependencyLifetime}");
    }
}

// .NET 10: Analyzer attributes for explicit lifetime requirements
[AttributeUsage(AttributeTargets.Class)]
public class RequiredLifetimeAttribute : Attribute
{
    public ServiceLifetime Lifetime { get; }
    public RequiredLifetimeAttribute(ServiceLifetime lifetime) => Lifetime = lifetime;
}
[RequiredLifetime(ServiceLifetime.Scoped)]
public class ShoppingCartRepository
{
    // This class MUST be registered as Scoped
    // The .NET 10 analyzer will warn if registered as Singleton
}

✅ What’s Fixed in .NET 10:

<img width="828" height="422" alt="image" src="https://github.com/user-attachments/assets/c8d6bc6d-04dc-45f0-bd29-35b7971aa27a" />

What Changed in .NET and Why It Matters
<img width="828" height="368" alt="image" src="https://github.com/user-attachments/assets/234663f6-1eaf-4c94-a834-1654e548cc0e" />

Analyzer in Action:
// In your IDE, .NET 10 shows this as a warning:
public class OrderProcessor
{
    private readonly AppDbContext _dbContext; // Scoped
    
    public OrderProcessor(AppDbContext dbContext) // Warning: Scoped dependency in Singleton
    {
        _dbContext = dbContext;
    }
}
// Warning message:
// "OrderProcessor is registered as Singleton but depends on Scoped service AppDbContext.
// This creates a captive dependency. Use IServiceScopeFactory to resolve scoped services."

Captive Dependency Trap Visualization

<img width="828" height="283" alt="image" src="https://github.com/user-attachments/assets/dd94c793-fea2-482e-8bfa-88c6972ed2fd" />

Prevention Checklist
// .NET 10: Safe Singleton Service Checklist
public class SafeSingletonService
{
    // ✅ SAFE - These are Singleton-safe
    private readonly ILogger<SafeSingletonService> _logger;        // Usually Singleton
    private readonly ICacheService _cache;                          // Singleton
    private readonly IConfiguration _configuration;                 // Singleton
    private readonly IOptionsMonitor<Settings> _optionsMonitor;     // Singleton-safe
    private readonly IHttpContextAccessor _httpContextAccessor;     // Singleton-safe
    private readonly IServiceScopeFactory _scopeFactory;            // Singleton-safe
    
    // ❌ UNSAFE - Never inject these directly
    // private readonly AppDbContext _dbContext;                    // Scoped - NO!
    // private readonly ICurrentUserService _currentUser;           // Scoped - NO!
    // private readonly IRequestCache _requestCache;                // Scoped - NO!
    
    public SafeSingletonService(
        ILogger<SafeSingletonService> logger,
        ICacheService cache,
        IConfiguration configuration,
        IOptionsMonitor<Settings> optionsMonitor,
        IHttpContextAccessor httpContextAccessor,
        IServiceScopeFactory scopeFactory)
    {
        _logger = logger;
        _cache = cache;
        _configuration = configuration;
        _optionsMonitor = optionsMonitor;
        _httpContextAccessor = httpContextAccessor;
        _scopeFactory = scopeFactory;
    }
    
    public async Task<T> DoWorkAsync<T>(Func<AppDbContext, Task<T>> dbWork)
    {
        // Create fresh scope for database work
        using var scope = _scopeFactory.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        return await dbWork(dbContext);
    }
}
Conclusion: The Evolution of DI in .NET 10
The Dependency Injection container in .NET 10 represents the culmination of years of refinement and real-world feedback. The core concepts remain exactly as Julio Casal’s diagram illustrates:

The Four Pillars of DI Lifetimes
Press enter or click to view image in full size

<img width="1167" height="353" alt="image" src="https://github.com/user-attachments/assets/e0d938ef-c64b-4e24-83bd-874fe49c4583" />

The .NET 10 Advantage
Performance: Resolution times have dropped by 60–80% across all lifetimes through:
Expression compilation and caching (no reflection)
Lock-free singleton access patterns
Pooled scope objects
Aggressive inlining of factory methods
Reduced memory allocations
Safety: Built-in analyzers catch lifetime mismatches before they reach production:
Compile-time warnings for captive dependencies
Startup validation with clear error messages
Health check integration for runtime monitoring
XML documentation with lifetime requirements
Automatic scope disposal guarantees
3. AOT Ready: Full support for Native AOT compilation:

No reflection at runtime
Pre-compiled service factories
Trimmable and optimized
Ultra-fast startup in cloud environments
4. Observability: Integrated health checks and validation tools:

Service graph visualization
Lifetime validation reports
Metrics for resolution times
Diagnostic logging for scope creation
Distributed tracing with Activity
5. Developer Experience: Clear warnings, suggestions, and documentation:

IDE squiggles for incorrect lifetimes
Quick fixes with proper patterns
Inline documentation with examples
Migration guides for legacy code
Analyzer attributes for explicit requirements
The Golden Rules to Live By
// Rule 1: Respect Lifetime Hierarchy
// Singleton → IServiceScopeFactory ✓
// Singleton → IHttpContextAccessor ✓
// Singleton → IOptionsMonitor ✓
// Singleton → Scoped/Transient ✗

// Rule 2: Choose the Right Lifetime
// Transient: Lightweight, stateless, no shared state
// Scoped: Unit of work, request context, DbContext
// Singleton: Global state, caching, configuration

// Rule 3: Validate Early and Often
builder.Services.ValidateOnBuild = true;
builder.Services.ValidateScopes = true;
builder.Services.ValidateCaptiveDependencies = true;

// Rule 4: Test Captive Dependencies
[Fact]
public void NoCaptiveDependencies()
{
    var services = new ServiceCollection();
    // Add your services
    var provider = services.BuildServiceProvider(new ServiceProviderOptions
    {
        ValidateScopes = true,
        ValidateOnBuild = true
    });
    // If there are captive dependencies, this throws
}
Summary of Fixes: Legacy vs .NET 10

<img width="828" height="331" alt="image" src="https://github.com/user-attachments/assets/1637f1cd-56d2-4e11-a067-5e59f196367f" />

Looking Forward
As .NET continues to evolve, the DI container becomes increasingly invisible — just doing its job efficiently in the background. The patterns remain the same, but the implementation gets smarter, faster, and safer with every release.

Remember Julio Casal’s diagram: Transient for isolation, Scoped for requests, Singleton for global state. And never, ever let a Singleton capture a Scoped service.

Final Thoughts
Dependency Injection in .NET 10 isn’t just about making code testable — it’s about creating predictable, scalable, and maintainable applications. The lifetimes you choose determine:

How your application scales (or doesn’t)
How memory is managed (or leaked)
How data is isolated (or corrupted)
How resources are used (or exhausted)
How bugs manifest (or are prevented)
With .NET 10’s advancements, you have the tools to make these choices confidently, with compile-time validation ensuring your architecture matches your intentions.

The container is your friend. Understand its lifetimes, and it will serve you well.

This comprehensive guide was inspired by the visual guide “How DI Lifetimes Actually Work in .NET” by Julio Casal. The concepts remain timeless, but the implementation keeps getting better with every .NET release.

Happy coding with .NET 10!













































