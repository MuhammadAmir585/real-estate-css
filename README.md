bootstrap.php file code:
<?php

require_once __DIR__ . '/../vendor/autoload.php';

use Medoo\Medoo;

define('APP_NAME', 'GameHull');

// Dynamic root that works both locally (subdirectory) and on production (domain root)
$root = (isset($_SERVER['HTTPS']) ? "https://" : "http://") . $_SERVER['HTTP_HOST'];
$root .= str_replace(basename($_SERVER['SCRIPT_NAME']), '', $_SERVER['SCRIPT_NAME']);
$root = rtrim($root, '/');

// Base path for internal links (e.g. "/projects/qasimhussain.com" or "")
$basePath = rtrim(str_replace(basename($_SERVER['SCRIPT_NAME']), '', $_SERVER['SCRIPT_NAME']), '/');

// Tell Bramus Router about the base path so route matching works in subdirectories
$_SERVER['SCRIPT_NAME'] = $basePath . '/index.php';

ini_set('session.cookie_httponly', '1');
ini_set('session.cookie_samesite', 'Lax');
ini_set('session.use_strict_mode', '1');
session_start();

$db = new Medoo([
    'type' => 'mysql',
    'host' => 'localhost',
    'port' => 3306,
    'database' => 'gamehull_db',
    'username' => 'root',
    'password' => '',
    'charset' => 'utf8mb4',
]);

function getSetting(string $key, string $default = ''): string {
    global $db;
    try {
        $row = $db->get('settings', 'setting_value', ['setting_key' => $key]);
        return $row ?: $default;
    } catch (\PDOException $e) {
        return $default;
    }
}

function setSetting(string $key, string $value): void {
    global $db;
    try {
        $exists = $db->has('settings', ['setting_key' => $key]);
        if ($exists) {
            $db->update('settings', ['setting_value' => $value], ['setting_key' => $key]);
        } else {
            $db->insert('settings', ['setting_key' => $key, 'setting_value' => $value]);
        }
    } catch (\PDOException $e) {
        // table not ready yet — silently ignore
    }
}

function verifyAdminLogin(string $email, string $password): bool {
    $dbEmail = getSetting('admin_email', 'admin@qasimhussain.com');
    $dbHash = getSetting('admin_password', '');
    if (strtolower($email) !== strtolower($dbEmail)) return false;
    if (!$dbHash) return false;
    return password_verify($password, $dbHash);
}

function isLoggedIn(): bool {
    return !empty($_SESSION['admin_logged_in']);
}

function requireAdmin(): void {
    global $root;
    if (!isLoggedIn()) {
        header('Location: ' . $root . '/login');
        exit;
    }
}

function csrfToken(): string {
    if (empty($_SESSION['csrf_token'])) {
        $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
    }
    return $_SESSION['csrf_token'];
}

function csrfField(): string {
    return '<input type="hidden" name="_csrf" value="' . csrfToken() . '">';
}

function verifyCsrf(): void {
    if (empty($_POST['_csrf']) || !hash_equals(csrfToken(), $_POST['_csrf'])) {
        http_response_code(403);
        die('Invalid CSRF token');
    }
}

function e(string $str): string {
    return htmlspecialchars($str, ENT_QUOTES, 'UTF-8');
}

function baseUrl(): string {
    global $root;
    return $root;
}

function currentPath(): string {
    global $basePath;
    $uri = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);
    return $basePath ? substr($uri, strlen($basePath)) : $uri;
}

function fullUrl(): string {
    global $root;
    return $root . currentPath();
}

function activeClass(string $path): string {
    global $basePath;
    $current = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);
    // Strip the basePath prefix to compare against route paths
    $relative = $basePath ? substr($current, strlen($basePath)) : $current;
    if (!$relative) $relative = '/';
    if ($path === '/') {
        return $relative === '/' ? 'active' : '';
    }
    return str_starts_with($relative, $path) ? 'active' : '';
}

// ─────────────────────────── SCHEMA AUTO-INIT ────────────────────────────

/**
 * Create an InnoDB table, handling orphaned .ibd files (errors 1813 / 1932).
 */
function createTableSafe(\PDO $pdo, string $tableName, string $ddl): void {
    try {
        $pdo->exec($ddl);
    } catch (\PDOException $e) {
        $code = (int)($e->errorInfo[1] ?? 0);
        if ($code !== 1813 && $code !== 1932) {
            throw $e;
        }
        // Orphaned tablespace file — drop definition (ignore failure), then
        // physically delete the .ibd file and retry.
        try { $pdo->exec("DROP TABLE IF EXISTS `{$tableName}`"); } catch (\Throwable $ignored) {}

        $row = $pdo->query("SELECT @@datadir AS d")->fetch(\PDO::FETCH_ASSOC);
        if ($row) {
            $ibd = rtrim($row['d'], '/\\') . DIRECTORY_SEPARATOR
                 . 'gamehull_db' . DIRECTORY_SEPARATOR
                 . $tableName . '.ibd';
            if (file_exists($ibd)) { @unlink($ibd); }
        }
        // Remove IF NOT EXISTS so it definitely creates fresh
        $ddl = str_replace('CREATE TABLE IF NOT EXISTS', 'CREATE TABLE', $ddl);
        $pdo->exec($ddl);
    }
}

function ensureSchemaExists(): void {
    global $db;
    $pdo = $db->pdo;

    createTableSafe($pdo, 'settings', "CREATE TABLE IF NOT EXISTS `settings` (
        `id` INT AUTO_INCREMENT PRIMARY KEY,
        `setting_key` VARCHAR(100) NOT NULL UNIQUE,
        `setting_value` TEXT NOT NULL DEFAULT '',
        `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");

    createTableSafe($pdo, 'users', "CREATE TABLE IF NOT EXISTS `users` (
        `id` INT AUTO_INCREMENT PRIMARY KEY,
        `name` VARCHAR(100) NOT NULL,
        `email` VARCHAR(255) NOT NULL UNIQUE,
        `username` VARCHAR(255) NOT NULL UNIQUE,
        `password` VARCHAR(255) NOT NULL,
        `role` ENUM('user','admin') NOT NULL DEFAULT 'user',
        `status` ENUM('active','banned') NOT NULL DEFAULT 'active',
        `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");

// ...but if the users table already exists, ensure it has the username column and index
    $hasUsernameColumn = $pdo->query("SHOW COLUMNS FROM `users` LIKE 'username'")->fetch(\PDO::FETCH_ASSOC);
    if (!$hasUsernameColumn) {
        $pdo->exec("ALTER TABLE `users` ADD COLUMN `username` VARCHAR(50) NULL AFTER `email`");
        $pdo->exec("UPDATE `users` SET `username` = CONCAT('user', `id`) WHERE `username` IS NULL OR `username` = ''");
        $pdo->exec("ALTER TABLE `users` MODIFY `username` VARCHAR(50) NOT NULL");
        $pdo->exec("ALTER TABLE `users` ADD UNIQUE KEY `username` (`username`)");
    } else {
        $indexExists = $pdo->query("SHOW INDEX FROM `users` WHERE Key_name = 'username'")->fetch(\PDO::FETCH_ASSOC);
        if (!$indexExists) {
            $pdo->exec("ALTER TABLE `users` ADD UNIQUE KEY `username` (`username`)");
        }
    }

    createTableSafe($pdo, 'password_resets', "CREATE TABLE IF NOT EXISTS `password_resets` (
        `id` INT AUTO_INCREMENT PRIMARY KEY,
        `email` VARCHAR(255) NOT NULL,
        `token` VARCHAR(64) NOT NULL UNIQUE,
        `expires_at` DATETIME NOT NULL,
        `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
}
ensureSchemaExists();

// ───────────────────────────── USER AUTH ─────────────────────────────────

function isUserLoggedIn(): bool {
    return !empty($_SESSION['user_id']);
}

function isAnyLoggedIn(): bool {
    return isLoggedIn() || isUserLoggedIn();
}

function currentUser(): ?array {
    global $db;
    if (!isUserLoggedIn()) return null;
    return $db->get('users', ['id', 'name', 'email', 'username', 'role', 'created_at'], ['id' => $_SESSION['user_id']]) ?: null;
}

function registerUser(string $name, string $email, string $password, string $username): array {
    global $db;
    $name     = trim($name);
    $email    = strtolower(trim($email));
    $username = trim($username);

    if (mb_strlen($name) < 2)                       return ['error' => 'Name must be at least 2 characters.'];
    if (!filter_var($email, FILTER_VALIDATE_EMAIL)) return ['error' => 'Please enter a valid email address.'];
    if (mb_strlen($username) < 3 || mb_strlen($username) > 24) {
        return ['error' => 'Username must be between 3 and 24 characters.'];
    }
    if (!preg_match('/^[A-Za-z0-9_]+$/', $username)) {
        return ['error' => 'Username can only contain letters, numbers, and underscores. No spaces are allowed.'];
    }
    $username = strtolower($username);
    if ($db->has('users', ['email' => $email]))      return ['error' => 'This email is already registered.'];
    if ($db->has('users', ['username' => $username])) return ['error' => 'This username is already taken.'];

    $db->insert('users', [
        'name'     => htmlspecialchars($name, ENT_QUOTES, 'UTF-8'),
        'email'    => $email,
        'username' => $username,
        'password' => password_hash($password, PASSWORD_DEFAULT),
        'role'     => 'user',
        'status'   => 'active',
    ]);
    return ['success' => true, 'id' => $db->id()];
}

function loginUser(string $email, string $password): array {
    global $db;
    $email = strtolower(trim($email));
    $user  = $db->get('users', '*', ['email' => $email]);
    if (!$user)                                       return ['error' => 'Invalid email or password.'];
    if (!password_verify($password, $user['password'])) return ['error' => 'Invalid email or password.'];
    if ($user['status'] === 'banned')                return ['error' => 'Your account has been suspended.'];
    $_SESSION['user_id']    = $user['id'];
    $_SESSION['user_name']  = $user['name'];
    $_SESSION['user_email'] = $user['email'];
    $_SESSION['user_role']  = $user['role'];
    return ['success' => true, 'user' => $user];
}

function logoutUser(): void {
    unset($_SESSION['user_id'], $_SESSION['user_name'], $_SESSION['user_email'], $_SESSION['user_role']);
}

function createPasswordResetToken(string $email): ?string {
    global $db;
    $email = strtolower(trim($email));
    if (!$db->has('users', ['email' => $email, 'status' => 'active'])) return null;
    // Delete old tokens for this email
    $db->delete('password_resets', ['email' => $email]);
    $token = bin2hex(random_bytes(32));
    $db->insert('password_resets', [
        'email'      => $email,
        'token'      => $token,
        'expires_at' => date('Y-m-d H:i:s', time() + 3600),
    ]);
    return $token;
}

function verifyPasswordResetToken(string $token): ?string {
    global $db;
    $row = $db->get('password_resets', ['email', 'expires_at'], ['token' => $token]);
    if (!$row) return null;
    if (strtotime($row['expires_at']) < time()) return null;
    return $row['email'];
}

function resetUserPassword(string $token, string $newPassword): array {
    global $db;
    if (mb_strlen($newPassword) < 8) return ['error' => 'Password must be at least 8 characters.'];
    $email = verifyPasswordResetToken($token);
    if (!$email) return ['error' => 'This reset link is invalid or has expired.'];
    $db->update('users', ['password' => password_hash($newPassword, PASSWORD_DEFAULT)], ['email' => $email]);
    $db->delete('password_resets', ['email' => $email]);
    return ['success' => true];
}

function isDev(): bool {
    $host = $_SERVER['HTTP_HOST'] ?? 'localhost';
    return in_array(explode(':', $host)[0], ['localhost', '127.0.0.1', '::1'], true);
}
-------------End file

--------------Rgister.php file code:
<?php

if (isAnyLoggedIn()) {
    header('Location: ' . $root . '/games'); exit;
}

$error   = '';
$success = '';
$post    = ['name' => '', 'email' => '', 'username' => ''];

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    verifyCsrf();
    $post['name']  = trim($_POST['name']  ?? '');
    $post['email'] = trim($_POST['email'] ?? '');
    $post['username']  = trim($_POST['username']  ?? '');
    $password      = $_POST['password']  ?? '';
    $confirm       = $_POST['confirm']   ?? '';
    $terms         = !empty($_POST['terms']);

    if (!$terms) {
        $error = 'You must agree to the Terms of Service to continue.';
    } elseif ($password !== $confirm) {
        $error = 'Passwords do not match.';
    } else {
        $result = registerUser($post['name'], $post['email'], $password, $post['username']);
        if (!empty($result['success'])) {
            // Auto-login after registration
            loginUser($post['email'], $password);
            header('Location: ' . $root . '/games'); exit;
        }
        $error = $result['error'] ?? 'Registration failed. Please try again.';
    }
}

include __DIR__ . '/../_header.php';
?>
<style>
.gh-auth-wrap{min-height:calc(100vh - 72px);display:flex;align-items:center;justify-content:center;padding:40px 16px;background:var(--hero-bg);}
.gh-auth-card{width:100%;max-width:460px;background:var(--card-bg);border:1px solid var(--card-border);border-radius:24px;padding:clamp(28px,5vw,44px);box-shadow:var(--card-shadow);}
.gh-auth-title{font-size:22px;font-weight:800;color:var(--text-heading);text-align:center;letter-spacing:-.03em;margin:0 0 6px;}
.gh-auth-sub{font-size:13px;color:var(--text-body);text-align:center;margin:0 0 28px;}
.gh-auth-label{display:block;font-size:13px;font-weight:600;color:var(--text-heading);margin-bottom:6px;}
.gh-auth-input{width:100%;padding:11px 14px;background:var(--bg-muted);border:1.5px solid var(--border);border-radius:12px;font-size:14px;color:var(--text);outline:none;transition:border-color .15s,box-shadow .15s;}
.gh-auth-input::placeholder{color:var(--text-dim);}
.gh-auth-input:focus{border-color:var(--primary);box-shadow:0 0 0 3px var(--primary-ring);}
.gh-auth-btn{width:100%;height:48px;border-radius:999px;background:var(--primary);color:#fff;font-size:15px;font-weight:700;border:none;cursor:pointer;transition:background .15s,transform .12s,box-shadow .12s,opacity .15s;display:flex;align-items:center;justify-content:center;gap:8px;}
.gh-auth-btn:hover:not(:disabled){background:var(--primary-hover);transform:translateY(-1px);box-shadow:0 6px 20px var(--primary-ring);}
.gh-auth-btn:active:not(:disabled){transform:scale(0.97) translateY(0);box-shadow:none;background:var(--primary-hover);}
.gh-auth-btn:disabled{opacity:.45;cursor:not-allowed;transform:none;}
.gh-auth-alert-err{background:rgba(239,68,68,0.1);border:1px solid rgba(239,68,68,0.3);color:#ef4444;border-radius:12px;padding:10px 14px;font-size:13px;margin-bottom:18px;}
.gh-auth-divider{display:flex;align-items:center;gap:12px;margin:22px 0;}
.gh-auth-divider span{font-size:11px;color:var(--text-dim);white-space:nowrap;}
.gh-auth-divider::before,.gh-auth-divider::after{content:'';flex:1;height:1px;background:var(--border);}
.gh-auth-link{color:var(--primary);text-decoration:none;font-weight:600;}
.gh-auth-link:hover{text-decoration:underline;}
.gh-pw-row{display:grid;grid-template-columns:1fr 1fr;gap:14px;}
@media(max-width:480px){.gh-pw-row{grid-template-columns:1fr;}}
.gh-check-row{display:flex;align-items:flex-start;gap:10px;}
.gh-check-row input[type=checkbox]{width:17px;height:17px;margin-top:2px;accent-color:var(--primary);flex-shrink:0;cursor:pointer;}
.gh-check-row label{font-size:12px;color:var(--text-body);line-height:1.5;cursor:pointer;}
</style>

<div class="gh-auth-wrap">
  <div class="gh-auth-card">

    <h1 class="gh-auth-title">Create your account</h1>
    <p class="gh-auth-sub">Join thousands of players &mdash; it&rsquo;s free</p>

    <?php if ($error): ?><div class="gh-auth-alert-err"><?= e($error) ?></div><?php endif; ?>

    <form method="POST" action="<?= $root ?>/register" style="display:flex;flex-direction:column;gap:16px;">
      <?= csrfField() ?>

      <div>
        <label class="gh-auth-label" for="reg-name">Full name</label>
        <input class="gh-auth-input" type="text" id="reg-name" name="name" required
               placeholder="Your name" value="<?= e($post['name']) ?>" autocomplete="name">
      </div>

      <div>
        <label class="gh-auth-label" for="reg-username">Username</label>
        <input class="gh-auth-input" type="text" id="reg-username" name="username" required
               placeholder="Your username" value="<?= e($post['username']) ?>" autocomplete="username"
               pattern="[A-Za-z0-9_]{3,24}" maxlength="24"
               title="Use letters, numbers, and underscores only. No spaces.">
      </div>

      <div>
        <label class="gh-auth-label" for="reg-email">Email address</label>
        <input class="gh-auth-input" type="email" id="reg-email" name="email" required
               placeholder="you@example.com" value="<?= e($post['email']) ?>" autocomplete="email">
      </div>

      <div class="gh-pw-row">
        <div>
          <label class="gh-auth-label" for="reg-password">Password</label>
          <input class="gh-auth-input" type="password" id="reg-password" name="password" required
                 placeholder="Min 8 characters" autocomplete="new-password" minlength="8">
        </div>
        <div>
          <label class="gh-auth-label" for="reg-confirm">Confirm password</label>
          <input class="gh-auth-input" type="password" id="reg-confirm" name="confirm" required
                 placeholder="Repeat password" autocomplete="new-password" minlength="8">
        </div>
      </div>

      <div class="gh-check-row">
        <input type="checkbox" id="reg-terms" name="terms" value="1" <?= !empty($_POST['terms']) ? 'checked' : '' ?>>
        <label for="reg-terms">
          I agree to the <a href="<?= $root ?>/terms-of-service" class="gh-auth-link">Terms of Service</a>
          and <a href="<?= $root ?>/privacy-policy" class="gh-auth-link">Privacy Policy</a>.
          I confirm I am 18+ and play responsibly.
        </label>
      </div>

      <button type="submit" id="reg-submit" class="gh-auth-btn" <?= empty($_POST['terms']) ? 'disabled' : '' ?>>
        <svg width="16" height="16" fill="none" stroke="currentColor" stroke-width="2.5" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" d="M13 10V3L4 14h7v7l9-11h-7z"/></svg>
        Create Account &amp; Play Now
      </button>
    </form>

    <div class="gh-auth-divider"><span>Already have an account?</span></div>

    <a href="<?= $root ?>/login"
       style="display:flex;align-items:center;justify-content:center;height:48px;border-radius:999px;font-size:15px;font-weight:700;color:var(--primary);border:1.5px solid var(--primary);text-decoration:none;transition:background .15s;"
       onmouseenter="this.style.background='var(--tag-bg)'" onmouseleave="this.style.background='transparent'">
      Sign In Instead
    </a>

  </div>
</div>

<script>
(function(){
  var cb  = document.getElementById('reg-terms');
  var btn = document.getElementById('reg-submit');
  if (!cb || !btn) return;
  function sync(){ btn.disabled = !cb.checked; }
  cb.addEventListener('change', sync);
  sync();

  var usernameInput = document.getElementById('reg-username');
  if (usernameInput) {
    usernameInput.addEventListener('input', function () {
      var cleaned = this.value.replace(/\s+/g, '');
      if (cleaned !== this.value) {
        var pos = this.selectionStart - 1;
        this.value = cleaned;
        this.setSelectionRange(pos, pos);
      }
    });
  }
})();
</script>

<?php include __DIR__ . '/../_footer.php'; ?>
------------End Register


----------------Profile file code:
<?php
$user = currentUser();
include __DIR__ . '/../_header.php';
?>
<style>
.gh-dash-wrap{min-height:calc(100vh - 144px);background:var(--hero-bg);padding:clamp(36px,5vw,60px) 24px;}
.gh-dash-inner{max-width:680px;margin:0 auto;}
.gh-dash-card{background:var(--card-bg);border:1px solid var(--card-border);border-radius:24px;padding:clamp(24px,4vw,40px);box-shadow:var(--card-shadow);}
.gh-dash-avatar{width:72px;height:72px;border-radius:999px;background:var(--primary);display:flex;align-items:center;justify-content:center;font-size:28px;font-weight:900;color:#fff;flex-shrink:0;box-shadow:0 4px 16px var(--primary-ring);}
.gh-dash-label{display:block;font-size:12px;font-weight:600;color:var(--text-dim);text-transform:uppercase;letter-spacing:.07em;margin-bottom:5px;}
.gh-dash-input{width:100%;padding:11px 14px;background:var(--bg-muted);border:1.5px solid var(--border);border-radius:12px;font-size:14px;color:var(--text);outline:none;transition:border-color .15s,box-shadow .15s;}
.gh-dash-input:focus{border-color:var(--primary);box-shadow:0 0 0 3px var(--primary-ring);}
.gh-dash-input[readonly]{opacity:.65;cursor:default;}
.gh-dash-btn{height:46px;padding:0 28px;border-radius:999px;background:var(--primary);color:#fff;font-size:14px;font-weight:700;border:none;cursor:pointer;transition:background .15s,transform .12s;}
.gh-dash-btn:hover{background:var(--primary-hover);transform:translateY(-1px);}
.gh-dash-alert-ok{background:rgba(34,197,94,0.1);border:1px solid rgba(34,197,94,0.3);color:#16a34a;border-radius:12px;padding:10px 14px;font-size:13px;margin-bottom:18px;}
.gh-dash-alert-err{background:rgba(239,68,68,0.1);border:1px solid rgba(239,68,68,0.3);color:#ef4444;border-radius:12px;padding:10px 14px;font-size:13px;margin-bottom:18px;}
</style>

<?php
$error   = '';
$success = '';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    verifyCsrf();
    $name    = trim($_POST['name']    ?? '');
    $newPass = $_POST['new_password'] ?? '';
    $confirm = $_POST['confirm']      ?? '';

    if ($name === '') {
        $error = 'Name cannot be empty.';
    } elseif ($newPass !== '' && $newPass !== $confirm) {
        $error = 'New passwords do not match.';
    } elseif ($newPass !== '' && strlen($newPass) < 8) {
        $error = 'Password must be at least 8 characters.';
    } else {
        $data = ['name' => $name];
        if ($newPass !== '') {
            $data['password'] = password_hash($newPass, PASSWORD_DEFAULT);
        }
        global $db;
        $db->update('users', $data, ['id' => $_SESSION['user_id']]);
        $_SESSION['user_name'] = $name;
        $success = 'Profile updated successfully.';
        $user = currentUser();
    }
}
?>

<div class="gh-dash-wrap">
  <div class="gh-dash-inner">

    <div style="display:flex;align-items:center;gap:16px;margin-bottom:32px;">
      <div class="gh-dash-avatar"><?= strtoupper(mb_substr(e($user['name'] ?? 'U'), 0, 1)) ?></div>
      <div>
        <h1 style="font-size:22px;font-weight:800;color:var(--text-heading);letter-spacing:-.03em;margin:0 0 4px;"><?= e($user['name'] ?? 'Player') ?></h1>
        <div style="font-size:13px;color:var(--text-dim);"><?= e($user['email'] ?? '') ?></div>
      </div>
    </div>

    <div class="gh-dash-card">
      <h2 style="font-size:16px;font-weight:700;color:var(--text-heading);margin:0 0 22px;">Account Details</h2>

      <?php if ($error): ?><div class="gh-dash-alert-err"><?= e($error) ?></div><?php endif; ?>
      <?php if ($success): ?><div class="gh-dash-alert-ok"><?= e($success) ?></div><?php endif; ?>

      <form method="POST" action="<?= $root ?>/profile" style="display:flex;flex-direction:column;gap:18px;">
        <?= csrfField() ?>

        <div>
          <label class="gh-dash-label" for="p-name">Display name</label>
          <input class="gh-dash-input" type="text" id="p-name" name="name"
                 value="<?= e($user['name'] ?? '') ?>" required maxlength="80">
        </div>

        <div>
          <label class="gh-dash-label" for="p-username">Username</label>
          <input class="gh-dash-input" type="text" id="p-username" value="@<?= e($user['username'] ?? '') ?>" readonly>
        </div>

        <div>
          <label class="gh-dash-label" for="p-email">Email address</label>
          <input class="gh-dash-input" type="email" id="p-email" value="<?= e($user['email'] ?? '') ?>" readonly>
        </div>

        <div style="height:1px;background:var(--border);margin:4px 0;"></div>
        <p style="font-size:12px;color:var(--text-dim);margin:0;">Leave password fields blank to keep your current password.</p>

        <div>
          <label class="gh-dash-label" for="p-newpass">New password</label>
          <input class="gh-dash-input" type="password" id="p-newpass" name="new_password"
                 placeholder="Min 8 characters" autocomplete="new-password" minlength="8">
        </div>

        <div>
          <label class="gh-dash-label" for="p-confirm">Confirm new password</label>
          <input class="gh-dash-input" type="password" id="p-confirm" name="confirm"
                 placeholder="Repeat new password" autocomplete="new-password" minlength="8">
        </div>

        <div style="display:flex;justify-content:flex-end;padding-top:4px;">
          <button type="submit" class="gh-dash-btn">Save Changes</button>
        </div>
      </form>
    </div>

  </div>
</div>
</div>

<?php include __DIR__ . '/../_footer.php'; ?>

-------------End profile 

DB file code:
-- phpMyAdmin SQL Dump
-- version 5.2.1
-- https://www.phpmyadmin.net/
--
-- Host: 127.0.0.1
-- Generation Time: May 02, 2026 at 01:25 PM
-- Server version: 10.4.32-MariaDB
-- PHP Version: 8.2.12

SET SQL_MODE = "NO_AUTO_VALUE_ON_ZERO";
START TRANSACTION;
SET time_zone = "+00:00";


/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
/*!40101 SET NAMES utf8mb4 */;

--
-- Database: `gamehull_db`
--

-- --------------------------------------------------------

--
-- Table structure for table `password_resets`
--

CREATE TABLE `password_resets` (
  `id` int(11) NOT NULL,
  `email` varchar(255) NOT NULL,
  `token` varchar(64) NOT NULL,
  `expires_at` datetime NOT NULL,
  `created_at` timestamp NOT NULL DEFAULT current_timestamp()
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;

-- --------------------------------------------------------

--
-- Table structure for table `settings`
--

CREATE TABLE `settings` (
  `id` int(11) NOT NULL,
  `setting_key` varchar(100) NOT NULL,
  `setting_value` text NOT NULL DEFAULT '',
  `created_at` timestamp NOT NULL DEFAULT current_timestamp()
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;

-- --------------------------------------------------------

--
-- Table structure for table `users`
--

CREATE TABLE `users` (
  `id` int(11) NOT NULL,
  `name` varchar(100) NOT NULL,
  `email` varchar(255) NOT NULL,
  `username` varchar(50) NOT NULL,
  `password` varchar(255) NOT NULL,
  `role` enum('user','admin') NOT NULL DEFAULT 'user',
  `status` enum('active','banned') NOT NULL DEFAULT 'active',
  `created_at` timestamp NOT NULL DEFAULT current_timestamp()
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;

--
-- Dumping data for table `users`
--

INSERT INTO `users` (`id`, `name`, `email`, `username`, `password`, `role`, `status`, `created_at`) VALUES
(1, 'Qasim Hussains', 'compoxition@gmail.com', 'compoxition', '$2y$10$EcYeiDD8FoneZZTvz2v49evfSd6sZYn8/wSuL0Du/Em4rJntLSkby', 'user', 'active', '2026-04-23 04:32:26');

--
-- Indexes for dumped tables
--

--
-- Indexes for table `password_resets`
--
ALTER TABLE `password_resets`
  ADD PRIMARY KEY (`id`),
  ADD UNIQUE KEY `token` (`token`);

--
-- Indexes for table `settings`
--
ALTER TABLE `settings`
  ADD PRIMARY KEY (`id`),
  ADD UNIQUE KEY `setting_key` (`setting_key`);

--
-- Indexes for table `users`
--
ALTER TABLE `users`
  ADD PRIMARY KEY (`id`),
  ADD UNIQUE KEY `email` (`email`),
  ADD UNIQUE KEY `username` (`username`);

--
-- AUTO_INCREMENT for dumped tables
--

--
-- AUTO_INCREMENT for table `password_resets`
--
ALTER TABLE `password_resets`
  MODIFY `id` int(11) NOT NULL AUTO_INCREMENT;

--
-- AUTO_INCREMENT for table `settings`
--
ALTER TABLE `settings`
  MODIFY `id` int(11) NOT NULL AUTO_INCREMENT;

--
-- AUTO_INCREMENT for table `users`
--
ALTER TABLE `users`
  MODIFY `id` int(11) NOT NULL AUTO_INCREMENT, AUTO_INCREMENT=2;
COMMIT;

/*!40101 SET CHARACTER_SET_CLIENT=@OLD_CHARACTER_SET_CLIENT */;
/*!40101 SET CHARACTER_SET_RESULTS=@OLD_CHARACTER_SET_RESULTS */;
/*!40101 SET COLLATION_CONNECTION=@OLD_COLLATION_CONNECTION */;

----------End DB



