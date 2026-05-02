---
---
# PostgreSQL

Install PostgreSQL (an open-source relational database).

## Install

Install, then the package will take care of everything.

```bash
sudo apt install postgresql
```

Now, it's ready to use.

- The Debian package is PostgreSQL 17. If you need the latest version, use the PostgreSQL Apt Repository.  
  <https://www.postgresql.org/download/linux/debian/>

## Interface for PHP/Ruby/Python

Interface modules to connect to PostgreSQL. For example, CMS (mainly by PHP) will require php-pgsql interface.

```bash
sudo apt install php-pgsql ruby-pg python3-psycopg2
```
