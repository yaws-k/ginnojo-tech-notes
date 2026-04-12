---
---
# MariaDB

Install MariaDB (an RDB forked from MySQL).

## Install

Install, then the package will take care of everything.

```console
sudo apt install mariadb-server
```

Now, it's ready to use.

### MariaDB in Debian is secure by default

You may find articles recommending `mysql_secure_installation`, but there is no need to run it.  
Check `/usr/share/doc/mariadb-server/README.Debian.gz` for more details.

> There is absolutely no need to run the script `mariadb-secure-installation` or (`mysql_secure_installation`) after installing MariaDB with `apt install mariadb-server`. The script is useless, and very misleading with reporting
> "Success!" after every step even if it did nothing.

## Interface for PHP/Ruby/Python

Interface modules to connect to MariaDB. For example, CMS (mainly by PHP) will require php-mysql interface.

```console
sudo apt install php-mysql ruby-mysql2 python3-mysqldb
```
