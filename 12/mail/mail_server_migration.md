---
---
# Mail server migration

{% include version_warning.html %}

If Dovecot is working on both the old and new servers, `doveadm` works well. It migrates (backups) all emails, directories, and Sieve scripts.

Prerequisites (in this article's case)

1. Dovecot is running on both servers
2. SSH access as root is allowed in the new server
3. Email users are defined as virtual users on /etc/dovecot/users

This article explains only about doveadm commands. Many steps to consider for the mail server migration are too much to document here, so please find some other sites...

## Preparation

### User list

The new server user list must be ready before copying emails. Copy `/etc/dovecot/users` file from old to new.

### SSH as root

SSH root login should be prohibited, but enable it temporarily for this migration. Enable key-pair only, but don't allow ID/Password login.

(If root login is prohibited,) Edit `/etc/ssh/sshd_config` to allow key-pair root login.

- `PermitRootLogin prohibit-password`
- `PubkeyAuthentication yes`
- `PasswordAuthentication no`

## Migration

Steps

1. Set the new server as the secondary MX
   - All emails should go to the old server
2. Initial migrate (copy)
3. Delete the old server from the MX and set the new server as the primary MX
   - Emails start going to the new server
4. Stop the old server mail service
   - Stop receiving emails even if other servers still try to send
5. Delta update from the old server to the new
   - Fetch emails received after the initial copy

### Initial migrate (copy)

The main process of data migration. `doveadm` command will copy all emails in all directories (including empty directories) and Sieve scripts.  
Be aware that any existing data on the new server will be deleted during this process.

```console
sudo doveadm backup -u user1@example.jp remote:new-server.example.jp
```

As the user is specified with `-u` parameter, this migrates one user's data. If you want to migrate all user emails, you can use `-A` paramter to migrate all users listed in the userdb.

### Delta update

To copy emails existing only on the old server, use `doveadm sync` command.

```console
sudo doveadm sync -1 -u user1@example.jp remote:new-server.example.jp
```

Technically this should work the same as `backup` command above, but the official document discouraged such usage. (The official document page has gone, so maybe it is ok now...?)
