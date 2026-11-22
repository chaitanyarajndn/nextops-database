# NextOps Database Setup Guide

## Introduction

This repository contains the PostgreSQL database schema and initialization scripts for the **NextOps Industrial Telemetry and Condition Monitoring** platform.

The database is the central data store in the NextOps workflow:

1. Industrial telemetry is stored in the `telemetry` table.
2. Alarm information is stored in the `alarms` table.
3. The NextOps Query Service connects to PostgreSQL and reads this data.
4. The Query Service provides API responses to the NextOps React frontend.
5. The frontend displays dashboards, equipment status, trends, regions, and active alarms.

For example, sensor readings stored as rows in `telemetry` can be queried by the backend and displayed as device trends or plant summaries in the dashboard.

There are **two ways to set up PostgreSQL for NextOps**:
1. **IaaS (Infrastructure as a Service)**: Install and manage PostgreSQL yourself on a virtual machine.
2. **PaaS (Platform as a Service)**: Use **Azure Database for PostgreSQL - Flexible Server**, where Azure manages the database infrastructure.

---

## Repository Structure

- `01-extensions.sql`: Enables the `pgcrypto` PostgreSQL extension used to generate UUID values.
- `02-telemetry.sql`: Creates the `telemetry` and `alarms` tables and adds indexes for faster device, time, region, and alarm queries.

---

## Option 1: IaaS Setup (PostgreSQL on a Linux VM with Public IP)

This approach installs and manages PostgreSQL on a Linux virtual machine. The steps below use an Ubuntu or Debian VM with a public IP address.

### Step 1: Configure Network Security / Firewall
Configure the VM Network Security Group (NSG) or firewall to allow inbound traffic on the PostgreSQL port:
- **Port**: `5432`
- **Protocol**: `TCP`
- **Source**: Your local workstation public IP is recommended. Use `0.0.0.0/0` only if connections must be allowed from any public IP.

### Step 2: Install PostgreSQL on the VM
SSH into your Linux VM and run:

```bash
# ==========================================
# PostgreSQL 18 Installation on Ubuntu
# Official PostgreSQL (PGDG) Repository
# ==========================================

# 1. Update Ubuntu packages
sudo apt update && sudo apt upgrade -y

# 2. Install prerequisites
sudo apt install -y postgresql-common ca-certificates

# 3. Add official PostgreSQL repository
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh

# 4. Update package list after adding repository
sudo apt update

# 5. Install PostgreSQL 18 server and client
sudo apt install -y postgresql-18 postgresql-client-18

# 6. Enable PostgreSQL service at system boot
sudo systemctl enable postgresql

# 7. Start PostgreSQL service
sudo systemctl start postgresql

# 8. Check PostgreSQL service status
sudo systemctl status postgresql

# 9. Check installed PostgreSQL version
psql --version
```

### Step 3: Configure PostgreSQL for Remote Connections

By default, PostgreSQL listens only on `localhost` (127.0.0.1). Update the configuration so it can accept remote connections.

1. **Edit `postgresql.conf`**:
   ```bash
   sudo nano /etc/postgresql/*/main/postgresql.conf
   ```
   Find `listen_addresses` and change it to:
   ```text
   listen_addresses = '*'
   ```

2. **Edit `pg_hba.conf` to allow remote client authentication**:
   ```bash
   sudo nano /etc/postgresql/*/main/pg_hba.conf
   ```
   Scroll to the end of the file and add the following entry:
   ```text
   # Allow remote password connections from any IP (or restrict to your specific client IP)
   host    all             all             0.0.0.0/0               scram-sha-256
   ```

3. **Restart PostgreSQL to apply changes**:
   ```bash
   sudo systemctl restart postgresql
   ```

### Step 4: Set the `postgres` User Password
Set a password for the default PostgreSQL administrator account:

```bash
sudo -u postgres psql -c "ALTER USER postgres PASSWORD 'YourStrongPasswordHere';"
```

---

### Step 5: Connect via pgAdmin 4 (IaaS)

1. Launch **pgAdmin 4** on your local machine.
2. In the Browser tree, right-click **Servers** > **Register** > **Server...**.
3. Under the **General** tab:
   - **Name**: `NextOps-IaaS-VM`
4. Under the **Connection** tab:
   - **Host name/address**: `<VM_PUBLIC_IP>`
   - **Port**: `5432`
   - **Maintenance database**: `postgres`
   - **Username**: `postgres`
   - **Password**: `YourStrongPasswordHere` (Check *Save password* if desired)
5. Select **Save**.

---

### Step 6: Create Database & Execute SQL Scripts (IaaS)

1. In pgAdmin, expand the connected server `NextOps-IaaS-VM`.
2. Right-click **Databases** > **Create** > **Database...**.
3. In the dialog, enter **Database**: `nextops`, then select **Save**.
4. Expand **Databases**, right-click **nextops**, and select **Query Tool**.
5. **Run Script 1 (`01-extensions.sql`)**:
   - Open `01-extensions.sql`, or copy and paste the following SQL:
     ```sql
     CREATE EXTENSION IF NOT EXISTS pgcrypto;
     ```
   - Click **Execute / Run** (`F5`).
6. **Run Script 2 (`02-telemetry.sql`)**:
   - Open `02-telemetry.sql`, or paste the script content.
   - Click **Execute / Run** (`F5`).
7. Expand `nextops` > **Schemas** > **public** > **Tables** to verify that `telemetry` and `alarms` were created.

*(Optional)* You can also run these directly on the VM via command line:
```bash
sudo -u postgres psql -c "CREATE DATABASE nextops;"
sudo -u postgres psql -d nextops -f /path/to/01-extensions.sql
sudo -u postgres psql -d nextops -f /path/to/02-telemetry.sql
```

---

## Option 2: PaaS Setup (Azure Database for PostgreSQL - Flexible Server)

This approach uses Azure Database for PostgreSQL - Flexible Server, a managed service that reduces database infrastructure administration.

### Step 1: Provision Azure Database for PostgreSQL Flexible Server

#### Via Azure Portal:
1. Sign in to the [Azure Portal](https://portal.azure.com/).
2. Search for **Azure Database for PostgreSQL flexible servers** and click **Create**.
3. Configure the **Basics** tab:
   - **Subscription**: Select your active Azure subscription.
   - **Resource group**: Select or create a resource group, such as `rg-nextops`.
   - **Server name**: Enter a unique name, such as `nextops-psql-server`.
   - **Region**: Choose a region close to the NextOps application servers.
   - **PostgreSQL version**: Select version `15` or `16`.
   - **Workload type**: Select **Development** or **Production** based on usage.
   - **Compute + storage**: The Burstable `B1ms` or `B2s` tier is suitable for development.
   - **Authentication method**: Select **PostgreSQL authentication only** or Microsoft Entra ID.
   - **Admin username**: For example, `nextopsadmin`.
   - **Admin password**: Enter and securely store a strong password.
4. Click **Next: Networking >**.

### Step 2: Configure Networking & Firewall Rules

1. Under **Network connectivity**, select **Public access (allowed IP addresses)**.
2. Under **Firewall rules**:
   - Select **+ Add current client IP address** to allow your local workstation to connect.
   - Optional: Enable **Allow public access from any Azure service within Azure to this server** if Azure-hosted applications need access.
3. Review and click **Review + create**, then click **Create**. Wait for deployment to complete (takes ~3–5 minutes).

---

### Step 3: Connect via pgAdmin 4 (PaaS)

1. Open **pgAdmin 4**.
2. Right-click **Servers** > **Register** > **Server...**.
3. Under the **General** tab:
   - **Name**: `NextOps-Azure-PaaS`.
4. Under the **Connection** tab:
   - **Host name/address**: `<server-name>.postgres.database.azure.com`. Find this under **Server name** in the Azure Portal Essentials section.
   - **Port**: `5432`
   - **Maintenance database**: `postgres`
   - **Username**: `<admin_username>`, for example `nextopsadmin`.
   - **Password**: `<admin_password>`.
5. Under the **Parameters** (or **SSL**) tab:
   - **SSL mode**: Select `Require`. Azure Flexible Server requires secure connections.
6. Click **Save**.

---

### Step 4: Create Database & Execute SQL Scripts (PaaS)

1. In pgAdmin, expand the connected server `NextOps-Azure-PaaS`.
2. Right-click **Databases** > **Create** > **Database...**.
3. Enter **Database**: `nextops`, then select **Save**.
4. Right-click the newly created `nextops` database and open **Query Tool**.
5. **Run `01-extensions.sql`**:
   - Paste or open `01-extensions.sql`.
   - Click **Execute** (`F5`).
6. **Run `02-telemetry.sql`**:
   - Paste or open `02-telemetry.sql`.
   - Click **Execute** (`F5`).
7. Refresh and open `nextops` > **Schemas** > **public** > **Tables** to confirm:
   - `telemetry`
   - `alarms`

---

## Verifying the Setup

Run the following test queries in pgAdmin Query Tool against the `nextops` database:

```sql
-- Check installed extensions
SELECT * FROM pg_extension WHERE extname = 'pgcrypto';

-- Check tables
SELECT table_name FROM information_schema.tables WHERE table_schema = 'public';

-- Test insert into telemetry
INSERT INTO telemetry (
    device_id,
    device_type,
    refinery_region,
    timestamp,
    temperature,
    pressure,
    flow_rate,
    component_metrics
) VALUES (
    'DEV-TURBINE-01',
    'TURBINE',
    'STATION_1',
    NOW(),
    485.5,
    165.2,
    3600.0,
    '{"bearing_vibration": 0.04, "oil_temp": 62.1}'::jsonb
);

-- View inserted telemetry
SELECT * FROM telemetry;
```

---

## Connection Strings for Applications

Use these formats in your backend application configuration (`.env` or application config):

### IaaS (VM)
```text
postgresql://postgres:<PASSWORD>@<VM_PUBLIC_IP>:5432/nextops?sslmode=disable
```

### PaaS (Azure Flexible Server)
```text
postgresql://<ADMIN_USERNAME>:<PASSWORD>@<SERVER_NAME>.postgres.database.azure.com:5432/nextops?sslmode=require
```
