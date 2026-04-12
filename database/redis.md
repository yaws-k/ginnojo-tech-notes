---
---
# Redis

Redis is a key-value database. Some applications, such as Rspamd, require Redis for data storage.

## Install

The package 'redis' will install required dependencies.

```console
sudo apt install redis
```

The Redis log `/var/log/redis/redis-server.log` will show the following warning if the OS overcommit config is not changed.

> \# WARNING Memory overcommit must be enabled! Without it, a background save or replication may fail under low memory condition. Being disabled, it can also cause failures without low memory condition, see <https://github.com/jemalloc/jemalloc/issues/1328>. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.

If the server is mainly for Redis, change the setting accordingly.

- In my case, I ignored the warning because the Redis workload is very low and the server provides multiple services.
- For more details, check [the official Redis documentation](https://redis.io/docs/latest/operate/oss_and_stack/management/admin/).

In case you want to change the setting, follow the instructions below.

`/etc/sysctl.conf` does not exist by default, so create a config file `/etc/sysctl.d/redis.conf` and add the following line.

```conf
vm.overcommit_memory = 1
```

Then change the setting immediately.

```console
sudo sysctl -p /etc/sysctl.d/redis.conf
```

## Multiple instances

Setting up a dedicated Redis instance for each service or application is recommended.  
By default, there is one `redis-server.service` daemon listening port 6379, so copy configuration files to start another when needed.

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

Create a new config, e.g. for RSpamd, `/etc/redis/redis-rspamd.conf` for the new instance.

```conf
# Include template file as default
include /etc/redis/redis-template.conf

pidfile /run/redis/redis-server-rspamd.pid
logfile /var/log/redis/redis-server-rspamd.log

# Specify TCP port
port 6380

dbfilename dump-rspamd.rdb
```

Change ownership of newly created config files to `redis:redis`.

```console
sudo chown redis:redis /etc/redis/*
```

### systemd config

Add a service file for new instance.

```console
sudo systemctl enable redis-server@rspamd
sudo systemctl start redis-server@rspamd
```

Check the new instance status.

```console
sudo systemctl status redis-server@rspamd
```
