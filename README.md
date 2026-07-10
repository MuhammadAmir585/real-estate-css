app\lib\Database.php -> from 331
self::$connection->exec("CREATE TABLE IF NOT EXISTS hotelier_checkins (
      id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
      hotel_row_id INT UNSIGNED NULL,
      hotel_id VARCHAR(64) NULL,
      hotel_slug VARCHAR(255) NULL,
      hotel_name VARCHAR(255) NULL,
      city_name VARCHAR(255) NULL,
      booking_hash VARCHAR(64) NULL,
      reference VARCHAR(64) NULL,
      guest_name VARCHAR(255) NULL,
      user_email VARCHAR(255) NULL,
      user_phone VARCHAR(64) NULL,
      room VARCHAR(255) NULL,
      room_no VARCHAR(32) NULL,
      checkin_date DATE NULL,
      arrival_time VARCHAR(16) NULL,
      nights INT UNSIGNED NOT NULL DEFAULT 0,
      guests INT UNSIGNED NOT NULL DEFAULT 1,
      id_status VARCHAR(32) NULL,
      payment_status VARCHAR(32) NULL,
      checkin_status VARCHAR(32) NULL,
      balance_due DECIMAL(12,2) NOT NULL DEFAULT 0,
      notes TEXT NULL,
      created_at DATETIME NULL,
      updated_at DATETIME NULL,
      UNIQUE KEY uq_hotelier_checkins_reference (reference),
      KEY idx_hotelier_checkins_hotel (hotel_row_id),
      KEY idx_hotelier_checkins_status (checkin_status),
      KEY idx_hotelier_checkins_date (checkin_date)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci");

    self::$connection->exec("CREATE TABLE IF NOT EXISTS hotelier_checkouts (
      id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
      hotel_row_id INT UNSIGNED NULL,
      hotel_id VARCHAR(64) NULL,
      hotel_slug VARCHAR(255) NULL,
      hotel_name VARCHAR(255) NULL,
      city_name VARCHAR(255) NULL,
      booking_hash VARCHAR(64) NULL,
      reference VARCHAR(64) NULL,
      guest_name VARCHAR(255) NULL,
      user_email VARCHAR(255) NULL,
      user_phone VARCHAR(64) NULL,
      room VARCHAR(255) NULL,
      room_no VARCHAR(32) NULL,
      checkout_date DATE NULL,
      checkout_time VARCHAR(16) NULL,
      nights INT UNSIGNED NOT NULL DEFAULT 0,
      guests INT UNSIGNED NOT NULL DEFAULT 1,
      payment_status VARCHAR(32) NULL,
      checkout_status VARCHAR(32) NULL,
      room_status VARCHAR(32) NULL,
      extras TEXT NULL,
      balance_due DECIMAL(12,2) NOT NULL DEFAULT 0,
      notes TEXT NULL,
      created_at DATETIME NULL,
      updated_at DATETIME NULL,
      UNIQUE KEY uq_hotelier_checkouts_reference (reference),
      KEY idx_hotelier_checkouts_hotel (hotel_row_id),
      KEY idx_hotelier_checkouts_status (checkout_status),
      KEY idx_hotelier_checkouts_date (checkout_date)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci");


app\lib\HotelierCheckinRepo.php
<?php
namespace Bookin;

/** Owner-scoped check-in data for the hotelier dashboard. */
final class HotelierCheckinRepo
{
  /** @return array<int,array<string,mixed>> */
  public static function hotelsForUser(int $userId): array
  {
    if ($userId <= 0) return [];

    $db = Database::medoo();
    try {
      return $db->select('hotels', [
        'id',
        'hotel_id',
        'slug',
        'name',
        'city_name',
      ], [
        'hotelier_id' => $userId,
        'ORDER' => ['id' => 'ASC'],
      ]) ?: [];
    } catch (\Throwable $e) {
      return [];
    }
  }

  /** @param array<int,array<string,mixed>> $hotels */
  public static function hotelRowIds(array $hotels): array
  {
    $ids = [];
    foreach ($hotels as $hotel) {
      $id = (int)($hotel['id'] ?? 0);
      if ($id > 0) $ids[] = $id;
    }
    return array_values(array_unique($ids));
  }

  /** @param array<int,array<string,mixed>> $hotels */
  public static function createDefaults(array $hotels): array
  {
    $hotel = $hotels[0] ?? [];
    $now = date('Y-m-d H:i:s');
    $reference = 'CI-' . strtoupper(substr(bin2hex(random_bytes(4)), 0, 8));

    return [
      'booking_hash' => bin2hex(random_bytes(16)),
      'reference' => $reference,
      'hotel_row_id' => (int)($hotel['id'] ?? 0),
      'hotel_id' => (string)($hotel['hotel_id'] ?? ''),
      'hotel_slug' => (string)($hotel['slug'] ?? ''),
      'hotel_name' => (string)($hotel['name'] ?? ''),
      'city_name' => (string)($hotel['city_name'] ?? ''),
      'id_status' => 'pending',
      'payment_status' => 'unpaid',
      'checkin_status' => 'ready',
      'balance_due' => 0,
      'created_at' => $now,
      'updated_at' => $now,
    ];
  }

  /** @return array<int,array<string,mixed>> */
  public static function demoRows(): array
  {
    $file = __DIR__ . '/../config/hotelier-checkins-demo.json';
    if (!is_file($file)) return [];

    $json = file_get_contents($file);
    $rows = json_decode((string)$json, true);
    return is_array($rows) ? array_values(array_filter($rows, 'is_array')) : [];
  }

  /** @param array<int,array<string,mixed>> $rows */
  public static function statsFromRows(array $rows): array
  {
    $stats = [
      'total' => count($rows),
      'ready' => 0,
      'checked_in' => 0,
      'pending_id' => 0,
      'balance_due' => 0.0,
    ];

    foreach ($rows as $row) {
      $status = strtolower((string)($row['checkin_status'] ?? ''));
      $idStatus = strtolower((string)($row['id_status'] ?? ''));
      if ($status === 'ready') $stats['ready']++;
      if ($status === 'checked_in') $stats['checked_in']++;
      if ($idStatus === 'pending') $stats['pending_id']++;
      if (is_numeric($row['balance_due'] ?? null)) {
        $stats['balance_due'] += (float)$row['balance_due'];
      }
    }

    return $stats;
  }

  /** @param array<int,int> $hotelRowIds */
  public static function stats(array $hotelRowIds): array
  {
    if (!$hotelRowIds) {
      return [
        'total' => 0,
        'ready' => 0,
        'checked_in' => 0,
        'pending_id' => 0,
        'balance_due' => 0.0,
      ];
    }

    return self::statsWhere(['hotel_row_id' => $hotelRowIds]);
  }

  public static function allStats(): array
  {
    return self::statsWhere([]);
  }

  private static function statsWhere(array $where): array
  {
    $db = Database::medoo();
    try {
      return [
        'total' => (int)$db->count('hotelier_checkins', $where),
        'ready' => (int)$db->count('hotelier_checkins', array_merge($where, ['checkin_status' => 'ready'])),
        'checked_in' => (int)$db->count('hotelier_checkins', array_merge($where, ['checkin_status' => 'checked_in'])),
        'pending_id' => (int)$db->count('hotelier_checkins', array_merge($where, ['id_status' => 'pending'])),
        'balance_due' => (float)($db->sum('hotelier_checkins', 'balance_due', $where) ?: 0),
      ];
    } catch (\Throwable $e) {
      return [
        'total' => 0,
        'ready' => 0,
        'checked_in' => 0,
        'pending_id' => 0,
        'balance_due' => 0.0,
      ];
    }
  }
}



app\lib\HotelierCheckoutRepo.php

<?php
namespace Bookin;

/** Owner-scoped checkout data for the hotelier dashboard. */
final class HotelierCheckoutRepo
{
  /** @return array<int,array<string,mixed>> */
  public static function hotelsForUser(int $userId): array
  {
    if ($userId <= 0) return [];

    $db = Database::medoo();
    try {
      return $db->select('hotels', [
        'id',
        'hotel_id',
        'slug',
        'name',
        'city_name',
      ], [
        'hotelier_id' => $userId,
        'ORDER' => ['id' => 'ASC'],
      ]) ?: [];
    } catch (\Throwable $e) {
      return [];
    }
  }

  /** @param array<int,array<string,mixed>> $hotels */
  public static function hotelRowIds(array $hotels): array
  {
    $ids = [];
    foreach ($hotels as $hotel) {
      $id = (int)($hotel['id'] ?? 0);
      if ($id > 0) $ids[] = $id;
    }
    return array_values(array_unique($ids));
  }

  /** @param array<int,array<string,mixed>> $hotels */
  public static function createDefaults(array $hotels): array
  {
    $hotel = $hotels[0] ?? [];
    $now = date('Y-m-d H:i:s');
    $reference = 'CO-' . strtoupper(substr(bin2hex(random_bytes(4)), 0, 8));

    return [
      'reference' => $reference,
      'hotel_row_id' => (int)($hotel['id'] ?? 0),
      'hotel_id' => (string)($hotel['hotel_id'] ?? ''),
      'hotel_slug' => (string)($hotel['slug'] ?? ''),
      'hotel_name' => (string)($hotel['name'] ?? ''),
      'city_name' => (string)($hotel['city_name'] ?? ''),
      'payment_status' => 'unpaid',
      'checkout_status' => 'ready',
      'room_status' => 'occupied',
      'balance_due' => 0,
      'created_at' => $now,
      'updated_at' => $now,
    ];
  }

  /** @return array<int,array<string,mixed>> */
  public static function demoRows(): array
  {
    $file = __DIR__ . '/../config/hotelier-checkouts-demo.json';
    if (!is_file($file)) return [];

    $json = file_get_contents($file);
    $rows = json_decode((string)$json, true);
    return is_array($rows) ? array_values(array_filter($rows, 'is_array')) : [];
  }

  public static function ensureSeeded(): void
  {
    $db = Database::medoo();
    try {
      $count = (int)$db->count('hotelier_checkouts');
      if ($count > 0) return;
    } catch (\Throwable $e) {
      return;
    }

    $rows = self::demoRows();
    foreach ($rows as $row) {
      $db->insert('hotelier_checkouts', [
        'reference' => (string)($row['reference'] ?? ''),
        'guest_name' => (string)($row['guest_name'] ?? ''),
        'user_phone' => (string)($row['phone'] ?? ''),
        'room' => (string)($row['room'] ?? ''),
        'room_no' => (string)($row['room_no'] ?? ''),
        'checkout_time' => (string)($row['checkout_time'] ?? ''),
        'nights' => (int)($row['nights'] ?? 0),
        'guests' => (int)($row['guests'] ?? 1),
        'payment_status' => (string)($row['payment_status'] ?? 'unpaid'),
        'checkout_status' => (string)($row['checkout_status'] ?? 'ready'),
        'room_status' => (string)($row['room_status'] ?? 'occupied'),
        'extras' => (string)($row['extras'] ?? ''),
        'balance_due' => (float)($row['balance_due'] ?? 0),
        'created_at' => date('Y-m-d H:i:s'),
        'updated_at' => date('Y-m-d H:i:s'),
      ]);
    }
  }

  /** @param array<int,array<string,mixed>> $rows */
  public static function statsFromRows(array $rows): array
  {
    $stats = [
      'total' => count($rows),
      'ready' => 0,
      'checked_out' => 0,
      'settlement_due' => 0,
      'balance_due' => 0.0,
    ];

    foreach ($rows as $row) {
      $status = strtolower((string)($row['checkout_status'] ?? ''));
      if ($status === 'ready') $stats['ready']++;
      if ($status === 'checked_out') $stats['checked_out']++;
      if (in_array($status, ['settlement_due', 'payment_due'], true)) $stats['settlement_due']++;
      if (is_numeric($row['balance_due'] ?? null)) {
        $stats['balance_due'] += (float)$row['balance_due'];
      }
    }

    return $stats;
  }

  /** @param array<int,int> $hotelRowIds */
  public static function stats(array $hotelRowIds): array
  {
    if (!$hotelRowIds) {
      return [
        'total' => 0,
        'ready' => 0,
        'checked_out' => 0,
        'settlement_due' => 0,
        'balance_due' => 0.0,
      ];
    }

    return self::statsWhere(['hotel_row_id' => $hotelRowIds]);
  }

  public static function allStats(): array
  {
    return self::statsWhere([]);
  }

  private static function statsWhere(array $where): array
  {
    $db = Database::medoo();
    try {
      return [
        'total' => (int)$db->count('hotelier_checkouts', $where),
        'ready' => (int)$db->count('hotelier_checkouts', array_merge($where, ['checkout_status' => 'ready'])),
        'checked_out' => (int)$db->count('hotelier_checkouts', array_merge($where, ['checkout_status' => 'checked_out'])),
        'settlement_due' => (int)$db->count('hotelier_checkouts', array_merge($where, ['checkout_status' => 'settlement_due'])),
        'balance_due' => (float)($db->sum('hotelier_checkouts', 'balance_due', $where) ?: 0),
      ];
    } catch (\Throwable $e) {
      return [
        'total' => 0,
        'ready' => 0,
        'checked_out' => 0,
        'settlement_due' => 0,
        'balance_due' => 0.0,
      ];
    }
  }
}


app\views\hoteliers\check-ins.php


<?php
/**
 * Hotelier check-ins today module.
 *
 * DB-backed CRUD over hotelier_checkins, scoped to the logged-in hotelier's hotels.
 */

use Bookin\Database;
use Bookin\HotelierCheckinRepo;

$hotelierId = (int)($user['id'] ?? 0);
$ownedHotels = empty($demoMode) ? HotelierCheckinRepo::hotelsForUser($hotelierId) : [];
$hotelRowIds = HotelierCheckinRepo::hotelRowIds($ownedHotels);
$jsonRows = HotelierCheckinRepo::demoRows();
$usingJsonData = !empty($demoMode);
$unscopedDbPreview = !$usingJsonData && !$hotelRowIds;
$stats = $usingJsonData
  ? HotelierCheckinRepo::statsFromRows($jsonRows)
  : ($unscopedDbPreview ? HotelierCheckinRepo::allStats() : HotelierCheckinRepo::stats($hotelRowIds));
$pkr = static fn($n): string => 'PKR ' . number_format((float)$n, 0);

$chip = static function (string $value, string $kind = 'status'): string {
  $value = strtolower(trim($value));
  $map = [
    'ready' => 'bg-emerald-50 text-emerald-700 ring-emerald-600/20',
    'checked_in' => 'bg-blue-50 text-blue-700 ring-blue-600/20',
    'awaiting_guest' => 'bg-slate-100 text-slate-700 ring-slate-500/20',
    'room_preparing' => 'bg-amber-50 text-amber-700 ring-amber-600/20',
    'verified' => 'bg-emerald-50 text-emerald-700 ring-emerald-600/20',
    'pending' => 'bg-amber-50 text-amber-700 ring-amber-600/20',
    'paid' => 'bg-emerald-50 text-emerald-700 ring-emerald-600/20',
    'partial' => 'bg-amber-50 text-amber-700 ring-amber-600/20',
    'unpaid' => 'bg-red-50 text-red-700 ring-red-600/20',
  ];
  $class = $map[$value] ?? 'bg-slate-100 text-slate-700 ring-slate-500/20';
  $label = $value !== '' ? ucwords(str_replace('_', ' ', $value)) : ucfirst($kind);
  return '<span class="inline-flex rounded-full px-2 py-1 text-xs font-semibold ring-1 ring-inset ' . $class . '">' . htmlspecialchars($label) . '</span>';
};
?>

<div class="mb-6 flex flex-col sm:flex-row sm:items-end sm:justify-between gap-3">
  <div>
    <h1 class="text-2xl font-extrabold text-ink">Check-ins Today</h1>
    <p class="text-sm text-muted mt-1">DB-backed front-desk check-in list rendered with the CRUD library.</p>
  </div>
</div>

<?php if ($usingJsonData): ?>
  <div class="mb-6 rounded-2xl border border-amber-200 bg-amber-50 p-4">
    <p class="text-sm font-semibold text-amber-900">Demo mode is active</p>
    <p class="text-sm text-amber-800 mt-1">
      This page is using sample check-in data from
      <code class="text-xs bg-white px-1 py-0.5 rounded">app/config/hotelier-checkins-demo.json</code> because hotelier demo mode is enabled.
    </p>
  </div>
<?php elseif ($unscopedDbPreview): ?>
  <div class="mb-6 rounded-2xl border border-blue-200 bg-blue-50 p-4">
    <p class="text-sm font-semibold text-blue-900">Showing DB check-ins without hotelier scope</p>
    <p class="text-sm text-blue-800 mt-1">
      This hotelier account has no linked hotel yet, so the table below is using records from
      <code class="text-xs bg-white px-1 py-0.5 rounded">hotelier_checkins</code> with no hotelier filter.
      Link a hotel in Admin > Hotels > Hotelier to enable scoped CRUD.
    </p>
  </div>
<?php endif; ?>

<div class="grid grid-cols-1 sm:grid-cols-2 xl:grid-cols-5 gap-4 mb-6">
  <div class="bg-white rounded-2xl border border-slate-200 p-4">
    <p class="text-xs font-bold uppercase tracking-wide text-muted">Arrivals</p>
    <p class="text-2xl font-extrabold text-ink mt-1"><?= number_format($stats['total']) ?></p>
  </div>
  <div class="bg-white rounded-2xl border border-slate-200 p-4">
    <p class="text-xs font-bold uppercase tracking-wide text-muted">Ready</p>
    <p class="text-2xl font-extrabold text-emerald-600 mt-1"><?= number_format($stats['ready']) ?></p>
  </div>
  <div class="bg-white rounded-2xl border border-slate-200 p-4">
    <p class="text-xs font-bold uppercase tracking-wide text-muted">Checked In</p>
    <p class="text-2xl font-extrabold text-blue-600 mt-1"><?= number_format($stats['checked_in']) ?></p>
  </div>
  <div class="bg-white rounded-2xl border border-slate-200 p-4">
    <p class="text-xs font-bold uppercase tracking-wide text-muted">ID Pending</p>
    <p class="text-2xl font-extrabold text-amber-600 mt-1"><?= number_format($stats['pending_id']) ?></p>
  </div>
  <div class="bg-white rounded-2xl border border-slate-200 p-4">
    <p class="text-xs font-bold uppercase tracking-wide text-muted">Balance Due</p>
    <p class="text-2xl font-extrabold text-ink mt-1"><?= htmlspecialchars($pkr($stats['balance_due'])) ?></p>
  </div>
</div>

<?php
require_once __DIR__ . '/../../lib/crud.php';

$crud = new \CRUD();
if ($usingJsonData) {
  $crud->data($jsonRows);
} else {
  $GLOBALS['db'] = Database::medoo();
  $crud->table('hotelier_checkins');
  if (!$unscopedDbPreview) {
    $crud
      ->create_defaults(HotelierCheckinRepo::createDefaults($ownedHotels))
      ->where(['hotel_row_id' => $hotelRowIds])
      ->form_col('guest_name,user_email,user_phone,room,room_no,checkin_date,arrival_time,nights,guests,id_status,payment_status,checkin_status,balance_due,notes')
      ->redirect_url(hotelier_url('check-ins'))
      ->action_urls([
        'add' => 'hoteliers/check-ins/create',
        'edit' => hotelier_url('check-ins') . '/edit/{id}',
      ]);
  }
}

$crud
  ->title($usingJsonData ? 'Today Check-ins - JSON Preview' : 'Today Check-ins')
  ->id_column('id')
  ->col('reference,guest_name,user_email,user_phone,room,room_no,arrival_time,nights,guests,id_status,payment_status,checkin_status,balance_due,notes')
  ->order('arrival_time', 'ASC')
  ->perPage(25)
  ->actions([
    'add' => !$usingJsonData && !$unscopedDbPreview,
    'status' => false,
    'view' => false,
    'edit' => !$usingJsonData && !$unscopedDbPreview,
    'delete' => false,
    'bulk_delete' => false,
    'search' => true,
  ])
  ->label([
    'reference' => 'Reference',
    'guest_name' => 'Guest',
    'user_email' => 'Email',
    'user_phone' => 'Phone',
    'room_no' => 'Room #',
    'arrival_time' => 'Arrival',
    'id_status' => 'ID',
    'payment_status' => 'Payment',
    'checkin_status' => 'Check-in',
    'balance_due' => 'Balance',
  ])
  ->row([
    'reference' => static fn(array $row): string => '<span class="font-semibold text-ink">' . htmlspecialchars((string)($row['reference'] ?? '')) . '</span>',
    'guest_name' => static function (array $row): string {
      return '<div class="font-semibold text-ink">' . htmlspecialchars((string)($row['guest_name'] ?? 'Guest')) . '</div>'
        . '<div class="text-xs text-muted">' . htmlspecialchars((string)($row['user_phone'] ?? $row['phone'] ?? '')) . '</div>';
    },
    'user_email' => static fn(array $row): string => htmlspecialchars((string)($row['user_email'] ?? '')),
    'user_phone' => static fn(array $row): string => htmlspecialchars((string)($row['user_phone'] ?? $row['phone'] ?? '')),
    'id_status' => static fn(array $row): string => $chip((string)($row['id_status'] ?? ''), 'id'),
    'payment_status' => static fn(array $row): string => $chip((string)($row['payment_status'] ?? ''), 'payment'),
    'checkin_status' => static fn(array $row): string => $chip((string)($row['checkin_status'] ?? ''), 'check-in'),
    'balance_due' => static fn(array $row): string => '<span class="font-semibold text-ink">' . htmlspecialchars($pkr($row['balance_due'] ?? 0)) . '</span>',
  ]);

echo $crud->render();
?>


app\views\hoteliers\check-outs.php

<?php
/**
 * Hotelier check-outs today module.
 *
 * DB-backed CRUD over hotelier_checkouts, scoped to the logged-in hotelier's hotels.
 */

use Bookin\Database;
use Bookin\HotelierCheckoutRepo;

$hotelierId = (int)($user['id'] ?? 0);
$ownedHotels = empty($demoMode) ? HotelierCheckoutRepo::hotelsForUser($hotelierId) : [];
$hotelRowIds = HotelierCheckoutRepo::hotelRowIds($ownedHotels);
$usingJsonData = !empty($demoMode);
$unscopedDbPreview = !$usingJsonData && !$hotelRowIds;

if (!$usingJsonData) {
  HotelierCheckoutRepo::ensureSeeded();
}

$jsonRows = HotelierCheckoutRepo::demoRows();
$stats = $usingJsonData
  ? HotelierCheckoutRepo::statsFromRows($jsonRows)
  : ($unscopedDbPreview ? HotelierCheckoutRepo::allStats() : HotelierCheckoutRepo::stats($hotelRowIds));

$pkr = static fn($n): string => 'PKR ' . number_format((float)$n, 0);

$chip = static function (string $value, string $kind = 'status'): string {
  $value = strtolower(trim($value));
  $map = [
    'ready' => 'bg-emerald-50 text-emerald-700 ring-emerald-600/20',
    'checked_out' => 'bg-blue-50 text-blue-700 ring-blue-600/20',
    'settlement_due' => 'bg-amber-50 text-amber-700 ring-amber-600/20',
    'payment_due' => 'bg-red-50 text-red-700 ring-red-600/20',
    'paid' => 'bg-emerald-50 text-emerald-700 ring-emerald-600/20',
    'partial' => 'bg-amber-50 text-amber-700 ring-amber-600/20',
    'unpaid' => 'bg-red-50 text-red-700 ring-red-600/20',
    'inspection_pending' => 'bg-amber-50 text-amber-700 ring-amber-600/20',
    'occupied' => 'bg-slate-100 text-slate-700 ring-slate-500/20',
    'cleaning' => 'bg-blue-50 text-blue-700 ring-blue-600/20',
  ];
  $class = $map[$value] ?? 'bg-slate-100 text-slate-700 ring-slate-500/20';
  $label = $value !== '' ? ucwords(str_replace('_', ' ', $value)) : ucfirst($kind);
  return '<span class="inline-flex rounded-full px-2 py-1 text-xs font-semibold ring-1 ring-inset ' . $class . '">' . htmlspecialchars($label) . '</span>';
};
?>

<div class="mb-6 flex flex-col sm:flex-row sm:items-end sm:justify-between gap-3">
  <div>
    <h1 class="text-2xl font-extrabold text-ink">Check-outs Today</h1>
    <p class="text-sm text-muted mt-1">DB-backed front-desk checkout list rendered with the CRUD library.</p>
  </div>
</div>

<?php if ($usingJsonData): ?>
  <div class="mb-6 rounded-2xl border border-amber-200 bg-amber-50 p-4">
    <p class="text-sm font-semibold text-amber-900">Demo mode is active</p>
    <p class="text-sm text-amber-800 mt-1">
      This page is using sample checkout data from
      <code class="text-xs bg-white px-1 py-0.5 rounded">app/config/hotelier-checkouts-demo.json</code> because hotelier demo mode is enabled.
    </p>
  </div>
<?php elseif ($unscopedDbPreview): ?>
  <div class="mb-6 rounded-2xl border border-blue-200 bg-blue-50 p-4">
    <p class="text-sm font-semibold text-blue-900">Showing DB check-outs without hotelier scope</p>
    <p class="text-sm text-blue-800 mt-1">
      This hotelier account has no linked hotel yet, so the table below is using records from
      <code class="text-xs bg-white px-1 py-0.5 rounded">hotelier_checkouts</code> with no hotelier filter.
      Link a hotel in Admin > Hotels > Hotelier to enable scoped CRUD.
    </p>
  </div>
<?php endif; ?>

<div class="grid grid-cols-1 sm:grid-cols-2 xl:grid-cols-5 gap-4 mb-6">
  <div class="bg-white rounded-2xl border border-slate-200 p-4">
    <p class="text-xs font-bold uppercase tracking-wide text-muted">Departures</p>
    <p class="text-2xl font-extrabold text-ink mt-1"><?= number_format($stats['total']) ?></p>
  </div>
  <div class="bg-white rounded-2xl border border-slate-200 p-4">
    <p class="text-xs font-bold uppercase tracking-wide text-muted">Ready</p>
    <p class="text-2xl font-extrabold text-emerald-600 mt-1"><?= number_format($stats['ready']) ?></p>
  </div>
  <div class="bg-white rounded-2xl border border-slate-200 p-4">
    <p class="text-xs font-bold uppercase tracking-wide text-muted">Checked Out</p>
    <p class="text-2xl font-extrabold text-blue-600 mt-1"><?= number_format($stats['checked_out']) ?></p>
  </div>
  <div class="bg-white rounded-2xl border border-slate-200 p-4">
    <p class="text-xs font-bold uppercase tracking-wide text-muted">Settlement Due</p>
    <p class="text-2xl font-extrabold text-amber-600 mt-1"><?= number_format($stats['settlement_due']) ?></p>
  </div>
  <div class="bg-white rounded-2xl border border-slate-200 p-4">
    <p class="text-xs font-bold uppercase tracking-wide text-muted">Balance Due</p>
    <p class="text-2xl font-extrabold text-ink mt-1"><?= htmlspecialchars($pkr($stats['balance_due'])) ?></p>
  </div>
</div>

<?php
require_once __DIR__ . '/../../lib/crud.php';

$crud = new \CRUD();
if ($usingJsonData) {
  $crud->data($jsonRows);
} else {
  $GLOBALS['db'] = Database::medoo();
  $crud->table('hotelier_checkouts');
  if (!$unscopedDbPreview) {
    $crud
      ->create_defaults(HotelierCheckoutRepo::createDefaults($ownedHotels))
      ->where(['hotel_row_id' => $hotelRowIds])
      ->form_col('guest_name,user_email,user_phone,room,room_no,checkout_date,checkout_time,nights,guests,payment_status,checkout_status,room_status,extras,balance_due,notes')
      ->redirect_url(hotelier_url('check-outs'))
      ->action_urls([
        'add' => 'hoteliers/check-outs/create',
        'edit' => hotelier_url('check-outs') . '/edit/{id}',
      ]);
  }
}

$crud
  ->title($usingJsonData ? 'Today Check-outs - JSON Preview' : 'Today Check-outs')
  ->id_column('id')
  ->col('reference,guest_name,user_email,user_phone,room,room_no,checkout_time,nights,guests,payment_status,checkout_status,room_status,balance_due,extras')
  ->order('checkout_time', 'ASC')
  ->perPage(25)
  ->actions([
    'add' => !$usingJsonData && !$unscopedDbPreview,
    'status' => false,
    'view' => false,
    'edit' => !$usingJsonData && !$unscopedDbPreview,
    'delete' => false,
    'bulk_delete' => false,
    'search' => true,
  ])
  ->label([
    'reference' => 'Reference',
    'guest_name' => 'Guest',
    'user_email' => 'Email',
    'user_phone' => 'Phone',
    'room_no' => 'Room #',
    'checkout_time' => 'Checkout',
    'payment_status' => 'Payment',
    'checkout_status' => 'Checkout',
    'room_status' => 'Room Status',
    'balance_due' => 'Balance',
  ])
  ->row([
    'reference' => static fn(array $row): string => '<span class="font-semibold text-ink">' . htmlspecialchars((string)($row['reference'] ?? '')) . '</span>',
    'guest_name' => static function (array $row): string {
      return '<div class="font-semibold text-ink">' . htmlspecialchars((string)($row['guest_name'] ?? 'Guest')) . '</div>'
        . '<div class="text-xs text-muted">' . htmlspecialchars((string)($row['user_phone'] ?? $row['phone'] ?? '')) . '</div>';
    },
    'user_email' => static fn(array $row): string => htmlspecialchars((string)($row['user_email'] ?? '')),
    'user_phone' => static fn(array $row): string => htmlspecialchars((string)($row['user_phone'] ?? $row['phone'] ?? '')),
    'payment_status' => static fn(array $row): string => $chip((string)($row['payment_status'] ?? ''), 'payment'),
    'checkout_status' => static fn(array $row): string => $chip((string)($row['checkout_status'] ?? ''), 'checkout'),
    'room_status' => static fn(array $row): string => $chip((string)($row['room_status'] ?? ''), 'room'),
    'balance_due' => static fn(array $row): string => '<span class="font-semibold text-ink">' . htmlspecialchars($pkr($row['balance_due'] ?? 0)) . '</span>',
  ]);

echo $crud->render();
?>

