
# SQLite-net

SQLite-net is an open source, minimal library to allow .NET, .NET Core, and Mono applications to store data in
[SQLite3 databases](http://www.sqlite.org). It was first designed to work with [Xamarin.iOS](http://xamarin.com),
but has since grown up to work on all the platforms (Xamarin.*, .NET, UWP, Azure, etc.).

SQLite-net was designed as a quick and convenient database layer. Its design follows from these *goals*:

* Very easy to integrate with existing projects and runs on all the .NET platforms.
  
* Thin wrapper over SQLite that is fast and efficient. (This library should not be the performance bottleneck of your queries.)
  
* Very simple methods for executing CRUD operations and queries safely (using parameters) and for retrieving the results of those query in a strongly typed fashion.
  
* Works with your data model without forcing you to change your classes. (Contains a small reflection-driven ORM layer.)
  

## Source Installation

SQLite-net is all contained in 1 file (I know, so cool right?) and is easy to add to your project. Just add [SQLite.cs](https://github.com/praeclarum/sqlite-net/blob/master/src/SQLite.cs) to your project, and you're ready to start creating tables. 


# Example Time!

Please consult the Wiki for the [complete documentation](https://github.com/praeclarum/sqlite-net/wiki)

The library contains simple attributes that you can use to control the construction of tables. In a simple stock program, you might use:

```csharp
public class Stock
{
	[PrimaryKey]
	public long Id { get; set; }
	public string Symbol { get; set; }
}

public class Valuation
{
	[PrimaryKey]
	public long Id { get; set; }
	[Indexed]
	public long StockId { get; set; }
	public DateTime Time { get; set; }
	public decimal Price { get; set; }
}
```

> On tables with integer primary keys the primary key column must be declared as `long` instead of `int` because LiteSync always use a 64-bit number for row ids.

> LiteSync does not support the `AutoIncrement` keyword.

Once you've defined the objects in your model you have a choice of APIs. You can use the "synchronous API" where calls
block one at a time, or you can use the "asynchronous API" where calls do not block. You may care to use the asynchronous
API for mobile applications in order to increase responsiveness.

Both APIs are explained in the two sections below.


## Synchronous API

Once you have defined your entity, you can automatically generate tables in your database by calling `CreateTable`:

```csharp
// Get an absolute path to the database file
var databasePath = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData), "MyData.db");
var uri = "file:" + databasePath + "?node=secondary&connect=tcp://123.45.67.89:1234";
var db = new SQLiteConnection(uri);

// wait until the db is ready
while (!db.IsReady()) {
    System.Threading.Thread.Sleep(250);
}

// now we can use the database
db.CreateTable<Stock>(CreateFlags.AutoIncPK);
db.CreateTable<Valuation>(CreateFlags.AutoIncPK);
```

LiteSync does not support the `AutoIncrement` keyword, but we can pass the `AutoIncPK` flag when creating a table.

You can insert rows in the database using `Insert`. If the table contains an auto-incremented primary key, then the value for that key will be available to you after the insert:

```csharp
public static void AddStock(SQLiteConnection db, string symbol) {
	var stock = new Stock() {
		Symbol = symbol
	};
	db.Insert(stock);
	Console.WriteLine("{0} == {1}", stock.Symbol, stock.Id);
}
```

Similar methods exist for `Update` and `Delete`.

The most straightforward way to query for data is using the `Table` method. This can take predicates for constraining via WHERE clauses and/or adding ORDER BY clauses:

```csharp
var query = db.Table<Stock>().Where(v => v.Symbol.StartsWith("A"));

foreach (var stock in query)
	Console.WriteLine("Stock: " + stock.Symbol);
```

You can also query the database at a low-level using the `Query` method:

```csharp
public static IEnumerable<Valuation> QueryValuations (SQLiteConnection db, Stock stock) {
	return db.Query<Valuation> ("select * from Valuation where StockId = ?", stock.Id);
}
```

The generic parameter to the `Query` method specifies the type of object to create for each row. It can be one of your table classes, or any other class whose public properties match the column returned by the query. For instance, we could rewrite the above query as:

```csharp
public class Val
{
	public decimal Money { get; set; }
	public DateTime Date { get; set; }
}

public static IEnumerable<Val> QueryVals (SQLiteConnection db, Stock stock) {
	return db.Query<Val> ("select \"Price\" as \"Money\", \"Time\" as \"Date\" from Valuation where StockId = ?", stock.Id);
}
```

You can perform low-level updates of the database using the `Execute` method.


## Asynchronous API

The asynchronous library uses the Task Parallel Library (TPL). As such, normal use of `Task` objects, and the `async` and `await` keywords 
will work for you.

Once you have defined your entity, you can automatically generate tables by calling `CreateTableAsync`:

```csharp
// Get an absolute path to the database file
var databasePath = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData), "MyData.db");
var uri = "file:" + databasePath + "?node=secondary&connect=tcp://123.45.67.89:1234";
var db = new SQLiteAsyncConnection(uri);

if (db.IsReady())
{
    // the database is ready to be accessed
    InitDatabase(db);
}
else
{
    // wait until the db is ready (and do not access the database)
    db.OnReady(() => {
        // the database is ready to be accessed
        InitDatabase(db);
    });
}

void InitDatabase(SQLiteAsyncConnection db) {
    await db.CreateTableAsync<Stock>(CreateFlags.AutoIncPK);
    await db.CreateTableAsync<Valuation>(CreateFlags.AutoIncPK);
}
```

LiteSync does not support the `AutoIncrement` keyword, but we can pass the `AutoIncPK` flag when creating a table.

You can insert rows in the database using `Insert`. If the table contains an auto-incremented primary key, then the value for that key will be available to you after the insert:

```csharp
var stock = new Stock()
{
	Symbol = "TSLA"
};

await db.InsertAsync(stock);

Console.WriteLine("Auto stock id: {0}", stock.Id);
```

Similar methods exist for `UpdateAsync` and `DeleteAsync`.

Querying for data is most straightforwardly done using the `Table` method. This will return an `AsyncTableQuery` instance back, whereupon
you can add predicates for constraining via WHERE clauses and/or adding ORDER BY. The database is not physically touched until one of the special 
retrieval methods - `ToListAsync`, `FirstAsync`, or `FirstOrDefaultAsync` - is called.

```csharp
var query = db.Table<Stock>().Where(s => s.Symbol.StartsWith("A"));

var result = await query.ToListAsync();

foreach (var s in result)
	Console.WriteLine("Stock: " + s.Symbol);
```

There are a number of low-level methods available. You can also query the database directly via the `QueryAsync` method. Over and above the change 
operations provided by `InsertAsync` etc you can issue `ExecuteAsync` methods to change sets of data directly within the database.

Another helpful method is `ExecuteScalarAsync`. This allows you to return a scalar value from the database easily:

```csharp
var count = await db.ExecuteScalarAsync<int>("select count(*) from Stock");

Console.WriteLine(string.Format("Found '{0}' stock items.", count));
```


## Sync Notifications

Your application can receive a notification when the database receives a sync/update

```csharp
db.OnSync(() => {
    // the db received an update. update the screen with new data
    UpdateScreen(db);
});
```


## Manual SQL

**sqlite-net** is normally used as a light ORM (object-relational-mapper) using the methods `CreateTable` and `Table`.
However, you can also use it as a convenient way to manually execute queries.

Here is an example of creating a table, inserting into it (with a parameterized command), and querying it without using ORM features.

```csharp
db.Execute ("create table Stock(Symbol text not null)");
db.Execute ("insert into Stock(Symbol) values (?)", "TSLA");
var stocks = db.Query<Stock> ("select * from Stock");
```
