<?php

require_once __DIR__ . '/../vendor/autoload.php';

use Medoo\Medoo;

define('APP_NAME', 'GameHull');

// ─────────────────────────── TIMEZONE ────────────────────────────────
// Set timezone for consistent date/time operations (e.g., 24-hour spin resets)
// Change to your server's timezone or use environment variable
date_default_timezone_set($_ENV['APP_TIMEZONE'] ?? 'UTC');

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

require_once __DIR__ . '/helpers/spin-helpers.php';

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
        `balance` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
        `role` ENUM('user','admin') NOT NULL DEFAULT 'user',
        `status` ENUM('active','banned') NOT NULL DEFAULT 'active',
        `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");

    createTableSafe($pdo, 'wallets', "CREATE TABLE IF NOT EXISTS `wallets` (
        `id` INT AUTO_INCREMENT PRIMARY KEY,
        `user_id` INT NOT NULL,
        `method` VARCHAR(100) NOT NULL,
        `address` VARCHAR(255) NOT NULL,
        `label` VARCHAR(255) NULL,
        `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        INDEX (`user_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");

    // but if the users table already exists, ensure it has the username column and index
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

    // Ensure balance column exists
    $hasBalanceColumn = $pdo->query("SHOW COLUMNS FROM `users` LIKE 'balance'")->fetch(\PDO::FETCH_ASSOC);
    if (!$hasBalanceColumn) {
        $pdo->exec("ALTER TABLE `users` ADD COLUMN `balance` DECIMAL(10,2) NOT NULL DEFAULT 0.00 AFTER `password`");
    }

    createTableSafe($pdo, 'password_resets', "CREATE TABLE IF NOT EXISTS `password_resets` (
        `id` INT AUTO_INCREMENT PRIMARY KEY,
        `email` VARCHAR(255) NOT NULL,
        `token` VARCHAR(64) NOT NULL UNIQUE,
        `expires_at` DATETIME NOT NULL,
        `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");

    createTableSafe($pdo, 'active_sessions', "CREATE TABLE IF NOT EXISTS `active_sessions` (
        `id` INT AUTO_INCREMENT PRIMARY KEY,
        `user_id` INT NOT NULL,
        `session_id` VARCHAR(128) NOT NULL,
        `ip` VARCHAR(45) NOT NULL,
        `user_agent` TEXT NOT NULL,
        `last_activity` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
        `created_at` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
        UNIQUE KEY `session_id` (`session_id`),
        UNIQUE KEY `user_session` (`user_id`, `session_id`),
        KEY `user_last_activity` (`user_id`, `last_activity`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
}
ensureSchemaExists();

function getCurrentRequestIp(): string {
    if (!empty($_SERVER['HTTP_CLIENT_IP'])) {
        return $_SERVER['HTTP_CLIENT_IP'];
    }
    if (!empty($_SERVER['HTTP_X_FORWARDED_FOR'])) {
        $ips = explode(',', $_SERVER['HTTP_X_FORWARDED_FOR']);
        return trim($ips[0]);
    }
    return $_SERVER['REMOTE_ADDR'] ?? '0.0.0.0';
}

function getCurrentSessionId(): string {
    $sessionId = session_id();
    if ($sessionId === '') {
        session_regenerate_id(true);
        $sessionId = session_id();
    }
    return $sessionId;
}

function registerUserSession(int $userId): void {
    global $db;
    $sessionId = getCurrentSessionId();
    $userAgent = $_SERVER['HTTP_USER_AGENT'] ?? 'Unknown';
    $ip = getCurrentRequestIp();
    // echo $ip ; exit;

    $existing = $db->get('active_sessions', 'id', [
        'user_id' => $userId,
        'session_id' => $sessionId,
    ]);

    $data = [
        'user_id'    => $userId,
        'session_id' => $sessionId,
        'ip'         => $ip,
        'user_agent' => $userAgent,
        'last_activity' => date('Y-m-d H:i:s'),
    ];

    if ($existing) {
        $db->update('active_sessions', $data, ['id' => $existing]);
    } else {
        $db->insert('active_sessions', $data);
    }
}

function updateActiveSessionActivity(): void {
    global $db;
    if (!isUserLoggedIn()) {
        return;
    }
    $sessionId = getCurrentSessionId();
    $userId = $_SESSION['user_id'];
    if ($db->has('active_sessions', ['user_id' => $userId, 'session_id' => $sessionId])) {
        $db->update('active_sessions', ['last_activity' => date('Y-m-d H:i:s')], [
            'user_id' => $userId,
            'session_id' => $sessionId,
        ]);
    }
}

function validateActiveSession(): void {
    global $db, $root;
    if (!isUserLoggedIn()) {
        return;
    }
    $sessionId = getCurrentSessionId();
    $userId = $_SESSION['user_id'];
    if (!$db->has('active_sessions', ['user_id' => $userId, 'session_id' => $sessionId])) {
        logoutUser();
        $_SESSION = [];
        session_destroy();
        header('Location: ' . $root . '/login');
        exit;
    }
}

function getActiveSessions(int $userId): array {
    global $db;
    return $db->select('active_sessions', '*', [
        'user_id' => $userId,
        'ORDER' => ['last_activity' => 'DESC'],
    ]);
}

function logoutUserSession(int $userId, string $sessionId): void {
    global $db;
    $db->delete('active_sessions', [
        'user_id' => $userId,
        'session_id' => $sessionId,
    ]);
}

function logoutOtherUserSessions(int $userId, string $currentSessionId): void {
    global $db;
    $db->delete('active_sessions', [
        'user_id' => $userId,
        'session_id[!]' => $currentSessionId,
    ]);
}

function parseBrowserName(string $userAgent): string {
    $ua = strtolower($userAgent);
    if (str_contains($ua, 'edg/')) return 'Edge';
    if (str_contains($ua, 'opr/') || str_contains($ua, 'opera')) return 'Opera';
    if (str_contains($ua, 'chrome/') && !str_contains($ua, 'chromium')) return 'Chrome';
    if (str_contains($ua, 'safari/') && !str_contains($ua, 'chrome/')) return 'Safari';
    if (str_contains($ua, 'firefox/')) return 'Firefox';
    if (str_contains($ua, 'msie') || str_contains($ua, 'trident/')) return 'Internet Explorer';
    return 'Browser';
}

function parsePlatformName(string $userAgent): string {
    $ua = $userAgent;
    if (str_contains($ua, 'Windows NT 10.0')) return 'Windows 10';
    if (str_contains($ua, 'Windows NT 6.3')) return 'Windows 8.1';
    if (str_contains($ua, 'Windows NT 6.2')) return 'Windows 8';
    if (str_contains($ua, 'Windows NT 6.1')) return 'Windows 7';
    if (str_contains($ua, 'Macintosh') || str_contains($ua, 'Mac OS X')) {
        if (preg_match('/Mac OS X ([0-9_\.]+)/', $ua, $matches)) {
            return 'macOS ' . str_replace('_', '.', $matches[1]);
        }
        return 'macOS';
    }
    if (str_contains($ua, 'Android')) {
        if (preg_match('/Android\s+([0-9\.]+)/', $ua, $matches)) {
            return 'Android ' . $matches[1];
        }
        return 'Android';
    }
    if (str_contains($ua, 'iPhone')) return 'iPhone';
    if (str_contains($ua, 'iPad')) return 'iPad';
    if (str_contains($ua, 'Linux')) return 'Linux';
    return 'Unknown OS';
}

function formatDeviceLabel(string $userAgent): string {
    $browser = parseBrowserName($userAgent);
    $platform = parsePlatformName($userAgent);
    return trim($browser . ' · ' . $platform);
}

function getRelativeTime(string $dateTime): string {
    $timestamp = strtotime($dateTime);
    if (!$timestamp) {
        return 'unknown time';
    }
    $diff = time() - $timestamp;
    if ($diff < 60) return 'just now';
    if ($diff < 3600) return floor($diff / 60) . ' min ago';
    if ($diff < 86400) return floor($diff / 3600) . ' hrs ago';
    return floor($diff / 86400) . ' days ago';
}

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
    return $db->get('users', ['id', 'name', 'email', 'username', 'balance', 'role', 'created_at'], ['id' => $_SESSION['user_id']]) ?: null;
}

function registerUser(string $name, string $email, string $password, string $username): array {
    global $db;
    $name  = trim($name);
    $email = strtolower(trim($email));
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
    if (mb_strlen($password) < 8)                   return ['error' => 'Password must be at least 8 characters.'];
    if ($db->has('users', ['email' => $email]))     return ['error' => 'This email is already registered.'];
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
    $_SESSION['user_balance'] = $user['balance'];
    $_SESSION['user_role']  = $user['role'];
    registerUserSession((int)$user['id']);
    return ['success' => true, 'user' => $user];
}

function logoutUser(): void {
    if (!empty($_SESSION['user_id'])) {
        logoutUserSession((int)$_SESSION['user_id'], getCurrentSessionId());
    }
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


validateActiveSession();
updateActiveSessionActivity(); bootstarap end


<?php

use Bramus\Router\Router;

$router = new Router();

// ─────────────── Route title / meta map ───────────────────────────
// Each entry: [pageTitle, metaDescription]
$routeMeta = [
    '/'                  => [APP_NAME . ' – Play & Win | Fish Games, Slots & Instant Payouts',   'Join 50,000+ players on ' . APP_NAME . '. Play Fire Kirin, Juwa, Ultra Panda, Vegas Sweeps and more. Instant deposits, fast withdrawals, daily free spins.'],
    '/games'             => ['All Games – ' . APP_NAME . ' | Fish Games, Slots & Jackpots',      'Browse all 13 games on ' . APP_NAME . ' – Fire Kirin, Juwa, Ultra Panda, Vegas Sweeps, Orion Stars, VBlink and more. Instant play, no download needed.'],
    '/free-spin'         => ['Free Spin – ' . APP_NAME . ' | Claim Your Daily Free Spins',       'Claim your free daily spin on ' . APP_NAME . '. New free spins every 24 hours – no deposit required.'],
    '/affiliate'         => ['Affiliate Program – ' . APP_NAME . ' | Earn While You Play',       'Join the ' . APP_NAME . ' affiliate program and earn commission for every player you refer. Unlimited earning potential.'],
    '/about'             => ['About ' . APP_NAME . ' – Our Story & Mission',                     'Learn about ' . APP_NAME . ', our mission to deliver the best online fish games and slots with instant payouts and a safe gaming environment.'],
    '/contact'           => ['Contact ' . APP_NAME . ' – Get in Touch',                          'Have a question or need support? Contact the ' . APP_NAME . ' team. We respond fast.'],
    '/support'           => ['Support – ' . APP_NAME . ' | Help Center',                         'Need help? Browse the ' . APP_NAME . ' support center for answers to common questions about deposits, withdrawals, games and accounts.'],
    '/privacy-policy'    => ['Privacy Policy – ' . APP_NAME,                                     'Read the ' . APP_NAME . ' privacy policy to understand how we collect, use and protect your personal data.'],
    '/terms-of-service'  => ['Terms of Service – ' . APP_NAME,                                   'Read the ' . APP_NAME . ' terms of service before playing. Know your rights and responsibilities as a player.'],
    '/responsible-gaming'=> ['Responsible Gaming – ' . APP_NAME . ' | Play Safely',              APP_NAME . ' is committed to responsible gaming. Learn about our tools and resources to help you play safely.'],
    '/login'             => ['Log In – ' . APP_NAME,                                             'Sign in to your ' . APP_NAME . ' account to start playing.'],
    '/register'          => ['Create Account – ' . APP_NAME . ' | Join Free & Play Now',         'Create a free ' . APP_NAME . ' account and start playing Fish Games, Slots & Jackpots instantly.'],
    '/forgot-password'   => ['Forgot Password – ' . APP_NAME,                                    'Reset your ' . APP_NAME . ' account password.'],
    '/reset-password'    => ['Reset Password – ' . APP_NAME,                                     'Set a new password for your ' . APP_NAME . ' account.'],
    '/admin/dashboard'   => ['Dashboard – Admin | ' . APP_NAME,                                  ''],
    '/admin/profile'     => ['Profile – Admin | ' . APP_NAME,                                    ''],
    '/profile'           => ['My Profile – ' . APP_NAME,                                         'Manage your ' . APP_NAME . ' account profile and settings.'],
    '/transactions'      => ['Transactions – ' . APP_NAME,                                       'View your ' . APP_NAME . ' transaction history, deposits and withdrawals.'],
    '/cashout'      => ['Cashout – ' . APP_NAME,                                                 'Withdraw & Manage Wallets ' . APP_NAME . ' Manage your cashout requests and wallets on ' . APP_NAME . '.'],
];

/**
 * Set page meta for the current route.
 */
function setRouteMeta(string $path): void {
    global $routeMeta;
    if (isset($routeMeta[$path])) {
        [$GLOBALS['__pageTitle'], $GLOBALS['__metaDescription']] = $routeMeta[$path];
    }
}

// ───────────────────────────── PUBLIC ─────────────────────────────

$router->get('/', function() use ($db, $root) {
    setRouteMeta('/');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/home/index.php';
});

$router->get('/games', function() use ($db, $root) {
    setRouteMeta('/games');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/games/index.php';
});

$router->get('/games/(\w[\w-]*)', function($slug) use ($db, $root) {
    // Static game catalogue (single source of truth)
    $games = [
        ['name'=>'Fire Kirin',      'slug'=>'fire-kirin',      'desc'=>'The legendary fish game platform featuring stunning graphics, multiple game rooms, and the chance to win big.',          'img'=>'fire-kirin.webp',      'badge'=>'HOT','badge_color'=>'#ef4444'],
        ['name'=>'Juwa',            'slug'=>'juwa',            'desc'=>'Experience the ultimate slot gaming with Juwa. Hundreds of exciting machines, progressive jackpots, and daily bonuses.', 'img'=>'juwa.webp',            'badge'=>'TOP','badge_color'=>'#f59e0b'],
        ['name'=>'Ultra Panda',     'slug'=>'ultra-panda',     'desc'=>'Ultimate panda power gaming! Next-level fish games and slots with ultra-smooth gameplay and massive rewards.',            'img'=>'ultra-panda.webp',     'badge'=>'NEW','badge_color'=>'#22c55e'],
        ['name'=>'Vegas Sweeps',    'slug'=>'vegas-sweeps',    'desc'=>'Bring the Vegas experience to your fingertips. Premium slot games, table games, and exclusive tournaments.',             'img'=>'vegas-sweeps.webp',    'badge'=>'HOT','badge_color'=>'#ef4444'],
        ['name'=>'Orion Stars',     'slug'=>'orion-stars',     'desc'=>'Reach for the stars with Orion Stars. A stellar collection of fish games and slots with out-of-this-world graphics.',    'img'=>'orion-stars.webp',     'badge'=>'TOP','badge_color'=>'#f59e0b'],
        ['name'=>'VBlink',          'slug'=>'vblink',          'desc'=>'Fast-paced gaming action in a blink! Experience lightning-quick gameplay with instant wins and exciting features.',        'img'=>'vblink.webp',          'badge'=>'HOT','badge_color'=>'#ef4444'],
        ['name'=>'Juwa 2.0',        'slug'=>'juwa-2',          'desc'=>'The next generation of Juwa gaming! Enhanced graphics, new games, and even bigger jackpots await you.',                  'img'=>'juwa-2.webp',          'badge'=>'NEW','badge_color'=>'#22c55e'],
        ['name'=>'Golden Treasure', 'slug'=>'golden-treasure', 'desc'=>'Hunt for golden treasures and big wins! Discover hidden riches with exciting slot games and legendary jackpots.',         'img'=>'golden-treasure.webp', 'badge'=>'HOT','badge_color'=>'#ef4444'],
        ['name'=>'Game Vault',      'slug'=>'game-vault',      'desc'=>'Your vault of endless entertainment. Premium casino games, exclusive titles, and a secure gaming environment.',           'img'=>'game-vault.webp',      'badge'=>'TOP','badge_color'=>'#f59e0b'],
        ['name'=>'Game Room',       'slug'=>'game-room',       'desc'=>'Your personal gaming room with endless entertainment! Premium slots and fish games in one sleek package.',               'img'=>'game-room.webp',       'badge'=>'NEW','badge_color'=>'#22c55e'],
        ['name'=>'Panda Master',    'slug'=>'panda-master',    'desc'=>'Master your gaming destiny with Panda Master. Classic fish games meet modern slot machines in one powerful platform.',   'img'=>'panda-master.webp',    'badge'=>'NEW','badge_color'=>'#22c55e'],
        ['name'=>'Cash Frenzy',     'slug'=>'cash-frenzy',     'desc'=>'Go crazy with cash prizes and bonuses! Non-stop action with the hottest slot games and massive jackpots.',               'img'=>'cash-frenzy.webp',     'badge'=>'HOT','badge_color'=>'#ef4444'],
        ['name'=>'Milky Way',       'slug'=>'milky-way',       'desc'=>'Explore a galaxy of gaming possibilities. Milky Way offers cosmic-themed games with stellar payouts.',                   'img'=>'milky-way.webp',       'badge'=>'TOP','badge_color'=>'#f59e0b'],
    ];

    // Find game by slug; 404 if not found
    $game = null;
    foreach ($games as $g) {
        if ($g['slug'] === $slug) { $game = $g; break; }
    }
    if (!$game) {
        http_response_code(404);
        include __DIR__ . '/views/errors/404.php';
        return;
    }

    $pageTitle       = $game['name'] . ' – ' . APP_NAME . ' | Play Online';
    $metaDescription = $game['desc'];
    include __DIR__ . '/views/games/detail.php';
});

$router->get('/free-spin', function() use ($db, $root) {
    setRouteMeta('/free-spin');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/pages/free-spin.php';
});

$router->get('/affiliate', function() use ($db, $root) {
    setRouteMeta('/affiliate');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/pages/affiliate.php';
});

$router->get('/about', function() use ($db, $root) {
    setRouteMeta('/about');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/pages/about.php';
});

$router->get('/contact', function() use ($db, $root) {
    setRouteMeta('/contact');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/pages/contact.php';
});

$router->post('/contact', function() use ($db, $root) {
    setRouteMeta('/contact');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/pages/contact.php';
});

$router->get('/support', function() use ($db, $root) {
    setRouteMeta('/support');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/pages/support.php';
});

$router->get('/privacy-policy', function() use ($db, $root) {
    setRouteMeta('/privacy-policy');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/pages/privacy-policy.php';
});

$router->get('/terms-of-service', function() use ($db, $root) {
    setRouteMeta('/terms-of-service');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/pages/terms-of-service.php';
});

$router->get('/responsible-gaming', function() use ($db, $root) {
    setRouteMeta('/responsible-gaming');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/pages/responsible-gaming.php';
});

$router->get('/responsible-gaming', function() use ($db, $root) {
    setRouteMeta('/responsible-gaming');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/pages/responsible-gaming.php';
});

$router->get('/cashout', function() use ($db, $root) {
    if (!isUserLoggedIn()) { header('Location: ' . $root . '/login'); exit; }
    setRouteMeta('/cashout');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/cashout/index.php';
});

$router->post('/cashout', function() use ($db, $root) {
    if (!isUserLoggedIn()) { header('Location: ' . $root . '/login'); exit; }
    setRouteMeta('/cashout');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/cashout/index.php';
});



// ───────────────────────────── AUTH ───────────────────────────────

$router->get('/login', function() use ($db, $root) {
    if (isAnyLoggedIn()) {
        $loc = isLoggedIn() ? $root . '/admin/dashboard' : $root . '/games';
        header('Location: ' . $loc); exit;
    }
    setRouteMeta('/login');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/auth/login.php';
});

$router->post('/login', function() use ($db, $root) {
    setRouteMeta('/login');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/auth/login.php';
});

$router->get('/register', function() use ($db, $root) {
    if (isAnyLoggedIn()) { header('Location: ' . $root . '/games'); exit; }
    setRouteMeta('/register');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/auth/register.php';
});

$router->post('/register', function() use ($db, $root) {
    setRouteMeta('/register');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/auth/register.php';
});

$router->get('/forgot-password', function() use ($db, $root) {
    if (isAnyLoggedIn()) { header('Location: ' . $root . '/games'); exit; }
    setRouteMeta('/forgot-password');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/auth/forgot-password.php';
});

$router->post('/forgot-password', function() use ($db, $root) {
    setRouteMeta('/forgot-password');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/auth/forgot-password.php';
});

$router->get('/reset-password', function() use ($db, $root) {
    if (isAnyLoggedIn()) { header('Location: ' . $root . '/games'); exit; }
    setRouteMeta('/reset-password');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/auth/reset-password.php';
});

$router->post('/reset-password', function() use ($db, $root) {
    setRouteMeta('/reset-password');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/auth/reset-password.php';
});

$router->get('/logout', function() use ($root) {
    logoutUser();
    $_SESSION = [];
    session_destroy();
    session_start();
    $_SESSION['success'] = 'You have been successfully logged out.';
    $ref = $_SERVER['HTTP_REFERER'] ?? '';
    // Redirect back only if the referer is on the same host and not the logout URL itself
    if ($ref && parse_url($ref, PHP_URL_HOST) === $_SERVER['HTTP_HOST'] && !str_contains($ref, '/logout')) {
        header('Location: ' . $ref); exit;
    }
    header('Location: ' . $root . '/'); exit;
});

$router->post('/admin/logout', function() use ($root) {
    verifyCsrf();
    logoutUser();
    $_SESSION = [];
    session_destroy();
    session_start();
    $_SESSION['success'] = 'Admin logged out successfully.';
    header('Location: ' . $root . '/');
    exit;
});

// ───────────────────────────── USER ACCOUNT ──────────────────────

$router->get('/api/devices', function() use ($db, $root) {
    ob_clean();
    header('Content-Type: application/json; charset=utf-8');
    
    if (!isUserLoggedIn()) {
        http_response_code(401);
        echo json_encode(['error' => 'Unauthorized']);
        return;
    }
    
    try {
        $devices = getActiveSessions($_SESSION['user_id']);
        $currentSessionId = getCurrentSessionId();
        $deviceCount = count($devices);
        $deviceCountLabel = $deviceCount . ' device' . ($deviceCount === 1 ? '' : 's') . ' connected';
        
        echo json_encode([
            'success' => true,
            'devices' => $devices,
            'currentSessionId' => $currentSessionId,
            'deviceCount' => $deviceCount,
            'deviceCountLabel' => $deviceCountLabel
        ]);
    } catch (\Exception $e) {
        http_response_code(500);
        echo json_encode(['error' => $e->getMessage()]);
    }
});

/**
 * ╔════════════════════════════════════════════════════════════════╗
 * ║                   FREE DAILY SPIN API POST /api/spin            ║
 * ╠════════════════════════════════════════════════════════════════╣
 *  Processes a spin request server-side.                                                 
 *  * Security layers (in order):
 *   1. Session auth  – must be a logged-in player
 *   2. canUserSpin() – daily free-spin gate + account status check
 *   3. recordSpin()  – re-runs canUserSpin() atomically before writing
 *   4. reward_id     – validated against DB; client value is just a hint  
 *   All checks happen server-side so a raw POST to this endpoint or a
 *   page refresh cannot bypass the once-per-day rule.
 *
 *   Response shape:
 *   { success, error_code, message, prize?, next_free_spin_at? }
 * ╚════════════════════════════════════════════════════════════════╝
 */

$router->post('/api/spin', function () use ($db, $root) {
    ob_clean();
    header('Content-Type: application/json; charset=utf-8');
 
    // ── 1. Authentication ─────────────────────────────────────────────
    if (!isUserLoggedIn()) {
        http_response_code(401);
        echo json_encode([
            'success'    => false,
            'error_code' => 'UNAUTHENTICATED',
            'message'    => 'You must be logged in to spin.',
        ]);
        return;
    }
 
    // ── 2. Parse JSON body ────────────────────────────────────────────
    $raw   = file_get_contents('php://input');
    $input = json_decode($raw, true);
 
    if (json_last_error() !== JSON_ERROR_NONE) {
        http_response_code(400);
        echo json_encode([
            'success'    => false,
            'error_code' => 'INVALID_JSON',
            'message'    => 'Invalid request body.',
        ]);
        return;
    }
 
    $userId          = (int)$_SESSION['user_id'];
    $rewardId        = (int)($input['reward_id']        ?? 0);
    $rotationDegrees = (int)($input['rotation_degrees'] ?? 0);
 
    // ── 3. Fast pre-flight eligibility check ─────────────────────────
    // recordSpin() repeats this, but doing it here lets us return early
    // and avoids touching the DB write path at all.
    $eligibility = canUserSpin($userId);
    if (!$eligibility['can_spin']) {
        http_response_code(403);
        echo json_encode([
            'success'           => false,
            'error_code'        => $eligibility['error_code'],
            'message'           => $eligibility['message'],
            'next_free_spin_at' => $eligibility['next_free_spin_at'] ?? getNextFreeSpinTimestamp(),
            'available_spins'   => $eligibility['available_spins'] ?? 0,
        ]);
        return;
    }
 
    // ── 4. Process the spin ───────────────────────────────────────────
    try {
        $result = recordSpin($userId, $rewardId, $rotationDegrees);
 
        if ($result['success']) {
            http_response_code(200);
        } else {
            // Business-logic failure (e.g. double-spin race condition)
            http_response_code(403);
        }
 
        echo json_encode([
            'success'           => $result['success'],
            'error_code'        => $result['error_code']        ?? null,
            'message'           => $result['message'],
            'prize'             => $result['prize']             ?? null,
            'next_free_spin_at' => $result['next_free_spin_at'] ?? null,
        ]);
 
    } catch (\Exception $e) {
        error_log('[/api/spin] unexpected: ' . $e->getMessage());
        http_response_code(500);
        echo json_encode([
            'success'           => false,
            'error_code'        => 'SERVER_ERROR',
            'message'           => 'Unexpected error. Please try again.',
            'next_free_spin_at' => getNextFreeSpinTimestamp(),
        ]);
    }
});

$router->get('/profile', function() use ($db, $root) {
    if (!isUserLoggedIn()) { header('Location: ' . $root . '/login'); exit; }
    setRouteMeta('/profile');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/user/profile.php';
});

$router->post('/profile', function() use ($db, $root) {
    if (!isUserLoggedIn()) { header('Location: ' . $root . '/login'); exit; }
    setRouteMeta('/profile');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/user/profile.php';
});

$router->get('/transactions', function() use ($db, $root) {
    if (!isUserLoggedIn()) { header('Location: ' . $root . '/login'); exit; }
    setRouteMeta('/transactions');
    $pageTitle = $GLOBALS['__pageTitle'];
    $metaDescription = $GLOBALS['__metaDescription'];
    include __DIR__ . '/views/user/transactions.php';
});


// ───────────────────────────── ADMIN ──────────────────────────────

$router->get('/admin/dashboard', function() use ($db, $root) {
    include __DIR__ . '/views/admin/dashboard.php';
});

$router->get('/admin/profile', function() use ($db, $root) {
    include __DIR__ . '/views/admin/profile.php';
});

$router->post('/admin/profile', function() use ($db, $root) {
    include __DIR__ . '/views/admin/profile.php';
});

// ───────────────────────────── SITEMAP / SEO ──────────────────────

$router->get('/sitemap\.xml', function() {
    $baseUrl = baseUrl();
    header('Content-Type: application/xml; charset=utf-8');
    echo '<?xml version="1.0" encoding="UTF-8"?>' . "\n";
    echo '<?xml-stylesheet type="text/xsl" href="' . $baseUrl . '/assets/sitemap.xsl"?>' . "\n";
    echo '<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">' . "\n";
    foreach (['/sitemap-pages.xml', '/sitemap-games.xml'] as $sm) {
        echo '<sitemap><loc>' . $baseUrl . $sm . '</loc><lastmod>' . date('Y-m-d') . '</lastmod></sitemap>' . "\n";
    }
    echo '</sitemapindex>';
});

$router->get('/sitemap-pages\.xml', function() {
    $baseUrl = baseUrl();
    header('Content-Type: application/xml; charset=utf-8');
    echo '<?xml version="1.0" encoding="UTF-8"?>' . "\n";
    echo '<?xml-stylesheet type="text/xsl" href="' . $baseUrl . '/assets/sitemap.xsl"?>' . "\n";
    echo '<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">' . "\n";
    $pages = [
        ['/', '1.0', 'daily'],
        ['/games', '0.9', 'daily'],
        ['/free-spin', '0.8', 'weekly'],
        ['/affiliate', '0.8', 'weekly'],
        ['/about', '0.6', 'monthly'],
        ['/contact', '0.6', 'monthly'],
        ['/support', '0.6', 'monthly'],
        ['/privacy-policy', '0.4', 'monthly'],
        ['/terms-of-service', '0.4', 'monthly'],
        ['/responsible-gaming', '0.4', 'monthly'],
    ];
    foreach ($pages as [$path, $priority, $freq]) {
        echo '<url><loc>' . $baseUrl . $path . '</loc><lastmod>' . date('Y-m-d') . '</lastmod><changefreq>' . $freq . '</changefreq><priority>' . $priority . '</priority></url>' . "\n";
    }
    echo '</urlset>';
});

$router->get('/sitemap-games\.xml', function() {
    $baseUrl = baseUrl();
    $games = [
        'fire-kirin', 'juwa', 'ultra-panda', 'vegas-sweeps', 'orion-stars',
        'vblink', 'juwa-2', 'golden-treasure', 'game-vault', 'game-room',
        'panda-master', 'cash-frenzy', 'milky-way',
    ];
    header('Content-Type: application/xml; charset=utf-8');
    echo '<?xml version="1.0" encoding="UTF-8"?>' . "\n";
    echo '<?xml-stylesheet type="text/xsl" href="' . $baseUrl . '/assets/sitemap.xsl"?>' . "\n";
    echo '<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">' . "\n";
    foreach ($games as $slug) {
        echo '<url><loc>' . $baseUrl . '/games/' . $slug . '</loc><lastmod>' . date('Y-m-d') . '</lastmod><changefreq>weekly</changefreq><priority>0.7</priority></url>' . "\n";
    }
    echo '</urlset>';
});

$router->get('/robots\.txt', function() {
    $baseUrl = baseUrl();
    header('Content-Type: text/plain');
    echo "User-agent: *\nAllow: /\nDisallow: /admin/\nDisallow: /login\n\nSitemap: $baseUrl/sitemap.xml\n";
});

// ──────────────────────────── 404 ─────────────────────────────────

$router->set404(function() use ($db, $root) {
    http_response_code(404);
    include __DIR__ . '/views/errors/404.php';
});

$router->run(); route end

<?php
/**
 * spin-helpers.php
 * ─────────────────────────────────────────────────────────────────────
 * All Free-Daily-Spin system functions live here so they are available
 * to EVERY route (page view AND /api/spin) without re-declaring them.
 *
 * ─────────────────────────────────────────────────────────────────────
 */


// ═════════════════════════════════════════════════════════════════════
// REWARD DETERMINATION (SERVER-SIDE ONLY)
// ═════════════════════════════════════════════════════════════════════

/**
 * Determine the correct reward based on ACTUAL rotation degrees.
 * This is the SINGLE SOURCE OF TRUTH for prize determination.
 *
 * @param int $rotationDegrees The actual rotation from the wheel (0-359)
 * @return array|null The reward object with id, segment_position, reward_value, reward_amount
 *
 * Logic:
 *   - Wheel has 8 segments, each = 45° (360 / 8)
 *   - Pointer is fixed at top (0°)
 *   - Segment = floor(rotation_degrees / 45) gives segment index (0-7)
 *   - segment_position in DB is 1-indexed, so add 1 to get DB ID
 */
function getRewardByRotation(int $rotationDegrees): ?array
{
    global $db;
    
    try {
        // Normalize rotation to 0-359
        $normalizedRotation = $rotationDegrees % 360;
        if ($normalizedRotation < 0) {
            $normalizedRotation += 360;
        }
        
        // Calculate which segment the pointer landed on
        // Each segment spans 45° (360 / 8 segments)
        // floor() ensures we get the correct segment index (0-7)
        $segmentIndex = floor($normalizedRotation / 45);
        
        // segment_position in DB is 1-indexed (1-8)
        $segmentPosition = $segmentIndex + 1;
        
        // Fetch reward by segment position (the source of truth)
        $reward = $db->get(
            'spin_wheel_rewards',
            '*',
            ['segment_position' => $segmentPosition, 'is_active' => 1]
        );
        
        if ($reward) {
            error_log("[Spin] getRewardByRotation: rotation={$rotationDegrees}° → segment={$segmentPosition} → reward={$reward['reward_value']}");
            return $reward;
        }
        
        // Fallback if segment not found (shouldn't happen in production)
        error_log("[Spin] getRewardByRotation: segment_position={$segmentPosition} not found, using random fallback");
        return null;
        
    } catch (\PDOException $e) {
        error_log('[Spin] getRewardByRotation: ' . $e->getMessage());
        return null;
    }
}

// ═════════════════════════════════════════════════════════════════════

/**
 * Single active reward by primary key, or null when not found / inactive.
 */
function getRewardById(int $rewardId): ?array
{
    global $db;
    try {
        return $db->get(
            'spin_wheel_rewards',
            '*',
            ['id' => $rewardId, 'is_active' => 1]
        ) ?: null;
    } catch (\PDOException $e) {
        error_log('[Spin] getRewardById: ' . $e->getMessage());
        return null;
    }
}

function getWheelRewards(): array
{
    global $db;

    try {
        return $db->select(
            'spin_wheel_rewards',
            '*',
            [
                'is_active' => 1,
                'ORDER' => ['segment_position' => 'ASC']
            ]
        ) ?: [];

    } catch (\PDOException $e) {
        error_log('[Spin] getWheelRewards: ' . $e->getMessage());
        return [];
    }
}

/**
 * Today's spin-stats row for a user.
 * Returns safe defaults when no row exists yet (first visit of the day).
 * Does NOT insert a row — recordSpin() does that.
 */
function getUserDailySpinStats(int $userId): array
{
    global $db;
    $today = date('Y-m-d');
    try {
        $row = $db->get(
            'user_daily_spin_stats',
            '*',
            ['user_id' => $userId, 'stats_date' => $today]
        );
        if ($row) {
            return $row;
        }
    } catch (\PDOException $e) {
        error_log('[Spin] getUserDailySpinStats: ' . $e->getMessage());
    }

    return [
        'user_id'               => $userId,
        'stats_date'            => $today,
        'total_deposited_today' => 0.00,
        'spins_available'       => 0,   // deposit-earned spins only
        'spins_used_today'      => 0,
        'total_won'             => 0.00,
        'best_win'              => 0.00,
        'lifetime_spins'        => 0,
        'lifetime_winnings'     => 0.00,
    ];
}

/**
 * Most-recent $limit spins for a user, newest first.
 */
function getUserSpinHistory(int $userId, int $limit = 10): array
{
    global $db;
    try {
        return $db->select(
            'spin_history',
            '*',
            ['user_id' => $userId, 'ORDER' => ['spun_at' => 'DESC'], 'LIMIT' => $limit]
        ) ?: [];
    } catch (\PDOException $e) {
        error_log('[Spin] getUserSpinHistory: ' . $e->getMessage());
        return [];
    }
}

/**
 * Aggregate lifetime stats across all days for a user.
 */
function getUserLifetimeSpinStats(int $userId): array
{
    global $db;
    try {
        $result = $db->select(
            'user_daily_spin_stats',
            [
                '[SUM]lifetime_spins'    => 'lifetime_spins',
                '[SUM]lifetime_winnings' => 'lifetime_winnings',
                '[SUM]total_won'         => 'total_won',
                '[MAX]best_win'          => 'best_win',
            ],
            ['user_id' => $userId]
        );
        $stats = ($result && isset($result[0])) ? $result[0] : [];
    } catch (\PDOException $e) {
        error_log('[Spin] getUserLifetimeSpinStats: ' . $e->getMessage());
        $stats = [];
    }

    return [
        'total_spins'        => (int)($stats['lifetime_spins']    ?? 0),
        'total_winnings'     => (float)($stats['lifetime_winnings'] ?? 0),
        'best_single_win'    => (float)($stats['best_win']         ?? 0),
        'total_won_all_time' => (float)($stats['total_won']        ?? 0),
    ];
}


// ═════════════════════════════════════════════════════════════════════
// FREE-SPIN GATE
// ═════════════════════════════════════════════════════════════════════

/**
 * True when the user has already taken their 1 free spin today.
 *
 * Uses spin_history as the source of truth — no separate column needed.
 * Resets every 24 hours based on server date (00:00:00 - 23:59:59 of current day).
 *
 * Fail-safe: returns true (= used) on DB error so we never double-award.
 */
function hasFreeSpinUsedToday(int $userId): bool
{
    global $db;
    try {
        $today = date('Y-m-d');

        // Check if ANY spin was made today (within 24-hour period: 00:00:00 to 23:59:59)
        // This automatically resets at midnight when the date changes
        return (bool) $db->has('spin_history', [
            'user_id'       => $userId,
            'spun_at[>=]'   => $today . ' 00:00:00',
            'spun_at[<=]'   => $today . ' 23:59:59',
        ]);
    } catch (\PDOException $e) {
        error_log('[Spin] hasFreeSpinUsedToday: ' . $e->getMessage());
        return true; // fail-safe: assume used to prevent double-award
    }
}

/**
 * Total spins the user may still take right now:
 *   (1 free spin if not yet used today) + (deposit-earned spins)
 */
function getTotalSpinsAvailable(int $userId): int
{
    $freeBonus    = hasFreeSpinUsedToday($userId) ? 0 : 1;
    $depositSpins = (int)(getUserDailySpinStats($userId)['spins_available'] ?? 0);
    return $freeBonus + $depositSpins;
}

/**
 * Unix timestamp of midnight tonight (start of tomorrow, server timezone).
 * Used by the UI countdown clock to show "Next free spin in: HH:MM:SS"
 * 
 * Resets every 24 hours at midnight. This timestamp is the exact moment
 * when a new free spin becomes available.
 */
function getNextFreeSpinTimestamp(): int
{
    return (int) strtotime('tomorrow 00:00:00');
}

// ═════════════════════════════════════════════════════════════════════
// ELIGIBILITY CHECK
// ═════════════════════════════════════════════════════════════════════

/**
 * Authoritative server-side check: can this user spin right now?
 *
 * Called by BOTH the page view (to seed the UI) and the API route
 * (as a security gate before any DB write).
 *
 * Error codes (used by JS errorMessages map in free-spin.php):
 *   INACTIVE_ACCOUNT      – user banned / not active
 *   DAILY_FREE_SPIN_USED  – free spin taken today, no deposit spins left
 *   NO_SPINS              – no spins of any kind available
 *
 * @return array{
 *   can_spin:           bool,
 *   error_code:         string|null,
 *   message:            string,
 *   available_spins:    int,
 *   is_using_free_spin: bool,
 *   next_free_spin_at:  int|null
 * }
 */
function canUserSpin(int $userId): array
{
    global $db;

    // ── 1. Account must be active ─────────────────────────────────────
    try {
        $user = $db->get('users', ['id', 'status'], ['id' => $userId]);
    } catch (\PDOException $e) {
        error_log('[Spin] canUserSpin db: ' . $e->getMessage());
        return _spinDenied('SERVER_ERROR', 'Server error. Please try again.');
    }

    if (!$user || $user['status'] !== 'active') {
        return _spinDenied('INACTIVE_ACCOUNT', 'Your account is not active.');
    }

    // ── 2. Calculate available spins ─────────────────────────────────
    $freeSpinUsedToday = hasFreeSpinUsedToday($userId);
    $depositSpins      = (int)(getUserDailySpinStats($userId)['spins_available'] ?? 0);
    $totalSpins        = ($freeSpinUsedToday ? 0 : 1) + $depositSpins;

    if ($totalSpins <= 0) {
        if ($freeSpinUsedToday) {
            return _spinDenied(
                'DAILY_FREE_SPIN_USED',
                'You have already used your free spin today. '
                    . 'Please come back tomorrow or complete a deposit to unlock more spins.',
                0,
                getNextFreeSpinTimestamp()
            );
        }
        return _spinDenied(
            'NO_SPINS',
            'No spins available. Deposit $50 to unlock a spin.'
        );
    }

    // ── 3. Eligible ───────────────────────────────────────────────────
    return [
        'can_spin'           => true,
        'error_code'         => null,
        'message'            => 'Ready to spin',
        'available_spins'    => $totalSpins,
        'is_using_free_spin' => !$freeSpinUsedToday,
        'next_free_spin_at'  => null,
    ];
}

/** Internal: build a denied-response without repeating array keys. */
function _spinDenied(
    string $errorCode,
    string $message,
    int    $availableSpins  = 0,
    ?int   $nextFreeSpinAt  = null
): array {
    return [
        'can_spin'           => false,
        'error_code'         => $errorCode,
        'message'            => $message,
        'available_spins'    => $availableSpins,
        'is_using_free_spin' => false,
        'next_free_spin_at'  => $nextFreeSpinAt,
    ];
}


// ═════════════════════════════════════════════════════════════════════
// RECORD SPIN
// ═════════════════════════════════════════════════════════════════════

/**
 * Validate, record, and reward a spin.
 *
 * Called ONLY from the /api/spin route.
 * Re-runs canUserSpin() internally so direct API calls cannot bypass
 * the daily free-spin gate.
 *
 * @return array{success: bool, error_code: string|null, message: string, prize: array|null}
 */
function recordSpin(int $userId, int $rewardId, int $rotationDegrees = 0): array
{
    global $db;

    // ── Re-validate (cannot trust frontend state) ─────────────────────
    $eligibility = canUserSpin($userId);
    if (!$eligibility['can_spin']) {
        return [
            'success'           => false,
            'error_code'        => $eligibility['error_code'],
            'message'           => $eligibility['message'],
            'prize'             => null,
            'next_free_spin_at' => $eligibility['next_free_spin_at'],
        ];
    }

    // ── Server-side reward selection ──────────────────────────────────
    // Validate the client-supplied reward_id; fall back to a random reward
    // so the spin always succeeds even if the client sent a bad id.
    $reward = getRewardById($rewardId);
    if (!$reward) {
        $all = getWheelRewards();
        if (empty($all)) {
            return [
                'success'    => false,
                'error_code' => 'NO_REWARDS',
                'message'    => 'No rewards configured.',
                'prize'      => null,
            ];
        }
        $reward = $all[array_rand($all)];
    }

    $today           = date('Y-m-d');
    $now             = date('Y-m-d H:i:s');
    $isUsingFreeSpin = $eligibility['is_using_free_spin'];

    try {
        // ── 1. Write spin history ──────────────────────────────────────
        $db->insert('spin_history', [
            'user_id'          => $userId,
            'reward_id'        => $reward['id'],
            'prize_value'      => $reward['reward_value'],
            'prize_amount'     => $reward['reward_amount'],
            'rotation_degrees' => $rotationDegrees,
            'spun_at'          => $now,
            // 'spin_type'     => $isUsingFreeSpin ? 'free' : 'deposit',  // uncomment if you add spin_type column
        ]);

        // ── 2. Deduct the correct spin bucket ─────────────────────────
        $stats = getUserDailySpinStats($userId);

        if (!$isUsingFreeSpin) {
            // Deduct one deposit-earned spin
            $newDepositSpins = max(0, (int)$stats['spins_available'] - 1);
            $statsExists = $db->has(
                'user_daily_spin_stats',
                ['user_id' => $userId, 'stats_date' => $today]
            );
            if ($statsExists) {
                $db->update(
                    'user_daily_spin_stats',
                    ['spins_available' => $newDepositSpins],
                    ['user_id' => $userId, 'stats_date' => $today]
                );
            }
        }
        // Free spin is tracked by the spin_history row written above —
        // hasFreeSpinUsedToday() will find it on the next request.

        // ── 3. Update daily aggregates ────────────────────────────────
        $newWon  = (float)$stats['total_won'] + (float)$reward['reward_amount'];
        $newBest = max((float)$stats['best_win'], (float)$reward['reward_amount']);

        $statsExists = $db->has(
            'user_daily_spin_stats',
            ['user_id' => $userId, 'stats_date' => $today]
        );

        if ($statsExists) {
            $db->update(
                'user_daily_spin_stats',
                [
                    'spins_used_today'  => (int)$stats['spins_used_today'] + 1,
                    'total_won'         => $newWon,
                    'best_win'          => $newBest,
                    'lifetime_spins'    => (int)$stats['lifetime_spins'] + 1,
                    'lifetime_winnings' => (float)$stats['lifetime_winnings'] + (float)$reward['reward_amount'],
                ],
                ['user_id' => $userId, 'stats_date' => $today]
            );
        } else {
            $db->insert('user_daily_spin_stats', [
                'user_id'               => $userId,
                'stats_date'            => $today,
                'total_deposited_today' => 0.00,
                'spins_available'       => 0,
                'spins_used_today'      => 1,
                'total_won'             => (float)$reward['reward_amount'],
                'best_win'              => (float)$reward['reward_amount'],
                'lifetime_spins'        => 1,
                'lifetime_winnings'     => (float)$reward['reward_amount'],
            ]);
        }

        // ── 4. Credit wallet ──────────────────────────────────────────
        $userRow    = $db->get('users', ['balance'], ['id' => $userId]);
        $newBalance = ((float)($userRow['balance'] ?? 0)) + (float)$reward['reward_amount'];
        $db->update('users', ['balance' => $newBalance], ['id' => $userId]);

        return [
            'success'           => true,
            'error_code'        => null,
            'message'           => "Congratulations! You won {$reward['reward_value']}!",
            'prize'             => $reward,
            'next_free_spin_at' => getNextFreeSpinTimestamp(),
        ];

    } catch (\PDOException $e) {
        error_log('[Spin] recordSpin: ' . $e->getMessage());
        return [
            'success'    => false,
            'error_code' => 'DB_ERROR',
            'message'    => 'Error processing spin. Please try again.',
            'prize'      => null,
        ];
    }
}


// ═════════════════════════════════════════════════════════════════════
// DEPOSIT → SPIN CREDIT
// ═════════════════════════════════════════════════════════════════════

/**
 * Award deposit-earned spins when a payment is confirmed.
 * Rule: every $50 deposited = +1 spin (separate from the free daily spin).
 * Call this from your deposit-processing route after confirming payment.
 */
function updateSpinsFromDeposit(int $userId, float $depositAmount): void
{
    global $db;
    $spinsToAdd = (int) floor($depositAmount / 50);
    if ($spinsToAdd <= 0) {
        return;
    }

    $today = date('Y-m-d');
    $stats = getUserDailySpinStats($userId);

    try {
        $exists = $db->has(
            'user_daily_spin_stats',
            ['user_id' => $userId, 'stats_date' => $today]
        );
        if ($exists) {
            $db->update(
                'user_daily_spin_stats',
                [
                    'total_deposited_today' => (float)$stats['total_deposited_today'] + $depositAmount,
                    'spins_available'       => (int)$stats['spins_available'] + $spinsToAdd,
                ],
                ['user_id' => $userId, 'stats_date' => $today]
            );
        } else {
            $db->insert('user_daily_spin_stats', [
                'user_id'               => $userId,
                'stats_date'            => $today,
                'total_deposited_today' => $depositAmount,
                'spins_available'       => $spinsToAdd,
                'spins_used_today'      => 0,
                'total_won'             => 0.00,
                'best_win'              => 0.00,
                'lifetime_spins'        => 0,
                'lifetime_winnings'     => 0.00,
            ]);
        }
    } catch (\PDOException $e) {
        error_log('[Spin] updateSpinsFromDeposit: ' . $e->getMessage());
    }
}spin-helper end


<?php
/**
 * Free Daily Spin  — views/pages/free-spin.php
 * ─────────────────────────────────────────────────────────────────────
 * Page view only.  All business logic lives in:
 *   • spin-helpers.php (functions, loaded by bootstrap.php)
 *   • /api/spin route  (POST endpoint)
 *
 * This file: fetches data, renders HTML + Alpine.js component.
 */

$pageTitle = 'Free Daily Spin';

// ── Auth guard ────────────────────────────────────────────────────────
// if (!isUserLoggedIn()) {
//     header('Location: ' . $root . '/login');
//     exit;
// }

include __DIR__ . '/../_header.php';

// ── Data for this page ────────────────────────────────────────────────
$currentUserId = (int)($_SESSION['user_id'] ?? 0);

$wheelRewards  = getWheelRewards();
$todayStats    = getUserDailySpinStats($currentUserId);
$spinHistory   = getUserSpinHistory($currentUserId, 10);
$lifetimeStats = getUserLifetimeSpinStats($currentUserId);

// Deposit progress
$targetDeposit   = 50;
$currentDeposit  = (float)($todayStats['total_deposited_today'] ?? 0);
$progressPercent = min(($currentDeposit / $targetDeposit) * 100, 100);

// Spin availability
$depositSpins        = (int)($todayStats['spins_available'] ?? 0);
$freeSpinUsedToday   = hasFreeSpinUsedToday($currentUserId);
$totalSpinsAvailable = getTotalSpinsAvailable($currentUserId);
$nextFreeSpinAt      = getNextFreeSpinTimestamp();
?>

<style type="text/tailwindcss">
  @layer utilities {
    .gh-gradient-border {
      background: linear-gradient(135deg, #2563eb 0%, #7c3aed 100%);
      padding: 3px;
    }
  }
</style>

<main
  class="mx-auto w-[calc(100%_-_48px)] max-w-[1140px] max-lg:w-[calc(100%_-_28px)] max-sm:w-[calc(100%_-_20px)] flex-1 pb-12 md:pb-16"
  x-data="spinPage()"
  x-init="init()">

  <!-- ══════════════════════════════════════════════════════════════════
       HERO
       ══════════════════════════════════════════════════════════════════ -->
  <section class="px-4 pt-10 md:pt-12 pb-12 text-center max-sm:px-0 max-sm:pt-8">
    <div class="inline-flex items-center gap-2 bg-tag-bg dark:bg-tag-dark-bg border border-tag-border dark:border-tag-dark-border rounded-full py-1.5 px-4 mb-5 text-[12px] font-bold text-primary dark:text-tag-dark-text uppercase tracking-widest transition-colors duration-200">
      <span class="relative flex h-2 w-2">
        <span class="animate-ping absolute inline-flex h-full w-full rounded-full bg-primary opacity-75"></span>
        <span class="relative inline-flex rounded-full h-2 w-2 bg-primary"></span>
      </span>
      Daily Reward Available
    </div>
    <h1 class="text-[clamp(32px,5vw,56px)] font-black text-text-heading dark:text-text-dark-heading mb-4 tracking-[-.04em] leading-[1.1] transition-colors duration-200">
      Free Daily <span class="text-primary">Spin</span>
    </h1>
    <p class="text-text-body dark:text-text-dark-body text-[16px] md:text-[18px] max-w-[600px] mx-auto leading-relaxed transition-colors duration-200">
      Spin the wheel once every 24 hours to win free play credits, bonuses, and other exciting rewards.
    </p>
  </section>

  <!-- ══════════════════════════════════════════════════════════════════
       ALREADY-SPUN BANNER  (server-rendered, shown when relevant)
       ══════════════════════════════════════════════════════════════════ -->
  <?php if ($freeSpinUsedToday && $depositSpins <= 0): ?>
    <div class="mb-8 rounded-2xl border border-amber-200 dark:border-amber-800 bg-amber-50 dark:bg-amber-950/40 px-6 py-5 flex flex-col sm:flex-row sm:items-center gap-4">
      <div class="w-10 h-10 rounded-full bg-amber-100 dark:bg-amber-900 flex items-center justify-center shrink-0">
        <svg width="20" height="20" fill="none" stroke="#f59e0b" stroke-width="2" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z"/>
        </svg>
      </div>
      <div class="flex-1">
        <p class="font-bold text-[15px] text-amber-800 dark:text-amber-300">Free spin already used today</p>
        <p class="text-[13px] text-amber-700 dark:text-amber-400 mt-0.5 leading-relaxed">
          Come back tomorrow for your next free spin, or deposit $50 to unlock an additional spin right now.
        </p>
      </div>
      <div class="shrink-0 text-center min-w-[100px]">
        <p class="text-[11px] font-bold text-amber-600 dark:text-amber-400 uppercase tracking-wider mb-1">Next free spin</p>
        <p class="text-[22px] font-black text-amber-700 dark:text-amber-300 tabular-nums" x-text="formatCountdown(countdown)"></p>
      </div>
    </div>
  <?php endif; ?>

  <!-- ══════════════════════════════════════════════════════════════════
       MAIN SPIN CARD
       ══════════════════════════════════════════════════════════════════ -->
  <section
    class="grid min-h-[381px] grid-cols-[minmax(0,1fr)_415px] items-center gap-[54px]
           rounded-[32px] border border-card-border dark:border-card-dark-border
           bg-white dark:bg-[#0e1629]
           bg-[linear-gradient(135deg,rgba(37,99,235,0.04)_0%,transparent_100%)]
           dark:bg-[linear-gradient(135deg,rgba(37,99,235,0.08)_0%,transparent_100%)]
           px-9 py-12 shadow-card dark:shadow-card-dark
           transition duration-300 hover:border-tag-border dark:hover:border-tag-dark-border
           max-lg:grid-cols-1 max-lg:gap-12 max-lg:p-8 max-sm:rounded-[24px] max-sm:p-6"
    aria-labelledby="spin-card-title">

    <!-- ── LEFT: copy + progress ─────────────────────────────────── -->
    <div>
      <span class="inline-flex h-[27px] items-center rounded-full border border-tag-border bg-tag-bg px-3.5 text-sm font-bold text-tag-text dark:border-tag-dark-border dark:bg-tag-dark-bg dark:text-tag-dark-text">
        Daily Bonus
      </span>

      <h2 id="spin-card-title" class="mt-[20px] mb-4 max-w-[580px] text-[clamp(28px,3vw,40px)] leading-[1.2] font-black tracking-[-0.03em] text-text-heading dark:text-text-dark-heading">
        Spin the Wheel &amp; Win
      </h2>

      <p class="max-w-[566px] text-[16px] leading-[1.6] text-text-body dark:text-text-dark-body">
        Every player gets <strong>1 free spin per day</strong>. Deposit $50 or more to unlock additional spins.
      </p>

      <!-- Spin availability pills -->
      <div class="mt-5 flex flex-wrap gap-2">
        <span class="inline-flex items-center gap-1.5 rounded-full px-3 py-1 text-[12px] font-bold
                     <?= !$freeSpinUsedToday
                       ? 'bg-emerald-50 dark:bg-emerald-950/50 border border-emerald-200 dark:border-emerald-800 text-emerald-700 dark:text-emerald-400'
                       : 'bg-bg-muted dark:bg-bg-dark-muted border border-card-border dark:border-card-dark-border text-text-dim dark:text-text-dark-dim line-through' ?>">
          <svg width="12" height="12" fill="none" stroke="currentColor" stroke-width="2.5" viewBox="0 0 24 24">
            <?php if (!$freeSpinUsedToday): ?>
              <path stroke-linecap="round" stroke-linejoin="round" d="M5 13l4 4L19 7"/>
            <?php else: ?>
              <path stroke-linecap="round" stroke-linejoin="round" d="M6 18L18 6M6 6l12 12"/>
            <?php endif; ?>
          </svg>
          Free daily spin
        </span>

        <?php if ($depositSpins > 0): ?>
          <span class="inline-flex items-center gap-1.5 rounded-full px-3 py-1 text-[12px] font-bold bg-blue-50 dark:bg-blue-950/50 border border-blue-200 dark:border-blue-800 text-blue-700 dark:text-blue-400">
            <svg width="12" height="12" fill="none" stroke="currentColor" stroke-width="2.5" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" d="M12 4v16m8-8H4"/>
            </svg>
            +<?= $depositSpins ?> deposit <?= $depositSpins === 1 ? 'spin' : 'spins' ?>
          </span>
        <?php endif; ?>
      </div>

      <!-- Deposit progress card -->
      <div
        id="dailyProgressCard"
        data-current="<?= number_format($currentDeposit, 2, '.', '') ?>"
        data-target="50"
        class="mt-[32px] w-[min(433px,100%)] rounded-2xl border border-card-border dark:border-card-dark-border bg-bg-muted dark:bg-bg-dark-muted px-[24px] py-[20px] transition duration-300 hover:border-tag-border dark:hover:border-tag-dark-border"
        aria-label="Daily deposit progress">

        <div class="flex items-center justify-between gap-4 text-xs font-bold text-text-muted dark:text-text-dark-muted uppercase tracking-widest max-sm:flex-col max-sm:items-start max-sm:gap-1.5">
          <span>Deposit Progress</span>
          <span class="tracking-normal normal-case font-semibold">
            <?= $totalSpinsAvailable ?> <?= $totalSpinsAvailable === 1 ? 'spin' : 'spins' ?> available
          </span>
        </div>

        <div class="mt-4 flex items-center gap-[18px]">
          <strong id="dailyProgressAmount" class="text-[24px] font-black text-text-heading dark:text-text-dark-heading">
            $<?= number_format($currentDeposit, 2) ?>
          </strong>
          <span class="text-text-dim dark:text-text-dark-dim">/ $50 for +1 spin</span>
        </div>

        <div
          class="mt-[20px] h-2 overflow-hidden rounded-full bg-bg-hover dark:bg-bg-dark-hover shadow-inner"
          role="progressbar"
          aria-label="Daily deposit progress"
          aria-valuemin="0"
          aria-valuemax="50"
          aria-valuenow="<?= round($currentDeposit) ?>">
          <div
            id="dailyProgressFill"
            class="h-full rounded-full bg-primary transition-all duration-1000 ease-out shadow-[0_0_12px_rgba(37,99,235,0.4)]"
            style="width: <?= $progressPercent ?>%">
          </div>
        </div>
      </div>
    </div>

    <!-- ── RIGHT: Prize wheel ─────────────────────────────────────── -->
    <div class="grid place-items-center max-lg:-order-1" aria-label="Prize wheel">

      <div class="relative grid aspect-square w-[320px] place-items-center rounded-full border shadow-card
                  transition-all duration-500 ease-out
                  <?= $totalSpinsAvailable > 0
                    ? 'border-[rgba(37,99,235,0.1)] bg-[rgba(37,99,235,0.03)] dark:bg-[rgba(37,99,235,0.05)] hover:border-primary/30'
                    : 'border-[rgba(0,0,0,0.06)] dark:border-[rgba(255,255,255,0.06)] bg-[rgba(0,0,0,0.02)] dark:bg-[rgba(255,255,255,0.02)]' ?>
                  before:absolute before:top-[-8px] before:left-1/2 before:z-30
                  before:h-0 before:w-0 before:-translate-x-1/2
                  before:border-x-[12px] before:border-t-[24px] before:border-x-transparent
                  <?= $totalSpinsAvailable > 0 ? 'before:border-t-primary' : 'before:border-t-text-dim' ?>
                  max-sm:w-[min(320px,86vw)]">

        <!-- Gradient-border ring -->
        <div class="relative w-[286px] h-[286px] rounded-full gh-gradient-border shadow-xl flex items-center justify-center
                    <?= $totalSpinsAvailable <= 0 ? 'opacity-50' : '' ?>">

          <!-- Rotating wheel disk -->
          <div
            id="prizeWheel"
            class="relative w-full h-full overflow-hidden rounded-full bg-white dark:bg-[#0f172a] transition-transform duration-[4000ms]"
            style="transition-timing-function: cubic-bezier(0.15,0,0.15,1);"
            :style="`transform: rotate(${rotation}deg)`">

            <div class="absolute inset-0">
              <!-- Divider lines -->
              <div class="absolute inset-0 rotate-[22.5deg]">
                <div class="absolute top-0 left-1/2 h-full w-px -translate-x-1/2 bg-[linear-gradient(180deg,transparent_0%,rgba(37,99,235,0.2)_50%,transparent_100%)]"></div>
                <div class="absolute top-0 left-1/2 h-full w-px -translate-x-1/2 rotate-45 bg-[linear-gradient(180deg,transparent_0%,rgba(37,99,235,0.2)_50%,transparent_100%)]"></div>
                <div class="absolute top-0 left-1/2 h-full w-px -translate-x-1/2 rotate-90 bg-[linear-gradient(180deg,transparent_0%,rgba(37,99,235,0.2)_50%,transparent_100%)]"></div>
                <div class="absolute top-0 left-1/2 h-full w-px -translate-x-1/2 rotate-[135deg] bg-[linear-gradient(180deg,transparent_0%,rgba(37,99,235,0.2)_50%,transparent_100%)]"></div>
              </div>
              <!-- Segment labels from DB -->
              <?php foreach ($wheelRewards as $index => $reward): ?>
                <span
                  class="absolute top-1/2 left-1/2 text-[22px] font-black text-[#64748b] dark:text-[#f1f5f9]/80 pointer-events-none select-none"
                  style="transform: translate(-50%, -50%) rotate(<?= (int)$index * 45 ?>deg) translateY(-100px)">
                  <?= e($reward['reward_value']) ?>
                </span>
              <?php endforeach; ?>
            </div>
          </div>
        </div>

        <!-- Centre spin button -->
        <button
          @click="spin()"
          :disabled="spinning || totalSpinsAvailable <= 0"
          class="absolute z-40 grid aspect-square w-[84px] place-items-center rounded-full
                 border-[6px] border-white dark:border-[#1e293b]
                 gh-gradient-border shadow-2xl
                 transition-all duration-300
                 hover:scale-110 active:scale-95
                 disabled:opacity-50 disabled:cursor-not-allowed disabled:hover:scale-100
                 group cursor-pointer">
          <div class="w-full h-full rounded-full bg-primary flex flex-col items-center justify-center text-white">
            <template x-if="spinning">
              <svg class="animate-spin w-6 h-6" fill="none" viewBox="0 0 24 24">
                <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z"></path>
              </svg>
            </template>
            <template x-if="!spinning">
              <span class="text-[14px] font-black leading-none group-hover:scale-110 transition-transform select-none"
                    x-text="totalSpinsAvailable > 0 ? 'SPIN' : 'NONE'"></span>
            </template>
          </div>
        </button>
      </div>

      <!-- Helper text when no spins left -->
      <?php if ($totalSpinsAvailable <= 0): ?>
        <p class="mt-4 text-center text-[13px] text-text-muted dark:text-text-dark-muted max-w-[260px] leading-relaxed">
          <?php if ($freeSpinUsedToday): ?>
            Free spin used. Next spin available in
            <span class="font-bold text-primary" x-text="formatCountdown(countdown)"></span>.
          <?php else: ?>
            Deposit $50 to unlock your first spin today.
          <?php endif; ?>
        </p>
      <?php endif; ?>
    </div>
  </section>

  <!-- ══════════════════════════════════════════════════════════════════
       STATS GRID
       ══════════════════════════════════════════════════════════════════ -->
  <section class="mt-12 grid grid-cols-2 md:grid-cols-3 lg:grid-cols-6 gap-4" aria-label="Spin statistics">
    <?php
    $statCards = [
        ['value' => '$' . number_format($currentDeposit, 2),                          'label' => 'Today'],
        ['value' => (string)$totalSpinsAvailable,                                      'label' => 'Available'],
        ['value' => '$' . number_format((float)($todayStats['total_won'] ?? 0), 2),   'label' => 'Total Won'],
        ['value' => (string)(int)($todayStats['spins_used_today'] ?? 0),              'label' => 'Used Today'],
        ['value' => (string)($lifetimeStats['total_spins'] ?? 0),                     'label' => 'Lifetime'],
        ['value' => '$' . number_format((float)($todayStats['best_win'] ?? 0), 2),    'label' => 'Best Win'],
    ];
    ?>
    <?php foreach ($statCards as $card): ?>
      <article class="bg-white dark:bg-[#0e1629] border border-card-border dark:border-card-dark-border rounded-2xl p-5 text-center shadow-card transition-all duration-300 hover:-translate-y-1 hover:border-primary/20">
        <strong class="block text-[22px] font-black text-primary mb-1"><?= e($card['value']) ?></strong>
        <span class="text-[13px] font-bold text-text-muted dark:text-text-dark-muted uppercase tracking-wider"><?= e($card['label']) ?></span>
      </article>
    <?php endforeach; ?>
  </section>

  <!-- ══════════════════════════════════════════════════════════════════
       PRIZE HISTORY
       ══════════════════════════════════════════════════════════════════ -->
  <section
    class="mt-12 rounded-[32px] border border-card-border dark:border-card-dark-border bg-white dark:bg-[#0e1629] p-8 shadow-card dark:shadow-card-dark transition duration-300 hover:border-tag-border dark:hover:border-tag-dark-border max-sm:rounded-[24px] max-sm:p-6"
    aria-labelledby="history-title">

    <div class="flex items-center justify-between gap-5 mb-8">
      <h2 id="history-title" class="text-[22px] font-black text-text-heading dark:text-text-dark-heading tracking-tight">
        Prize History
      </h2>
      <button
        type="button"
        class="inline-flex h-9 cursor-pointer items-center justify-center gap-2 rounded-xl border border-primary/10 bg-primary/5 px-4 text-[13px] font-bold text-primary hover:bg-primary/10 transition-all dark:border-primary/20 dark:bg-primary/10 dark:text-primary-dark-hover dark:hover:bg-primary/20">
        View All
      </button>
    </div>

    <?php if (!empty($spinHistory)): ?>
      <div class="overflow-x-auto">
        <table class="w-full text-sm">
          <thead>
            <tr class="border-b border-card-border dark:border-card-dark-border">
              <th class="text-left py-3 px-4 font-bold text-text-muted dark:text-text-dark-muted uppercase text-xs tracking-wide">Prize</th>
              <th class="text-left py-3 px-4 font-bold text-text-muted dark:text-text-dark-muted uppercase text-xs tracking-wide">Amount</th>
              <th class="text-left py-3 px-4 font-bold text-text-muted dark:text-text-dark-muted uppercase text-xs tracking-wide">Time</th>
            </tr>
          </thead>
          <tbody>
            <?php foreach ($spinHistory as $spin): ?>
              <tr class="border-b border-card-border dark:border-card-dark-border hover:bg-bg-muted dark:hover:bg-bg-dark-muted transition">
                <td class="py-3 px-4 font-semibold text-text-body dark:text-text-dark-body"><?= e($spin['prize_value']) ?></td>
                <td class="py-3 px-4 font-bold text-emerald-600 dark:text-emerald-400">+$<?= number_format((float)$spin['prize_amount'], 2) ?></td>
                <td class="py-3 px-4 text-text-muted dark:text-text-dark-muted text-xs tabular-nums"><?= date('M d, Y H:i', strtotime($spin['spun_at'])) ?></td>
              </tr>
            <?php endforeach; ?>
          </tbody>
        </table>
      </div>
    <?php else: ?>
      <div class="flex flex-col items-center justify-center py-12 text-center">
        <div class="w-16 h-16 bg-primary/5 dark:bg-primary/10 rounded-full flex items-center justify-center mb-4">
          <svg width="32" height="32" viewBox="0 0 24 24" fill="none" stroke="#2563eb" stroke-width="1.5" class="opacity-40">
            <path d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z"/>
          </svg>
        </div>
        <p class="text-[16px] text-text-body dark:text-text-dark-body">No prize history yet. Spin the wheel to win!</p>
      </div>
    <?php endif; ?>
  </section>

</main>

<!-- ══════════════════════════════════════════════════════════════════
     ALPINE COMPONENT
     ══════════════════════════════════════════════════════════════════ -->
<script>
/**
 * spinPage() – Alpine.js data component
 *
 * PHP seeds injected below keep the JS in sync with the server on first load.
 * After a spin the JS updates its own state; a full page reload follows on
 * success to refresh all stats from the DB.
 */
function spinPage() {
  return {

    // ── PHP-seeded state ──────────────────────────────────────────
    spinning:            false,
    rotation:            0,
    totalSpinsAvailable: <?= (int) $totalSpinsAvailable ?>,
    freeSpinUsedToday:   <?= $freeSpinUsedToday ? 'true' : 'false' ?>,
    nextFreeSpinAt:      <?= (int) $nextFreeSpinAt ?>,
    countdown:           0,

    _countdownTimer: null,

    // ── Lifecycle ─────────────────────────────────────────────────
    init() {
      this._tickCountdown();
      this._countdownTimer = setInterval(() => this._tickCountdown(), 1000);
    },

    // ── Countdown ────────────────────────────────────────────────
    _tickCountdown() {
      const now = Math.floor(Date.now() / 1000);
      this.countdown = Math.max(0, this.nextFreeSpinAt - now);
    },

    formatCountdown(secs) {
      if (secs <= 0) return '00:00:00';
      const h = Math.floor(secs / 3600);
      const m = Math.floor((secs % 3600) / 60);
      const s = secs % 60;
      return [h, m, s].map(v => String(v).padStart(2, '0')).join(':');
    },

    // ── Global Toast via Window Event ──────────────────────────────
    showToast(type, title, message, duration = 5000) {
      window.dispatchEvent(new CustomEvent('toast', {
        detail: { type, title, message, duration }
      }));
    },

    // ── Spin ──────────────────────────────────────────────────────
    async spin() {
      if (this.spinning) return;

      // Client-side fast-fail (avoids a round-trip when clearly blocked)
      if (this.totalSpinsAvailable <= 0) {
        const msg = this.freeSpinUsedToday
          ? 'You have already used your free spin today. Come back tomorrow or deposit $50 for more spins.'
          : 'Deposit $50 to unlock a spin.';
        this.showToast('warning', 'No spins available', msg, 0);
        return;
      }

      // ── Start wheel animation ────────────────────────────────
      this.spinning = true;
      const rounds  = 5 + Math.floor(Math.random() * 5);
      const extra   = Math.floor(Math.random() * 360);
      this.rotation += (rounds * 360) + extra;

      // ── Call API after the animation completes (4 s) ─────────
      await new Promise(resolve => setTimeout(resolve, 4000));

      try {
        // ── Calculate which segment the pointer is pointing at ──────────
        // Wheel has 8 segments, each segment is 45° (360 / 8)
        // The pointer is at top (0°), so determine which segment is there
        const finalRotation = this.rotation % 360;
        // Round to nearest segment (so pointer points to segment center)
        const segmentIndex = Math.round(finalRotation / 45) % 8;
        const rewardId = segmentIndex === 0 ? 8 : segmentIndex;

        const res = await fetch('<?= $root ?>/api/spin', {
          method:  'POST',
          headers: { 'Content-Type': 'application/json' },
          body:    JSON.stringify({
            reward_id:        rewardId,
            rotation_degrees: Math.round(finalRotation),
          }),
        });

        // Guard: make sure we got JSON back, not a PHP error page
        const contentType = res.headers.get('content-type') || '';
        if (!contentType.includes('application/json')) {
          const text = await res.text();
          console.error('[spin] non-JSON response:', text);
          this.showToast('error', 'Server error', 'The server returned an unexpected response. Please try again or contact support.', 6000);
          return;
        }
        
        const data = await res.json();
        
        if (data.success) {
          // ── Success ────────────────────────────────────────
          this.totalSpinsAvailable = Math.max(0, this.totalSpinsAvailable - 1);
          this.freeSpinUsedToday   = true;
          
          // Extract prize amount and format currency
          let prizeDisplay = 'a prize';
          if (data.prize) {
            if (data.prize.reward_value) {
              prizeDisplay = data.prize.reward_value;
            } else if (data.prize.reward_amount) {
              prizeDisplay = '$' + parseFloat(data.prize.reward_amount).toFixed(2);
            }
          }
          
          this.showToast('success', '🎉 Congratulations!', `You won ${prizeDisplay}!`, 8000);

          // Reload after toast so stats update from DB
          setTimeout(() => location.reload(), 8000);

        } else {
          // ── Server denied (daily limit, no spins, etc.) ───
          if (typeof data.available_spins === 'number') {
            this.totalSpinsAvailable = data.available_spins;
          }
          if (data.next_free_spin_at) {
            this.nextFreeSpinAt = data.next_free_spin_at;
            this._tickCountdown();
          }

          // Human-readable messages per error code
          const messages = {
            DAILY_FREE_SPIN_USED:
              'You have already used your free spin today. '
              + 'Please come back tomorrow or complete a deposit to unlock more spins.',
            NO_SPINS:
              'No spins available. Deposit $50 to unlock a spin.',
            INACTIVE_ACCOUNT:
              'Your account is not active. Please contact support.',
            UNAUTHENTICATED:
              'Your session has expired. Please refresh the page and log in again.',
            DB_ERROR:
              'A database error occurred. Please try again in a moment.',
          };

          const types = {
            DAILY_FREE_SPIN_USED: 'warning',
            NO_SPINS:             'info',
            INACTIVE_ACCOUNT:     'error',
            UNAUTHENTICATED:      'error',
            DB_ERROR:             'error',
          };

          const isDailyLimit = data.error_code === 'DAILY_FREE_SPIN_USED';

          this.showToast(
            types[data.error_code]    ?? 'error',
            isDailyLimit ? 'Come back tomorrow' : 'Spin not allowed',
            messages[data.error_code] ?? data.message,
            isDailyLimit ? 0 : 8000   // keep daily-limit toast until dismissed
          );
        }

      } catch (err) {
        // Network error (offline, CORS, timeout, etc.)
        console.error('[spin] fetch error:', err);
        this.showToast(
          'error',
          'Network error',
          'Could not reach the server. Please check your connection and try again.',
          6000
        );
      } finally {
        this.spinning = false;
      }
    },
  };
}

// ── Progress bar (plain JS, no framework needed) ──────────────────────
(function () {
  const card = document.getElementById('dailyProgressCard');
  const fill = document.getElementById('dailyProgressFill');
  if (!card || !fill) return;

  const current = parseFloat(card.dataset.current) || 0;
  const target  = parseFloat(card.dataset.target)  || 50;
  const pct     = Math.min((current / target) * 100, 100);

  fill.style.width = pct + '%';
  const bar = fill.parentElement;
  if (bar) {
    bar.setAttribute('aria-valuenow', String(Math.round(current)));
    bar.setAttribute('aria-valuemax', String(target));
  }
  const amountEl = document.getElementById('dailyProgressAmount');
  if (amountEl) amountEl.textContent = '$' + current.toFixed(2);
})();
</script>

<?php include __DIR__ . '/../_footer.php'; ?>



