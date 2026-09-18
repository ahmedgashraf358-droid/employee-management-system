# Employee Management System (EMS)

A simple, dynamic PHP + MySQL Employee Management System. This application
is **not** the CloudSafe project itself — it is the **test workload** that
CloudSafe (an AWS backup and disaster recovery solution) uses to validate
EC2 backups, RDS snapshots, and full-stack restore procedures.

---

## 1. Project Overview

EMS is a small internal-style web application for managing employee
records: viewing, searching, filtering, adding, editing, and deleting
employees. It is deliberately simple — plain PHP, Apache, and MySQL, with
no frameworks — so that:

* It deploys quickly on a single EC2 instance.
* Its data lives entirely in one RDS MySQL database, making backup and
  restore behavior easy to observe and verify.
* Its state (row counts, specific records) is easy to check before and
  after a disaster-recovery drill.

EMS exists purely to give CloudSafe something realistic to protect, break,
and restore.

---

## 2. Architecture

```
User Browser
     |
     v
Amazon EC2 (Apache + PHP application)
     |
     v
Amazon RDS for MySQL (private subnet)
```

* The browser only ever talks to the EC2 instance over HTTP(S).
* The EC2 instance runs Apache + PHP and hosts this codebase.
* The EC2 instance connects to RDS MySQL using credentials supplied via
  environment variables — RDS is **not** exposed to the public internet
  and should live in a private subnet, reachable only from the
  application security group.

---

## 3. Technology Stack

* **Backend:** PHP (plain, no framework), PDO for MySQL access
* **Web server:** Apache
* **Database:** MySQL / Amazon RDS for MySQL
* **Frontend:** HTML5, CSS3, Bootstrap 5 (loaded from CDN) + a small
  custom stylesheet for the console-style UI
* **Deployment target:** Amazon EC2 (Amazon Linux or Ubuntu)

---

## 4. Project Structure

```
employee-management-system/
├── index.php               Dashboard
├── health.php               Health check endpoint
├── config/
│   └── database.php         DB connection (env-based, fails gracefully)
├── employees/
│   ├── index.php             List / search / filter
│   ├── create.php            Add employee
│   ├── edit.php               Edit employee
│   └── delete.php              Delete employee (POST only)
├── includes/
│   ├── header.php, footer.php  Shared layout
│   └── functions.php            Validation, queries, helpers
├── assets/
│   ├── css/style.css
│   └── js/app.js
├── database/
│   ├── schema.sql             Table definition
│   └── seed.sql                 ~27 sample employees
├── .env.example
└── README.md
```

---

## 5. Data Model

Table: `employees`

| Column      | Type                | Notes                          |
|-------------|---------------------|---------------------------------|
| id          | INT UNSIGNED, PK, AI | Primary key                    |
| name        | VARCHAR(100)         | Required                        |
| email       | VARCHAR(150), UNIQUE | Required, validated             |
| department  | VARCHAR(60)          | Required (IT, Cyber Security, Network, HR, Finance, Operations) |
| salary      | DECIMAL(10,2)        | Required, must be >= 0          |
| created_at  | TIMESTAMP            | Auto-set on insert              |

See `database/schema.sql` for the full DDL.

---

## 6. Local Setup

**Requirements:** PHP 8.x with the `pdo_mysql` extension, Apache (or PHP's
built-in server for quick testing), and MySQL/MariaDB.

1. Copy the environment template and fill in local values:
   ```bash
   cp .env.example .env
   ```
2. Create the schema and load sample data:
   ```bash
   mysql -u root -p < database/schema.sql
   mysql -u root -p < database/seed.sql
   ```
3. Run the app with PHP's built-in server for quick local testing:
   ```bash
   php -S localhost:8000
   ```
4. Visit `http://localhost:8000/` for the dashboard and
   `http://localhost:8000/health.php` for the health check.

The `.env` file is loaded automatically by `config/database.php` for local
development. On a real server, prefer setting real environment variables
instead of relying on `.env`.

---

## 7. AWS Deployment (EC2)

1. Launch an EC2 instance (Amazon Linux 2023 or Ubuntu) in a public
   subnet with a security group allowing inbound HTTP/HTTPS from your
   users and SSH from your admin IP only.
2. Install Apache and PHP:
   ```bash
   sudo dnf install -y httpd php php-mysqlnd
   sudo systemctl enable --now httpd
   ```
3. Deploy this codebase to `/var/www/html/` (or a vhost document root).
4. Set the required environment variables for the web server process
   (e.g. in `/etc/environment`, an Apache `SetEnv` directive, or a
   systemd drop-in for `httpd`) — see Section 8.
5. Ensure the EC2 instance's security group is allowed by the RDS
   security group on port 3306 (private connectivity only — RDS should
   not have a public endpoint).
6. Restart Apache and browse to the instance's public IP/DNS.
7. Confirm `/health.php` returns `HEALTHY`.

This EC2 instance is what CloudSafe will image (AMI) and later restore
from, so keep the deployment simple and reproducible.

---

## 8. RDS Configuration

Create an Amazon RDS MySQL instance in a private subnet, then set these
environment variables on the EC2 instance (never hardcode them in code):

| Variable      | Description                          |
|---------------|---------------------------------------|
| `DB_HOST`     | RDS endpoint hostname                 |
| `DB_PORT`     | Usually `3306`                        |
| `DB_NAME`     | Database name (e.g. `ems_db`)         |
| `DB_USER`     | Application database user             |
| `DB_PASSWORD` | Application database password         |

Load `database/schema.sql` and `database/seed.sql` against the RDS
instance (e.g. via a bastion host or an EC2 instance in the same VPC)
before first use.

---

## 9. Health Check

`GET /health.php` — human-readable status page showing:

1. Web application running
2. PHP runtime working
3. RDS connection established
4. A sample query (`SELECT COUNT(*) FROM employees`) succeeding

`GET /health.php?format=json` — machine-readable version for automation,
returning HTTP 200 with `{"status":"HEALTHY", ...}` when all checks pass,
or HTTP 503 with `{"status":"UNHEALTHY", ...}` otherwise. No credentials
or internal error detail are ever included in either response.

---

## 10. Backup / Restore Testing (CloudSafe)

EMS is designed to make the following CloudSafe test scenario easy to run
and verify:

1. **Baseline:** Load `schema.sql` + `seed.sql`. Note the employee count
   on the dashboard (should be 27).
2. **Backup:** CloudSafe creates an EC2 AMI and an RDS snapshot.
3. **Simulate data loss:** Use the Employees page to edit or delete a
   few records (or run test DML directly against RDS).
4. **Restore RDS:** CloudSafe restores the RDS database from the
   snapshot taken in step 2.
5. **Verify data:** Reload the dashboard/Employees page and confirm the
   original records (and original employee count) are back.
6. **Restore/launch EC2:** CloudSafe launches a new instance from the
   AMI (or restores the original instance).
7. **Run health check:** Hit `/health.php?format=json` on the restored
   instance and confirm `"status":"HEALTHY"`.
8. **Confirm functionality:** Use the UI to add/edit/delete an employee
   on the restored stack to confirm full CRUD functionality works
   end-to-end after restore.

Because the entire application state lives in the `employees` table, the
employee count and specific record values are simple, reliable signals
for verifying a successful restore.

---

## 11. Security Notes

* All database queries use PDO prepared statements.
* All user input is validated server-side; all output is HTML-escaped.
* Database credentials are read only from environment variables and are
  never logged or displayed to end users.
* Internal errors (e.g. connection failures) are logged server-side via
  `error_log()` and shown to users only as generic status messages.
* RDS should be deployed in a private subnet with a security group that
  only allows inbound MySQL traffic from the EC2 application security
  group — never expose RDS directly to the internet.

---

## 12. Out of Scope

By design, this application does **not** include: authentication, user
registration, payments, role-based access control, analytics/reporting,
a mobile app, microservices, Docker/Kubernetes, CI/CD, Terraform, AWS
Backup, or cross-region DR. These are intentionally excluded to keep EMS
a simple, stable workload for CloudSafe to protect — not a project in
its own right.
