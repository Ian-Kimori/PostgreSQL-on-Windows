# PostgreSQL-on-Windows

Run everything in **PowerShell as Administrator**. Replace `18` with a newer version if one shows up in step 1.

**1. Find the package**

```powershell
# Broad: everything PostgreSQL-related (server, pgAdmin, drivers, clients)
winget search PostgreSQL

# Narrow: only the official server packages, pick the highest version
winget search PostgreSQL.PostgreSQL

# Exact: confirm the one ID you're about to install
winget search --id PostgreSQL.PostgreSQL.18 --exact
```

**2. Install**

```powershell
winget install --id PostgreSQL.PostgreSQL.18 --exact --override "--mode unattended --unattendedmodeui minimal --superpassword YourStrongPass --serverport 5432"
```

This installs the server, pgAdmin 4 and the command-line tools, sets your `postgres` password and registers the Windows service.

**3. Add the tools to PATH**

```powershell
[Environment]::SetEnvironmentVariable("Path", [Environment]::GetEnvironmentVariable("Path", "Machine") + ";C:\Program Files\PostgreSQL\18\bin", "Machine")
```

Close PowerShell and open a new window.

**4. Check the service**

```powershell
Get-Service postgresql*
```

It should show `postgresql-x64-18` as `Running`. If it doesn't:

```powershell
Start-Service postgresql-x64-18
Set-Service postgresql-x64-18 -StartupType Automatic
```

**5. Test the connection**

```powershell
psql --version
psql -U postgres -h localhost -c "SELECT version();"
```

Enter `YourStrongPass` when it asks for a password.

**6. Create a working user and database (optional but recommended)**

```powershell
psql -U postgres -h localhost -c "CREATE ROLE lacan WITH LOGIN PASSWORD 'AnotherStrongPass' CREATEDB;"
psql -U postgres -h localhost -c "CREATE DATABASE labdb OWNER lacan;"
psql -U lacan -h localhost -d labdb
```

This way you don't use the superuser for everyday work.

**7. Managing the service later**

```powershell
Stop-Service postgresql-x64-18
Start-Service postgresql-x64-18
Restart-Service postgresql-x64-18
```

**8. Allow connections from other machines (only if you need it)**

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

**9. Upgrading or uninstalling later**

```powershell
winget upgrade --id PostgreSQL.PostgreSQL.18 --exact
winget uninstall --id PostgreSQL.PostgreSQL.18 --exact
```

`upgrade` only applies minor updates (18.x → 18.y). Moving to a new major version (18 → 19) means installing the new package and migrating your data with `pg_upgrade` or with `pg_dump`/`pg_restore`. Uninstalling leaves the `data` folder in place, so delete it by hand if you want a clean removal.

**Troubleshooting**

| Problem | Fix |
|---|---|
| `psql` not recognized | PATH wasn't picked up. Open a new terminal, or check that the `bin` folder exists. |
| Install hangs or fails | Run it again with `--verbose-logs` added, or remove `--override` to get the normal wizard. |
| Port 5432 already in use | Run `netstat -ano \| findstr 5432` to find what's using it, or install with `--serverport 5433`. |
| Strange characters in the psql console | Run `chcp 1252` first, or use Windows Terminal. |
| Forgot the `postgres` password | In `pg_hba.conf`, temporarily set the localhost lines to `trust`, restart, run `ALTER USER postgres PASSWORD '...';`, then change them back to `scram-sha-256` and restart. |
