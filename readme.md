# Metis - ETL tools for PHP
Metis gives you the ability to extract data from different sources, transform data and load to different destinations.

## Installation
### Require package
In your application's folder, run:
```
composer require marquine\metis
```

### Configuration
Global application configuration can be set using the `config` method. This configuration is optional.
```PHP
$config = [
    // If not provided, you can use the full path when working with files.
    'default_path' => '/etl',
];

Metis::config($config);
```

### Database Setup
Metis uses [doctrine/dbal](https://github.com/doctrine/dbal). Add connections with the `addConnection` method.

Adding the default connection:
```PHP
Metis::addConnection(['driver' => 'pdo_sqlite', 'database' => 'db.sqlite']);
```

Adding a named connection:
```PHP
Metis::addConnection(['driver' => 'pdo_sqlite', 'database' => 'db.sqlite'], 'connection_name');
```

### Laravel Integration
If you are using Laravel, you can skip 'Global Configuration' and 'Database Setup' steps. Metis provides a default configuration file and will register all connections of your application.

Add the ServiceProvider to the `providers` array in `config/app.php` file:
```PHP
Marquine\Metis\Providers\Laravel\MetisServiceProvider::class,
```

Publish the configuration file using the artisan command:
```
php artisan vendor:publish --provider="Marquine\Metis\Providers\Laravel\MetisServiceProvider"
```


## Usage

Create a new Metis instance and start chaining methods:
```PHP
$metis = new Metis;

$metis->extract(...)->transform(...)->load(...);
```

You can also call the `start` static method to create a new Metis instance:
```PHP
Metis::start()->extract(...)->transform(...)->load(...);
```

### Extracting
```PHP
$metis->extract($type, $source, $columns = null, $options = []);
```

#### Array
Get all array columns:
```PHP
$metis->extract('array', $array);
```
Get specific columns:
```PHP
$columns = ['id', 'name']; //columns names

$metis->extract('array', $array, $columns);
```
The Array extractor does not have any option.


#### CSV
Extract from a CSV file with columns header:
```PHP
$metis->extract('csv', '/path/to/file.csv');
```
Extract from a CSV file without columns header:
```PHP
$columns = [
    'id' => 1, // Index of the column. The first column is 1.
    'name' => 2,
];

$metis->extract('csv', '/path/to/file.csv', $columns);
```
Options:

| Name      | Type   | Default | Description |
| --------- |------- | ------- | ----------- |
| delimiter | string | ';'     | Delimiter that separates items. |
| enclosure | string | '"'     | The value enclosure. |


#### FixedWidth
Extract from a fixed width text file:
```PHP
$columns = [
    'id' => [0, 5], // Start position and length of column.
    'name' => [5, 40],
];

$metis->extract('fixedWidth', '/path/to/file.txt', $columns);
```
The FixedWidth extractor does not have any option.


#### Json
Extract from a Json file:
```PHP
$metis->extract('json', '/path/to/file.json');
```
Extract from a Json file with custom attributes path:
```PHP
$columns = [
    'id' => '$..bindings[*].id.value',
    'name' => '$..bindings[*].name.value',
];

$metis->extract('json', '/path/to/file.json', $columns);
```
The Json extractor does not have any option.

#### Query
Extract from a database table using a custom query:
```PHP
$query = 'SELECT * FROM users';

$metis->extract('query', $query);
```
Extract from a database table using a custom query and bindings:
```PHP
$query = 'SELECT * FROM users WHERE status = ?';
$bindings = ['active']

$metis->extract('query', $query, $bindings);
```
Options:

| Name       | Type   | Default   | Description |
| ---------- | ------ |---------- | ----------- |
| connection | string | 'default' | Name of the database connection to use. |


#### Table
Extract from a database table:
```PHP
$metis->extract('table', 'table_name');
```
Extract specific columns from a database table:
```PHP
$columns = ['id', 'nome'];

$metis->extract('table', 'table_name', $columns);
```
Options:

| Name       | Type   | Default   | Description |
| ---------- | ------ |---------- | ----------- |
| connection | string | 'default' | Name of the database connection to use. |
| where      | array  | []        | Array of where clause. For Example: ['status' => 'active']. |


#### XML
Extract from a XML file:
```PHP
$metis->extract('xml', '/path/to/file.xml');
```
Extract from a XML file with custom attributes and loop path:
```PHP
$columns = [
    'id' => 'id/value',
    'name' => 'name/value',
];
$options = ['loop' => '/users/user'];

$metis->extract('xml', '/path/to/file.xml', $columns, $options);
```
Options:

| Name | Type   | Default | Description |
| ---- | ------ |-------- | ----------- |
| loop | string | '/'     | The path to loop. |



### Transforming
```PHP
$metis->extract(...)->transform($type, $columns = null, $options = []);
```
#### Trim
Strip whitespace from the beginning and end of a string in all transformation columns:
```PHP
$metis->extract(...)->transform('trim');
```
Strip pipes from the beginning of a string in specific transformation columns:
```PHP
$columns = ['id', 'name'];
$options = ['type' => 'left', 'mask' => '|'];

$metis->extract(...)->transform('trim', $columns, $options);
```
Options:

| Name | Type   | Default              | Description |
| ---- | ------ |--------------------- | ----------- |
| type | string | 'both'               | The options `trim` &#124; `all` &#124; `both` will trim both sides, `ltrim` &#124; `start` &#124; `left` will trim the left side and `rtrim` &#124; `end` &#124; `right` will trim the right side of the string.  |
| mask | string | "&nbsp;\t\n\r\0\x0B" | The stripped characters. Simply list all characters that you want to be stripped. With .. you can specify a range of characters. |



### Loading
```PHP
$metis->extract(...)->transform(...)->load($type, $destination, $options = []);
```

#### Table
Load data to a database table:
```PHP
$metis->extract(...)->transform(...)->load('table', 'table_name');
```
Load data to a database table using timestamps and custom primary key:
```PHP
$options = [
    'timestamps' => true,
    'keys' => ['id', 'company_id']
];

$metis->extract(...)->transform(...)->load('table', 'table_name', $options);
```
Options:

| Name         | Type                 | Default   | Description |
| ------------- | ------------------- |---------- | ----------- |
| connection    | string              | 'default' | Name of the database connection.  |
| keys          | array&#124;string   | ['id']    | List of primary keys or identifiers of the table. |
| insert        | boolean             | true      | Insert rows that are in the source but not in the destination table. |
| update        | boolean             | true      | Update rows (based on primary_key option) that are in both source and destination and have new values from the source. |
| delete        | boolean&#124;string | false     | Delete rows that are in destination table but not in the source. If set to `soft`, the row will not be deleted and the column deleted_at will be set to the current timestamp. |
| skipDataCheck | boolean             | false     | Do not check table current data before `insert`, `update` and `delete` statements execution. |
| forceUpdate   | boolean             | false     | Do not check for differences between source and destination when updating. |
| timestamps    | boolean             | false     | Use `created_at` and `updated_at` columns when inserting or updating. |
| transaction   | boolean&#124;int    | 100       | Transaction size. Set to `false` to execute statements without transactions. |



### Utilities
```PHP
$metis->utility($type, $options);
```

#### Command
Execute a command:
```PHP
$options = [
    'command' => 'cp /path/to/file.csv /new/path/file.csv'
];

$metis->utility('command', $options);
```
Execute multiple commands:
```PHP
$options = [
    'command' => [
        'cp /path/to/file.csv /new/path/file.csv',
        'chmod 777 /new/path/file.csv'
    ]
];

$metis->utility('command', $options);
```
Options:

| Name    | Type              | Default | Description |
| ------- | ----------------- |-------- | ----------- |
| command | array&#124;string | null    | Command or array of commands to be executed. |
