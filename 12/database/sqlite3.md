---
---
# SQLite3

{% include version_warning.html %}

SQLite3 is a small database for light usage.

## Install

```console
sudo apt install sqlite3 libsqlite3-dev
```

No configuration is required.
SQLite3 is a file-based database. Unlike large-scale DBMS, backup/restore is very easy.

- Full backup: copy sqlite file
- Restore: replace sqlite file

## Interface for PHP/Ruby

PHP and Ruby require interfaces to use SQLite3. Python3 includes SQLite3 interface by default.

```console
sudo apt install php-sqlite3 ruby-sqlite3
```