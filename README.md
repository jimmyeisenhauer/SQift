# SQift

SQift is a lightweight Swift wrapper for SQLite.

## Features

- [X] On-Disk, In-Memory and Temporary Database Connections
- [X] SQL Statement Execution
- [X] Generic Parameter Binding and Value Extraction
- [X] Simple Row Iteration through `SequenceType` Protocol
- [X] Fetch a Single Row without Iteration
- [X] Query a Single Column Value from a Single Row
- [X] ConnectionQueue for Serial Execution per Database Connection
- [X] ConnectionPool for Parallel Execution of Read-Only Connections
- [X] Top-Level Database to Simplify Thread-Safe Reads and Writes
- [X] Database Migrations Support
- [X] Database Encryption and Exporting Decrypted or Encrypted Variations
- [x] Comprehensive Unit Test Coverage
- [x] Complete Documentation

## Requirements

- iOS 8.0+, OSX 10.10+, tvOS 9.0+, watchOS 2.0+
- Xcode 7.0+

## Dependencies

- [SQLCipher](https://github.com/sqlcipher/sqlcipher) - 256 bit AES Encryption

## Communication

- Need help? Open a [Question](https://jira.nike.com/browse/bmd). (Component => `SQift`)
- Have a feature request? Open a [Feature Request](https://jira.nike.com/browse/bmd). (Component => `SQift`)
- Find a bug? Open a [Bug](https://jira.nike.com/browse/bmd). (Component => `SQift`)
- Want to contribute? Fork the repo and submit a pull request.

> These tickets go directly to the developers of SQift who are very adament about providing top notch support for this library. Please don't hesitate to open tickets for any type of issue. If we don't know about it, we can't fix it, support it or build it.

---

## Installation

### CocoaPods

[CocoaPods](http://cocoapods.org) is a dependency manager for Cocoa projects. You can install it with the following command:

```bash
$ gem install cocoapods
```

> CocoaPods 0.39.0+ is required to build SQift.

To integrate SQift into your Xcode project using CocoaPods, specify it in your `Podfile`:

```ruby
platform :ios, '8.0'
use_frameworks!

# Spec sources
source 'ssh://git@stash.nikedev.com/ncps/nike-private-spec.git'
source 'https://github.com/CocoaPods/Specs.git'

pod 'SQift', '~> 0.0.1'
```

Then, run the following command:

```bash
$ pod install
```

---

## Usage

SQift is designed to make it as easy as possible to work with SQLite from Swift. It does not, however, eliminate the need to understand how SQLite actually works. Before diving into SQift, it is recommended to first have a firm grasp on what SQLite is, how it works and how to use it.

- [SQLite Docs](https://www.sqlite.org/docs.html)
- [SQLite Tutorial](http://zetcode.com/db/sqlite/)

SQift heavily leverages the new error handling model released with Swift 2.0. It was designed from the ground up to throw in all applicable cases. This makes it easy to wrap all your SQift calls in the do / catch paradigm.

### Creating a Database Connection

Creating a database connection is simple.

```swift
let onDiskConnection = try Connection(storageLocation: .OnDisk("path_to_db"))
let inMemoryConnection = try Connection(storageLocation: .InMemory)
let tempConnection = try Connection(storageLocation: .Temporary)
```

There are also convenience parameters to make it easy to customize the flags when initializing the database connection:

```swift
let connection = try Connection(
	storageLocation: .OnDisk("path_to_db"),
	readOnly: true,
	multiThreaded: false,
	sharedCache: false
)
```

> In most cases, the default values are preferred. For more details about creating a database connection, please refer to the SQLite [documentation](https://www.sqlite.org/c3ref/open.html).

### Executing Statements

To execute a SQL statement on the `Connection`, you need to first create a `Connection`, then call `execute`.

```swift
let connection = try Connection(storageLocation: .OnDisk("path_to_db"))

try connection.execute("PRAGMA foreign_keys = true")
try connection.execute("PRAGMA journal_mode = WAL")

try connection.execute("CREATE TABLE cars(id INTEGER PRIMARY KEY, name TEXT, price INTEGER)")

try connection.execute("INSERT INTO cars VALUES(1, 'Audi', 52642)")
try connection.execute("INSERT INTO cars VALUES(2, 'Mercedes', 57127)")

try connection.execute("UPDATE cars SET name = 'Honda' where id = 1")
try connection.execute("UPDATE cars SET price = 61_999 where name = 'Mercedes'")

try connection.execute("DELETE FROM cars where name = 'Mercedes'")
try connection.execute("DROP TABLE cars")
```

### Bindings

Most Swift data types cannot be directly stored inside the database. They need to be converted to a data type supported by SQLite. In order to support moving Swift data types into the database and back out again, SQift leverage three powerful protocol: `Bindable`, `Extractable` and `Binding`.

#### Bindable Protocol

The `Bindable` protocol handles converting Swift data types into a `BindingValue` enumeration type which can be stored in the database.

```swift
public protocol Bindable {
    var bindingValue: BindingValue { get }
}
```

#### Extractable Protocol

While the `Bindable` protocol helps move Swift data types into the database, the `Extractable` protocol allows SQift to extract the values from the database `Connection` and convert them back to the requested Swift data type.

```swift
public protocol Extractable {
    typealias BindingType
    typealias DataType = Self
    static func fromBindingValue(value: Any) -> DataType
}
```

#### Binding Protocol

To extend Swift data types to be able to be inserted into the database and also be extracted safely, the `Binding` protocol forces the data type to conform to both the `Bindable` and `Extractable` protocols.

```swift
public protocol Binding: Bindable, Extractable {}
```

In order to make it as easy as possible to use SQift, SQift extends the following Swift data types to conform to the `Binding` protocol:

- **NULL:** `NSNull`
- **INTEGER:** `Bool`, `Int8`, `Int16`, `Int32`, `Int64`, `Int`, `UInt8`, `UInt16`, `UInt32`, `UInt64`, `UInt`
- **REAL:** `Float`, `Double`
- **TEXT:** `String`, `NSDate`
- **BLOB:** `NSData`

> Additional Swift data types can easily add `Bindable` protocol conformance if necessary.

### Binding Parameters to a Statement

Safely binding parameters to a `Statement` is easy thanks to the `Binding` protocol. First you need to `prepare` a `Statement` object, then `bind` the parameters and `run` it using method chaining.

```swift
let connection = try Connection(storageLocation: .OnDisk("path_to_db"))

try connection.prepare("INSERT INTO cars VALUES(?, ?, ?)").bind(1, "Audi", 52_642).run()
try connection.prepare("INSERT INTO cars VALUES(:id, :name, :price)").bind([":id": 1, ":name": "Audi", ":price": 52_642]).run()
```

There are also convenience methods on the `Connection` for preparing a `Statement`, binding parameters and running it all in a single method named `run`.

```swift
let connection = try Connection(storageLocation: .OnDisk("path_to_db"))

try connection.run("INSERT INTO cars VALUES(?, ?, ?)", 1, "Audi", 52_642)
try connection.run("INSERT INTO cars VALUES(:id, :name, :price)", parameters: [":id": 1, ":name": "Audi", ":price": 52_642])
```

> It is very important to properly esacpe all parameter values provided in a SQL statement. When in doubt, always use the provided bind functionality.

### Fetching Data

Fetching data from the database also leverages the `Binding` protocol extensively to extract data using generics.

#### Iterating through Rows

To iterate through all the rows of a `Statement`, the `Statement` class conforms to the `SequenceType` protocol. This allows you to iterate through a `Statement` using fast enumeration.

```swift
for row in try connection.prepare("SELECT * FROM cars") {
    print(row)
}
```

#### Row Subscripts

Extracting the values from the row is handled through the `Row` subscript methods.

```swift
struct Car {
    let name: String
    let price: UInt
}

var cars: [Car] = []

for row in try connection.prepare("SELECT * FROM cars") {
    let car = Car(name: row[1], price: row[2])
    cars.append(car)
}
```

The `SequenceType` conformance also let's you use methods like `map` on the `Statement`.

```swift
let cars: [Car] = try connection.prepare("SELECT name, price FROM cars").map { Car(name: $0[0], price: $0[1]) }
```

You can also extract the `Row` values using the column name.

```swift
for row in try connection.prepare("SELECT * FROM cars WHERE price > ?", 20_000) {
	let name: String = row["name"]
	let price: UInt = row["price"]

	print("Name: \(name), Price: \(price)")
}
```

#### Fetching a Single Row

Sometimes you only want to fetch a single `Row`. Maybe you know beforehand that there will only be 1 row, or maybe you specified a LIMIT of 1. Either way, you can easily fetch a single `Row` without having to iterate over the `Statement`.

```swift
let row = try connection.fetch("SELECT * FROM cars WHERE name = ?", "Audi")
print(row)
```

### Querying Data

When querying the database for a single value, you want to use the `query` method on the `Connection`. It is designed to extract the first column value from the first `Row` returned. This is very useful for certain SELECT and PRAGMA statements.

```swift
let avgHighEndPrice: UInt = try db.query("SELECT avg(price) FROM cars WHERE price > ?", 40_000)
let synchronous: Int = try db.query("PRAGMA synchronous")
```

> You MUST be careful when using the `query` method. You should always use the version of the `query` API where the return type is optional unless you are 100% sure that the value will exist.

### Dates

Since SQLite already has builtin support for dates, it's critical that SQift leverage that capability for things like being able to store dates in the database, run range queries against them and be able to extract them back out. SQift handles all this functionality through the `NSDate` binding as well as the `BindingDateFormatter`. By default, all `NSDate` types will be stored in the database as `TEXT`, so make sure to set your column types accordingly.

```swift
let date1975: NSDate!
let date1983: NSDate!
let date1992: NSDate!
let date2001: NSDate!

try connection.execute(
	"CREATE TABLE cars(id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT, release_date TEXT)"
)

try connection.execute("INSERT INTO cars(?, ?)", "70s car", date1975)
try connection.execute("INSERT INTO cars(?, ?)", "80s car", date1983)
try connection.execute("INSERT INTO cars(?, ?)", "90s car", date1992)
try connection.execute("INSERT INTO cars(?, ?)", "00s car", date2001)
```

Once you have your dates stored in the database, you can run date range queries to narrow down your data.

```swift
let date1980: NSDate!
let date2000: NSDate!

let carCount: Int = try connection.query(
	"SELECT count(*) FROM cars WHERE release_date >= date(?) AND release_date <= date(?)",
	date1980, 
	date2000
)

print("Total Cars from the 80s and 90s: \(carCount)") // should equal 2
```

> You can swap the default date formatting, but be careful when doing so. You need to make sure the new date format complies with the SQLite [requirements](https://www.sqlite.org/lang_datefunc.html) so date range queries will continue to work as expected.

You can also extract dates out of each row as an `NSDate`.

```swift
let releaseDate: NSDate = try connection.fetch(
	"SELECT release_date WHERE name = ? LIMIT 1", "80s car"
)[0]
```

---

## Advanced

### Thread Safety

Thread-safety is a complex topic when it comes to SQLite. As a general rule, it is NEVER safe to access a database `Connection` from multiple threads simultaneously. Each connection should be accessed serially to guarantee safety. 

If you wish to access the database in parallel, there are a few things you need to know. First off, you'll need to use Write-Ahead Logging by setting the journal mode to `WAL`. By changing the database to a `WAL` journal mode, the database can be read during a write and written during a read in parallel using multiple connections.

```swift
try connection.execute("PRAGMA journal_mode = WAL")
```

Another important note is that SQLite can only perform write operations serially, no matter how many connections you create. Therefore, you should only ever create a single connection for writing if possible. You can use as many reader connections as you wish. For more information about thread-safety and WAL journal modes, please refer to the following:

- [Write-Ahead Logging](https://www.sqlite.org/wal.html)
- [SQLite and Multiple Threads](http://www.sqlite.org/threadsafe.html)
- [SQLite WAL mode with multiple transactions in multiple threads](http://stackoverflow.com/questions/14234007/sqlite-wal-mode-with-multiple-transactions-in-multiple-threads)

#### Connection Queue

The `ConnectionQueue` class in SQift was designed to help guarantee thread-safety for a database `Connection` that could be accessed from multiple threads. It executes all operations on an internal serial dispatch queue. This ensures all operations on the connection operation in a serial fashion. The `ConnectionQueue` also supports executing logic inside a transaction and savepoint.

```swift
let queue = try ConnectionQueue(connection: Connection(storageLocation: .OnDisk("path_to_db")))

try queue.execute { connection in
    try connection.execute("PRAGMA foreign_keys = true")
    try connection.execute("PRAGMA journal_mode = WAL")
    try connection.execute("CREATE TABLE cars(id INTEGER PRIMARY KEY, name TEXT, price INTEGER)")
}

try queue.executeInTransaction { connection in
    try connection.execute("INSERT INTO cars VALUES(1, 'Audi', 52642)")
    try connection.execute("INSERT INTO cars VALUES(2, 'Mercedes', 57127)")
}

try queue.executeInSavepoint("drop_cars_table") { connection in
	try connection.execute("DROP TABLE cars")	
}
```

#### Connection Pool

The `ConnectionPool` class allows multiple read-only connections to access a database simultaneously in a thread-safe manner. Internally, the pool manages two different sets of connections, ones that are available and ones that are currently busy executing SQL logic. The pool will reuse available connections when they are available, and initializes new connections when all available connections are busy until the max connection count is reached.

```swift
let pool = try ConnectionPool(storageLocation: .OnDisk("path_to_db"))

try pool.execute { connection in
    let count: Int = try connection.query("SELECT count(*) FROM cars")
}
```

Since SQLite has no limit on the maximum number of open connections to a single database, the `ConnectionPool` will initialize as many connections as needed within a small amount of time. Each time a connection is executed, the internal drain delay timer starts up. When the drain delay timer fires, it will drain the available connections if there are no more busy connections. If there are still busy connections, the timer is restarted. This allows the `ConnectionPool` to spin up as many connections as necessary for very small amounts of time.

> The thread-safety is guaranteed by the connection pool by always executing the SQL closure inside a connection queue. This ensures all SQL closures executed on the connection are done so in a serial fashion, thus guaranteeing the thread-safety of each connection.

#### Database

The `Database` class is a lightweight way to create a single writable connection queue and connection pool for all read statements. The read and write APIs are designed to make it simple to execute SQL statements on the appropriate type of `Connection` in a thread-safe manner.

```swift
let database = try Database(storageLocation: .OnDisk("path_to_db"))

try database.executeWrite { connection in
    try connection.execute("PRAGMA foreign_keys = true")
    try connection.execute("PRAGMA journal_mode = WAL")
    try connection.execute("CREATE TABLE cars(id INTEGER PRIMARY KEY, name TEXT, price INTEGER)")
}

try database.executeRead { connection in
	let count: Int = try connection.query("SELECT count(*) FROM cars")
}
```

This is the easiest way to operate in a 100% thread-safe manner without having to deal with the underlying complexities of the `ConnectionQueue` and `ConnectionPool` classes.

> We would like to encourage everyone to use a `Database` object rather than working directly with connection queues or connection pools.

### Migrations

Production applications generally need to migrate the database schema from time-to-time. Whether it requires some new tables or possibly alterations to a table, you need to have a way to manage the migration logic. SQift has migration support already built-in for you through the `Migrator` class. All you need to do is create the `Migrator` instance and tell it to run. Everything else is handled internally by SQift.

```swift
let connection = try Connection(storageLocation: .OnDisk("path_to_db"))
let migrator = Migrator(connection: connection, desiredSchemaVersion: 2)

try migrator.runMigrationsIfNecessary(
	migrationSQLForSchemaVersion: { version in
		var SQL: String = ""

		switch version {
		case 1:
			return "CREATE TABLE cars(id INTEGER PRIMARY KEY, name TEXT, price INTEGER)"
		case 2:
			return "CREATE TABLE person(id INTEGER PRIMARY KEY, name TEXT, address TEXT)"
		default:
			break
		}

		return SQL
	},
	willMigrateToSchemaVersion: { version in
		print("Will migrate to schema version: \(version)")
	},
	didMigrateToSchemaVersion: { version in
		print("Did migrate to schema version: \(version)")
	}
)
```

All migrations must start at 1 and increment by 1 with each iteration. For example, the first time you create a `Migrator`, you want to set the `desiredSchemaVersion` to 1 and implement the `migrationSQLForSchemaVersion` closure to return your initial database schema SQL. Then, each time you need to migrate your database, bump the `desiredSchemaVersion` by 1 and add the new case to your `migrationSQLForSchemaVersion` schema closure. In a production application, it would be easiest to write actual SQL files, add them to your bundle and load the SQL string from the file for the required version.

### Encryption

Encrypting a database can help safeguard vulnerabilities and keep your data safe. SQift uses [SQLCipher](https://www.zetetic.net/sqlcipher/design/) to encrypt all data in a database using an encryption passphrase. SQLCipher will lazily convert the passphrase to a key using [PBKDF2 key derivation](http://en.wikipedia.org/wiki/PBKDF2). Once the database has been encrypted, the only way to access the data inside the database is to set the encryption passphrase before accessing the database through a `Connection`.

#### Encrypting a Database Connection

Encrypting a database is simple.

```swift
let passphrase = "1234567890"

let connection = try Connection(connectionType: .OnDisk("path_to_db"))
try connection.setEncryptionPassphrase(passphrase)
```

> Make sure to set the encryption passphrase immediately after opening the `Connection`.

#### Updating the Encryption Passphrase

Updating the encryption passphrase of a database can be done after the current encryption passphrase has been set. Otherwise you are essentially attempting to change the passphrase without first confirming you should have access to do so.

```swift
let currentPassphrase = "1234567890"
let newPassphrase = "ABCDEFGHIJKLMNOP"

let connection = try Connection(connectionType: .OnDisk("path_to_db"))
try connection.setEncryptionPassphrase(currentPassphrase)

try connection.updateEncryptionPassphrase(newPassphrase)
```

#### Exporting an Decrypted Form of an Encrypted Database

In the event that you need to inspect a decrypted form of an encrypted database, you can do so by using the `exportDecryptedDatabaseToPath()` method in SQift.

```swift
let passphrase = "1234567890"

let connection = try Connection(connectionType: .OnDisk("path_to_db"))
try connection.setEncryptionPassphrase(currentPassphrase)

try connection.exportDecryptedDatabaseToPath("path_to_decrypted_db")
```

For more information about exporting encrypted or decrypted forms of a database, please refer to the SQLCipher [export documentation](https://www.zetetic.net/sqlcipher/sqlcipher-api/index.html#sqlcipher_export).

#### Inspecting Encrypted Database

So now you've encrypted your database and you're super safe, but how do you inspect your newly encrypted super safe database? The `sqlite3` command line tool does not support encryption. Neither do the widely popular [SQLPro for SQLite](https://itunes.apple.com/us/app/sqlpro-for-sqlite-sql-database/id586001240?mt=12) or [Base - SQLite Editor](https://itunes.apple.com/us/app/base-sqlite-editor/id402383384?mt=12) apps. While this is unfortunate, you still have several options.

First off, you should install the [sqlcipher](http://brewformulas.org/Sqlcipher) Homebrew package. It allows you to inspect encrypted databases using the `sqlcipher` command line tool. Here are some steps to get you going.

```bash
# Install Homebrew
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"

# Update Homebrew
brew update
brew doctor

# Install sqlcipher package
brew install sqlcipher

# Start using sqlcipher
$ sqlcipher encrypted_stuff.db
sqlite> PRAGMA key = 'passphrase';
sqlite> CREATE TABLE agents(id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT NOT NULL);
sqlite> SELECT count(*) FROM sqlite_master;
```

If you prefer graphical interfaces, you could install [SQLiteManager 4](https://itunes.apple.com/us/app/sqlitemanager-4/id604707021?mt=12). While it's not quite as nice as `SQLPro` or `Base`, it's still a fairly solid application. Also, you can always use the `sqlcipher` command line tool to export a decrypted form of the database if you really wanted to a different application.

> You could also write a simple bash script to wrap `sqlcipher` to export the decrypted database by passing in the passphrase.

---

## Roadmap

- Get new podspec published for SQLCipher 3.3.1 - [Open Issue](https://github.com/sqlcipher/sqlcipher/issues/141)
- Add Full-Text Search Support (FST4)
- Create a full DSL leveraging property and method chaining similar to SnapKit
  - The goal here would be to eliminate the need to ever write a single line of SQL

---

## FAQ

### Why not use CoreData?

There are many trade-offs between CoreData and SQift. SQift was certainly not created as a replacement for CoreData. It was created to make working with SQLite from Swift as easy and painless as possible. Anyone trying to decide between using CoreData and using SQift needs to consider the pros and cons carefully before making a decision. Both have significant learning curves and require significant amounts of forethought and architectural design before being integrated to an application or framework.

### Why not use FMDB?

SQift is designed from the start to support the latest Swift features and syntax to make the library as easy to use as possible. While FMDB is a fantastic library, it was originally designed for Objective-C making it less desirable from an API standpoint. It also has issues being embedded within a third-party Swift framework as a dependency with how the project is currently structured.

### Why not use any of the other open-source Swift libraries already out there?

There are many Swift SQLite libraries current on GitHub. Unfortunatey, none of them are production ready with the exception of SQLite.swift. SQLite.swift is a very well designed library that is fully featured, but has several drawbacks.

- The full DSL has serious performance problems converting the Swift types to SQL.
- The entire library is locked to serial execution on a single dispatch queue. Write-Ahead Logging (WAL) journal modes are not supported.
- Makes heavy use of functional operators in the DSL making it difficult to understand exactly what APIs are being called.

## Creators

- [Dave Camp](http://mobile-stash.nike.com:7990/users/dcam15) ([@thinbits](https://twitter.com/thinbits))
- [Christian Noon](http://mobile-stash.nike.com:7990/users/cnoon) ([@Christian_Noon](https://twitter.com/Christian_Noon))
