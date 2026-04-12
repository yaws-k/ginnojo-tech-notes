---
---
# Redis

{% include version_warning.html %}

Redis is a key-value database. Some applications, such as Rspamd, require Redis for data storage.

## Install

The debian package is simply 'redis'.

```console
sudo apt install redis
```

If Redis will handle a large amount of data, consider changing the OS overcommit config. This warning will show up in the log file.

Add the following line to `/etc/sysctl.conf`

```conf
vm.overcommit_memory = 1
```

Then restart the server.

## Multiple instances

Setting up a dedicated Redis instance for each service or application is often recommended.  
By default, there is one `redis-server.service` daemon listening port 6379, so copy configuration files to start another.

### Config file

Create a config template to include.

```console
sudo cp /etc/redis/redis.conf /etc/redis/redis-template.conf
```

Then update the template.

Set `port 0` to stop listening TCP socket as default.

```conf
# Accept connections on the specified port, default is 6379 (IANA #815344).
# If port 0 is specified Redis will not listen on a TCP socket.
port 0
```

Create a new config `/etc/redis/redis-new.conf` for the new instance. (Use a better name in the actual situation.)

```conf
# Incluede template file as default
include /etc/redis/redis-template.conf

pidfile /run/redis/redis-server-new.pid
logfile /var/log/redis/redis-server-new.log

# Specify TCP port
port 6378

dbfilename dump-new.rdb
```

Change ownership of newly created config files to `redis:redis`.

```console
sudo chown redis:redis /etc/redis/*
```

### systemd config

Add a service file for new instance in `/etc/systemd/system/`.  
Copy the existing file.

```console
cd /lib/systemd/system
sudo cp ./redis-server.service ./redis-server-new.service
```

Update the lines as below.

```conf
Description=Advanced key-value store on port 6378
ExecStart=/usr/bin/redis-server /etc/redis/redis-new.conf --supervised systemd --daemonize no
PIDFile=/var/run/redis/redis-server-new.pid
Alias=redis-new.service
```

Enable and start the service.

```console
sudo systemctl enable redis-server-new
sudo systemctl daemon-reload
sudo systemctl start redis-server-new
```