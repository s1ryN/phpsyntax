# README

Tento dokument popisuje základní funkcionalitu pro PHP backend:
- Hashování hesel
- CRUD (Create, Read, Update, Delete) s MySQLi
- Výpis z databáze
- Zpracování a výpis chyb při přihlášení

## Požadavky
- PHP >= 7.4
- MySQL nebo MariaDB
- Rozšíření MySQLi

## Konfigurace připojení k DB
Ukázka souboru `config.php` (object-oriented MySQLi):
```php
<?php
$host = '127.0.0.1';
$db   = 'nazev_databaze';
$user = 'uzivatel';
$pass = 'heslo';
$port = 3306;

// Aktivace reportingu chyb MySQLi
mysqli_report(MYSQLI_REPORT_ERROR | MYSQLI_REPORT_STRICT);

$mysqli = new mysqli($host, $user, $pass, $db, $port);
$mysqli->set_charset('utf8mb4');

if ($mysqli->connect_errno) {
    die('Chyba připojení k DB: ' . $mysqli->connect_error);
}
```

## Hashování hesel
Použijte `password_hash` a `password_verify`:
```php
// Registrace
$heslo = $_POST['heslo'];
$hash = password_hash($heslo, PASSWORD_DEFAULT);
// Uložení $hash do DB...
```
```php
// Přihlášení
if (password_verify($zadaneHeslo, $ulozenyHash)) {
    // OK
} else {
    // Chybné heslo
}
```

## CRUD operace
### Vytvoření (Create)
```php
$stmt = $mysqli->prepare("INSERT INTO uzivatele (jmeno, email, heslo) VALUES (?, ?, ?)");
$stmt->bind_param('sss', $jmeno, $email, $hash);
$stmt->execute();
$stmt->close();
```
### Čtení (Read)
```php
$stmt = $mysqli->prepare("SELECT * FROM uzivatele WHERE id = ?");
$stmt->bind_param('i', $id);
$stmt->execute();
$result = $stmt->get_result();
$user = $result->fetch_assoc();
$stmt->close();
```
### Aktualizace (Update)
```php
$stmt = $mysqli->prepare("UPDATE uzivatele SET email = ? WHERE id = ?");
$stmt->bind_param('si', $novyEmail, $id);
$stmt->execute();
$stmt->close();
```
### Mazání (Delete)
```php
$stmt = $mysqli->prepare("DELETE FROM uzivatele WHERE id = ?");
$stmt->bind_param('i', $id);
$stmt->execute();
$stmt->close();
```

## Výpis z databáze
```php
$query = "SELECT id, jmeno, email FROM uzivatele";
if ($result = $mysqli->query($query)) {
    while ($row = $result->fetch_assoc()) {
        echo $row['id'] . ' - ' . $row['jmeno'] . ' (' . $row['email'] . ')<br>';
    }
    $result->free();
}
```

## Zpracování chyb při přihlášení
```php
try {
    $stmt = $mysqli->prepare("SELECT heslo FROM uzivatele WHERE email = ?");
    $stmt->bind_param('s', $email);
    $stmt->execute();
    $result = $stmt->get_result();
    $user = $result->fetch_assoc();
    $stmt->close();

    if (!$user) {
        throw new Exception('Uživatel nenalezen.');
    }
    if (!password_verify($heslo, $user['heslo'])) {
        throw new Exception('Neplatné heslo.');
    }
    // Přihlášení úspěšné
} catch (Exception $e) {
    echo 'Chyba: ' . $e->getMessage();
}
```

## Dodatečné poznámky
- Vždy ošetřujte vstupy od uživatele (např. pomocí `filter_input`)
- Používejte HTTPS pro přenos dat
- V produkci zakazujte zobrazování chyb (`display_errors = Off`)


## Příklady funkcionality

### Registrace uživatele
```php
function registerUser($mysqli, $jmeno, $email, $heslo) {
    $hash = password_hash($heslo, PASSWORD_DEFAULT);
    $stmt = $mysqli->prepare("INSERT INTO uzivatele (jmeno, email, heslo) VALUES (?, ?, ?)");
    $stmt->bind_param('sss', $jmeno, $email, $hash);
    $stmt->execute();
    $stmt->close();
}
```

### Přihlášení uživatele
```php
function loginUser($mysqli, $email, $heslo) {
    $stmt = $mysqli->prepare("SELECT id, heslo FROM uzivatele WHERE email = ?");
    $stmt->bind_param('s', $email);
    $stmt->execute();
    $result = $stmt->get_result();
    $user = $result->fetch_assoc();
    $stmt->close();
    if (!$user) {
        throw new Exception('Uživatel nenalezen.');
    }
    if (!password_verify($heslo, $user['heslo'])) {
        throw new Exception('Neplatné heslo.');
    }
    return $user['id'];
}
```

### Změna hesla
```php
function changePassword($mysqli, $userId, $oldPassword, $newPassword) {
    // Získání současného hashe
    $stmt = $mysqli->prepare("SELECT heslo FROM uzivatele WHERE id = ?");
    $stmt->bind_param('i', $userId);
    $stmt->execute();
    $result = $stmt->get_result();
    $user = $result->fetch_assoc();
    $stmt->close();
    if (!password_verify($oldPassword, $user['heslo'])) {
        throw new Exception('Staré heslo nesouhlasí.');
    }
    $newHash = password_hash($newPassword, PASSWORD_DEFAULT);
    $stmt = $mysqli->prepare("UPDATE uzivatele SET heslo = ? WHERE id = ?");
    $stmt->bind_param('si', $newHash, $userId);
    $stmt->execute();
    $stmt->close();
}
```

### Odstranění uživatele
```php
function deleteUser($mysqli, $userId) {
    $stmt = $mysqli->prepare("DELETE FROM uzivatele WHERE id = ?");
    $stmt->bind_param('i', $userId);
    $stmt->execute();
    $stmt->close();
}
```


## Cookies a session

### Inicializace session
```php
<?php
session_start();
```

### Nastavení a čtení session proměnných
```php
// Nastavení
$_SESSION['user_id'] = $userId;

// Čtení
if (isset($_SESSION['user_id'])) {
    // Uživatel přihlášen
}
```

### Odhlášení uživatele
```php
function logoutUser() {
    session_start();
    session_unset();
    session_destroy();
    // Vymazání cookie remember-me
    setcookie('rememberme', '', time() - 3600, '/', '', true, true);
}
```

### Nastavení cookies (např. "remember me")
```php
// Vytvoření tokenu a uložení do DB...
setcookie('rememberme', $token, time() + (86400 * 30), '/', '', true, true);
```

### Čtení cookies
```php
if (isset($_COOKIE['rememberme'])) {
    $token = $_COOKIE['rememberme'];
    // Hledání v DB a automatické přihlášení...
}
```

### Vymazání cookies
```php
setcookie('rememberme', '', time() - 3600, '/', '', true, true);
```
