---
---
# MongoDB

{% include version_warning.html %}

MongoDB is a document DB. It has both commercial and free (community edition) licenses.  
There is no official Debian package for MongoDB. [The official document](https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-debian/) explains how to install the current (as of July 2024, 7.0) version of the community edition.

## Preparation

Import the gpg key.

```console
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor
```

Add MongoDB apt-line.

```console
echo "deb [ signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] http://repo.mongodb.org/apt/debian bookworm/mongodb-org/7.0 main" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
```

Update apt sources.

```console
sudo apt update
```

## Install

Now apt-line is ready.

```console
sudo apt install mongodb-org
```

MongoDB engine (WiredTiger) strongly recommends the XFS filesystem.  
In my case, XFS partition is made during the Debian installation and mounted on `/var/xfs`. So I'll change the data directory from default `/var/lib/mongodb` to `/var/xfs/mongodb`.

Make a new MongoDB data directory and change the ownership.

```console
cd /var/xfs
sudo mkdir mongodb
sudo chown mongodb:mongodb mongodb
```

Change `/etc/mongod.conf` to let MongoDB use the new directory.

```config
# Where and how to store data.
storage:
  dbPath: /var/xfs/mongodb
```

- The `storage.journal.enabled` option is not available from MongoDB 6.1 because it's enabled by default. See [Journaling](https://www.mongodb.com/docs/manual/core/journaling/) for more details.

Reload daemons, enable Mongod and start.

```console
sudo systemctl daemon-reload
sudo systemctl enable mongod
sudo systemctl start mongod
```

## Security configuration

By default, MongoDB accepts access only from localhost, but anybody with access can read all databases.  
It is strongly recommended to enable authentication.

### Add an admin user

Add a superuser (admin/root) before enabling authentication.  
For example, a user "mongo" with the password "password".

```console
$ mongosh
(snip)
------
   The server generated these startup warnings when booting
   (snip) Access control is not enabled for the database. Read and write access to data and configuration is unrestricted
------
test> use admin
switched to db admin
admin> db.createUser({user:"mongo", pwd:"password", roles:["root"]})
{ ok: 1 }
admin> exit
```

## Enable authentication

Enable user authentication. Change security settings in `/etc/mongod.conf`.

```config
security
```

If you need to connect from servers other than localhost, add IP addresses separated by commas.

```config
net:
  port: 27017
  bindIp: 127.0.0.1, 192.168.10.1
```

Restart MongoDB.

```console
sudo systemctl restart mongod
```

## Add a normal user

Add a normal user with read/write privileges to the specific database.  
For example, "user01" to "doc01" collection with "readWrite" role.

```console
$ mongosh
test> use admin
switched to db admin
admin> db.auth("mongo", "password")
{ ok: 1 }
admin> use doc01
switched to db doc01
doc01> db.createUser({user: "user01", pwd: "password", roles: [{role: "readWrite", db: "doc01"}] })
{ ok: 1 }
doc01> exit
```

- `use doc01` automatically makes a new DB "doc 01" unless it exists