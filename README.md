# DB Workship 2022
Copy Pasta notes to follow along during the 2022 Grad DB workshop
### Requirements
* vscode
* VLC or any other video stream player to follow the life stream
  * Life stream url: `udp://@224.0.0.111:1111`

## Part 1 - Setup

### Create project

```
dotnet new webapi -o db1
```

### Test

```
dotnet build
```
```
dotnet run
```

### Add EF

```
dotnet add package Microsoft.EntityFrameworkCore.InMemory
```

#### or (if you are running .net 5)
```
dotnet add package -v 5.0.4 Microsoft.EntityFrameworkCore.InMemory
```

### Some house keeping
* create Models folder
* move `WeatherForecast.cs` to `Models` folder
* update namespace to `db1.Models`
* add `using db1.Models;` to `WeatherForecastController.cs`

### Add an id property to the `Weatherforecast` model
```cs
[Key]
[DatabaseGenerated(DatabaseGeneratedOption.Identity)]
public string Id { get; set; }
```

### Create a Data folder and create a new file `DataContext.cs`
```cs
using db1.Models;
using Microsoft.EntityFrameworkCore;

namespace db1.Data
{
  public class DataContext : DbContext
  {
    public DataContext(DbContextOptions<DataContext> options) : base(options) { }

    public DbSet<WeatherForecast> Forecasts { get; set; }
  }
}
```

### Edit `startup.cs` change the function `ConfigureServices` by adding these two lines
```cs
services.AddDbContext<DataContext>(opt => opt.UseInMemoryDatabase("db1"));
services.AddScoped<DataContext, DataContext>();
```

### Change `WeatherForecastController.cs`
```cs
public class WeatherForecastController : ControllerBase
{
  private DataContext _ctx;
  private readonly ILogger<WeatherForecastController> _logger;

  public WeatherForecastController(ILogger<WeatherForecastController> logger, DataContext ctx)
  {
    _logger = logger;
    _ctx = ctx;
  }

  [HttpGet]
  public async Task<ActionResult<List<WeatherForecast>>> Get()
  {
    return await _ctx.Forecasts.ToListAsync();
  }

  [HttpPost]
  public async Task<ActionResult<WeatherForecast>> Add([FromBody] WeatherForecast forecast)
  {
    _ctx.Forecasts.Add(forecast);
    await _ctx.SaveChangesAsync();
    return Ok(forecast);
  }
}
```

### Run the project
```
dotnet run
```

### Using swagger or postman post some weather forecasts
Url for **POST** `http://127.0.0.1:5000/weatherforecast`

example JSON for body:
```json
{
  "date": "2021-01-01",
  "temperaturec": 44,
  "summary": "hot as hell"
}
```

### Fetch content
Url for **GET** `http://127.0.0.1:5000/weatherforecast`

## Part 2 - Parameters
### Add 2 Get methods to `WeatherForecastController.cs`
```cs
[HttpGet("{id}")]
public async Task<ActionResult<WeatherForecast>> GetById(int id)
{
  var retVal = await _ctx.Forecasts.FindAsync(id);

  if (retVal == null)
  {
    return NotFound();
  }

  return Ok(retVal);
}

[HttpGet("filter")]
public async Task<ActionResult<List<WeatherForecast>>> FilterByName([FromQuery] string name)
{
  var retVal = await _ctx.Forecasts
    .Where(x => x.city.Contains(name))
    //.Where(x => x.city.ToLower().Contains(name.ToLower()))
    .ToListAsync();

  if (retVal == null)
  {
    return NotFound();
  }

  return Ok(retVal);
}
```

### Fetch content - ID
Url for **GET** `http://127.0.0.1:5000/weatherforecast/1`

### Fetch content - Filter
Url for **GET** `http://127.0.0.1:5000/weatherforecast/filter?name=abc`

## Part 3 – Cache
Type 1: https://github.com/VahidN/EFCoreSecondLevelCacheInterceptor

Type 2: https://referbruv.com/blog/posts/configuring-and-integrating-redis-cache-in-aspnet-core

## Part 4 – Connect to a remote db
### Install vscode extension - PostgreSQL
### Connect to a remote PostgreSQL db
```
ip: <My ip here>
usr: dev
pwd: 12345
db: weather
```

### Some sql exercises
* Hottest temp at 9:00 (exclude 999.9)
* Coldest temp at 19:00
* Hottest temp on a Wednesday
* Average temp on a Friday at 21:00 for every station
* Average of the hottest 5 temps for every station

## Part 5 – Connect code to a remote db
### Create a new model `HourlyData.cs`
```cs
using System;
using System.ComponentModel.DataAnnotations.Schema;

namespace db1.Models
{
  [Table("hourly_data")]
  public class HourlyData
  {
    [Column("station")]
    public string Station { get; set; }

    [Column("date")]
    public DateTime Date { get; set; }

    [Column("rn")]
    public long Rn { get; set; }

    [Column("tmp")]
    public double Tmp { get; set; }
  }
}
```

### Update `DataContext.cs`
```cs
  public DbSet<HourlyData> HourlyData { get; set; }

  protected override void OnModelCreating(ModelBuilder modelBuilder)
  {
    modelBuilder.Entity<HourlyData>()
      .HasKey(hd => new { hd.Station, hd.Date, hd.Rn });
  }
```

### Add Get method to `WeatherForecastController.cs`
```cs
[HttpGet("hourly")]
public async Task<ActionResult<List<HourlyData>>> GetHourlyData()
{
  return await _ctx.HourlyData.Take(100).ToListAsync();
}
```

### Add postgres nuget package
```
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL --version 5
```
#### or if you run .net 6
```
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
```

### Change `startup.cs` to point to pg
```cs
  //services.AddDbContext<DataContext>(opt => opt.UseInMemoryDatabase("db1"));
  services.AddDbContext<DataContext>(opt => opt.UseNpgsql(@"Host=127.0.0.1;Username=dev;Password=12345;Database=weather"));
```

## Part 6 – Mongo
### Start Mongo
```
docker run -d -p 27017:27017 mongo:5.0.5-focal
```

### Add mongo package
```
dotnet add package MongoDB.Driver
```

### Modify `WeatherForecast.cs`
```cs
[Key]
[DatabaseGenerated(DatabaseGeneratedOption.Identity)]
[BsonId]
[BsonRepresentation(BsonType.ObjectId)]
public string Id { get; set; }
```

### Add a **GET** method to `WeatherForecastController.cs`
```cs
[HttpGet("mongo")]
public async Task<ActionResult<List<WeatherForecast>>> GetMongoForecast()
{
  var mongoClient = new MongoClient("mongodb://localhost:27017");
  var mongoDatabase = mongoClient.GetDatabase("weather");
  var collection = mongoDatabase.GetCollection<WeatherForecast>("forecasts");

  return await collection.Find(b => true).ToListAsync();
}
```

### Add a **GET** method to `WeatherForecastController.cs`
```cs
[HttpPost("mongo")]
public async Task<ActionResult<WeatherForecast>> AddMongoForecast([FromBody] WeatherForecast forecast)
{
  var mongoClient = new MongoClient("mongodb://localhost:27017");
  var mongoDatabase = mongoClient.GetDatabase("weather");
  var collection = mongoDatabase.GetCollection<WeatherForecast>("forecasts");

  await collection.InsertOneAsync(forecast);

  return Ok(forecast);
}
```

### Clean-up on isle 5 - Add a Services folder
### Add a ForecastService.cs
```cs
using System.Collections.Generic;
using System.Threading.Tasks;
using db1.Models;
using MongoDB.Driver;

namespace db1.Services
{
    public class ForecastService
    {
        private readonly IMongoCollection<WeatherForecast> _collection;

        public ForecastService()
        {
            // TODO: for you to practice: inject config from application.json here
            var mongoClient = new MongoClient("mongodb://localhost:27017");
            var mongoDatabase = mongoClient.GetDatabase("weather");
            _collection = mongoDatabase.GetCollection<WeatherForecast>("forecasts");
        }

        public async Task<List<WeatherForecast>> GetTop100Async()
        {
            return await _collection.Find(f => true).ToListAsync();
        }

        public async Task Add(WeatherForecast forecast)
        {
            await _collection.InsertOneAsync(forecast);
        }
    }
}
```

### Update `Program.cs`
```cs
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(webBuilder =>
            webBuilder.UseStartup<Startup>()
        )
        .ConfigureServices(s =>
            s.AddSingleton<ForecastService>()
        );
```

### Inject the service into `WeatherForecastController.cs`
```cs
private readonly ForecastService _service;

public WeatherForecastController(ILogger<WeatherForecastController> logger, DataContext ctx, ForecastService service)
{
    _logger = logger;
    _ctx = ctx;
    _service = service;
}
```

### Update the mongo **GET** methods in `WeatherForecastController.cs` to use the service
```cs
[HttpGet("mongo")]
public async Task<ActionResult<List<WeatherForecast>>> GetMongoForcast()
{
    return await _service.GetTop100Async();
}

[HttpPost("mongo")]
public async Task<ActionResult<WeatherForecast>> AddMongoForcast([FromBody] WeatherForecast forecast)
{
    await _service.Add(forecast);

    return Ok(forecast);
}
```