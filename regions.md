# Adding Regions With PostgreSQL and Redis

Your application's performance may be improved by running it in multiple regions. This guide will walk you through the process of adding regions to a Fly.io application and configuring a PostgreSQL database and Redis cache to work across multiple regions. 

## Adding Regions

Fly.io runs applications from datacenters located throughout the world. These datacenters are identified by region. Fly.io will automatically route traffic to the region closest to the user. 

To add a region, you'll need its three-letter Region ID. The list of regions can be found [here](https://fly.io/docs/reference/regions/). This demonstration will walk you through adding the region `nrt` (Tokyo, Japan).

1. First, run `fly regions add nrt` to add the region to your application's region pool. Nearby regions may also be added as a backup.
2. Next, run `fly scale count 2` to add a new VM. You'll need one VM per region.
3. Finally, run `fly status` to see your VMs.

## PostgreSQL

It's straightforward to create a multi-region PostgreSQL cluster on Fly.io. This guide will walk you through the steps to create a PostgreSQL cluster in one region and a read replica in another. For the purposes of demonstration, we'll use the regions `sea` and `nrt`. 

Please note that this guide assumes that the application is structured so that only some requests include writes. Furthermore, it assumes that HTTP requests are the primary method of interacting with the database. If your application writes to the database on every request or makes heavy use of long-lived connections like websockets, please see additional guidance [here](https://fly.io/docs/getting-started/multi-region-databases/#this-is-wrong-for-some-apps).

### Creating A Cluster

There are a few steps to get a multi-region PostgreSQL cluster up and running.

1. First, run `fly pg create --name example-postgres --region sea` to create a PostgreSQL cluster. Note that this creates two databases, a write primary and a read replica. 
2. Second, run `fly volumes create pg_data -a example-postgres --size 10 --region nrt` to create a read replica in the `nrt` region.
3. Next, run `fly scale count 3 -a example-postgres` to add new VMs. You'll need one VM per database.
4. At this point, run `fly status -a example-postgres` to see your database cluster.
5. Finally, run `fly pg attach --example-app example-postgres` to attach your cluster to your application. This creates a `DATABASE_URL` secret in your application and makes it available as an environment variable.

At this point, the database cluster is running! However, in order to get all of the performance benefits of multi-region PostgreSQL, you may need to configure the connection string in your application logic. 

### Configuring the Connection String

By default, the generated connection string uses port `5432` to connect to PostgreSQL. This port always forwards to a writable instance. Thus, you'll need to update your application logic to connect on port `5433` in regions which are not the primary.

The basic logic to connect is:

1. Set a `PRIMARY_REGION` environment variable in your `fly.toml` file noting the region where the write leader is located.

```toml
// fly.toml

//...
[env]
  PRIMARY_REGION = "sea"
//...
```

2. Check the `FLY_REGION` environment variable at connect time. 
    A. When `FLY_REGION` is the primary region, use `DATABASE_URL` as-is. 
    B. When `FLY_REGION` is in another region, modify the `DATABASE_URL` as follows:
        a. Change the hostname to `<region>.example-postgres.internal`
        b. Change the port to `5433`

This is an example of the logic implemented in Ruby:

```ruby
class Fly
  def self.database_url
    primary = ENV["PRIMARY_REGION"]
    current = ENV["FLY_REGION"]
    db_url = ENV["DATABASE_URL"]

    if primary.blank? || current.blank? || primary == current
      return db_url
    end

    u = URI.parse(db_url)
    u.hostname = "#{current}.#{u.hostname}"
    u.port = 5433

    return u.to_s
  end
end
```

For advanced information on working with multi-region PostgreSQL, please consult [this guide](https://fly.io/docs/getting-started/multi-region-databases/).

## Configuring Redis

Adding Redis to your application is a straightforward procedure. This guide will walk you through creating a Redis cluster, once again using `sea` and `nrt`.

Before we begin, we'll need to create a couple files and update our `fly.toml` file. The first file we need to create is a `start-redis-server.sh` script. The Fly.io engineering team has created a sample one:

```sh
#!/bin/sh
sysctl vm.overcommit_memory=1
sysctl net.core.somaxconn=1024

# write region to local redis
nohup /bin/sh -c "sleep 10 && redis-cli -a $REDIS_PASSWORD set fly_region $FLY_REGION" >> /dev/null 2>&1 &

if [ "$PRIMARY_REGION" = "$FLY_REGION" ]; then
    # start master
    echo "Starting primary in $PFLY_REGION"
    redis-server /usr/local/etc/redis/redis.conf \
        --requirepass $REDIS_PASSWORD
else
    # start replica
    echo "Starting replica: $FLY_REGION <- $PRIMARY_REGION"

    # redis is dumb and can't replicate from an ipv6 only hostname
    ip=$(dig aaaa $PRIMARY_REGION.$FLY_APP_NAME.internal +short | head -n 1)

    echo "" >> /usr/local/etc/redis/redis.conf
    echo "replicaof $ip 6379" >> /usr/local/etc/redis/redis.conf
    redis-server /usr/local/etc/redis/redis.conf \
        --requirepass $REDIS_PASSWORD \
        --masterauth $REDIS_PASSWORD
fi
```

We also need to create `redis.conf` with the following:

```conf
# write to /data/
dir /data/

# snapshot settings
save 900 1
save 300 10
save 60 10000

# allow writes to replicas
replica-read-only no

# never become master
replica-priority 0

# replica should come after this
```

And finally, we need to update our `fly.toml` file. Remember that `sea` is the primary region we're using for the purposes of this demonstration: if your primary is located in a different region, you'll have to update the value.

```toml
[experimental]
  auto_rollback = false

[env]
  PRIMARY_REGION = "sea"

[mount]
  destination = "/data"
  source = "redis_server"
```

With that setup out of the way, we're ready to create our cluster!

### Creating a Cluster

There are only a few steps necessary to get a multi-region Redis cluster up and running:

1. First, run `fly volumes create redis_server --size 10 --region sea` to create the primary volume.
2. Next, run `fly volumes create redis_server --size 10 --region nrt` to create the read replica. 
3. Finally, run `fly scale count 2` to tell Fly.io to add more machines. You'll need one machine per region, so if you're adding multiple regions, your `count` may be higher.

The Redis cluster is now ready to use! However, as with PostgreSQL, we'll need to add some logic so that writes which need to propagate to all regions are sent to the primary, while writes with ephemeral data are sent to the nearest replica.

### Configuring Multi-Region Redis

The primary Redis URL is:

```conf
# format
redis://x:<password>@<primary-region>.<appname>.internal:6379

# example in sea
redis://x:password@sea.example-app.internal:6379
```
To generate the URL for a local read replica in Ruby, you might do something like:

```ruby
class Fly
  def self.local_redis_url
    primary = ENV["PRIMARY_REGION"]
    current = ENV["FLY_REGION"]
    redis_url = ENV["FLY_REDIS_CACHE_URL"]

    if primary.blank? || current.blank? || primary == current
      return redis_url
    end

    u = URI.parse(redis_url)
    u.hostname = "#{current}.example-app.internal"

    return u.to_s
  end
end
```
You can find additional guidance on configuring and using Redis [here](https://fly.io/docs/reference/redis/).

