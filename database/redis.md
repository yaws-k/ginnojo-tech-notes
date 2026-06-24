---
---
# Redis

Redis is a key-value database. Some applications, such as Rspamd, require Redis for data storage.

## Install

The package 'redis' will install required dependencies.

```bash
sudo apt install redis
```

Now a Redis instance is running and listening on port 6379.

### Memory overcommit

The Redis log `/var/log/redis/redis-server.log` will show the following warning.

\# WARNING Memory overcommit must be enabled! Without it, a background save or replication may fail under low memory condition. Being disabled, it can also cause failures without low memory condition, see <https://github.com/jemalloc/jemalloc/issues/1328>. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
{: .notice--info}

If the server is mainly for Redis, change the setting accordingly.

- In my case, I ignored the warning because the Redis workload is very low and the server provides multiple services (Other DB, Web, Mail...).
- For more details, check [the official Redis documentation](https://redis.io/docs/latest/operate/oss_and_stack/management/admin/).

In case you want to change the setting, follow the instructions below.

`/etc/sysctl.conf` does not exist on Debian 13, so create a config file `/etc/sysctl.d/redis.conf` and add the following line.

```conf
vm.overcommit_memory = 1
```

Activate the setting immediately.

```bash
sudo sysctl -p /etc/sysctl.d/redis.conf
```

## Multiple instances

Instead of using multiple logical databases within a single instance, run a dedicated Redis instance for each service or application.  
To run multiple instances, stop the default `redis-server.service` and run dedicated ones based on the default configuration.

### Stop and disable the default instance

Stop the default instance and use the default configuration as a template.

```bash
sudo systemctl stop redis-server
sudo systemctl disable redis-server
```

### Update configuration to use it as a template

Update `/etc/redis/redis.conf` to stop listening on the TCP socket by default.

```conf
# Accept connections on the specified port, default is 6379 (IANA #815344).
# If port 0 is specified Redis will not listen on a TCP socket.
port 0
```

### Create a new configuration for each instance

Create a new configuration file for the new instance, such as `/etc/redis/redis-rspamd.conf` for Rspamd.

```conf
# Include template file as default
include /etc/redis/redis.conf

pidfile /run/redis/redis-server-rspamd.pid
logfile /var/log/redis/redis-server-rspamd.log

# Specify TCP port
port 6380

# Specify database dump file name
dbfilename dump-rspamd.rdb

# Limit memory usage
# - Max memory depends on the data and server memory size
# - Policy depends on the application (check the official documentation)
maxmemory 256mb
maxmemory-policy volatile-ttl
```

Change ownership of the newly created config file to `redis:redis`.

```bash
sudo chown redis:redis /etc/redis/redis-rspamd.conf
sudo chmod 640 /etc/redis/redis-rspamd.conf
```

### systemd config

Add a service file for the new instance.

```bash
sudo systemctl enable redis-server@rspamd
sudo systemctl start redis-server@rspamd
```

Check the new instance status.

```bash
sudo systemctl status redis-server@rspamd
```
