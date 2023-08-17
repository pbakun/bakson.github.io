---
layout: post
title: Integration tests with Entity Framework Core, Testcontainers & Respawn
tags: [dotnet, testing, database]
# feature-img: "assets/img/articles/2023/deploy-azure-dagger/header.webp"
excerpt_separator: <!--more-->
---

Integration tests are advertised as the best choice if you are to have only one type of tests. They are a&nbsp;powerful tool that can test the full scope of application, but keeping them clean and easy-to-read cam bring challenges. One of which is how to keep the test database state clean.

<!--more-->

## The problem overview

Integration tests are meant to check how the application behaves as a whole (including some IOs). Almost every program utilizes database, therefore this is one of the factors a developer wants to test against. In&nbsp;order for test to serve also as documentation the data should be seeded before each execution. At the end of the check after assertions data should be cleaned, otherwise the next test in sequence would be polluted with residue from the previous one.

Keeping test setup clean may introduce additional overhead when reverting database to its initial state. It can be done manually with truncating tables or simply deleting the data added at the beginning of the test, but this adds a lot of boilerplate code and tedious work. Someone has to put in the time and effort to write code that takes care of the cleanup process after each test.

### Solution

Jimmy Bogard struggled with this problem and decided to help us all. He developed a solution which cleans a database state on demand - [Respawn](https://github.com/jbogard/Respawn). The repository's readme explains pretty well how to use it when with standard ADO.NET connection, but things get a bit more complicated when it comes to widely used Entity Framework.

In the example for database I used PostgreSQL hosted with [Testcontainers](https://testcontainers.com/). This tool spins up Docker container automatically from any given image on test application startup. There are also specific database [modules](https://dotnet.testcontainers.org/modules/) available, but in my example I used PostgreSQL with postgis extension. For this configuration there is no module available, thus I used Docker image directly.

#### Setup

Let's start with defining the `WebApplicationFactory` that will be using with xUnit's `CollectionFixture`. First thing necessary is the database setup. It couldn't be easier. All you need is to build the container and define a wait strategy. After the container is ready the execution moves on to launch the test application fixture. The `CustomWebApplicationFactory` implements `IAsyncLifetime`, which has 2 methods: `InitializeAsync` and `DisposeAsync`. I think the names are self-explanatory. On initialization which runs just after constructor the Testcontainer is started. After test execution finishes it needs to be cleaned up by removing Testcontainer.

```csharp
public class CustomWebApplicationFactory<TStartup> : WebApplicationFactory<TStartup>, IAsyncLifetime where TStartup : class
{
    public string DbConnectionString { get; private set; } = string.Empty;

    private struct DatabaseAccess
    {
        public const string Username = "root";
        public const string Password = "password";
        public const string DatabaseName = "integration_tests";
        public const int Port = 5432;
    }

    private readonly IContainer _dbContainer;

    public CustomWebApplicationFactory()
    {
        _dbContainer = new ContainerBuilder().WithImage("postgis/postgis:latest")
                                            .WithPortBinding(DatabaseAccess.Port, true)
                                            .WithEnvironment("POSTGRES_USER", DatabaseAccess.Username)
                                            .WithEnvironment("POSTGRES_PASSWORD", DatabaseAccess.Password)
                                            .WithEnvironment("POSTGRES_DB", DatabaseAccess.DatabaseName)
                                            .WithEnvironment("PGPORT", DatabaseAccess.Port.ToString())
                                            .WithWaitStrategy(Wait.ForUnixContainer().UntilPortIsAvailable(DatabaseAccess.Port))
                                            .WithCleanUp(true)
                                            .Build();
    }

    public async Task InitializeAsync()
    {
        await _dbContainer.StartAsync().ConfigureAwait(false);
        DbConnectionString = $"Host={_dbContainer.Hostname};Port={_dbContainer.GetMappedPublicPort(DatabaseAccess.Port)};" +
                             $"Database={DatabaseAccess.DatabaseName};Username={DatabaseAccess.Username};Password={DatabaseAccess.Password};" +
                              "Pooling=true;Maximum Pool Size=1024;";"
    }

    async Task IAsyncLifetime.DisposeAsync()
    {
        await _dbContainer.StopAsync();
        await _dbContainer.DisposeAsync();
    }
}
```

The fixture will take the same setup as our main application. It is after all the same program, only slightly modified for testing purposes.

The first thing is to remove existing `ApplicationDbContext` which uses connection string to real database, taken from configuration (at least it should take if from there). Immediately after that the new infrastructure is setup, the only difference is new connection string to our Testcontainer. The next step is to create a `ServiceScope` and retrieve context added couple lines above. The `DatabaseFacade` is stored in a private property, since it is used inside `GetOpenedDbConnectionAsync` method, which ensured that the connection to database is open. It will provide us the connection to the database, so it doesn't have to be passed from each test's scope. At this point Entity Framework migrations can be executed to setup the database schema. The last step is to initialize Respawn, this must be done only after database is seeded with complete schema.

```csharp
protected override void ConfigureWebHost(IWebHostBuilder builder)
{
    private DatabaseFacade _databaseFacade = default!;
    builder.ConfigureTestServices(services =>
    {
        var descriptor = services.SingleOrDefault(d => d.ServiceType == typeof(DbContextOptions<ApplicationDbContext>));
        if (descriptor != null)
            services.Remove(descriptor);
        string connectionString = $"Host={_dbContainer.Hostname};Port={_dbContainer.GetMappedPublicPort(PostgresPort)};Database=integration_tests; Username=root; Password=mypassword;Pooling=true;Maximum Pool Size=1024;";
        services.AddInfrastructure(connectionString);

        var provider = services.BuildServiceProvider();
        using var scope = provider.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();

        _databaseFacade = context.Database;

        context.Database.Migrate();
        InitRespawner().Wait();
    });
}
```

Respawner itself is setup only once. Using an opened connection it can be instantiated, additionally in the `RespawnerOptions` I pass adapter for the database (adjust to the one you use) and set which tables should be excluded from cleanup. For Entity Framework its the `__EFMigrationHistory`, which is used to track if the database schema is up to date with code. In my case the is one more table, but this is specific for the PostgreSQL flavor I'm using - postgis.

The other method will be called after test execution. It will instruct Respawn to clean the database.

```csharp
private Respawner respawner = default!;

public async Task InitRespawner()
{
    DbConnection conn = await GetOpenedDbConnectionAsync();
    respawner = await Respawner.CreateAsync(conn, new RespawnerOptions
    {
        SchemasToInclude = new[] { "public" },
        TablesToIgnore = new Table[]
        {
            "__EFMigrationsHistory",
            "spatial_ref_sys"
        },
        DbAdapter = DbAdapter.Postgres
    });
}

public async Task ResetDatabaseAsync()
{
    DbConnection conn = await GetOpenedDbConnectionAsync();
    await respawner.ResetAsync(conn);
}

private async Task<DbConnection> GetOpenedDbConnectionAsync()
{
    var conn = _databaseFacade.GetDbConnection();
    if (conn.State != System.Data.ConnectionState.Open)
        await conn.OpenAsync();
    return conn;
}
```

To limit code duplications the `IntegrationTest` abstraction is defined. Similarly to `CustomWebApplicationFactory` it implements `IAsyncLifetime`. On the `DisposeAsync` method `ResetDatabaseAsync` from `CustomWebApplicationFactory` is called to restart the database to configured state with Respawn.

One more important thing is that new `ApplicationDbContext` is created. The main purpose is not to use the same context in tested application and test itself. It provides more separation and simulates real life situation better. Additionally `QueryTrackingBehavior` is set to `NoTracking` so the test won't care that much about the data mocked before calling application's endpoint.

> To test `QueryTrackingBehavior` impact try `UpdateCity` test with `NoTracking` and without the line changing it commented out.

```csharp
[Collection(nameof(IntegrationTestCollection))]
public abstract class IntegrationTest : IAsyncLifetime
{
    public readonly CustomWebApplicationFactory<Program> WebApplicationFactory;
    public readonly IServiceScope Scope;
    public readonly ApplicationDbContext Db;
    public readonly HttpClient HttpClient;

    public IntegrationTest(CustomWebApplicationFactory<Program> webApplicationFactory)
    {
        WebApplicationFactory = webApplicationFactory;
        Scope = webApplicationFactory.Services.CreateScope();
        HttpClient = webApplicationFactory.CreateClient();

        var builder = new DbContextOptionsBuilder();
        builder.ConfigureOptions(WebApplicationFactory.DbConnectionString);
        Db = new ApplicationDbContext(builder.Options);
        //Disable tracking enables reads on updated entities (UpdateCity tests)
        Db.ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
    }

    public virtual Task InitializeAsync() => Task.CompletedTask;

    public virtual async Task DisposeAsync()
    {
        await WebApplicationFactory.ResetDatabaseAsync();
        Scope.Dispose();
    }
}
```

The test is attributed with `Collection` of name `IntegrationTestCollection`, meaning it will use this collection unless otherwise instructed. The derived class will inherit this attribute, but it can also have its own with different collection, which may use another `CustomWebApplicationFactory`.

The `CollectionDefinition` is an empty class, which only implements interface `ICollectionFixture<WebApplicationFactory<TProgram>`. This part bounds `CustomWebApplicationFactory` with `IntegrationTest` and passes the factory in the test constructor.

The test parallelization is disabled, otherwise multiple tests might try to access database at the same time and/or restart it while other test runs.

```csharp
[CollectionDefinition(nameof(IntegrationTestCollection), DisableParallelization = true)]
public class IntegrationTestCollection : ICollectionFixture<CustomWebApplicationFactory<Program>>
{
}
```

#### Test

Finally sample test utilizing previously made setup:

```csharp
public class UpdateCity : IntegrationTest
{
    public UpdateCity(CustomWebApplicationFactory<Program> webApplicationFactory) : base(webApplicationFactory)
    {
    }

    [Fact]
    public async Task Ok()
    {
        //arrange
        var city = new City("Klagenfurt") { Location = new NetTopologySuite.Geometries.Point(46.62472, 14.30528) };
        await Db.Cities.AddAsync(city);
        await Db.SaveChangesAsync();

        //act
        var payload = new Presentation.Model.City.UpdateCity
        {
            Name = "Klagenfurt am Wörthersee"
        };

        var response = await HttpClient.PatchAsJsonAsync($"/api/cities/{city.Id}", payload);

        //assert
        response.EnsureSuccessStatusCode();

        var updated = await Db.Cities.FirstOrDefaultAsync(c => c.Id == city.Id);
        updated?.Name.Should().Be(payload.Name);
    }

}
```

Nothing fancy in the test itself. There are 3 steps:
- arrange - setup everything prior testing the system: write necessary data into database, mock services, prepare inputs
- act - execute SUT (system under test); in this case call application's endpoint
- assert - verify outcome: response, database state

After test method is finished `IDisposeAsync` from derived `IntegrationTest` is called and database is restarted to clean state.

In the [repository]() the full project can be found. It contains more tests including adding, listing and removing data. Without clearing the database each time most of them would fail due to data inconsistency.