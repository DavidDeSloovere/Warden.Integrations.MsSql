# Warden MsSql Integration

![Warden](http://spetz.github.io/img/warden_logo.png)

**OPEN SOURCE & CROSS-PLATFORM TOOL FOR SIMPLIFIED MONITORING**

**[getwarden.net](http://getwarden.net)**

|Branch             |Build status                                                  
|-------------------|-----------------------------------------------------
|master             |[![master branch build status](https://api.travis-ci.org/warden-stack/Warden.Integrations.MsSql.svg?branch=master)](https://travis-ci.org/warden-stack/Warden.Integrations.MsSql)
|develop            |[![develop branch build status](https://api.travis-ci.org/warden-stack/Warden.Integrations.MsSql.svg?branch=develop)](https://travis-ci.org/warden-stack/Warden.Integrations.MsSql/branches)

**MsSqlIntegration** can be used for querying or saving the data using the MSSQL database. 

### Installation:

Available as a **[NuGet package](https://www.nuget.org/packages/Warden.Integrations.MsSql)**. 
```
dotnet add package Warden.Integrations.MsSql
```

### Configuration:

- **WithCommand()** - Sets the SQL command and its parameters (optional).
- **WithCommandTimeout()** - Sets the timeout for the SQL command execution.
- **WithQuery()** - Sets the SQL query and its parameters (optional).
- **WithQueryTimeout()** -  Sets the timeout for the SQL query execution.
- **WithConnectionProvider()** - Sets the custom provider for the *IDbConnection*.
- **WithMsSqlServiceProvider()** - Sets the custom provider for the *IMsSqlService*.



### Initialization:

In order to register and resolve **MsSqlIntegration** make use of the available extension methods while configuring the **Warden**:

```csharp
var wardenConfiguration = WardenConfiguration
    .Create()
    .IntegrateWithMsSql(@"Data Source=.\sqlexpress;Initial Catalog=MyDatabase;Integrated Security=True")
    .SetHooks((hooks, integrations) =>
    {
	hooks.OnIterationCompletedAsync(iteration => integrations.MsSql()
     	     .QueryAsync<int>("select * from users where id = @id", GetSqlQueryParams()))
     	     .OnIterationCompletedAsync(iteration => integrations.MsSql()
     	     .ExecuteAsync("insert into messages values(@message)", GetSqlCommandParams()));
    })
    //Configure watchers, hooks etc..

private static IDictionary<string, object> GetSqlQueryParams()
    => new Dictionary<string, object> {["id"] = 1};

private static IDictionary<string, object> GetSqlCommandParams()
    => new Dictionary<string, object> {["message"] = "Iteration completed"};
```

Besides the generic methods for executing the custom SQL commands, this integration also provides the built-in function *SaveIterationAsync()* to store the *IWardenIteration* object based on the table schema listed below.

```csharp
var wardenConfiguration = WardenConfiguration
    .Create()
    .IntegrateWithMsSql(@"Data Source=.\sqlexpress;Initial Catalog=MyDatabase;Integrated Security=True")
    .SetHooks((hooks, integrations) =>
    {
        hooks.OnIterationCompletedAsync(iteration => 
              OnIterationCompletedMsSqlAsync(iteration, integrations.MsSql()));
    })
    //Configure watchers, hooks etc..

private static async Task OnIterationCompletedMsSqlAsync(IWardenIteration wardenIteration,
    MsSqlIntegration integration)
{
    await integration.SaveIterationAsync(wardenIteration);
}
```

Database schema for storing *IWardenIteration*:

```
CREATE TABLE WardenIterations
(
	Id bigint primary key identity not null,
	WardenName nvarchar(MAX) not null,
	Ordinal bigint not null,
	StartedAt datetime not null,
	CompletedAt datetime not null,
	ExecutionTime time not null,
	IsValid bit not null
)

CREATE TABLE WardenCheckResults
(
	Id bigint primary key identity not null,
	WardenIteration_Id bigint not null,
	IsValid bit not null,
	StartedAt datetime not null,
	CompletedAt datetime not null,
	ExecutionTime time not null,
	foreign key (WardenIteration_Id) references WardenIterations(Id)
)

CREATE TABLE WatcherCheckResults
(
	Id bigint primary key identity not null,
	WardenCheckResult_Id bigint not null,
	WatcherName nvarchar(MAX) not null,
	WatcherType nvarchar(MAX) not null,
	Description nvarchar(MAX) not null,
	IsValid bit not null,
	foreign key (WardenCheckResult_Id) references WardenCheckResults(Id)
)

CREATE TABLE Exceptions
(
	Id bigint primary key identity not null,
	WardenCheckResult_Id bigint not null,
	ParentException_Id bigint null,
	Message nvarchar(MAX) null,
	Source nvarchar(MAX) null,
	StackTrace nvarchar(MAX) null,
	foreign key (WardenCheckResult_Id) references WardenCheckResults(Id),
	foreign key (ParentException_Id) references Exceptions(Id)
)
```

### Custom interfaces:
```csharp
public interface IMsSqlService
{
    Task<IEnumerable<T>> QueryAsync<T>(IDbConnection connection, string query,
            IDictionary<string, object> parameters, TimeSpan? timeout = null);

    Task<int> ExecuteAsync(IDbConnection connection, string command,
            IDictionary<string, object> parameters, TimeSpan? timeout = null);
}
```

**IMsSqlService** is responsible for both executing the query and command on a database. It can be configured via the *WithMsSqlServiceProvider()* method. By default it is based on the **[Dapper](https://github.com/StackExchange/dapper-dot-net)**.