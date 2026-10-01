# Part 09: File System & File Handling
## ขั้นตอนที่ 211-240: จัดการไฟล์และโฟลเดอร์

---

## ขั้นตอนที่ 211: File Reading & Writing

```php
<?php
declare(strict_types=1);

// ========================
// Basic File I/O
// ========================

// อ่านทั้งไฟล์
$content = file_get_contents('/path/to/file.txt');

// อ่านและแยกเป็น array (แต่ละบรรทัด)
$lines = file('/path/to/file.txt', FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES);

// เขียนไฟล์ (overwrite)
file_put_contents('/path/to/file.txt', "Hello, World!\n");

// เพิ่มต่อท้ายไฟล์
file_put_contents('/path/to/file.txt', "New line\n", FILE_APPEND);

// เขียนแบบ atomic (ป้องกัน race condition)
file_put_contents('/path/to/file.txt', $content, LOCK_EX);

// ========================
// File Handle (fopen)
// ========================
// Modes: r, r+, w, w+, a, a+, x, x+
$handle = fopen('/path/to/file.txt', 'r');

if ($handle) {
    while (!feof($handle)) {
        $line = fgets($handle, 4096); // อ่านทีละบรรทัด
        if ($line !== false) {
            echo $line;
        }
    }
    fclose($handle);
}

// อ่านข้อมูล binary
$handle = fopen('/path/to/image.jpg', 'rb');
$data = fread($handle, filesize('/path/to/image.jpg'));
fclose($handle);

// เขียนทีละ chunk
$handle = fopen('/path/to/large.csv', 'w');
fwrite($handle, "id,name,email\n");
for ($i = 1; $i <= 100000; $i++) {
    fwrite($handle, "{$i},User{$i},user{$i}@example.com\n");
}
fclose($handle);

// File Locking
$handle = fopen('/path/to/counter.txt', 'c+');
if (flock($handle, LOCK_EX)) {
    $count = (int) fread($handle, 100);
    $count++;
    ftruncate($handle, 0);
    rewind($handle);
    fwrite($handle, (string) $count);
    fflush($handle);
    flock($handle, LOCK_UN);
}
fclose($handle);
```

---

## ขั้นตอนที่ 212: File & Directory Info

```php
<?php
declare(strict_types=1);

$path = '/var/www/html/index.php';

// File Info
echo file_exists($path) ? "Exists\n" : "Not found\n";
echo is_file($path) ? "Is file\n" : "Not a file\n";
echo is_dir($path) ? "Is dir\n" : "Not a dir\n";
echo is_readable($path) ? "Readable\n" : "Not readable\n";
echo is_writable($path) ? "Writable\n" : "Not writable\n";
echo is_executable($path) ? "Executable\n" : "Not executable\n";

echo filesize($path) . " bytes\n";
echo date('Y-m-d H:i:s', filemtime($path)) . "\n"; // Modified time
echo date('Y-m-d H:i:s', filectime($path)) . "\n"; // Changed time
echo date('Y-m-d H:i:s', fileatime($path)) . "\n"; // Access time

// pathinfo
$info = pathinfo($path);
echo $info['dirname'];    // /var/www/html
echo $info['basename'];   // index.php
echo $info['filename'];   // index
echo $info['extension'];  // php

// realpath - ตามหา absolute path
echo realpath('../config/database.php');

// ========================
// Directory Operations
// ========================
// สร้าง directory
mkdir('/path/to/new/dir', 0755, true); // recursive

// ลบ directory (ต้องว่างเปล่า)
rmdir('/path/to/empty/dir');

// List files in directory
$files = scandir('/path/to/dir');
$files = array_diff($files, ['.', '..']); // ลบ . และ ..

// glob - Pattern matching
$phpFiles = glob('/var/www/html/*.php');
$images = glob('/uploads/**/*.{jpg,png,gif}', GLOB_BRACE);

// ========================
// SPL Directory Iterator
// ========================
$dir = new DirectoryIterator('/path/to/dir');
foreach ($dir as $file) {
    if ($file->isDot()) continue;
    echo $file->getFilename() . "\n";
    echo $file->getSize() . " bytes\n";
    echo $file->getExtension() . "\n";
}

// Recursive
$it = new RecursiveDirectoryIterator('/path/to/dir', FilesystemIterator::SKIP_DOTS);
$rit = new RecursiveIteratorIterator($it);
$files = new RegexIterator($rit, '/\.php$/');

foreach ($files as $file) {
    echo $file->getRealPath() . "\n";
}
```

---

## ขั้นตอนที่ 213: File Manager Class

```php
<?php
declare(strict_types=1);

class FileManager {
    private string $basePath;
    
    public function __construct(string $basePath) {
        $this->basePath = rtrim(realpath($basePath) ?: $basePath, '/');
    }
    
    // Secure path ป้องกัน directory traversal
    private function securePath(string $path): string {
        $fullPath = $this->basePath . '/' . ltrim($path, '/');
        $realPath = realpath($fullPath);
        
        if ($realPath === false) {
            $realPath = $fullPath; // ไฟล์ยังไม่มี
        }
        
        // ตรวจสอบว่าอยู่ใน base path
        if (!str_starts_with($realPath, $this->basePath)) {
            throw new RuntimeException("Access denied: path outside base directory");
        }
        
        return $realPath;
    }
    
    public function read(string $path): string {
        $fullPath = $this->securePath($path);
        
        if (!file_exists($fullPath)) {
            throw new RuntimeException("File not found: {$path}");
        }
        
        return file_get_contents($fullPath);
    }
    
    public function write(string $path, string $content): bool {
        $fullPath = $this->securePath($path);
        $dir = dirname($fullPath);
        
        if (!is_dir($dir)) {
            mkdir($dir, 0755, true);
        }
        
        return file_put_contents($fullPath, $content, LOCK_EX) !== false;
    }
    
    public function append(string $path, string $content): bool {
        $fullPath = $this->securePath($path);
        return file_put_contents($fullPath, $content, FILE_APPEND | LOCK_EX) !== false;
    }
    
    public function delete(string $path): bool {
        $fullPath = $this->securePath($path);
        
        if (is_dir($fullPath)) {
            return $this->deleteDirectory($fullPath);
        }
        
        return file_exists($fullPath) ? unlink($fullPath) : true;
    }
    
    private function deleteDirectory(string $dir): bool {
        $files = scandir($dir);
        foreach ($files as $file) {
            if ($file === '.' || $file === '..') continue;
            $filePath = $dir . '/' . $file;
            is_dir($filePath) ? $this->deleteDirectory($filePath) : unlink($filePath);
        }
        return rmdir($dir);
    }
    
    public function copy(string $from, string $to): bool {
        $fromPath = $this->securePath($from);
        $toPath = $this->securePath($to);
        
        $toDir = dirname($toPath);
        if (!is_dir($toDir)) {
            mkdir($toDir, 0755, true);
        }
        
        return copy($fromPath, $toPath);
    }
    
    public function move(string $from, string $to): bool {
        $fromPath = $this->securePath($from);
        $toPath = $this->securePath($to);
        
        $toDir = dirname($toPath);
        if (!is_dir($toDir)) {
            mkdir($toDir, 0755, true);
        }
        
        return rename($fromPath, $toPath);
    }
    
    public function exists(string $path): bool {
        try {
            return file_exists($this->securePath($path));
        } catch (RuntimeException) {
            return false;
        }
    }
    
    public function size(string $path): int {
        return filesize($this->securePath($path));
    }
    
    public function list(string $dir = '', bool $recursive = false): array {
        $fullPath = $dir ? $this->securePath($dir) : $this->basePath;
        
        if (!is_dir($fullPath)) {
            throw new RuntimeException("Directory not found");
        }
        
        $files = [];
        
        if ($recursive) {
            $it = new RecursiveDirectoryIterator($fullPath, FilesystemIterator::SKIP_DOTS);
            $rit = new RecursiveIteratorIterator($it);
            
            foreach ($rit as $file) {
                $files[] = $this->fileInfo($file);
            }
        } else {
            foreach (new DirectoryIterator($fullPath) as $file) {
                if ($file->isDot()) continue;
                $files[] = $this->fileInfo($file);
            }
        }
        
        usort($files, fn($a, $b) => ($b['is_dir'] <=> $a['is_dir']) ?: strcmp($a['name'], $b['name']));
        
        return $files;
    }
    
    private function fileInfo(SplFileInfo $file): array {
        return [
            'name' => $file->getFilename(),
            'path' => str_replace($this->basePath . '/', '', $file->getRealPath()),
            'size' => $file->isFile() ? $file->getSize() : 0,
            'is_dir' => $file->isDir(),
            'extension' => $file->getExtension(),
            'modified' => date('Y-m-d H:i:s', $file->getMTime()),
            'permissions' => substr(sprintf('%o', $file->getPerms()), -4),
        ];
    }
    
    // Human readable size
    public static function formatSize(int $bytes): string {
        $units = ['B', 'KB', 'MB', 'GB', 'TB'];
        $bytes = max($bytes, 0);
        $pow = floor(($bytes ? log($bytes) : 0) / log(1024));
        $pow = min($pow, count($units) - 1);
        
        return round($bytes / (1024 ** $pow), 2) . ' ' . $units[$pow];
    }
    
    // CSV operations
    public function readCsv(string $path, bool $hasHeader = true): array {
        $fullPath = $this->securePath($path);
        $rows = [];
        $headers = [];
        
        $handle = fopen($fullPath, 'r');
        if (!$handle) throw new RuntimeException("Cannot open file");
        
        while (($row = fgetcsv($handle, 0, ',', '"', '\\')) !== false) {
            if ($hasHeader && empty($headers)) {
                $headers = $row;
                continue;
            }
            
            $rows[] = $hasHeader ? array_combine($headers, $row) : $row;
        }
        
        fclose($handle);
        return $rows;
    }
    
    public function writeCsv(string $path, array $data, array $headers = []): bool {
        $fullPath = $this->securePath($path);
        
        $handle = fopen($fullPath, 'w');
        if (!$handle) return false;
        
        // BOM for Excel UTF-8
        fputs($handle, "\xEF\xBB\xBF");
        
        if (!empty($headers)) {
            fputcsv($handle, $headers);
        } elseif (!empty($data)) {
            fputcsv($handle, array_keys(reset($data)));
        }
        
        foreach ($data as $row) {
            fputcsv($handle, $row);
        }
        
        fclose($handle);
        return true;
    }
}

// ========================
// Usage
// ========================
$fm = new FileManager('/var/www/html/storage');

// Write
$fm->write('logs/app.log', date('Y-m-d H:i:s') . " Application started\n");

// Read
$log = $fm->read('logs/app.log');

// List files
$files = $fm->list('uploads', recursive: true);
foreach ($files as $file) {
    echo $file['name'] . ' - ' . FileManager::formatSize($file['size']) . "\n";
}

// CSV
$users = [
    ['id' => 1, 'name' => 'สมชาย', 'email' => 'somchai@example.com'],
    ['id' => 2, 'name' => 'สมหญิง', 'email' => 'somying@example.com'],
];
$fm->writeCsv('exports/users.csv', $users);
```

---

## ขั้นตอนที่ 214: JSON File Storage

```php
<?php
declare(strict_types=1);

class JsonStorage {
    private string $path;
    private array $data = [];
    
    public function __construct(string $path) {
        $this->path = $path;
        $this->load();
    }
    
    private function load(): void {
        if (!file_exists($this->path)) {
            $this->data = [];
            return;
        }
        
        $content = file_get_contents($this->path);
        $this->data = json_decode($content, true) ?? [];
    }
    
    private function save(): void {
        $dir = dirname($this->path);
        if (!is_dir($dir)) {
            mkdir($dir, 0755, true);
        }
        
        file_put_contents(
            $this->path,
            json_encode($this->data, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE),
            LOCK_EX
        );
    }
    
    public function get(string $key, mixed $default = null): mixed {
        return $this->data[$key] ?? $default;
    }
    
    public function set(string $key, mixed $value): self {
        $this->data[$key] = $value;
        $this->save();
        return $this;
    }
    
    public function delete(string $key): self {
        unset($this->data[$key]);
        $this->save();
        return $this;
    }
    
    public function has(string $key): bool {
        return isset($this->data[$key]);
    }
    
    public function all(): array {
        return $this->data;
    }
    
    public function clear(): self {
        $this->data = [];
        $this->save();
        return $this;
    }
}

// Simple key-value config file
$config = new JsonStorage('/var/www/html/storage/config.json');
$config->set('site_name', 'My Website');
$config->set('maintenance_mode', false);
echo $config->get('site_name'); // My Website
```

---

## ขั้นตอนที่ 215: Log Writer

```php
<?php
declare(strict_types=1);

enum LogLevel: string {
    case DEBUG = 'DEBUG';
    case INFO = 'INFO';
    case WARNING = 'WARNING';
    case ERROR = 'ERROR';
    case CRITICAL = 'CRITICAL';
    
    public function color(): string {
        return match($this) {
            self::DEBUG => "\033[37m",   // White
            self::INFO => "\033[32m",    // Green
            self::WARNING => "\033[33m", // Yellow
            self::ERROR => "\033[31m",   // Red
            self::CRITICAL => "\033[35m", // Magenta
        };
    }
}

class Logger {
    private string $logDir;
    private string $dateFormat;
    private int $maxFileSize;  // bytes
    
    public function __construct(
        string $logDir = '/var/log/app',
        int $maxFileSize = 10 * 1024 * 1024  // 10MB
    ) {
        $this->logDir = rtrim($logDir, '/');
        $this->maxFileSize = $maxFileSize;
        
        if (!is_dir($this->logDir)) {
            mkdir($this->logDir, 0755, true);
        }
    }
    
    public function log(LogLevel $level, string $message, array $context = []): void {
        $logFile = $this->logDir . '/' . date('Y-m-d') . '.log';
        
        // Rotate if needed
        if (file_exists($logFile) && filesize($logFile) >= $this->maxFileSize) {
            $this->rotate($logFile);
        }
        
        $entry = sprintf(
            "[%s] [%s] %s%s\n",
            date('Y-m-d H:i:s'),
            $level->value,
            $message,
            !empty($context) ? ' | ' . json_encode($context, JSON_UNESCAPED_UNICODE) : ''
        );
        
        file_put_contents($logFile, $entry, FILE_APPEND | LOCK_EX);
    }
    
    private function rotate(string $logFile): void {
        $rotated = $logFile . '.' . time();
        rename($logFile, $rotated);
        
        // Compress old log
        if (function_exists('gzopen')) {
            $gz = gzopen($rotated . '.gz', 'wb9');
            gzwrite($gz, file_get_contents($rotated));
            gzclose($gz);
            unlink($rotated);
        }
        
        // Keep only last 30 files
        $pattern = dirname($logFile) . '/*.log.*';
        $files = glob($pattern);
        if (count($files) > 30) {
            usort($files, fn($a, $b) => filemtime($a) - filemtime($b));
            $toDelete = array_slice($files, 0, count($files) - 30);
            array_walk($toDelete, fn($f) => unlink($f));
        }
    }
    
    public function debug(string $message, array $context = []): void {
        $this->log(LogLevel::DEBUG, $message, $context);
    }
    
    public function info(string $message, array $context = []): void {
        $this->log(LogLevel::INFO, $message, $context);
    }
    
    public function warning(string $message, array $context = []): void {
        $this->log(LogLevel::WARNING, $message, $context);
    }
    
    public function error(string $message, array $context = []): void {
        $this->log(LogLevel::ERROR, $message, $context);
    }
    
    public function critical(string $message, array $context = []): void {
        $this->log(LogLevel::CRITICAL, $message, $context);
    }
    
    public function tail(int $lines = 50): array {
        $logFile = $this->logDir . '/' . date('Y-m-d') . '.log';
        
        if (!file_exists($logFile)) return [];
        
        $file = new SplFileObject($logFile);
        $file->seek(PHP_INT_MAX);
        $total = $file->key();
        
        $result = [];
        $start = max(0, $total - $lines);
        
        for ($i = $start; $i < $total; $i++) {
            $file->seek($i);
            $result[] = rtrim($file->current());
        }
        
        return $result;
    }
}

// Usage
$logger = new Logger('/var/log/myapp');

$logger->info('Application started', ['version' => '1.0.0', 'env' => 'production']);
$logger->warning('High memory usage', ['memory' => memory_get_usage(true)]);
$logger->error('Database connection failed', [
    'host' => 'localhost',
    'error' => 'Connection refused',
]);

// View last 20 log entries
$recent = $logger->tail(20);
foreach ($recent as $entry) {
    echo $entry . "\n";
}
```

---

## 🎯 สรุป Part 09

| Function/Class | การใช้งาน |
|----------------|-----------|
| file_get_contents / file_put_contents | อ่าน/เขียนไฟล์ง่าย |
| fopen / fread / fwrite | Control แบบละเอียด |
| FileManager | Secure file operations |
| JsonStorage | JSON-based data storage |
| Logger | Application logging |
| glob / DirectoryIterator | List files |

**ถัดไป → Part 10: Sessions & Cookies**
