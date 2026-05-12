---
---
# Mail server migration

If Dovecot is working on both the old and new servers, `doveadm` works well. It migrates (backups) all emails, directories, and Sieve scripts.

Prerequisites (in this article's case)

1. Dovecot is running on both servers
2. SSH access as root is allowed in the new server
3. Email users are defined as virtual users on /etc/dovecot/users

This article explains only about doveadm commands. Many steps to consider for the mail server migration are too much to document here, so please find some other sites...
{: .notice--info}

It looks like `doveadm` command also had some breaking changes and doesn't work as expected when migrating from Dvecot 2.3 to 2.4. Here are some workarounds that I used, but not sure if it properly works on your environment.
{: .notice--warning}

## Preparation

### SSH as root

SSH root login should be prohibited, but enable it temporarily for this migration. Enable key-pair only, but don't allow ID/Password login.

Edit `/etc/ssh/sshd_config` to allow key-pair root login.

- `PermitRootLogin prohibit-password`
- `PubkeyAuthentication yes`
- `PasswordAuthentication no`

### User list

The new server user list must be ready before copying emails. Copy `/etc/dovecot/users` file from old to new.

## Email Migration

Steps

1. Set the new server as the secondary MX
   - All emails should go to the old server (primary MX)
2. Initial email copy from old to new server
3. Delete the old server from the MX and set the new server as the primary MX
   - Emails start going to the new server
4. Stop the old server mail service
   - Stop receiving emails even if other servers still try to send
5. Delta update from the old server to the new
   - Fetch emails received after the initial copy

### Initial migrate (copy)

The main process of data migration. `doveadm` command will copy all emails in all directories (including empty directories) and Sieve scripts.  
Be aware that any existing data on the new server will be deleted during this process.

Between compatible versions:

```console
# doveadm backup -u user1@example.jp remote:new-server.example.jp
```

From 2.3 to 2.4:

```console
# doveadm -o "dsync_remote_cmd=ssh -l root new-server.example.jp doveadm dsync-server -u user1@example.jp" backup -u user1@example.jp remote:new-server.example.jp
```

- It seems `doveadm backup` command sends `-U` option that is not available in Dovecot 2.4. So specify `dsync_remote_cmd` option to avoid this issue.

If you simply want to copy all emails for all users, you can use `-A` option.  
(It's only available between compatible versions because it loops commands with `-u` internally.)

```console
# doveadm backup -A remote:new-server.example.jp
```

### Delta update

To copy emails existing only on the old server, use `doveadm sync` command. With `-1` option, it only copies emails that are not on the new server.

Between compatible versions:

```console
# doveadm sync -1 -u user1@example.jp remote:new-server.example.jp
```

From 2.3 to 2.4:

```console
# doveadm -o "dsync_remote_cmd=ssh -l root new-server.example.jp doveadm dsync-server -u user1@example.jp" sync -1 -u user1@example.jp remote:new-server.example.jp
```

Technically this should work the same as `backup` command above, but [the official document](https://doc.dovecot.org/2.4.1/core/man/doveadm-backup.1.html) discourages such usage.
