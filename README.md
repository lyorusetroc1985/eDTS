# eDTS Implementation Blueprint

## 📐 System Architecture Overview (folder structure, request flow, tech decisions)

**Assumptions**
- Organization: **[Your Organization Name]**
- Default roles: **Admin, DeptHead, Staff, Viewer**
- Workflow example: **Draft → Review → Approve → Archive**
- Storage: **Local filesystem / S3-compatible**

**Architecture style:** Modular MVC, PSR-4 autoloading, PHP 8.2 strict typing, MySQL 8.0, PDO prepared statements.

**Request flow**
1. `public/index.php` bootstraps app, env, session hardening.
2. Router maps URI to controller action.
3. Middleware stack: Auth → RBAC → CSRF → Validation.
4. Controller delegates to services/models.
5. Model/Repository uses PDO + transactions.
6. View template renders escaped output.
7. Audit + notification events are persisted asynchronously.

**Tech decisions**
- Minimum supported database: MySQL **8.0+** (for `utf8mb4_0900_ai_ci`, JSON, and modern indexing).
- If legacy compatibility is required (MySQL 5.7), use `utf8mb4_unicode_ci` instead of `utf8mb4_0900_ai_ci`.
- Target platform is Oracle MySQL 8.0+ (not MariaDB) for full feature parity.
- MySQL InnoDB + FK constraints for integrity.
- UTC timestamps (`TIMESTAMP`) everywhere.
- Immutable audit table + append-only history.
- Files stored outside web root (`storage/uploads`) with hashed paths and metadata in DB.
- Background cron for overdue checks + email notifications.

---

## 🗃️ Complete MySQL Schema (SQL DDL with comments, indexes, and relationships)

```sql
-- MySQL 8.0+
SET NAMES utf8mb4;
SET time_zone = '+00:00';

CREATE DATABASE IF NOT EXISTS edts
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;
USE edts;

-- MySQL BOOLEAN is an alias of TINYINT(1): 0=false, non-zero=true.
CREATE TABLE roles (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(50) NOT NULL UNIQUE,
  description VARCHAR(255) NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

CREATE TABLE departments (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  code VARCHAR(30) NOT NULL UNIQUE,
  name VARCHAR(120) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

CREATE TABLE users (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(80) NOT NULL UNIQUE,
  email VARCHAR(191) NOT NULL UNIQUE,
  password_hash VARCHAR(255) NOT NULL,
  full_name VARCHAR(150) NOT NULL,
  department_id BIGINT UNSIGNED NULL,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  failed_logins INT UNSIGNED NOT NULL DEFAULT 0,
  locked_until TIMESTAMP NULL,
  last_login_at TIMESTAMP NULL,
  deleted_at TIMESTAMP NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  CONSTRAINT fk_users_department FOREIGN KEY (department_id) REFERENCES departments(id)
) ENGINE=InnoDB;

CREATE TABLE user_roles (
  user_id BIGINT UNSIGNED NOT NULL,
  role_id BIGINT UNSIGNED NOT NULL,
  assigned_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (user_id, role_id),
  CONSTRAINT fk_user_roles_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  CONSTRAINT fk_user_roles_role FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE
) ENGINE=InnoDB;

CREATE TABLE workflows (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(120) NOT NULL,
  description VARCHAR(255) NULL,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_by BIGINT UNSIGNED NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  CONSTRAINT fk_workflows_user FOREIGN KEY (created_by) REFERENCES users(id),
  UNIQUE KEY uq_workflow_name (name)
) ENGINE=InnoDB;

CREATE TABLE workflow_steps (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  workflow_id BIGINT UNSIGNED NOT NULL,
  step_order INT UNSIGNED NOT NULL,
  step_name VARCHAR(120) NOT NULL,
  assigned_role_id BIGINT UNSIGNED NULL,
  assigned_department_id BIGINT UNSIGNED NULL,
  is_final BOOLEAN NOT NULL DEFAULT FALSE,
  condition_expression JSON NULL,
  sla_hours INT UNSIGNED NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT fk_steps_workflow FOREIGN KEY (workflow_id) REFERENCES workflows(id) ON DELETE CASCADE,
  CONSTRAINT fk_steps_role FOREIGN KEY (assigned_role_id) REFERENCES roles(id),
  CONSTRAINT fk_steps_department FOREIGN KEY (assigned_department_id) REFERENCES departments(id),
  UNIQUE KEY uq_workflow_step_order (workflow_id, step_order)
) ENGINE=InnoDB;

CREATE TABLE documents (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  reference_no VARCHAR(60) NOT NULL UNIQUE,
  title VARCHAR(255) NOT NULL,
  type VARCHAR(80) NOT NULL,
  priority ENUM('low','normal','high','urgent') NOT NULL DEFAULT 'normal',
  description TEXT NULL,
  origin_department_id BIGINT UNSIGNED NOT NULL,
  current_department_id BIGINT UNSIGNED NULL,
  current_custodian_id BIGINT UNSIGNED NULL,
  status ENUM('draft','in_review','approved','rejected','completed','archived') NOT NULL DEFAULT 'draft',
  workflow_id BIGINT UNSIGNED NULL,
  current_step_id BIGINT UNSIGNED NULL,
  due_at TIMESTAMP NULL,
  deleted_at TIMESTAMP NULL,
  created_by BIGINT UNSIGNED NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  CONSTRAINT fk_documents_origin_dept FOREIGN KEY (origin_department_id) REFERENCES departments(id),
  CONSTRAINT fk_documents_current_dept FOREIGN KEY (current_department_id) REFERENCES departments(id),
  CONSTRAINT fk_documents_custodian FOREIGN KEY (current_custodian_id) REFERENCES users(id),
  CONSTRAINT fk_documents_workflow FOREIGN KEY (workflow_id) REFERENCES workflows(id),
  CONSTRAINT fk_documents_current_step FOREIGN KEY (current_step_id) REFERENCES workflow_steps(id),
  CONSTRAINT fk_documents_creator FOREIGN KEY (created_by) REFERENCES users(id),
  KEY idx_documents_status (status),
  KEY idx_documents_created_at (created_at),
  KEY idx_documents_current_department (current_department_id),
  FULLTEXT KEY ft_documents_title_desc (title, description)
) ENGINE=InnoDB;

CREATE TABLE document_metadata (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  document_id BIGINT UNSIGNED NOT NULL,
  meta_key VARCHAR(100) NOT NULL,
  meta_value TEXT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT fk_metadata_document FOREIGN KEY (document_id) REFERENCES documents(id) ON DELETE CASCADE,
  UNIQUE KEY uq_document_meta (document_id, meta_key)
) ENGINE=InnoDB;

CREATE TABLE document_assignments (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  document_id BIGINT UNSIGNED NOT NULL,
  workflow_step_id BIGINT UNSIGNED NULL,
  assigned_to_user_id BIGINT UNSIGNED NULL,
  assigned_to_department_id BIGINT UNSIGNED NULL,
  assigned_by BIGINT UNSIGNED NOT NULL,
  assigned_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  due_at TIMESTAMP NULL,
  status ENUM('pending','accepted','forwarded','completed','rejected','overdue') NOT NULL DEFAULT 'pending',
  completed_at TIMESTAMP NULL,
  remarks TEXT NULL,
  CONSTRAINT fk_assign_doc FOREIGN KEY (document_id) REFERENCES documents(id) ON DELETE CASCADE,
  CONSTRAINT fk_assign_step FOREIGN KEY (workflow_step_id) REFERENCES workflow_steps(id),
  CONSTRAINT fk_assign_user FOREIGN KEY (assigned_to_user_id) REFERENCES users(id),
  CONSTRAINT fk_assign_dept FOREIGN KEY (assigned_to_department_id) REFERENCES departments(id),
  CONSTRAINT fk_assign_by FOREIGN KEY (assigned_by) REFERENCES users(id),
  KEY idx_assign_doc_status (document_id, status),
  KEY idx_assign_due (due_at)
) ENGINE=InnoDB;

CREATE TABLE document_history (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  document_id BIGINT UNSIGNED NOT NULL,
  action VARCHAR(80) NOT NULL,
  action_by BIGINT UNSIGNED NOT NULL,
  from_status VARCHAR(30) NULL,
  to_status VARCHAR(30) NULL,
  remarks TEXT NULL,
  ip_address VARBINARY(16) NULL COMMENT 'Store packed IP bytes (INET6_ATON/inet_pton): IPv4=4 bytes, IPv6=16 bytes',
  user_agent VARCHAR(255) NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT fk_history_document FOREIGN KEY (document_id) REFERENCES documents(id) ON DELETE CASCADE,
  CONSTRAINT fk_history_user FOREIGN KEY (action_by) REFERENCES users(id),
  KEY idx_history_doc_created (document_id, created_at)
) ENGINE=InnoDB;

CREATE TABLE audit_logs (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  actor_user_id BIGINT UNSIGNED NULL,
  entity_type VARCHAR(60) NOT NULL,
  entity_id BIGINT UNSIGNED NULL,
  action VARCHAR(60) NOT NULL,
  before_state JSON NULL,
  after_state JSON NULL,
  ip_address VARBINARY(16) NULL COMMENT 'Store packed IP bytes (INET6_ATON/inet_pton): IPv4=4 bytes, IPv6=16 bytes',
  request_id CHAR(36) NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT fk_audit_actor FOREIGN KEY (actor_user_id) REFERENCES users(id),
  KEY idx_audit_entity (entity_type, entity_id),
  KEY idx_audit_actor_created (actor_user_id, created_at)
) ENGINE=InnoDB;

CREATE TABLE notifications (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT UNSIGNED NOT NULL,
  document_id BIGINT UNSIGNED NULL,
  channel ENUM('in_app','email') NOT NULL DEFAULT 'in_app',
  subject VARCHAR(180) NOT NULL,
  body TEXT NOT NULL,
  is_read BOOLEAN NOT NULL DEFAULT FALSE,
  sent_at TIMESTAMP NULL,
  read_at TIMESTAMP NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT fk_notifications_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  CONSTRAINT fk_notifications_document FOREIGN KEY (document_id) REFERENCES documents(id) ON DELETE SET NULL,
  KEY idx_notifications_user_read (user_id, is_read, created_at)
) ENGINE=InnoDB;

CREATE TABLE system_settings (
  `key` VARCHAR(100) PRIMARY KEY,
  `value` JSON NOT NULL,
  updated_by BIGINT UNSIGNED NULL,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  CONSTRAINT fk_settings_user FOREIGN KEY (updated_by) REFERENCES users(id)
) ENGINE=InnoDB;

-- Seed data
INSERT INTO roles (name, description) VALUES
('Admin','Full system access'),
('DeptHead','Department-level approvals'),
('Staff','Operational routing and updates'),
('Viewer','Read-only access');

INSERT INTO departments (code, name) VALUES
('HR','Human Resources'),
('FIN','Finance'),
('OPS','Operations');
```

---

## 📁 PHP Project Structure (MVC/modular layout with file purposes)

```text
/app
  /Config           # env, db, app settings
  /Core             # Router, Request, Response, BaseController, View, DB
  /Controllers      # AuthController, DocumentController, WorkflowController, AdminController
  /Models           # User, Role, Document, Workflow, Notification, AuditLog
  /Services         # AuthService, DocumentService, WorkflowService, NotificationService
  /Middleware       # AuthMiddleware, RbacMiddleware, CsrfMiddleware, RateLimitMiddleware
  /Validation       # Request validators
  /Views            # Twig/PHP templates (layout + modules)
/public             # index.php, assets, front controller only
/storage
  /logs             # app/error/audit export logs
  /uploads          # non-public uploaded files (hashed path)
  /cache
/database
  schema.sql        # DDL + seed
  migrations        # optional incremental SQL migrations
/tests              # PHPUnit unit/feature tests
/vendor
```

---

## 🔑 Core PHP Implementation (key classes: DB connection, Auth, Document CRUD, Routing/Tracking, Security middleware)

```php
<?php
// app/Core/Database.php
declare(strict_types=1);

namespace App\Core;

use PDO;
use PDOException;

final class Database
{
    private static ?PDO $pdo = null;

    public static function connection(array $cfg): PDO
    {
        if (self::$pdo instanceof PDO) {
            return self::$pdo;
        }

        $dsn = sprintf(
            'mysql:host=%s;port=%d;dbname=%s;charset=utf8mb4',
            $cfg['host'],
            (int)$cfg['port'],
            $cfg['database']
        );

        self::$pdo = new PDO($dsn, $cfg['username'], $cfg['password'], [
            PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
            PDO::ATTR_EMULATE_PREPARES => false,
        ]);

        self::$pdo->exec("SET time_zone = '+00:00'");
        return self::$pdo;
    }
}
```

```php
<?php
// app/Services/AuthService.php
declare(strict_types=1);

namespace App\Services;

use DateTimeImmutable;
use PDO;

final class AuthService
{
    public function __construct(private PDO $db) {}

    public function login(string $identity, string $password, string $ip): bool
    {
        $stmt = $this->db->prepare(
            'SELECT id, password_hash, is_active, locked_until FROM users WHERE (email=:i OR username=:i) AND deleted_at IS NULL LIMIT 1'
        );
        $stmt->execute([':i' => $identity]);
        $user = $stmt->fetch();

        if (!$user || !(bool)$user['is_active']) {
            return false;
        }

        if ($user['locked_until'] !== null) {
            $lockedUntil = new DateTimeImmutable((string)$user['locked_until']);
            if ($lockedUntil > new DateTimeImmutable('now')) {
                return false;
            }
        }

        if (!password_verify($password, (string)$user['password_hash'])) {
            $this->db->prepare('UPDATE users SET failed_logins = failed_logins + 1 WHERE id = :id')
                ->execute([':id' => (int)$user['id']]);

            $this->db->prepare(
                'UPDATE users SET locked_until = DATE_ADD(UTC_TIMESTAMP(), INTERVAL 15 MINUTE)
                 WHERE id = :id AND failed_logins >= 5'
            )->execute([':id' => (int)$user['id']]);
            // Locks account when failed_logins reaches 5.
            return false;
        }

        $this->db->prepare('UPDATE users SET failed_logins = 0, locked_until = NULL WHERE id = :id')
            ->execute([':id' => (int)$user['id']]);

        session_regenerate_id(true);
        $_SESSION['uid'] = (int)$user['id'];
        $_SESSION['csrf'] = bin2hex(random_bytes(32));
        return true;
    }
}
```

```php
<?php
// During user registration / password reset
// Requires PHP Argon2 support (common in PHP 8.2 builds).
$passwordHash = password_hash($plainPassword, PASSWORD_ARGON2ID, [
    'memory_cost' => 1 << 17,
    'time_cost' => 4,
    'threads' => 2,
]);
```

```php
<?php
// IP storage/retrieval with INET6_ATON / INET6_NTOA
$ipBytes = isset($_SERVER['REMOTE_ADDR']) ? inet_pton($_SERVER['REMOTE_ADDR']) : null;
$insert = $pdo->prepare('INSERT INTO audit_logs (actor_user_id, entity_type, action, ip_address) VALUES (:uid,:type,:action,:ip)');
$insert->bindValue(':uid', 1, PDO::PARAM_INT);
$insert->bindValue(':type', 'document', PDO::PARAM_STR);
$insert->bindValue(':action', 'viewed', PDO::PARAM_STR);
$insert->bindValue(':ip', $ipBytes, $ipBytes === null ? PDO::PARAM_NULL : PDO::PARAM_LOB);
$insert->execute();

$read = $pdo->prepare('SELECT INET6_NTOA(ip_address) AS ip_text FROM audit_logs ORDER BY id DESC LIMIT 1');
$read->execute();
$row = $read->fetch();
```

```php
<?php
// app/Services/DocumentService.php
declare(strict_types=1);

namespace App\Services;

use PDO;

final class DocumentService
{
    public function __construct(private PDO $db) {}

    public function create(array $payload, int $actorId): int
    {
        $this->db->beginTransaction();
        try {
            $stmt = $this->db->prepare('INSERT INTO documents
                (reference_no, title, type, priority, description, origin_department_id, current_department_id, status, created_by)
                VALUES (:ref, :title, :type, :priority, :description, :origin, :current, :status, :creator)');

            $stmt->execute([
                ':ref' => $payload['reference_no'],
                ':title' => $payload['title'],
                ':type' => $payload['type'],
                ':priority' => $payload['priority'] ?? 'normal',
                ':description' => $payload['description'] ?? null,
                ':origin' => (int)$payload['origin_department_id'],
                ':current' => (int)$payload['origin_department_id'],
                ':status' => 'draft',
                ':creator' => $actorId,
            ]);

            $documentId = (int)$this->db->lastInsertId();

            $hist = $this->db->prepare('INSERT INTO document_history (document_id, action, action_by, to_status) VALUES (:d,:a,:u,:s)');
            $hist->execute([':d' => $documentId, ':a' => 'created', ':u' => $actorId, ':s' => 'draft']);

            $this->db->commit();
            return $documentId;
        } catch (\Throwable $e) {
            $this->db->rollBack();
            throw $e;
        }
    }
}
```

```php
<?php
// app/Middleware/CsrfMiddleware.php
declare(strict_types=1);

namespace App\Middleware;

final class CsrfMiddleware
{
    public function handle(string $token): void
    {
        $sessionToken = $_SESSION['csrf'] ?? '';
        if (!is_string($sessionToken) || !hash_equals($sessionToken, $token)) {
            http_response_code(419);
            throw new \RuntimeException('Invalid CSRF token.');
        }
    }
}
```

**Secure upload rules**
- Validate by MIME (`finfo`), extension allowlist, max size.
- Generate `sha256_file()` for path: `storage/uploads/ab/cd/<hash>.bin`.
- Store original filename + mime + size + hash in metadata.
- Never execute uploaded files; serve via download controller with auth checks and disable script execution in upload directories.

---

## 🌐 Frontend & Routing Logic (basic UI templates, form handlers, AJAX/fetch examples for status updates)

```php
// public/index.php (simplified)
$router->get('/documents', [DocumentController::class, 'index']);
$router->post('/documents', [DocumentController::class, 'store']);
$router->post('/documents/{id}/route', [WorkflowController::class, 'route']);
$router->get('/documents/{id}/timeline', [DocumentController::class, 'timeline']);
```

```html
<!-- app/Views/documents/show.php -->
<form id="routeForm" method="post" action="/documents/<?= (int)$doc['id'] ?>/route">
  <input type="hidden" name="_csrf" value="<?= htmlspecialchars($csrf, ENT_QUOTES, 'UTF-8') ?>">
  <select name="action" required>
    <option value="forward">Forward</option>
    <option value="approve">Approve</option>
    <option value="reject">Reject</option>
    <option value="complete">Complete</option>
  </select>
  <button type="submit">Update Status</button>
</form>

<script>
async function markRead(notificationId, csrf) {
  await fetch(`/notifications/${notificationId}/read`, {
    method: 'POST',
    headers: {'Content-Type': 'application/json', 'X-CSRF-Token': csrf},
    body: JSON.stringify({read: true})
  });
}
</script>
```

---

## 📊 Dashboard & Reporting Queries (MySQL views/queries for analytics)

```sql
CREATE OR REPLACE VIEW vw_document_status_distribution AS
SELECT status, COUNT(*) AS total
FROM documents
WHERE deleted_at IS NULL
GROUP BY status;

CREATE OR REPLACE VIEW vw_department_pending AS
SELECT d.current_department_id AS department_id, dep.name AS department_name, COUNT(*) AS pending_count
FROM documents d
JOIN departments dep ON dep.id = d.current_department_id
WHERE d.status IN ('draft','in_review') AND d.deleted_at IS NULL
GROUP BY d.current_department_id, dep.name;

-- Overdue assignments
SELECT da.id, da.document_id, da.assigned_to_user_id, da.due_at
FROM document_assignments da
WHERE da.status = 'pending' AND da.due_at < UTC_TIMESTAMP();

-- Timeline for one document
SELECT dh.action, dh.from_status, dh.to_status, u.full_name, dh.created_at
FROM document_history dh
JOIN users u ON u.id = dh.action_by
WHERE dh.document_id = :document_id
ORDER BY dh.created_at ASC;
```

---

## 🛡️ Security Checklist & Deployment Guide (environment setup, .htaccess/nginx config, cron for notifications, backup strategy)

**Security checklist**
- [x] PDO prepared statements only
- [x] Password hashing (`password_hash`, `PASSWORD_ARGON2ID`)
- [x] CSRF token per session + write requests validation
- [x] Output escaping (`htmlspecialchars`)
- [x] Auth rate limiting + lockout window
- [x] Session hardening (`httponly`, `secure`, `samesite=strict`)
- [x] Upload MIME/size/path validation
- [x] RBAC middleware on all protected routes
- [x] Immutable audit logs + restricted DB permissions

**Nginx**
(`public/` is the web root in this layout)
```nginx
server {
  root /var/www/edts/public;
  index index.php;
  location / { try_files $uri /index.php?$query_string; }
  location ~ \.php$ {
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
    fastcgi_pass unix:/run/php/php8.2-fpm.sock;
  }
  location ~ ^/(?:storage|app|config|database|vendor|tests)/ { deny all; }
  location ~ /\.env { deny all; }
}
```

**Cron jobs**
- `*/5 * * * * php /var/www/edts/bin/notify-overdue.php`
- `0 2 * * * mysqldump --defaults-extra-file=/etc/edts/.my.cnf --single-transaction edts | gzip > /backups/edts_$(date +\%F).sql.gz`

**Deployment**
1. Provision Linux + PHP 8.2 + MySQL 8 + Nginx.
2. Create least-privilege DB user (no `DROP`, no `SUPER`).
3. Configure `.env` and rotate secrets.
4. Run schema + seed scripts.
5. Set writable permissions only for `/storage`.
6. Enable TLS, HSTS, and centralized logs.

---

## 🔄 Step-by-Step Implementation Plan (phased rollout from setup to testing to production)

1. **Foundation**: initialize repo, PSR-4 autoload, env/config loader, router, DB bootstrap.
2. **Identity & Access**: login/logout, role mapping, middleware, session + CSRF + rate limiting.
3. **Document Core**: registration, metadata, secure uploads, search/filter APIs.
4. **Workflow Engine**: workflow templates, steps, routing logic, SLA handling, assignment queue.
5. **Tracking & Audit**: timeline UI, immutable logs, actor/IP capture, state diffing.
6. **Notifications**: in-app bell feed, email SMTP adapter, overdue cron.
7. **Admin & Reports**: settings, user/role CRUD, dashboard widgets, CSV/PDF exports.
8. **Quality & Release**: PHPUnit tests, security tests, load test, backups, monitoring, go-live checklist.
