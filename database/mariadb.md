# MariaDB

Install MariaDB (an RDB forked from MySQL).

## Install

Install, then the package will take care of everything.

```console
# apt install mariadb-server
```

Now, MariaDB is working out of the box.

### MariaDB in Debian is secure by default

There is no need to run `mysql_secure_installation`. Check `/usr/share/doc/mariadb-server.README.Debian.gz` for more details.

## Interface for PHP/Ruby/Python

Interface modules to connect to MariaDB. For example, CMS (mainly by PHP) will require php-mysql interface.

```console
# apt install php-mysql ruby-mysql2 python3-mysqldb
```

These are Debian-packaged versions aligned with packaged versions of programming languages. If you have multiple versions, then install interfaces for each of them.
