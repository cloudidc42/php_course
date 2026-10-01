# Part 01: PHP คืออะไร + การติดตั้ง Environment
## ขั้นตอนที่ 1-30: เริ่มต้นกับ PHP

---

## 📋 สิ่งที่จะได้เรียนในบทนี้
- PHP คืออะไร และทำงานอย่างไร
- การติดตั้ง PHP บน Windows, macOS, Linux
- การตั้งค่า Development Environment
- เขียนโปรแกรม PHP แรกของคุณ
- โครงสร้างพื้นฐานของ PHP
- การแสดงผลข้อมูลแบบต่างๆ

---

## ขั้นตอนที่ 1: PHP คืออะไร?

PHP (PHP: Hypertext Preprocessor) คือภาษาสคริปต์ที่ทำงานฝั่ง Server (Server-side scripting language) ที่ถูกออกแบบมาเพื่อการพัฒนาเว็บโดยเฉพาะ

### ประวัติของ PHP
- **1994**: Rasmus Lerdorf สร้าง PHP/FI (Personal Home Page/Form Interpreter)
- **1997**: PHP 3.0 เปิดตัว ทีมนักพัฒนาเริ่มเข้ามาร่วม
- **2000**: PHP 4.0 ใช้ Zend Engine
- **2004**: PHP 5.0 รองรับ OOP อย่างเต็มรูปแบบ
- **2015**: PHP 7.0 เร็วขึ้น 2x จาก PHP 5
- **2020**: PHP 8.0 JIT Compiler, Named Arguments, Match expression
- **2023**: PHP 8.3 ล่าสุด (Typed class constants, Override attribute)

### PHP ทำงานอย่างไร?

```
Browser → HTTP Request → Web Server (Apache/Nginx)
                              ↓
                         PHP Interpreter
                              ↓
                    Database (MySQL/PostgreSQL)
                              ↓
                         PHP Interpreter
                              ↓
                      HTML Response ← Web Server ← Browser
```

### ทำไมต้องเรียน PHP?
1. **ง่ายต่อการเรียนรู้** - Syntax เข้าใจง่าย เหมาะสำหรับผู้เริ่มต้น
2. **ใช้งานอย่างแพร่หลาย** - 77% ของเว็บไซต์ทั่วโลกใช้ PHP (รวม WordPress)
3. **Community ใหญ่** - มีเอกสารและ Tutorial มากมาย
4. **Framework ที่ดี** - Laravel, Symfony, CodeIgniter
5. **รายได้ดี** - PHP Developer มีความต้องการสูงในตลาด

---

## ขั้นตอนที่ 2: การติดตั้ง PHP บน Windows

### วิธีที่ 1: ใช้ XAMPP (แนะนำสำหรับผู้เริ่มต้น)

XAMPP คือ Package ที่รวม Apache, MySQL, PHP, phpMyAdmin ไว้ด้วยกัน

1. ดาวน์โหลด XAMPP จาก [https://www.apachefriends.org](https://www.apachefriends.org)
2. เลือกเวอร์ชัน PHP 8.x
3. ติดตั้งตาม Wizard
4. เปิด XAMPP Control Panel
5. Start Apache และ MySQL

```bash
# ตรวจสอบ PHP Version
php -v

# ผลลัพธ์ที่ควรได้:
# PHP 8.3.x (cli) (built: ...)
# Copyright (c) The PHP Group
# Zend Engine v4.3.x
```

### วิธีที่ 2: ใช้ Laragon (แนะนำ)

Laragon เป็นทางเลือกที่ดีกว่า XAMPP สำหรับ Windows

```bash
# 1. ดาวน์โหลดจาก https://laragon.org
# 2. ติดตั้งและเปิด Laragon
# 3. คลิก Start All
# 4. PHP พร้อมใช้งาน

# ตรวจสอบ
php -v
composer -v
```

### วิธีที่ 3: ใช้ WSL2 + Ubuntu (สำหรับมืออาชีพ)

```bash
# เปิด PowerShell as Administrator
wsl --install -d Ubuntu-22.04

# หลังจาก Restart เปิด Ubuntu แล้วรัน:
sudo apt update && sudo apt upgrade -y
sudo apt install php8.3 php8.3-cli php8.3-fpm php8.3-mysql \
    php8.3-xml php8.3-curl php8.3-zip php8.3-gd php8.3-mbstring \
    php8.3-intl php8.3-bcmath -y

# ติดตั้ง Composer
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer

# ตรวจสอบ
php -v
composer -v
```

---

## ขั้นตอนที่ 3: การติดตั้ง PHP บน macOS

### วิธีที่ 1: ใช้ Homebrew (แนะนำ)

```bash
# ติดตั้ง Homebrew ก่อน (ถ้ายังไม่มี)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ติดตั้ง PHP
brew install php

# ติดตั้ง Extensions เพิ่มเติม
brew install php-mysql
brew install php-redis

# ตรวจสอบ
php -v
which php
```

### วิธีที่ 2: ใช้ Laravel Herd (แนะนำสำหรับ macOS)

```bash
# ดาวน์โหลดจาก https://herd.laravel.com
# ติดตั้งแล้วเปิดใช้งาน
# รองรับ PHP 8.x และ Laravel โดยตรง

# ตรวจสอบ
herd php -v
```

### วิธีที่ 3: ใช้ Valet

```bash
# ติดตั้ง Valet
composer global require laravel/valet
valet install

# Park folder
cd ~/Sites
valet park

# ตรวจสอบ
valet --version
```

---

## ขั้นตอนที่ 4: การติดตั้ง PHP บน Ubuntu/Linux

```bash
# อัพเดท Package list
sudo apt update

# ติดตั้ง PHP 8.3
sudo apt install software-properties-common -y
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update

# ติดตั้ง PHP และ Extensions ที่จำเป็น
sudo apt install php8.3 php8.3-cli php8.3-fpm \
    php8.3-mysql php8.3-pgsql \
    php8.3-xml php8.3-curl php8.3-zip \
    php8.3-gd php8.3-mbstring php8.3-intl \
    php8.3-bcmath php8.3-redis php8.3-imagick -y

# ติดตั้ง Apache
sudo apt install apache2 libapache2-mod-php8.3 -y
sudo a2enmod rewrite
sudo systemctl restart apache2

# ติดตั้ง MySQL
sudo apt install mysql-server mysql-client -y
sudo mysql_secure_installation

# ติดตั้ง Composer
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
sudo chmod +x /usr/local/bin/composer

# ตรวจสอบทั้งหมด
php -v
mysql --version
composer -v
apache2 -v
```

---

## ขั้นตอนที่ 5: การตั้งค่า VS Code สำหรับ PHP

### Extensions ที่แนะนำ

```json
// .vscode/extensions.json
{
    "recommendations": [
        "bmewburn.vscode-intelephense-client",
        "xdebug.php-debug",
        "recca0120.vscode-phpunit",
        "onecentlin.laravel5-snippets",
        "amiralizadeh9480.laravel-extra-intellisense",
        "mikestead.dotenv",
        "formulahendry.auto-close-tag",
        "esbenp.prettier-vscode",
        "dbaeumer.vscode-eslint",
        "github.copilot"
    ]
}
```

### การตั้งค่า VS Code สำหรับ PHP

```json
// settings.json
{
    "editor.tabSize": 4,
    "editor.insertSpaces": true,
    "editor.formatOnSave": true,
    "php.validate.executablePath": "/usr/bin/php",
    "intelephense.environment.phpVersion": "8.3.0",
    "intelephense.files.maxSize": 5000000,
    "php.suggest.basic": false,
    "[php]": {
        "editor.defaultFormatter": "bmewburn.vscode-intelephense-client"
    }
}
```

### การตั้งค่า Xdebug

```ini
; /etc/php/8.3/cli/conf.d/20-xdebug.ini
[xdebug]
zend_extension=xdebug.so
xdebug.mode=debug
xdebug.start_with_request=yes
xdebug.client_host=127.0.0.1
xdebug.client_port=9003
xdebug.idekey=VSCODE
```

```json
// .vscode/launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Listen for Xdebug",
            "type": "php",
            "request": "launch",
            "port": 9003
        },
        {
            "name": "Launch currently open script",
            "type": "php",
            "request": "launch",
            "program": "${file}",
            "cwd": "${fileDirname}",
            "port": 9003,
            "runtimeArgs": [
                "-dxdebug.start_with_request=yes"
            ],
            "env": {
                "XDEBUG_SESSION": "VSCODE"
            }
        }
    ]
}
```

---

## ขั้นตอนที่ 6: โปรแกรม PHP แรกของคุณ

### 6.1 โครงสร้างพื้นฐาน PHP File

```php
<?php
// นี่คือ PHP ไฟล์แรกของคุณ!
// PHP Tags เริ่มต้นด้วย <?php และจบด้วย ?>

echo "Hello, World!";  // แสดงข้อความ
echo PHP_EOL;          // ขึ้นบรรทัดใหม่

// หรือใช้ print
print("Hello from PHP!");
print(PHP_EOL);

// PHP_EOL = End of Line (\n บน Linux, \r\n บน Windows)
?>
```

### 6.2 การรัน PHP

```bash
# รันผ่าน Command Line
php hello.php

# รันผ่าน Built-in Server
php -S localhost:8000

# รันผ่าน Web Server (วางไฟล์ใน /var/www/html/)
# แล้วเปิด http://localhost/hello.php
```

### 6.3 Hello World แบบ HTML

```php
<?php
// ไฟล์: index.php
$name = "PHP Developer";
$version = phpversion();
?>
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Hello PHP!</title>
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            max-width: 800px;
            margin: 50px auto;
            padding: 20px;
            background: #f0f4f8;
        }
        .card {
            background: white;
            padding: 30px;
            border-radius: 10px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
        }
        h1 { color: #4a90e2; }
        .badge {
            display: inline-block;
            background: #4a90e2;
            color: white;
            padding: 4px 12px;
            border-radius: 20px;
            font-size: 14px;
        }
    </style>
</head>
<body>
    <div class="card">
        <h1>สวัสดี, <?php echo htmlspecialchars($name); ?>!</h1>
        <p>ยินดีต้อนรับสู่โลกของ PHP</p>
        <p>คุณกำลังใช้ PHP Version: 
            <span class="badge"><?php echo $version; ?></span>
        </p>
        <p>วันที่วันนี้: <?php echo date('Y-m-d H:i:s'); ?></p>
    </div>
</body>
</html>
```

---

## ขั้นตอนที่ 7: การแสดงผลข้อมูล

### 7.1 echo และ print

```php
<?php
// echo - แสดงผลหนึ่งหรือหลายค่า (ไม่มี return value)
echo "Hello, World!";
echo "Hello", " ", "World"; // แสดงหลายค่าพร้อมกัน
echo "<br>";                 // HTML tag

// print - แสดงผลค่าเดียว (return 1 เสมอ)
print "Hello PHP!";
print("<p>HTML with print</p>");

// echo ทำงานเร็วกว่า print เล็กน้อย
// แต่ในทางปฏิบัติไม่มีผลต่อ Performance

// Short echo tag
$name = "PHP";
?>
<p>Hello <?= $name ?>!</p>  <!-- เทียบเท่า <?php echo $name ?> -->

<?php
// var_dump - แสดงข้อมูลแบบละเอียด (type + value)
$number = 42;
$text = "Hello";
$decimal = 3.14;
$boolean = true;
$null_val = null;
$array = [1, 2, 3];

var_dump($number);    // int(42)
var_dump($text);      // string(5) "Hello"
var_dump($decimal);   // float(3.14)
var_dump($boolean);   // bool(true)
var_dump($null_val);  // NULL
var_dump($array);     // array(3) { [0]=> int(1) [1]=> int(2) [2]=> int(3) }

echo "<br>";

// print_r - แสดงข้อมูลแบบ Human-readable
print_r($array);
/*
Array
(
    [0] => 1
    [1] => 2
    [2] => 3
)
*/

// var_export - แสดงข้อมูลในรูปแบบที่ PHP Code ใช้ได้
var_export($array);  // array ( 0 => 1, 1 => 2, 2 => 3, )

// การ Debug แบบสวยงาม
echo "<pre>";
print_r($array);
echo "</pre>";
```

### 7.2 printf และ sprintf

```php
<?php
// printf - แสดงผลแบบ Formatted
printf("สวัสดี %s คุณอายุ %d ปี\n", "สมชาย", 25);
printf("ราคา: %.2f บาท\n", 1234.5);
printf("เลขฐาน 16: %x\n", 255);    // ff
printf("เลขฐาน 2: %b\n", 10);      // 1010

// sprintf - เหมือน printf แต่ return string แทนการแสดงผล
$message = sprintf("ยินดีต้อนรับ %s!", "คุณสมชาย");
echo $message . "\n";

$price = sprintf("%.2f", 1234.5);
echo "ราคา: $price บาท\n";

// Format Specifiers
// %s = string
// %d = integer (decimal)
// %f = float
// %e = scientific notation
// %x = hexadecimal lowercase
// %X = hexadecimal uppercase
// %b = binary
// %o = octal
// %u = unsigned integer
// %% = literal %

// Padding
printf("%010d\n", 42);       // 0000000042 (pad with zeros)
printf("%-10s|\n", "left");  // left       | (left align)
printf("%10s|\n", "right");  //      right | (right align)

// Number formatting
$amount = 1234567.89;
echo number_format($amount) . "\n";           // 1,234,568
echo number_format($amount, 2) . "\n";        // 1,234,567.89
echo number_format($amount, 2, '.', ',') . "\n"; // 1,234,567.89
echo number_format($amount, 2, ',', '.') . "\n"; // 1.234.567,89 (European)
```

---

## ขั้นตอนที่ 8: PHP Configuration (php.ini)

```bash
# หา php.ini
php --ini

# ดู PHP Configuration
php -i | head -50
php -i | grep "php.ini"
```

### php.ini สำคัญที่ควรรู้

```ini
; /etc/php/8.3/cli/php.ini

; ========================
; Error Reporting
; ========================
; Development (แสดง Error ทุกอย่าง)
error_reporting = E_ALL
display_errors = On
display_startup_errors = On

; Production (ซ่อน Error จาก User)
; error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT
; display_errors = Off
; log_errors = On
; error_log = /var/log/php/error.log

; ========================
; Performance
; ========================
max_execution_time = 30      ; วินาที
max_input_time = 60          ; วินาที
memory_limit = 256M          ; Memory สูงสุด
post_max_size = 64M          ; ขนาด POST สูงสุด
upload_max_filesize = 64M    ; ขนาด Upload สูงสุด

; ========================
; Date & Time
; ========================
date.timezone = Asia/Bangkok

; ========================
; Session
; ========================
session.save_path = /var/lib/php/sessions
session.cookie_httponly = On
session.cookie_secure = On    ; เฉพาะ HTTPS
session.use_strict_mode = On

; ========================
; OPcache (Performance)
; ========================
opcache.enable = 1
opcache.memory_consumption = 128
opcache.interned_strings_buffer = 8
opcache.max_accelerated_files = 10000
opcache.revalidate_freq = 2
opcache.fast_shutdown = 1
```

---

## ขั้นตอนที่ 9: การใช้ PHP Built-in Functions พื้นฐาน

```php
<?php
// ========================
// PHP Version และ Info
// ========================
echo phpversion() . "\n";          // "8.3.x"
echo PHP_VERSION . "\n";           // "8.3.x"
echo PHP_MAJOR_VERSION . "\n";     // 8
echo PHP_MINOR_VERSION . "\n";     // 3
echo PHP_OS . "\n";                // Linux/Darwin/Windows
echo PHP_EOL;                      // ขึ้นบรรทัดใหม่ตาม OS
echo PHP_INT_MAX . "\n";           // 9223372036854775807
echo PHP_INT_MIN . "\n";           // -9223372036854775808
echo PHP_FLOAT_MAX . "\n";         // 1.7976931348623E+308
echo PHP_FLOAT_EPSILON . "\n";     // 2.2204460492503E-16

// ตรวจสอบ Extension
var_dump(extension_loaded('pdo'));    // bool(true)
var_dump(extension_loaded('gd'));     // bool(true)
print_r(get_loaded_extensions());     // รายการ Extensions ทั้งหมด

// ========================
// Math Functions
// ========================
echo abs(-42) . "\n";          // 42
echo ceil(4.1) . "\n";         // 5
echo floor(4.9) . "\n";        // 4
echo round(4.5) . "\n";        // 5
echo round(4.55, 1) . "\n";    // 4.6
echo max(1, 5, 3) . "\n";      // 5
echo min(1, 5, 3) . "\n";      // 1
echo pow(2, 10) . "\n";        // 1024
echo sqrt(16) . "\n";          // 4
echo log(M_E) . "\n";          // 1
echo M_PI . "\n";               // 3.1415926535898

// Random Numbers
echo rand() . "\n";            // Random int
echo rand(1, 100) . "\n";      // Random int 1-100
echo mt_rand(1, 100) . "\n";   // Mersenne Twister (เร็วกว่า)
echo random_int(1, 100) . "\n"; // Cryptographically secure random

// ========================
// Type Functions
// ========================
$value = "42";
echo gettype($value) . "\n";         // string
echo is_string($value) . "\n";       // 1 (true)
echo is_int($value) . "\n";          // (false)
echo is_numeric($value) . "\n";      // 1 (true) - string ที่เป็นตัวเลข

// Type Casting
echo (int)"42abc" . "\n";     // 42
echo (int)"abc" . "\n";       // 0
echo (float)"3.14" . "\n";    // 3.14
echo (string)42 . "\n";       // "42"
echo (bool)"" . "\n";         // (false)
echo (bool)"0" . "\n";        // (false)
echo (bool)"false" . "\n";    // 1 (true!) - string ที่ไม่ใช่ "" หรือ "0"
echo (bool)[] . "\n";         // (false) - array ว่าง

// settype
$num = "42";
settype($num, "integer");
var_dump($num);  // int(42)

// intval, floatval, strval
echo intval("42.9") . "\n";   // 42
echo floatval("3.14px") . "\n"; // 3.14
echo strval(42) . "\n";        // "42"
```

---

## ขั้นตอนที่ 10: Project แรก - Calculator

สร้าง Web Calculator อย่างง่ายด้วย PHP

```php
<?php
// ไฟล์: calculator.php

// กำหนด Error Reporting สำหรับ Development
error_reporting(E_ALL);
ini_set('display_errors', 1);

// รับค่าจาก Form
$num1 = isset($_POST['num1']) ? (float)$_POST['num1'] : '';
$num2 = isset($_POST['num2']) ? (float)$_POST['num2'] : '';
$operator = isset($_POST['operator']) ? $_POST['operator'] : '';
$result = null;
$error = null;

// คำนวณ
if ($_SERVER['REQUEST_METHOD'] === 'POST' && $num1 !== '' && $num2 !== '') {
    switch ($operator) {
        case '+':
            $result = $num1 + $num2;
            break;
        case '-':
            $result = $num1 - $num2;
            break;
        case '*':
            $result = $num1 * $num2;
            break;
        case '/':
            if ($num2 == 0) {
                $error = "ไม่สามารถหารด้วย 0 ได้";
            } else {
                $result = $num1 / $num2;
            }
            break;
        case '%':
            if ($num2 == 0) {
                $error = "ไม่สามารถหารด้วย 0 ได้";
            } else {
                $result = $num1 % $num2;
            }
            break;
        case '**':
            $result = $num1 ** $num2;
            break;
        default:
            $error = "กรุณาเลือกตัวดำเนินการ";
    }
}

// Format result
$formatted_result = null;
if ($result !== null) {
    $formatted_result = is_int($result) ? $result : round($result, 10);
}
?>
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>PHP Calculator</title>
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            display: flex;
            align-items: center;
            justify-content: center;
        }
        .calculator {
            background: white;
            padding: 40px;
            border-radius: 20px;
            box-shadow: 0 20px 60px rgba(0,0,0,0.3);
            width: 400px;
        }
        h1 {
            text-align: center;
            color: #4a4a4a;
            margin-bottom: 30px;
            font-size: 24px;
        }
        .form-group {
            margin-bottom: 20px;
        }
        label {
            display: block;
            margin-bottom: 8px;
            color: #666;
            font-weight: 500;
        }
        input[type="number"], select {
            width: 100%;
            padding: 12px 16px;
            border: 2px solid #e1e5ee;
            border-radius: 10px;
            font-size: 16px;
            transition: border-color 0.3s;
            outline: none;
        }
        input[type="number"]:focus, select:focus {
            border-color: #667eea;
        }
        button {
            width: 100%;
            padding: 14px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            border: none;
            border-radius: 10px;
            font-size: 18px;
            cursor: pointer;
            transition: opacity 0.3s;
            font-weight: 600;
        }
        button:hover { opacity: 0.9; }
        .result {
            margin-top: 25px;
            padding: 20px;
            background: #f8f9ff;
            border-radius: 10px;
            text-align: center;
        }
        .result-value {
            font-size: 32px;
            font-weight: bold;
            color: #667eea;
        }
        .result-equation {
            color: #888;
            margin-top: 5px;
        }
        .error {
            background: #fff5f5;
            border: 1px solid #feb2b2;
            color: #c53030;
            padding: 15px;
            border-radius: 10px;
            margin-top: 20px;
            text-align: center;
        }
        .history {
            margin-top: 20px;
            font-size: 13px;
            color: #888;
        }
    </style>
</head>
<body>
    <div class="calculator">
        <h1>🧮 PHP Calculator</h1>
        
        <form method="POST" action="">
            <div class="form-group">
                <label>ตัวเลขที่ 1:</label>
                <input type="number" 
                       name="num1" 
                       step="any" 
                       value="<?= htmlspecialchars($num1) ?>" 
                       placeholder="กรอกตัวเลข"
                       required>
            </div>
            
            <div class="form-group">
                <label>ตัวดำเนินการ (Operator):</label>
                <select name="operator">
                    <option value="">-- เลือก --</option>
                    <option value="+" <?= $operator === '+' ? 'selected' : '' ?>>+ (บวก)</option>
                    <option value="-" <?= $operator === '-' ? 'selected' : '' ?>>- (ลบ)</option>
                    <option value="*" <?= $operator === '*' ? 'selected' : '' ?>>× (คูณ)</option>
                    <option value="/" <?= $operator === '/' ? 'selected' : '' ?>>÷ (หาร)</option>
                    <option value="%" <?= $operator === '%' ? 'selected' : '' ?>>% (เศษ)</option>
                    <option value="**" <?= $operator === '**' ? 'selected' : '' ?>>^ (ยกกำลัง)</option>
                </select>
            </div>
            
            <div class="form-group">
                <label>ตัวเลขที่ 2:</label>
                <input type="number" 
                       name="num2" 
                       step="any" 
                       value="<?= htmlspecialchars($num2) ?>" 
                       placeholder="กรอกตัวเลข"
                       required>
            </div>
            
            <button type="submit">คำนวณ</button>
        </form>
        
        <?php if ($error): ?>
            <div class="error">
                ❌ <?= htmlspecialchars($error) ?>
            </div>
        <?php endif; ?>
        
        <?php if ($result !== null): ?>
            <div class="result">
                <div class="result-equation">
                    <?= htmlspecialchars($num1) ?> 
                    <?= htmlspecialchars($operator) ?> 
                    <?= htmlspecialchars($num2) ?> =
                </div>
                <div class="result-value">
                    <?= htmlspecialchars(number_format($formatted_result, 
                        (floor($formatted_result) == $formatted_result) ? 0 : 6, 
                        '.', ',')) ?>
                </div>
            </div>
        <?php endif; ?>
    </div>
</body>
</html>
```

---

## ขั้นตอนที่ 11: PHP Tags ทุกรูปแบบ

```php
<?php
// Standard Tag (แนะนำ)
echo "Standard PHP Tag\n";
?>

<?= "Short Echo Tag - PHP 5.4+" ?>

<?php
// Heredoc - สำหรับ String หลายบรรทัด
$name = "PHP Developer";
$text = <<<EOT
สวัสดีคุณ $name,
ยินดีต้อนรับสู่โลกของ PHP!
วันนี้คือ: <?= date('Y-m-d') ?>
EOT;

echo $text;

// Nowdoc - เหมือน Heredoc แต่ไม่ Parse Variables
$code = <<<'EOT'
$variable = "ค่านี้จะไม่ถูก Parse";
echo $variable;  // แสดงตัวอักษรตรงๆ
EOT;

echo $code;
```

---

## ขั้นตอนที่ 12: Comments ใน PHP

```php
<?php
// Single-line comment แบบ C++

# Single-line comment แบบ Shell

/*
 * Multi-line comment
 * ใช้สำหรับอธิบายหลายบรรทัด
 * 
 * @author Your Name
 * @version 1.0
 */

/**
 * PHPDoc Comment - สำหรับ IDE และการสร้าง Documentation
 * 
 * @param string $name ชื่อผู้ใช้
 * @param int $age อายุ
 * @return string ข้อความทักทาย
 * @throws InvalidArgumentException ถ้าอายุน้อยกว่า 0
 * @since 1.0.0
 * @example
 * $greeting = greet('สมชาย', 25);
 * echo $greeting; // "สวัสดีคุณ สมชาย อายุ 25 ปี"
 */
function greet(string $name, int $age): string
{
    if ($age < 0) {
        throw new \InvalidArgumentException("อายุต้องมากกว่า 0");
    }
    return "สวัสดีคุณ {$name} อายุ {$age} ปี";
}

echo greet("สมชาย", 25);
```

---

## ขั้นตอนที่ 13: Whitespace และ Semicolons

```php
<?php
// Semicolon จำเป็นทุก Statement
echo "Hello";  // ถูกต้อง
echo "World";

// Whitespace ไม่สำคัญ (นอกจากใน String)
$x=1+2;  // ได้
$x = 1 + 2;  // แนะนำ (อ่านง่ายกว่า)

$y =
    1 +
    2 +
    3;  // ก็ได้เช่นกัน

// ระวัง: ไม่ต้องใส่ ; ตอน Close Tag ?>
?>
</html>
<?php
// ต่อจาก HTML ได้เลย
echo "Back in PHP";
?>
```

---

## ขั้นตอนที่ 14: ข้อผิดพลาดพื้นฐาน (Common Mistakes)

```php
<?php
// ❌ ข้อผิดพลาดที่พบบ่อย

// 1. ลืม Semicolon
// echo "Hello"  // Parse error: syntax error

// 2. ใช้ = แทน == ใน Condition
$x = 5;
if ($x = 10) {  // ❌ นี่คือ Assignment ไม่ใช่ Comparison!
    echo "x เท่ากับ 10"; // จะแสดงเสมอ!
}

if ($x == 10) {  // ✅ Comparison
    echo "x เท่ากับ 10";
}

// 3. Case-sensitive Variable Names
$Name = "PHP";
// echo $name;  // ❌ Undefined variable: name

// 4. ใช้ Variable ก่อน Declare
// echo $undeclared;  // ❌ Warning: Undefined variable

// 5. String vs Integer
$num = "5";
echo $num + 3;   // 8 (PHP auto-convert)
echo $num . 3;   // 53 (String concatenation)

// ✅ วิธีป้องกัน: ใช้ Strict Types
declare(strict_types=1);  // ต้องอยู่บรรทัดแรก!

// 6. Division by Zero
$a = 10;
$b = 0;
// echo $a / $b;  // ❌ DivisionByZeroError ใน PHP 8

// ✅ วิธีป้องกัน
if ($b !== 0) {
    echo $a / $b;
} else {
    echo "ไม่สามารถหารด้วย 0";
}

// หรือใช้ intdiv()
// intdiv(10, 0);  // Throws DivisionByZeroError
```

---

## ขั้นตอนที่ 15: การตรวจสอบ PHP Environment

```php
<?php
// ตรวจสอบ PHP Environment
echo "=== PHP Environment Info ===\n\n";

// Version
echo "PHP Version: " . PHP_VERSION . "\n";
echo "PHP Major: " . PHP_MAJOR_VERSION . "\n";

// OS
echo "OS: " . PHP_OS . "\n";
echo "OS Family: " . PHP_OS_FAMILY . "\n";

// SAPI (Server API)
echo "SAPI: " . PHP_SAPI . "\n";
// cli = Command Line
// apache2handler = Apache Module
// fpm-fcgi = PHP-FPM

// Paths
echo "PHP Binary: " . PHP_BINARY . "\n";
echo "Extension Dir: " . PHP_EXTENSION_DIR . "\n";
echo "Config Dir: " . PHP_CONFIG_FILE_PATH . "\n";

// Memory
echo "\nMemory Limit: " . ini_get('memory_limit') . "\n";
echo "Memory Usage: " . round(memory_get_usage(true) / 1024 / 1024, 2) . " MB\n";
echo "Peak Memory: " . round(memory_get_peak_usage(true) / 1024 / 1024, 2) . " MB\n";

// Time
echo "\nServer Time: " . date('Y-m-d H:i:s') . "\n";
echo "Timezone: " . date_default_timezone_get() . "\n";

// Extensions
echo "\nLoaded Extensions: " . implode(', ', get_loaded_extensions()) . "\n";

// CPU Info (Linux)
if (PHP_OS_FAMILY === 'Linux') {
    $cpu = shell_exec('nproc');
    echo "CPU Cores: " . trim($cpu) . "\n";
}
```

---

## 📝 แบบฝึกหัด Part 01

### แบบฝึกหัดที่ 1: Hello PHP
สร้างไฟล์ `hello.php` ที่แสดง:
- "Hello, World!" เป็นภาษาไทย
- วันที่และเวลาปัจจุบัน
- PHP Version ที่ใช้

### แบบฝึกหัดที่ 2: Personal Info Page
สร้างหน้าเว็บ HTML+PHP ที่แสดงข้อมูลส่วนตัว:
- ชื่อ-นามสกุล
- อายุ
- อาชีพ
- Hobby

### แบบฝึกหัดที่ 3: Simple Form
สร้างฟอร์มที่:
- รับชื่อและอีเมล
- แสดงผลการกรอก

---

## 🎯 สรุป Part 01

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| PHP คืออะไร | ประวัติ, การทำงาน, ข้อดี |
| การติดตั้ง | Windows (XAMPP/Laragon), macOS (Homebrew/Herd), Linux |
| VS Code Setup | Extensions, Xdebug, Settings |
| PHP Basics | Tags, echo/print, Comments, Semicolons |
| Functions พื้นฐาน | var_dump, print_r, printf, number_format |
| Common Mistakes | ข้อผิดพลาดที่พบบ่อย |

**ถัดไป → Part 02: Variables, Data Types, Constants**
