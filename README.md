# PostgreSQL-on-Windows

Run everything in **PowerShell as Administrator**.

**0. Remove the current install first (you already have one)**

The installer won't change the password on a data folder that already exists, so start clean:

```powershell
winget uninstall --id PostgreSQL.PostgreSQL.18 --exact
Remove-Item "C:\Program Files\PostgreSQL\18" -Recurse -Force
```

This deletes any databases in that install. Skip this step if you'd rather keep them and fix the password with the `pg_hba.conf` reset instead.

**1. Find the package**

```powershell
# Broad: everything PostgreSQL-related (server, pgAdmin, drivers, clients)
winget search PostgreSQL

# Narrow: only the official server packages, pick the highest version
winget search PostgreSQL.PostgreSQL

# Exact: confirm the one ID you're about to install
winget search --id PostgreSQL.PostgreSQL.18 --exact
```

**2. Check that it's the EDB installer**

```powershell
winget show --id PostgreSQL.PostgreSQL.18 --exact
```

Look at the **Installer Url** line. It should point to an `enterprisedb.com` download of a file like `postgresql-18.x-x-windows-x64.exe`. This is the same installer postgresql.org links to, and it's why the `--override` flags in the next step work: `--mode`, `--superpassword` and `--serverport` are options of the EDB installer, not of winget.

**3. Install**

```powershell
winget install --id PostgreSQL.PostgreSQL.18 --exact --override '--mode unattended --unattendedmodeui minimal --superpassword YourStrongPass --serverport 5432'
```

Single quotes make PowerShell pass the string exactly as written, so `$`, `` ` `` and `"` in the password survive. The password just can't contain a single quote (`'`).

**4. Add the tools to PATH**

```powershell
[Environment]::SetEnvironmentVariable("Path", [Environment]::GetEnvironmentVariable("Path", "Machine") + ";C:\Program Files\PostgreSQL\18\bin", "Machine")
```

If PATH already has this folder from the earlier install, skip this step. Close PowerShell and open a new window.

**5. Check the service**

```powershell
Get-Service postgresql*
```

It should show `postgresql-x64-18` as `Running`. If it doesn't:

```powershell
Start-Service postgresql-x64-18
Set-Service postgresql-x64-18 -StartupType Automatic
```

**6. Test the connection**

```powershell
psql --version
psql -U postgres -h localhost -c "SELECT version();"
```

Enter `YourStrongPass` when it asks for a password.

**7. Create a working user and database (optional but recommended)**

```powershell
psql -U postgres -h localhost -c "CREATE ROLE lacan WITH LOGIN PASSWORD 'AnotherStrongPass' CREATEDB;"
psql -U postgres -h localhost -c "CREATE DATABASE labdb OWNER lacan;"
psql -U lacan -h localhost -d labdb
```

**8. Managing the service later**

```powershell
Stop-Service postgresql-x64-18
Start-Service postgresql-x64-18
Restart-Service postgresql-x64-18
```

**9. Allow connections from other machines (only if you need it)**

1. Edit `C:\Program Files\PostgreSQL\18\data\postgresql.conf` and set:
   ```
   listen_addresses = '*'
   ```
2. Add this line to the end of `C:\Program Files\PostgreSQL\18\data\pg_hba.conf`, using your own subnet:
   ```
   host    all    all    192.168.1.0/24    scram-sha-256
   ```
3. Open the firewall port and restart the service:
   ```powershell
   New-NetFirewallRule -DisplayName "PostgreSQL 5432" -Direction Inbound -Protocol TCP -LocalPort 5432 -Action Allow
   Restart-Service postgresql-x64-18
   ```

**10. Upgrading or uninstalling later**

```powershell
winget upgrade --id PostgreSQL.PostgreSQL.18 --exact
winget uninstall --id PostgreSQL.PostgreSQL.18 --exact
```

`upgrade` only applies minor updates (18.x → 18.y). Moving to a new major version means installing the new package and migrating your data with `pg_upgrade` or with `pg_dump`/`pg_restore`. Uninstalling leaves the `data` folder in place, so delete it by hand if you want a clean removal.

**Troubleshooting**

| Problem | Fix |
|---|---|
| `password authentication failed` | The `data` folder from an earlier install was reused. Do step 0, or reset the password using `trust` in `pg_hba.conf` and then `\password postgres`. |
| `psql` not recognized | PATH wasn't picked up. Open a new terminal, or check that the `bin` folder exists. |
| Install hangs or fails | Run it again with `--verbose-logs` added, or remove `--override` to get the normal wizard. |
| Port 5432 already in use | Run `netstat -ano \| findstr 5432` to find what's using it, or install with `--serverport 5433`. |
| Strange characters in the psql console | Run `chcp 1252` first, or use Windows Terminal. |
