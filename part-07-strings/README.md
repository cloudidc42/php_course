# Part 07: String Functions & การจัดการ String
## ขั้นตอนที่ 157-180: PHP String Mastery

---

## ขั้นตอนที่ 157: String Basics และ Syntax

```php
<?php
declare(strict_types=1);

// Single Quotes - ไม่ parse variables
$name = 'สมชาย';
$greeting = 'สวัสดี $name';   // ผล: สวัสดี $name (literal)
$path = 'C:\Users\admin';      // ไม่ต้อง escape

// Double Quotes - parse variables และ escape sequences
$greeting = "สวัสดี $name";   // ผล: สวัสดี สมชาย
$greeting = "สวัสดี {$name}"; // แบบ explicit (แนะนำ)
$tab = "คอลัมน์1\tคอลัมน์2";
$newline = "บรรทัดที่1\nบรรทัดที่2";
$unicode = "\u{0E2A}\u{0E27}\u{0E31}\u{0E2A}\u{0E14}\u{0E35}"; // สวัสดี

// Heredoc
$html = <<<EOT
<div class="card">
    <h1>$name</h1>
    <p>Hello World</p>
</div>
EOT;

// Nowdoc (like single quotes)
$template = <<<'EOT'
Hello $name - this won't be parsed
EOT;

// String as Array
$str = "Hello";
echo $str[0];  // H
echo $str[-1]; // o (PHP 7.1+)

// Multiline String
$sql = "SELECT users.id, users.name, 
        orders.total, orders.created_at
        FROM users
        LEFT JOIN orders ON users.id = orders.user_id
        WHERE users.active = 1";
```

---

## ขั้นตอนที่ 158: String Searching และ Position

```php
<?php
declare(strict_types=1);

$text = "The quick brown fox jumps over the lazy dog";

// หาตำแหน่ง (case sensitive)
$pos = strpos($text, "fox");      // 16
$last = strrpos($text, "the");    // 31

// Case insensitive
$pos = stripos($text, "THE");     // 0
$last = strripos($text, "THE");   // 31

// ตรวจสอบการมีอยู่
if (str_contains($text, "fox")) {   // PHP 8.0+
    echo "Found fox!";
}

if (str_starts_with($text, "The")) {  // PHP 8.0+
    echo "Starts with The";
}

if (str_ends_with($text, "dog")) {    // PHP 8.0+
    echo "Ends with dog";
}

// แบบ Classic (PHP 7)
if (strpos($text, "fox") !== false) {
    echo "Found!";
}

// substr_count - นับจำนวนครั้งที่พบ
$count = substr_count($text, "the"); // 1 (case sensitive)
$count = substr_count(strtolower($text), "the"); // 2

// similar_text - เปรียบเทียบความคล้ายคลึง
similar_text("Hello", "World", $percent);
echo round($percent, 2); // 40.00

// levenshtein - จำนวน operations ที่ต้องทำ
echo levenshtein("kitten", "sitting"); // 3

// soundex / metaphone - เสียง
echo soundex("Robert");   // R163
echo metaphone("Smith");  // SM0
```

---

## ขั้นตอนที่ 159: String Extraction และ Slicing

```php
<?php
declare(strict_types=1);

$str = "Hello, World!";

// substr(string, start, length?)
echo substr($str, 7);      // World!
echo substr($str, 7, 5);   // World
echo substr($str, -6);     // orld!
echo substr($str, -6, 4);  // orld

// strstr - ดึงจากตำแหน่งที่พบ
echo strstr($str, ",");            // , World!
echo strstr($str, ",", true);     // Hello  (before needle)
echo stristr($str, "world");       // World! (case insensitive)

// substr_replace
echo substr_replace($str, "PHP", 7, 5); // Hello, PHP!

// str_split - แยกเป็น Array
$chars = str_split("Hello");       // ['H','e','l','l','o']
$chunks = str_split("Hello World", 3); // ['Hel','lo ','Wor','ld']

// chunk_split - แยกแล้วใส่ delimiter
echo chunk_split("ABCDEFGH", 2, "-"); // AB-CD-EF-GH-

// wordwrap
$long = "The quick brown fox jumped over the lazy dog";
echo wordwrap($long, 15, "\n", true);

// strrchr - หา character สุดท้ายที่พบ
$path = "/var/www/html/index.php";
echo strrchr($path, "/"); // /index.php
echo ltrim(strrchr($path, "/"), "/"); // index.php
```

---

## ขั้นตอนที่ 160: String Modification

```php
<?php
declare(strict_types=1);

$str = "  Hello, World!  ";

// Case functions
echo strtolower("HELLO");      // hello
echo strtoupper("hello");      // HELLO
echo ucfirst("hello world");   // Hello world
echo lcfirst("HELLO World");   // hELLO World
echo ucwords("hello world php"); // Hello World Php
echo mb_strtolower("สวัสดี");  // สวัสดี (Unicode safe)
echo mb_strtoupper("สวัสดี");  // สวัสดี

// Trim
echo trim("  Hello  ");           // "Hello"
echo ltrim("  Hello  ");          // "Hello  "
echo rtrim("  Hello  ");          // "  Hello"
echo trim("***Hello***", "*");    // "Hello"

// Padding
echo str_pad("42", 5);              // "42   "
echo str_pad("42", 5, "0", STR_PAD_LEFT);  // "00042"
echo str_pad("Hi", 10, "-", STR_PAD_BOTH); // "----Hi----"

// Repeat
echo str_repeat("ab", 3);   // ababab
echo str_repeat("-", 20);   // --------------------

// Reverse
echo strrev("Hello");   // olleH

// String Replace
echo str_replace("World", "PHP", "Hello, World!"); // Hello, PHP!
echo str_replace(["Hello", "World"], ["Hi", "PHP"], "Hello, World!"); // Hi, PHP!
echo str_ireplace("WORLD", "PHP", "Hello, World!"); // Hello, PHP! (case insensitive)

// Count replacements
$count = 0;
str_replace("a", "b", "abcabc", $count);
echo $count; // 2

// substr_replace
$email = "user@example.com";
$masked = substr_replace($email, str_repeat('*', 4), 2, 4); // us****@example.com
// Better:
$atPos = strpos($email, '@');
$masked = substr($email, 0, 2) . str_repeat('*', $atPos - 2) . substr($email, $atPos);
```

---

## ขั้นตอนที่ 161: String Formatting

```php
<?php
declare(strict_types=1);

// sprintf
$name = "สมชาย";
$age = 30;
$price = 1234.5678;

echo sprintf("ชื่อ: %s, อายุ: %d ปี", $name, $age);
echo sprintf("ราคา: %.2f บาท", $price);           // 1234.57
echo sprintf("ราคา: %'.10.2f บาท", $price);        // ......1234.57
echo sprintf("Binary: %b", 255);                   // 11111111
echo sprintf("Hex: %x", 255);                      // ff
echo sprintf("Octal: %o", 255);                    // 377
echo sprintf("Scientific: %e", 123456789.0);       // 1.234568e+8
echo sprintf("Padded: %05d", 42);                  // 00042

// number_format
echo number_format(1234567.891, 2, '.', ',');  // 1,234,567.89
echo number_format(1234567.891, 2, ',', '.');  // 1.234.567,89 (European)

// Thai number format
function formatThaiPrice(float $amount): string {
    return '฿' . number_format($amount, 2, '.', ',');
}
echo formatThaiPrice(1234.5); // ฿1,234.50

// printf (print formatted)
printf("Hello, %s! You are %d years old.\n", "World", 30);

// sscanf (inverse of sprintf)
[$year, $month, $day] = sscanf("2024-01-15", "%d-%d-%d");

// money_format (deprecated, use NumberFormatter)
$fmt = new NumberFormatter('th_TH', NumberFormatter::CURRENCY);
echo $fmt->formatCurrency(1234.56, 'THB'); // ฿1,234.56

// date formatting
echo date('d/m/Y H:i:s');           // 15/01/2024 14:30:00
echo date('D, d M Y');              // Mon, 15 Jan 2024
echo strftime('%A %d %B %Y');       // Monday 15 January 2024
```

---

## ขั้นตอนที่ 162: Regular Expressions

```php
<?php
declare(strict_types=1);

// Basic Patterns
$text = "Hello World 123";

// preg_match - หา match แรก
if (preg_match('/(\d+)/', $text, $matches)) {
    echo $matches[0]; // 123 (full match)
    echo $matches[1]; // 123 (group 1)
}

// preg_match_all - หาทุก match
$html = '<a href="url1">Link1</a> <a href="url2">Link2</a>';
preg_match_all('/<a href="([^"]+)">([^<]+)<\/a>/', $html, $matches);
print_r($matches[1]); // ['url1', 'url2']
print_r($matches[2]); // ['Link1', 'Link2']

// Named Groups
$date = "2024-01-15";
if (preg_match('/(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})/', $date, $m)) {
    echo $m['year'];  // 2024
    echo $m['month']; // 01
    echo $m['day'];   // 15
}

// preg_replace - แทนที่
$text = "Hello World";
echo preg_replace('/World/', 'PHP', $text); // Hello PHP

// With backreference
$date = "15/01/2024";
echo preg_replace('/(\d{2})\/(\d{2})\/(\d{4})/', '$3-$2-$1', $date); // 2024-01-15

// preg_replace_callback
$text = "hello world";
$result = preg_replace_callback('/\w+/', function($matches) {
    return ucfirst($matches[0]);
}, $text);
echo $result; // Hello World

// preg_split
$str = "one,two;three|four";
$parts = preg_split('/[,;|]/', $str);
print_r($parts); // ['one', 'two', 'three', 'four']

// Common Patterns
class RegexPatterns {
    const EMAIL = '/^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/';
    const PHONE_TH = '/^(0[689]\d{8}|0[2-9]\d{7})$/';
    const URL = '/^https?:\/\/(www\.)?[-a-zA-Z0-9@:%._\+~#=]{2,256}\.[a-z]{2,6}\b([-a-zA-Z0-9@:%_\+.~#?&\/\/=]*)$/';
    const PASSWORD = '/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]{8,}$/';
    const IP_V4 = '/^(\d{1,3}\.){3}\d{1,3}$/';
    const THAI_ID = '/^\d{13}$/';
    
    public static function validate(string $pattern, string $value): bool {
        return (bool) preg_match($pattern, $value);
    }
    
    public static function validateEmail(string $email): bool {
        return filter_var($email, FILTER_VALIDATE_EMAIL) !== false;
    }
    
    public static function validateUrl(string $url): bool {
        return filter_var($url, FILTER_VALIDATE_URL) !== false;
    }
    
    public static function validateThaiPhone(string $phone): bool {
        $phone = preg_replace('/[\s\-\(\)]/', '', $phone);
        return self::validate(self::PHONE_TH, $phone);
    }
}

// Test
var_dump(RegexPatterns::validateEmail("user@example.com")); // true
var_dump(RegexPatterns::validateThaiPhone("0812345678")); // true
```

---

## ขั้นตอนที่ 163: Multibyte String (Thai/Unicode)

```php
<?php
declare(strict_types=1);

// ปัญหาของ String ปกติกับภาษาไทย
$thai = "สวัสดี";
echo strlen($thai);     // 18 (bytes, ไม่ใช่ตัวอักษร)
echo mb_strlen($thai);  // 6 (ตัวอักษรจริง)

// mb_ functions
echo mb_strtolower("HELLO สวัสดี");  // hello สวัสดี
echo mb_strtoupper("hello สวัสดี");  // HELLO สวัสดี
echo mb_substr("สวัสดีครับ", 0, 6);  // สวัสดี
echo mb_strpos("สวัสดีครับ", "ครับ"); // 6

// ตั้ง Encoding
mb_internal_encoding('UTF-8');
mb_regex_encoding('UTF-8');

// Convert Encoding
$utf8 = mb_convert_encoding($tis620_string, 'UTF-8', 'TIS-620');
$tis620 = mb_convert_encoding($utf8_string, 'TIS-620', 'UTF-8');

// Check Encoding
$encoding = mb_detect_encoding($str, ['UTF-8', 'TIS-620', 'ISO-8859-1']);

// mb_str_split (PHP 7.4+)
$chars = mb_str_split("สวัสดี"); // ['ส','ว','ั','ส','ด','ี']
$chars = mb_str_split("Hello", 2); // ['He','ll','o']

// Slug for Thai
function createSlug(string $title): string {
    // Replace Thai characters with transliteration or just keep them
    $title = mb_strtolower($title, 'UTF-8');
    
    // Replace spaces and special chars
    $title = preg_replace('/[^\p{L}\p{N}\s-]/u', '', $title);
    $title = preg_replace('/[\s]+/', '-', trim($title));
    
    return $title;
}

echo createSlug("สวัสดี World! Test 123");  // สวัสดี-world-test-123

// WordCount with Thai
function countWords(string $text): int {
    // Thai doesn't use spaces between words, this is simplified
    return count(preg_split('/[\s,]+/u', trim($text), -1, PREG_SPLIT_NO_EMPTY));
}

// Truncate Thai text properly
function truncate(string $text, int $length = 100, string $append = '...'): string {
    if (mb_strlen($text) <= $length) {
        return $text;
    }
    return mb_substr($text, 0, $length) . $append;
}
```

---

## ขั้นตอนที่ 164: String Security

```php
<?php
declare(strict_types=1);

// ========================
// HTML Encoding (XSS Prevention)
// ========================
$userInput = '<script>alert("XSS")</script>';

// htmlspecialchars - สำหรับ HTML context
echo htmlspecialchars($userInput, ENT_QUOTES | ENT_HTML5, 'UTF-8');
// &lt;script&gt;alert(&quot;XSS&quot;)&lt;/script&gt;

// htmlentities - แปลงทุกอย่างเป็น HTML entity
echo htmlentities($userInput, ENT_QUOTES, 'UTF-8');

// Decode
echo htmlspecialchars_decode("&lt;b&gt;Bold&lt;/b&gt;"); // <b>Bold</b>
echo html_entity_decode("&copy; &amp; &lt;");

// strip_tags - ลบ HTML tags
echo strip_tags("<p>Hello <b>World</b></p>"); // Hello World
echo strip_tags("<p>Hello <b>World</b></p>", '<p>'); // <p>Hello World</p>
echo strip_tags("<p>Hello <b>World</b></p>", ['p', 'b']); // PHP 7.4+

// ========================
// URL Encoding
// ========================
$url = "https://example.com/search?q=สวัสดี โลก&page=1";

echo urlencode("สวัสดี โลก"); // %E0%B8%AA%E0%B8%A7%E0%B8%B1%E0%B8%AA%E0%B8%94%E0%B8%B5+%E0%B9%82%E0%B8%A5%E0%B8%81
echo rawurlencode("สวัสดี โลก"); // spaces become %20 not +

echo urldecode($encoded);
echo rawurldecode($encoded);

// http_build_query
$params = [
    'q' => 'สวัสดี',
    'page' => 1,
    'filters' => ['type' => 'post', 'status' => 'active'],
];
echo http_build_query($params); // q=%E0%B8%AA%...&page=1&filters%5Btype%5D=post&...

// parse_str
parse_str("name=John&age=30&skills[]=PHP&skills[]=Python", $output);
print_r($output);
// ['name' => 'John', 'age' => '30', 'skills' => ['PHP', 'Python']]

// ========================
// SQL Encoding (แต่ใช้ PDO prepared statements ดีกว่า)
// ========================
// addslashes / stripslashes (ไม่แนะนำสำหรับ SQL)
$str = "It's a test";
echo addslashes($str); // It\'s a test

// ========================
// Sanitization Helpers
// ========================
class Sanitizer {
    public static function html(string $value): string {
        return htmlspecialchars(trim($value), ENT_QUOTES | ENT_HTML5, 'UTF-8');
    }
    
    public static function email(string $email): string|false {
        return filter_var(trim($email), FILTER_SANITIZE_EMAIL);
    }
    
    public static function url(string $url): string|false {
        return filter_var(trim($url), FILTER_SANITIZE_URL);
    }
    
    public static function int(mixed $value): int {
        return (int) filter_var($value, FILTER_SANITIZE_NUMBER_INT);
    }
    
    public static function float(mixed $value): float {
        return (float) filter_var($value, FILTER_SANITIZE_NUMBER_FLOAT, FILTER_FLAG_ALLOW_FRACTION);
    }
    
    public static function filename(string $filename): string {
        $filename = preg_replace('/[^\w\-.]/', '_', $filename);
        $filename = preg_replace('/_{2,}/', '_', $filename);
        return ltrim($filename, '._');
    }
    
    public static function slug(string $text): string {
        $text = mb_strtolower(trim($text), 'UTF-8');
        $text = preg_replace('/[^\p{L}\p{N}\s-]/u', '', $text);
        $text = preg_replace('/[\s_]+/', '-', $text);
        $text = trim($text, '-');
        return $text;
    }
    
    public static function phone(string $phone): string {
        return preg_replace('/[^\d+]/', '', $phone);
    }
}
```

---

## ขั้นตอนที่ 165: String Utilities Class

```php
<?php
declare(strict_types=1);

class Str {
    // ตัดคำ
    public static function limit(string $str, int $limit = 100, string $end = '...'): string {
        if (mb_strlen($str) <= $limit) {
            return $str;
        }
        return rtrim(mb_substr($str, 0, $limit)) . $end;
    }
    
    // ตัดตามคำ
    public static function words(string $str, int $words = 100, string $end = '...'): string {
        preg_match('/^\s*+(?:\S+\s*+){1,' . $words . '}/u', $str, $matches);
        if (!isset($matches[0]) || mb_strlen($str) === mb_strlen($matches[0])) {
            return $str;
        }
        return rtrim($matches[0]) . $end;
    }
    
    // สุ่ม String
    public static function random(int $length = 16): string {
        $chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
        $result = '';
        for ($i = 0; $i < $length; $i++) {
            $result .= $chars[random_int(0, strlen($chars) - 1)];
        }
        return $result;
    }
    
    // UUID v4
    public static function uuid(): string {
        $data = random_bytes(16);
        $data[6] = chr(ord($data[6]) & 0x0f | 0x40);
        $data[8] = chr(ord($data[8]) & 0x3f | 0x80);
        return vsprintf('%s%s-%s-%s-%s-%s%s%s', str_split(bin2hex($data), 4));
    }
    
    // Slug
    public static function slug(string $title, string $separator = '-'): string {
        $title = mb_strtolower(trim($title), 'UTF-8');
        $title = preg_replace('/[^\p{L}\p{N}\s-]/u', '', $title);
        $title = preg_replace('/[\s_]+/', $separator, $title);
        return trim($title, $separator);
    }
    
    // Camel Case
    public static function camel(string $str): string {
        return lcfirst(static::studly($str));
    }
    
    // StudlyCaps
    public static function studly(string $str): string {
        return str_replace(' ', '', ucwords(str_replace(['-', '_'], ' ', $str)));
    }
    
    // Snake Case
    public static function snake(string $str, string $delimiter = '_'): string {
        $str = preg_replace('/\s+/u', '', ucwords($str));
        return mb_strtolower(preg_replace('/(.)(?=[A-Z])/u', '$1' . $delimiter, $str));
    }
    
    // Kebab Case
    public static function kebab(string $str): string {
        return static::snake($str, '-');
    }
    
    // Mask
    public static function mask(string $str, string $char = '*', int $start = 0, int $length = 0): string {
        $strLen = mb_strlen($str);
        $maskLen = $length ?: $strLen - $start;
        
        if ($start < 0) {
            $start = max(0, $strLen + $start);
        }
        
        return mb_substr($str, 0, $start)
            . str_repeat($char, min($maskLen, $strLen - $start))
            . mb_substr($str, $start + $maskLen);
    }
    
    // Between
    public static function between(string $str, string $from, string $to): string {
        $startPos = mb_strpos($str, $from);
        if ($startPos === false) return $str;
        
        $startPos += mb_strlen($from);
        $endPos = mb_strpos($str, $to, $startPos);
        
        if ($endPos === false) return mb_substr($str, $startPos);
        return mb_substr($str, $startPos, $endPos - $startPos);
    }
    
    // Count Words
    public static function wordCount(string $str): int {
        return str_word_count(strip_tags($str));
    }
    
    // Highlight search term
    public static function highlight(string $text, string $term, string $tag = 'mark'): string {
        $pattern = '/(' . preg_quote($term, '/') . ')/ui';
        return preg_replace($pattern, "<{$tag}>$1</{$tag}>", $text);
    }
    
    // Title Case
    public static function title(string $str): string {
        return mb_convert_case($str, MB_CASE_TITLE, 'UTF-8');
    }
    
    // Pad (unicode safe)
    public static function padLeft(string $str, int $length, string $pad = ' '): string {
        $padLen = $length - mb_strlen($str);
        if ($padLen <= 0) return $str;
        return str_repeat($pad, (int) ceil($padLen / mb_strlen($pad))) . $str;
    }
    
    public static function padRight(string $str, int $length, string $pad = ' '): string {
        $padLen = $length - mb_strlen($str);
        if ($padLen <= 0) return $str;
        return $str . str_repeat($pad, (int) ceil($padLen / mb_strlen($pad)));
    }
}

// Tests
echo Str::limit("The quick brown fox", 10);       // The quick...
echo Str::words("The quick brown fox", 2);         // The quick...
echo Str::random(8);                               // e.g. aB3xKp9Z
echo Str::uuid();                                  // e.g. 550e8400-e29b-41d4-a716-446655440000
echo Str::slug("Hello World PHP");                 // hello-world-php
echo Str::camel("foo_bar_baz");                    // fooBarBaz
echo Str::studly("foo_bar_baz");                   // FooBarBaz
echo Str::snake("FooBarBaz");                      // foo_bar_baz
echo Str::kebab("FooBarBaz");                      // foo-bar-baz
echo Str::mask("user@example.com", '*', 2, 6);    // us******ample.com
echo Str::between("Hello [World] PHP", "[", "]"); // World
echo Str::highlight("Hello World", "World");       // Hello <mark>World</mark>
```

---

## 🎯 สรุป Part 07

| Function | การใช้งาน |
|----------|-----------|
| strlen / mb_strlen | ความยาว String |
| strpos / str_contains | หาตำแหน่ง/ตรวจสอบ |
| substr / mb_substr | ดึงส่วนของ String |
| str_replace / preg_replace | แทนที่ |
| sprintf / number_format | Format String |
| htmlspecialchars | ป้องกัน XSS |
| mb_* functions | Unicode/Thai support |

**ถัดไป → Part 08: Forms & User Input Validation**
