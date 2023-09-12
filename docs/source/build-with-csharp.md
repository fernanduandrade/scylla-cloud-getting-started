# Quick Start: Csharp

In this tutorial we're gonna build a simple Media Player to store our songs and build playlists

## 1. Setup the Environment

### 1.1 Downloading chsarp and .NET dependencies:

If you don't have csharp installed already on your machine, you can install from two possible sources:

1. [Ruby main website](https://www.ruby-lang.org/en/downloads/)
2. [Rbenv](https://github.com/rbenv/rbenv)

After installing the language, make sure to install the `bundler` library so we can manage our projects with:

```sh
gem install bundler
```

### 1.2 Starting the project

Now with the ruby and bundler gem installed, let's create a new project with the following command:

```sh
mkdir media_player && cd media_player && bundle init
```

### 1.3 Setting the project dependencies

First we'll install the required gem to connect to scyllaDB with the following command:

```sh
bundle add cassandra-driver
```

This gem can be found at [github](https://github.com/datastax/ruby-driver/)

> Disclaimer: This gem require system wide dependencies with the cassandra client, so it's required to install on your system (or run the whole application under a docker image). You can find the installation guide at: https://cassandra.apache.org/doc/latest/cassandra/getting_started/installing.html

A sample `Gemfile` will be like this:

```ruby
# frozen_string_literal: true

source 'https://rubygems.org'

gem 'cassandra-driver', '~> 3.2'
```

## 2. Connecting to the Cluster

Make sure to get the right credentials on your [ScyllaDB Cloud Dashboard](https://cloud.scylladb.com/clusters) in the tab `Connect`.

```ruby
# frozen_string_literal: true

require 'cassandra'

cluster = Cassandra.cluster(
  username: 'scylla',
  password: 'a-very-secure-password',
  hosts: [
    'node-0.aws-sa-east-1.xxx.clusters.scylla.cloud',
    'node-1.aws-sa-east-1.xxx.clusters.scylla.cloud',
    'node-2.aws-sa-east-1.xxx.clusters.scylla.cloud'
  ]
)
```

> If the connection got refused, check if your IP Address is added into allowed IPs.

## 3. Handling Queries

Using the `cassandra` gem you can instantiate a session and then run fully asynchronous queries.

```ruby
session  = cluster.connect

future = session.execute_async('SELECT address, port, connection_stage FROM system.clients LIMIT 5')
future.on_success do |rows|
  rows.each do |row|
    puts "IP -> #{row[:address]}, Port -> #{row[:port]}, CS -> #{row[:connection_stage]}"
  end
end

future.join
```

The output should look something like:

```
IP -> 170.244.28.189, Port -> 49096, CS -> AUTHENTICATING
```

### 3.1 Creating a Keyspace

The `keyspace` inside the ScyllaDB ecossystem can be interpreted as your `database` or `collection`.

On your connection boot, you don't need to provide it but you will use it later and also is able to create when you need.

```ruby
# frozen_string_literal: true

require 'cassandra'

cluster = Cassandra.cluster(
  username: 'scylla',
  password: 'a-strong-password',
  hosts: [
    'node-0.aws-us-east-1.first.clusters.scylla.cloud',
    'node-1.aws-us-east-1.second.clusters.scylla.cloud',
    'node-2.aws-us-east-1.third.clusters.scylla.cloud'
  ]
)

keyspace = 'media_player'

session = cluster.connect

# Verify if the Keyspace already exists in your Cluster
has_keyspace = session.execute_async('select keyspace_name from system_schema.keyspaces WHERE keyspace_name=?',
                                     arguments: [keyspace]).join.rows.size

if has_keyspace.zero?
  new_keyspace_query = <<~SQL
    CREATE KEYSPACE #{keyspace}
    WITH replication = {
      'class': 'NetworkTopologyStrategy',
      'replication_factor': '3'
    }
    AND durable_writes = true
  SQL

  session.execute_async(new_keyspace_query).join

  puts "Keyspace #{keyspace} created!"
else
  puts "Keyspace #{keyspace} already created!"
end

# Reconnecting to the cluster with the correct keyspace
session = cluster.connect(keyspace)
```

### 3.2 Creating a table

A table is used to store part or all the data of your app (depends on how you will build it). 
Remember to add your `keyspace` into your connection and let's create a table to store our liked songs.

```ruby
# frozen_string_literal: true

require 'cassandra'

cluster = Cassandra.cluster(
  username: 'scylla',
  password: 'a-strong-password',
  hosts: [
    'node-0.aws-us-east-1.first.clusters.scylla.cloud',
    'node-1.aws-us-east-1.second.clusters.scylla.cloud',
    'node-2.aws-us-east-1.third.clusters.scylla.cloud'
  ]
)

keyspace = 'media_player'
table = 'playlist'

session = cluster.connect(keyspace)

# Verify if the table already exists in the specific Keyspace inside your Cluster
has_table = session.execute_async('select keyspace_name, table_name from system_schema.tables where keyspace_name = ? AND table_name = ?', arguments: [keyspace, table]).join.rows.size

if has_table.zero?
  new_table_query = <<~SQL
    CREATE TABLE #{keyspace}.#{table} (
      id uuid,
      title text,
      album text,
      artist text,
      created_at timestamp
      PRIMARY KEY (id, created_at)
    )
  SQL

  session.execute_async(new_table_query).join

  puts "Table #{table} created!"
else
  puts "Table #{table} already created!"
end
```

### 3.3 Inserting data

Now that we have the keyspace and a table inside of it, we need to bring some good songs and populate it. 

```ruby
# frozen_string_literal: true

require 'cassandra'

cluster = Cassandra.cluster(
  username: 'scylla',
  password: 'a-strong-password',
  hosts: [
    'node-0.aws-us-east-1.first.clusters.scylla.cloud',
    'node-1.aws-us-east-1.second.clusters.scylla.cloud',
    'node-2.aws-us-east-1.third.clusters.scylla.cloud'
  ]
)

keyspace = 'media_player'
table = 'playlist'

session = cluster.connect(keyspace)

song_list = [
  {
    title: 'Bohemian Rhapsody',
    album: 'A Night at the Opera',
    artist: 'Queen',
    created_at: Time.now
  },
  {
    title: 'Hotel California',
    album: 'Hotel California',
    artist: 'Eagles',
    created_at: Time.now
  },
  {
    title: 'Smells Like Teen Spirit',
    album: 'Nevermind',
    artist: 'Nirvana',
    created_at: Time.now
  }
]

insert_query = "INSERT INTO #{table} (now(),title,album,artist,created_at) VALUES (?,?,?,?)"

song_list.each do |song|
  session.execute_async(insert_query,
                        arguments: [song[:title], song[:album], song[:artist],
                                    song[:created_at]]).join.rows.size
end
```

### 3.4 Reading data

Since probably we added more than 3 songs into our database, let's list it into our terminal.

```ruby
# frozen_string_literal: true

require 'cassandra'

cluster = Cassandra.cluster(
  username: 'scylla',
  password: 'a-strong-password',
  hosts: [
    'node-0.aws-us-east-1.first.clusters.scylla.cloud',
    'node-1.aws-us-east-1.second.clusters.scylla.cloud',
    'node-2.aws-us-east-1.third.clusters.scylla.cloud'
  ]
)

keyspace = 'media_player'
table = 'playlist'

session = cluster.connect(keyspace)

future = session.execute_async("SELECT id, title, album, artist, created_at FROM #{table}")
future.on_success do |rows|
  rows.each do |row|
    puts "ID: #{song['id']} | Song: #{song['title']} | Album: #{song['album']} | Created At: #{song['created_at']}"
  end
end

future.join
```

The result will look like:

```
ID: 40450211-42cc-11ee-b14c-3da98b5024c0 | Song: Smells Like Teen Spirit | Album: Nevermind | Created At: 2023-08-24 19:19:11 -0300
ID: 3ab84321-42cc-11ee-b14c-3da98b5024c0 | Song: Hotel California | Album: Hotel California | Created At: 2023-08-24 19:19:01 -0300
ID: 354f11c1-42cc-11ee-b14c-3da98b5024c0 | Song: Bohemian Rhapsody | Album: A Night at the Opera | Created At: 2023-08-24 19:18:52 -0300
```

### 3.5 Updating data

Ok, almost there! Now we're going to learn about update but here's a disclaimer: 
> INSERT and UPDATES are not equals!

There's a myth in Scylla/Cassandra community that it's the same for the fact that you just need the `Partition Key` and `Clustering Key` (if you have one) and query it.

If you want to read more about it, [click here.](https://docs.scylladb.com/stable/using-scylla/cdc/cdc-basic-operations.html)

As we can see, the `UPDATE QUERY` takes two fields on `WHERE` (PK and CK). Check the snippet below: 

```ruby
# frozen_string_literal: true

require 'cassandra'
require 'securerandom'

cluster = Cassandra.cluster(
  username: 'scylla',
  password: 'a-strong-password',
  hosts: [
    'node-0.aws-us-east-1.first.clusters.scylla.cloud',
    'node-1.aws-us-east-1.second.clusters.scylla.cloud',
    'node-2.aws-us-east-1.third.clusters.scylla.cloud'
  ]
)

keyspace = 'media_player'
table = 'playlist'

session = cluster.connect(keyspace)

song = {
    title: 'Smells Like Teen Spirit Updated',
    album: 'Nevermind Updaetd',
    artist: 'Nirvana Updated',
    created_at: Time.new('2023-08-24 22:19:11.091000+0000')
}

update_query = "UPDATE #{keyspace}.#{table} SET title = ?, album = ?, artist = ? where id = ? and created_at = ?"

session.execute_async(update_query, arguments: [song[:title], song[:album], song[:artist], '40450211-42cc-11ee-b14c-3da98b5024c0', song[:created_at]]).join

puts 'Song Updated'
```

After updated, let's query for the ID and see the results:

```
scylla@cqlsh> select * from media_player.playlist where id = 40450211-42cc-11ee-b14c-3da98b5024c0;

 id                                   | created_at                      | album             | artist          | title
--------------------------------------+---------------------------------+-------------------+-----------------+---------------------------------
 40450211-42cc-11ee-b14c-3da98b5024c0 | 2023-08-24 22:19:11.091000+0000 | Nevermind Updated | Nirvana Updated | Smells Like Teen Spirit Updated

(1 rows)
```

It only "updated" the field `title` and `updated_at` (that is our Clustering Key) and since we didn't inputted the rest of the data, it will not be replicated as expected.

### 3.5 Deleting data

Let's understand what we can DELETE with this statement. There's the normal `DELETE` statement that focus on `ROWS` and other one that delete data only from `COLUMNS` and the syntax is very similar.

```sql 
-- Deletes a single row
DELETE FROM songs WHERE id = d754f8d5-e037-4898-af75-44587b9cc424;

-- Deletes a whole column
DELETE artist FROM songs WHERE id = d754f8d5-e037-4898-af75-44587b9cc424;
```

If you want to erase a specific column, you also should pass as parameter the `Clustering Key` and be very specific in which register you want to delete something. 
On the other hand, the "normal delete" just need the `Partition Key` to handle it. Just remember: if you use the statement "DELETE FROM keyspace.table_name" it will delete ALL the rows that you stored with that ID. 

```ruby
# frozen_string_literal: true

require 'cassandra'
require 'securerandom'

cluster = Cassandra.cluster(
  username: 'scylla',
  password: 'a-strong-password',
  hosts: [
    'node-0.aws-us-east-1.first.clusters.scylla.cloud',
    'node-1.aws-us-east-1.second.clusters.scylla.cloud',
    'node-2.aws-us-east-1.third.clusters.scylla.cloud'
  ]
)

keyspace = 'media_player'
table = 'playlist'

session = cluster.connect(keyspace)

song = {
    title: 'Smells Like Teen Spirit Updated',
    album: 'Nevermind Updaetd',
    artist: 'Nirvana Updated',
    created_at: Time.new('2023-08-24 22:19:11.091000+0000')
}

delete_query = "DELETE FROM #{keyspace}.#{table} where id = ? and created_at = ?"

session.execute_async(delete_query, arguments: ['40450211-42cc-11ee-b14c-3da98b5024c0', song[:created_at]]).join

puts 'Song deleted!'
```

## Conclusion

Yay! You now have the knowledge to use the basics of ScyllaDB with Ruby.

If you thinks that something can be improved, please open an issue and let's make it happen!

Did you like the content? Dont forget to star the repo and follow us on socials.
