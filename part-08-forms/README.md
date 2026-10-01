# Part 08: Forms & User Input Validation
## ขั้นตอนที่ 181-210: การจัดการ Forms อย่างมืออาชีพ

---

## ขั้นตอนที่ 181: HTML Forms กับ PHP

```php
<?php
declare(strict_types=1);
// form_handler.php

// Handle Form Submission
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    // รับข้อมูลจาก Form
    $name = trim($_POST['name'] ?? '');
    $email = trim($_POST['email'] ?? '');
    $message = trim($_POST['message'] ?? '');
    
    $errors = [];
    
    // Validate
    if (empty($name)) {
        $errors['name'] = 'กรุณากรอกชื่อ';
    } elseif (mb_strlen($name) < 2) {
        $errors['name'] = 'ชื่อต้องมีอย่างน้อย 2 ตัวอักษร';
    }
    
    if (empty($email)) {
        $errors['email'] = 'กรุณากรอก Email';
    } elseif (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
        $errors['email'] = 'Email ไม่ถูกต้อง';
    }
    
    if (empty($errors)) {
        // บันทึกข้อมูลหรือส่ง Email
        $success = true;
    }
}

// esc helper
function esc(string $value): string {
    return htmlspecialchars($value, ENT_QUOTES | ENT_HTML5, 'UTF-8');
}
?>
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Contact Form</title>
</head>
<body>
<?php if (isset($success)): ?>
    <div class="success">ส่งข้อความสำเร็จ!</div>
<?php else: ?>
    <form method="POST" action="">
        <?php if (!empty($errors)): ?>
            <div class="errors">
                <ul>
                    <?php foreach ($errors as $error): ?>
                        <li><?= esc($error) ?></li>
                    <?php endforeach; ?>
                </ul>
            </div>
        <?php endif; ?>
        
        <div class="field">
            <label for="name">ชื่อ-นามสกุล *</label>
            <input type="text" 
                   id="name" 
                   name="name" 
                   value="<?= esc($name ?? '') ?>"
                   class="<?= isset($errors['name']) ? 'error' : '' ?>">
            <?php if (isset($errors['name'])): ?>
                <span class="error-msg"><?= esc($errors['name']) ?></span>
            <?php endif; ?>
        </div>
        
        <div class="field">
            <label for="email">Email *</label>
            <input type="email" id="email" name="email" value="<?= esc($email ?? '') ?>">
            <?php if (isset($errors['email'])): ?>
                <span class="error-msg"><?= esc($errors['email']) ?></span>
            <?php endif; ?>
        </div>
        
        <div class="field">
            <label for="message">ข้อความ</label>
            <textarea id="message" name="message" rows="5"><?= esc($message ?? '') ?></textarea>
        </div>
        
        <!-- CSRF Token -->
        <input type="hidden" name="_token" value="<?= generateCsrfToken() ?>">
        
        <button type="submit">ส่งข้อความ</button>
    </form>
<?php endif; ?>
</body>
</html>
```

---

## ขั้นตอนที่ 182: CSRF Protection

```php
<?php
declare(strict_types=1);

class CsrfProtection {
    private const TOKEN_LENGTH = 32;
    private const SESSION_KEY = '_csrf_tokens';
    
    public static function generateToken(string $formId = 'default'): string {
        if (session_status() !== PHP_SESSION_ACTIVE) {
            session_start();
        }
        
        $token = bin2hex(random_bytes(self::TOKEN_LENGTH));
        
        if (!isset($_SESSION[self::SESSION_KEY])) {
            $_SESSION[self::SESSION_KEY] = [];
        }
        
        $_SESSION[self::SESSION_KEY][$formId] = [
            'token' => $token,
            'expires' => time() + 3600, // 1 hour
        ];
        
        return $token;
    }
    
    public static function validateToken(string $token, string $formId = 'default'): bool {
        if (session_status() !== PHP_SESSION_ACTIVE) {
            session_start();
        }
        
        $stored = $_SESSION[self::SESSION_KEY][$formId] ?? null;
        
        if (!$stored) {
            return false;
        }
        
        // ลบ Token หลังใช้แล้ว (one-time use)
        unset($_SESSION[self::SESSION_KEY][$formId]);
        
        // Check expiry
        if ($stored['expires'] < time()) {
            return false;
        }
        
        // Constant-time comparison (ป้องกัน timing attack)
        return hash_equals($stored['token'], $token);
    }
    
    public static function getHiddenInput(string $formId = 'default'): string {
        $token = static::generateToken($formId);
        return sprintf(
            '<input type="hidden" name="_csrf_token" value="%s">',
            htmlspecialchars($token, ENT_QUOTES, 'UTF-8')
        );
    }
}

// Middleware สำหรับตรวจสอบ CSRF
function requireCsrf(string $formId = 'default'): void {
    if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
        return;
    }
    
    $token = $_POST['_csrf_token'] ?? '';
    
    if (!CsrfProtection::validateToken($token, $formId)) {
        http_response_code(403);
        die('CSRF token mismatch');
    }
}

// ใน form
echo CsrfProtection::getHiddenInput('contact_form');

// ใน handler
requireCsrf('contact_form');
```

---

## ขั้นตอนที่ 183: Form Validation Class

```php
<?php
declare(strict_types=1);

class Validator {
    private array $data;
    private array $errors = [];
    private array $rules;
    
    private array $messages = [
        'required' => ':field จำเป็นต้องกรอก',
        'email' => ':field ต้องเป็น email ที่ถูกต้อง',
        'min' => ':field ต้องมีความยาวอย่างน้อย :param ตัวอักษร',
        'max' => ':field ต้องมีความยาวไม่เกิน :param ตัวอักษร',
        'numeric' => ':field ต้องเป็นตัวเลข',
        'integer' => ':field ต้องเป็นจำนวนเต็ม',
        'between' => ':field ต้องอยู่ระหว่าง :min และ :max',
        'in' => ':field ต้องเป็นหนึ่งใน: :values',
        'confirmed' => ':field ไม่ตรงกัน',
        'url' => ':field ต้องเป็น URL ที่ถูกต้อง',
        'regex' => ':field รูปแบบไม่ถูกต้อง',
        'unique' => ':field นี้มีอยู่แล้ว',
        'phone' => ':field ต้องเป็นเบอร์โทรที่ถูกต้อง',
        'date' => ':field ต้องเป็นวันที่ที่ถูกต้อง',
        'before' => ':field ต้องเป็นวันที่ก่อน :param',
        'after' => ':field ต้องเป็นวันที่หลัง :param',
        'file_size' => 'ไฟล์ :field ต้องมีขนาดไม่เกิน :param MB',
        'file_type' => 'ไฟล์ :field ต้องเป็นประเภท: :values',
    ];
    
    public function __construct(array $data, array $rules) {
        $this->data = $data;
        $this->rules = $rules;
    }
    
    public static function make(array $data, array $rules): self {
        return new self($data, $rules);
    }
    
    public function validate(): bool {
        foreach ($this->rules as $field => $ruleSet) {
            $rules = is_string($ruleSet) ? explode('|', $ruleSet) : $ruleSet;
            $value = $this->getValue($field);
            
            foreach ($rules as $rule) {
                $this->applyRule($field, $value, $rule);
            }
        }
        
        return empty($this->errors);
    }
    
    private function getValue(string $field): mixed {
        $keys = explode('.', $field);
        $data = $this->data;
        
        foreach ($keys as $key) {
            if (!isset($data[$key])) return null;
            $data = $data[$key];
        }
        
        return $data;
    }
    
    private function applyRule(string $field, mixed $value, string $rule): void {
        // Parse rule and params
        [$ruleName, $params] = str_contains($rule, ':')
            ? explode(':', $rule, 2)
            : [$rule, null];
        
        $method = 'validate' . ucfirst($ruleName);
        
        if (method_exists($this, $method)) {
            if (!$this->$method($value, $params, $field)) {
                $this->addError($field, $ruleName, $params);
            }
        }
    }
    
    private function addError(string $field, string $rule, ?string $params): void {
        $template = $this->messages[$rule] ?? ":field ไม่ถูกต้อง";
        
        $message = str_replace(':field', $this->formatFieldName($field), $template);
        
        if ($params !== null) {
            if (str_contains($template, ':min') && str_contains($template, ':max')) {
                [$min, $max] = explode(',', $params);
                $message = str_replace([':min', ':max'], [$min, $max], $message);
            } elseif (str_contains($template, ':values')) {
                $message = str_replace(':values', $params, $message);
            } else {
                $message = str_replace(':param', $params, $message);
            }
        }
        
        $this->errors[$field][] = $message;
    }
    
    private function formatFieldName(string $field): string {
        return ucfirst(str_replace(['_', '.'], ' ', $field));
    }
    
    // Validation Methods
    protected function validateRequired(mixed $value): bool {
        if (is_null($value)) return false;
        if (is_string($value)) return trim($value) !== '';
        if (is_array($value)) return count($value) > 0;
        return true;
    }
    
    protected function validateEmail(mixed $value): bool {
        if (empty($value)) return true; // Required handles empty
        return filter_var($value, FILTER_VALIDATE_EMAIL) !== false;
    }
    
    protected function validateMin(mixed $value, ?string $min): bool {
        if (empty($value)) return true;
        if (is_numeric($value)) return (float)$value >= (float)$min;
        return mb_strlen((string)$value) >= (int)$min;
    }
    
    protected function validateMax(mixed $value, ?string $max): bool {
        if (empty($value)) return true;
        if (is_numeric($value)) return (float)$value <= (float)$max;
        return mb_strlen((string)$value) <= (int)$max;
    }
    
    protected function validateNumeric(mixed $value): bool {
        if (empty($value)) return true;
        return is_numeric($value);
    }
    
    protected function validateInteger(mixed $value): bool {
        if (empty($value)) return true;
        return filter_var($value, FILTER_VALIDATE_INT) !== false;
    }
    
    protected function validateBetween(mixed $value, ?string $params): bool {
        if (empty($value)) return true;
        [$min, $max] = explode(',', $params ?? '0,100');
        $num = (float)$value;
        return $num >= (float)$min && $num <= (float)$max;
    }
    
    protected function validateIn(mixed $value, ?string $params): bool {
        if (empty($value)) return true;
        $allowed = explode(',', $params ?? '');
        return in_array($value, $allowed, true);
    }
    
    protected function validateConfirmed(mixed $value, ?string $params, string $field): bool {
        $confirmField = $params ?? $field . '_confirmation';
        return $value === $this->getValue($confirmField);
    }
    
    protected function validateUrl(mixed $value): bool {
        if (empty($value)) return true;
        return filter_var($value, FILTER_VALIDATE_URL) !== false;
    }
    
    protected function validateRegex(mixed $value, ?string $pattern): bool {
        if (empty($value)) return true;
        return (bool) preg_match($pattern ?? '/./', (string)$value);
    }
    
    protected function validatePhone(mixed $value): bool {
        if (empty($value)) return true;
        $phone = preg_replace('/[\s\-\(\)]/', '', (string)$value);
        return (bool) preg_match('/^(0[689]\d{8}|0[2-9]\d{7}|\+66\d{9})$/', $phone);
    }
    
    protected function validateDate(mixed $value): bool {
        if (empty($value)) return true;
        $date = date_create($value);
        return $date !== false;
    }
    
    // Getters
    public function errors(): array { return $this->errors; }
    public function firstError(string $field): ?string {
        return $this->errors[$field][0] ?? null;
    }
    public function passes(): bool { return empty($this->errors); }
    public function fails(): bool { return !$this->passes(); }
    
    public function validated(): array {
        if ($this->fails()) {
            throw new RuntimeException('Cannot get validated data with errors');
        }
        return array_intersect_key($this->data, $this->rules);
    }
}

// ========================
// Usage
// ========================
$data = [
    'name' => 'สมชาย ใจดี',
    'email' => 'somchai@example.com',
    'age' => 25,
    'phone' => '0812345678',
    'password' => 'Secret123!',
    'password_confirmation' => 'Secret123!',
    'website' => 'https://example.com',
];

$validator = Validator::make($data, [
    'name' => 'required|min:2|max:100',
    'email' => 'required|email',
    'age' => 'required|integer|between:18,99',
    'phone' => 'required|phone',
    'password' => 'required|min:8|confirmed',
    'website' => 'url',
]);

if ($validator->fails()) {
    print_r($validator->errors());
} else {
    $validated = $validator->validated();
    echo "Validation passed!";
}
```

---

## ขั้นตอนที่ 184: File Upload Handling

```php
<?php
declare(strict_types=1);

class FileUploader {
    private array $config;
    private array $errors = [];
    
    public function __construct(array $config = []) {
        $this->config = array_merge([
            'max_size' => 5 * 1024 * 1024,  // 5MB
            'allowed_types' => ['image/jpeg', 'image/png', 'image/gif', 'image/webp'],
            'allowed_extensions' => ['jpg', 'jpeg', 'png', 'gif', 'webp'],
            'upload_dir' => '/var/www/html/uploads/',
            'create_dir' => true,
            'randomize_name' => true,
            'max_width' => 2000,
            'max_height' => 2000,
        ], $config);
    }
    
    public function upload(array $file, string $subDir = ''): ?string {
        $this->errors = [];
        
        // Check for upload errors
        if ($file['error'] !== UPLOAD_ERR_OK) {
            $this->errors[] = $this->getUploadError($file['error']);
            return null;
        }
        
        // Validate file size
        if ($file['size'] > $this->config['max_size']) {
            $maxMb = $this->config['max_size'] / 1024 / 1024;
            $this->errors[] = "ขนาดไฟล์ต้องไม่เกิน {$maxMb}MB";
            return null;
        }
        
        // Validate MIME type (อย่าเชื่อ $_FILES['type'])
        $finfo = new finfo(FILEINFO_MIME_TYPE);
        $mimeType = $finfo->file($file['tmp_name']);
        
        if (!in_array($mimeType, $this->config['allowed_types'])) {
            $this->errors[] = 'ประเภทไฟล์ไม่ได้รับอนุญาต';
            return null;
        }
        
        // Validate extension
        $ext = strtolower(pathinfo($file['name'], PATHINFO_EXTENSION));
        if (!in_array($ext, $this->config['allowed_extensions'])) {
            $this->errors[] = 'นามสกุลไฟล์ไม่ได้รับอนุญาต';
            return null;
        }
        
        // Validate image dimensions (for images)
        if (str_starts_with($mimeType, 'image/')) {
            $imageInfo = getimagesize($file['tmp_name']);
            if ($imageInfo === false) {
                $this->errors[] = 'ไฟล์รูปภาพไม่ถูกต้อง';
                return null;
            }
            
            [$width, $height] = $imageInfo;
            if ($width > $this->config['max_width'] || $height > $this->config['max_height']) {
                $this->errors[] = "รูปภาพต้องมีขนาดไม่เกิน {$this->config['max_width']}x{$this->config['max_height']} pixels";
                return null;
            }
        }
        
        // Prepare upload directory
        $uploadDir = rtrim($this->config['upload_dir'], '/');
        if ($subDir) {
            $uploadDir .= '/' . trim($subDir, '/');
        }
        
        if (!is_dir($uploadDir)) {
            if ($this->config['create_dir']) {
                if (!mkdir($uploadDir, 0755, true)) {
                    $this->errors[] = 'ไม่สามารถสร้างโฟลเดอร์ได้';
                    return null;
                }
            } else {
                $this->errors[] = 'โฟลเดอร์ปลายทางไม่มีอยู่';
                return null;
            }
        }
        
        // Generate filename
        $filename = $this->config['randomize_name']
            ? bin2hex(random_bytes(16)) . '.' . $ext
            : $this->sanitizeFilename($file['name']);
        
        $destination = $uploadDir . '/' . $filename;
        
        // Move file
        if (!move_uploaded_file($file['tmp_name'], $destination)) {
            $this->errors[] = 'ไม่สามารถย้ายไฟล์ได้';
            return null;
        }
        
        // Add date-based subfolder to return path
        return ($subDir ? $subDir . '/' : '') . $filename;
    }
    
    public function uploadMultiple(array $files, string $subDir = ''): array {
        $uploaded = [];
        $allFiles = $this->reindexFiles($files);
        
        foreach ($allFiles as $file) {
            $path = $this->upload($file, $subDir);
            if ($path !== null) {
                $uploaded[] = $path;
            }
        }
        
        return $uploaded;
    }
    
    private function reindexFiles(array $files): array {
        $result = [];
        $count = count($files['name']);
        
        for ($i = 0; $i < $count; $i++) {
            $result[] = [
                'name' => $files['name'][$i],
                'type' => $files['type'][$i],
                'tmp_name' => $files['tmp_name'][$i],
                'error' => $files['error'][$i],
                'size' => $files['size'][$i],
            ];
        }
        
        return $result;
    }
    
    private function sanitizeFilename(string $filename): string {
        $ext = strtolower(pathinfo($filename, PATHINFO_EXTENSION));
        $name = pathinfo($filename, PATHINFO_FILENAME);
        $name = preg_replace('/[^a-zA-Z0-9_\-]/', '_', $name);
        $name = preg_replace('/_+/', '_', trim($name, '_'));
        return $name . '_' . time() . '.' . $ext;
    }
    
    private function getUploadError(int $code): string {
        return match($code) {
            UPLOAD_ERR_INI_SIZE => 'ไฟล์ใหญ่เกิน upload_max_filesize ใน php.ini',
            UPLOAD_ERR_FORM_SIZE => 'ไฟล์ใหญ่เกิน MAX_FILE_SIZE ใน form',
            UPLOAD_ERR_PARTIAL => 'ไฟล์ถูก Upload บางส่วนเท่านั้น',
            UPLOAD_ERR_NO_FILE => 'ไม่มีไฟล์ถูก Upload',
            UPLOAD_ERR_NO_TMP_DIR => 'ไม่มี temporary folder',
            UPLOAD_ERR_CANT_WRITE => 'ไม่สามารถเขียนไฟล์ลงดิสก์ได้',
            UPLOAD_ERR_EXTENSION => 'Extension ของ PHP หยุด Upload',
            default => 'เกิดข้อผิดพลาดในการ Upload',
        };
    }
    
    public function errors(): array { return $this->errors; }
    public function hasErrors(): bool { return !empty($this->errors); }
}

// ========================
// Usage
// ========================
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_FILES['avatar'])) {
    $uploader = new FileUploader([
        'max_size' => 2 * 1024 * 1024, // 2MB
        'upload_dir' => __DIR__ . '/uploads',
    ]);
    
    $path = $uploader->upload($_FILES['avatar'], date('Y/m'));
    
    if ($uploader->hasErrors()) {
        foreach ($uploader->errors() as $error) {
            echo $error . "\n";
        }
    } else {
        echo "Uploaded: " . $path;
    }
}
```

---

## ขั้นตอนที่ 185: Form Builder Class

```php
<?php
declare(strict_types=1);

class FormBuilder {
    private string $html = '';
    private array $errors = [];
    private array $old = [];
    
    public function __construct(array $errors = [], array $old = []) {
        $this->errors = $errors;
        $this->old = $old;
    }
    
    public function open(string $action = '', string $method = 'POST', array $attrs = []): self {
        $multipart = in_array('enctype', array_keys($attrs)) ? '' : '';
        
        $html = sprintf(
            '<form action="%s" method="%s"%s>',
            $this->esc($action),
            strtoupper($method) === 'GET' ? 'GET' : 'POST',
            $this->buildAttrs($attrs)
        );
        
        if (strtoupper($method) === 'POST') {
            $html .= CsrfProtection::getHiddenInput();
        }
        
        if (!in_array(strtoupper($method), ['GET', 'POST'])) {
            $html .= sprintf('<input type="hidden" name="_method" value="%s">', strtoupper($method));
        }
        
        $this->html .= $html;
        return $this;
    }
    
    public function close(): self {
        $this->html .= '</form>';
        return $this;
    }
    
    public function input(
        string $type,
        string $name,
        string $label,
        array $attrs = []
    ): self {
        $value = $this->old($name) ?? ($attrs['value'] ?? '');
        $error = $this->errors[$name][0] ?? null;
        $id = $attrs['id'] ?? $name;
        
        unset($attrs['value']);
        
        $this->html .= sprintf(
            '<div class="form-group%s">
                <label for="%s">%s</label>
                <input type="%s" id="%s" name="%s" value="%s"%s>
                %s
            </div>',
            $error ? ' has-error' : '',
            $this->esc($id),
            $this->esc($label),
            $this->esc($type),
            $this->esc($id),
            $this->esc($name),
            $this->esc((string)$value),
            $this->buildAttrs($attrs),
            $error ? sprintf('<span class="error-message">%s</span>', $this->esc($error)) : ''
        );
        
        return $this;
    }
    
    public function text(string $name, string $label, array $attrs = []): self {
        return $this->input('text', $name, $label, $attrs);
    }
    
    public function email(string $name, string $label, array $attrs = []): self {
        return $this->input('email', $name, $label, $attrs);
    }
    
    public function password(string $name, string $label, array $attrs = []): self {
        return $this->input('password', $name, $label, $attrs);
    }
    
    public function textarea(string $name, string $label, array $attrs = []): self {
        $value = $this->old($name) ?? ($attrs['value'] ?? '');
        $error = $this->errors[$name][0] ?? null;
        $id = $attrs['id'] ?? $name;
        $rows = $attrs['rows'] ?? 5;
        
        unset($attrs['value'], $attrs['rows']);
        
        $this->html .= sprintf(
            '<div class="form-group%s">
                <label for="%s">%s</label>
                <textarea id="%s" name="%s" rows="%d"%s>%s</textarea>
                %s
            </div>',
            $error ? ' has-error' : '',
            $this->esc($id),
            $this->esc($label),
            $this->esc($id),
            $this->esc($name),
            $rows,
            $this->buildAttrs($attrs),
            $this->esc((string)$value),
            $error ? sprintf('<span class="error-message">%s</span>', $this->esc($error)) : ''
        );
        
        return $this;
    }
    
    public function select(string $name, string $label, array $options, array $attrs = []): self {
        $selected = $this->old($name) ?? ($attrs['selected'] ?? null);
        $error = $this->errors[$name][0] ?? null;
        $id = $attrs['id'] ?? $name;
        
        unset($attrs['selected']);
        
        $optionsHtml = '<option value="">-- เลือก --</option>';
        foreach ($options as $value => $text) {
            $optionsHtml .= sprintf(
                '<option value="%s"%s>%s</option>',
                $this->esc((string)$value),
                $selected == $value ? ' selected' : '',
                $this->esc($text)
            );
        }
        
        $this->html .= sprintf(
            '<div class="form-group%s">
                <label for="%s">%s</label>
                <select id="%s" name="%s"%s>%s</select>
                %s
            </div>',
            $error ? ' has-error' : '',
            $this->esc($id),
            $this->esc($label),
            $this->esc($id),
            $this->esc($name),
            $this->buildAttrs($attrs),
            $optionsHtml,
            $error ? sprintf('<span class="error-message">%s</span>', $this->esc($error)) : ''
        );
        
        return $this;
    }
    
    public function submit(string $label = 'บันทึก', array $attrs = []): self {
        $this->html .= sprintf(
            '<button type="submit"%s>%s</button>',
            $this->buildAttrs($attrs),
            $this->esc($label)
        );
        return $this;
    }
    
    public function render(): string {
        return $this->html;
    }
    
    public function __toString(): string {
        return $this->render();
    }
    
    private function old(string $name): mixed {
        return $this->old[$name] ?? null;
    }
    
    private function esc(string $value): string {
        return htmlspecialchars($value, ENT_QUOTES | ENT_HTML5, 'UTF-8');
    }
    
    private function buildAttrs(array $attrs): string {
        $result = '';
        foreach ($attrs as $key => $value) {
            if ($value === true) {
                $result .= " {$key}";
            } elseif ($value !== false) {
                $result .= sprintf(' %s="%s"', $key, $this->esc((string)$value));
            }
        }
        return $result;
    }
}

// Usage
$form = new FormBuilder($errors ?? [], $_POST);

echo $form
    ->open('/users/store', 'POST')
    ->text('name', 'ชื่อ-นามสกุล', ['required' => true, 'placeholder' => 'กรอกชื่อ'])
    ->email('email', 'Email', ['required' => true])
    ->select('role', 'บทบาท', ['admin' => 'Admin', 'user' => 'User'])
    ->textarea('bio', 'ประวัติ', ['rows' => 3])
    ->submit('บันทึก', ['class' => 'btn btn-primary'])
    ->close();
```

---

## 🎯 สรุป Part 08

| หัวข้อ | สิ่งที่สำคัญ |
|--------|-------------|
| Form Handling | $_POST, $_GET, REQUEST_METHOD |
| CSRF Protection | Token generation, hash_equals |
| Validation | Rules, error messages, sanitization |
| File Upload | MIME check, size limit, sanitize filename |
| Form Builder | Fluent interface, error display |

**ถัดไป → Part 09: File System & File Handling**
