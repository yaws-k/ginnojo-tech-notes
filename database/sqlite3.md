---
---
# SQLite3

SQLite3 is a lightweight database engine for simple use cases.

## Install

```bash
sudo apt install sqlite3
```

No configuration is required.
SQLite3 is a file-based database. Unlike large-scale DBMSs, backup and restore are very straightforward.

- Full backup: copy the SQLite file
- Restore: replace the SQLite file

## Interface for PHP/Ruby

PHP and Ruby require interfaces to use SQLite3. Python3 includes the SQLite3 interface by default.

```bash
sudo apt install php-sqlite3 ruby-sqlite3
```
