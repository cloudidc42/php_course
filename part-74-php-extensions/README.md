# Part 74: PHP Extensions และ Libraries

## ขั้นตอนที่ 2081-2110: Popular PHP Extensions และ Libraries

PHP มี extensions และ libraries ที่ทรงพลังสำหรับงานต่างๆ ตั้งแต่ image processing ไปจนถึง spreadsheets

---

## ขั้นตอนที่ 2081: Image Processing กับ GD

```bash
# GD มาพร้อมกับ PHP ส่วนใหญ่
php -m | grep gd

# ถ้าไม่มี:
sudo apt install php8.3-gd
```

```php
<?php

declare(strict_types=1);

// app/Services/ImageService.php
namespace App\Services;

class ImageService
{
    /**
     * Resize image ด้วย GD
     */
    public function resize(string $sourcePath, string $destPath, int $width, int $height): void
    {
        $info     = getimagesize($sourcePath);
        $mimeType = $info['mime'];

        // Load source image
        $source = match ($mimeType) {
            'image/jpeg' => imagecreatefromjpeg($sourcePath),
            'image/png'  => imagecreatefrompng($sourcePath),
            'image/gif'  => imagecreatefromgif($sourcePath),
            'image/webp' => imagecreatefromwebp($sourcePath),
            default      => throw new \InvalidArgumentException("Unsupported: {$mimeType}"),
        };

        $origWidth  = imagesx($source);
        $origHeight = imagesy($source);

        // Calculate dimensions ให้ maintain aspect ratio
        $ratio = min($width / $origWidth, $height / $origHeight);
        $newWidth  = (int) round($origWidth * $ratio);
        $newHeight = (int) round($origHeight * $ratio);

        // Create destination image
        $dest = imagecreatetruecolor($newWidth, $newHeight);

        // Preserve transparency สำหรับ PNG/GIF
        if (in_array($mimeType, ['image/png', 'image/gif'])) {
            imagealphablending($dest, false);
            imagesavealpha($dest, true);
            $transparent = imagecolorallocatealpha($dest, 255, 255, 255, 127);
            imagefilledrectangle($dest, 0, 0, $newWidth, $newHeight, $transparent);
        }

        // Resize
        imagecopyresampled($dest, $source, 0, 0, 0, 0, $newWidth, $newHeight, $origWidth, $origHeight);

        // Save
        match ($mimeType) {
            'image/jpeg' => imagejpeg($dest, $destPath, 85),
            'image/png'  => imagepng($dest, $destPath, 6),
            'image/gif'  => imagegif($dest, $destPath),
            'image/webp' => imagewebp($dest, $destPath, 85),
        };

        imagedestroy($source);
        imagedestroy($dest);
    }

    /**
     * สร้าง thumbnail
     */
    public function createThumbnail(string $sourcePath, string $destPath, int $size = 200): void
    {
        $this->resize($sourcePath, $destPath, $size, $size);
    }

    /**
     * เพิ่ม watermark
     */
    public function addWatermark(string $imagePath, string $watermarkText, string $destPath): void
    {
        $image = imagecreatefromjpeg($imagePath);
        $width  = imagesx($image);
        $height = imagesy($image);

        // สร้าง color กับ transparency
        $white = imagecolorallocatealpha($image, 255, 255, 255, 80);

        // คำนวณ position (bottom-right)
        $fontSize = 5; // built-in font size
        $textWidth = imagefontwidth($fontSize) * strlen($watermarkText);
        $x = $width - $textWidth - 10;
        $y = $height - imagefontheight($fontSize) - 10;

        imagestring($image, $fontSize, $x, $y, $watermarkText, $white);

        imagejpeg($image, $destPath, 90);
        imagedestroy($image);
    }

    /**
     * Convert to WebP
     */
    public function convertToWebp(string $sourcePath, string $destPath, int $quality = 80): void
    {
        $info   = getimagesize($sourcePath);
        $source = match ($info['mime']) {
            'image/jpeg' => imagecreatefromjpeg($sourcePath),
            'image/png'  => imagecreatefrompng($sourcePath),
            default      => throw new \InvalidArgumentException("Cannot convert from: {$info['mime']}"),
        };

        imagewebp($source, $destPath, $quality);
        imagedestroy($source);
    }
}
```

---

## ขั้นตอนที่ 2082: Imagick (ImageMagick)

```bash
# ติดตั้ง ImageMagick + PHP extension
sudo apt install imagemagick php8.3-imagick
```

```php
<?php

declare(strict_types=1);

// app/Services/ImagickService.php
namespace App\Services;

use Imagick;
use ImagickDraw;
use ImagickPixel;

class ImagickService
{
    /**
     * Advanced image processing
     */
    public function processImage(string $sourcePath): Imagick
    {
        $image = new Imagick($sourcePath);

        // Auto-orient (แก้ rotation จาก EXIF)
        $image->autoOrient();

        // Strip EXIF data (เพื่อ privacy)
        $image->stripImage();

        return $image;
    }

    /**
     * Smart crop กับ face detection concept
     */
    public function smartCrop(string $sourcePath, int $width, int $height, string $destPath): void
    {
        $image = new Imagick($sourcePath);

        // Thumbnail แบบ smart (crop จาก center)
        $image->cropThumbnailImage($width, $height);

        // Sharp
        $image->sharpenImage(0, 0.5);

        $image->writeImage($destPath);
        $image->clear();
    }

    /**
     * สร้าง image collage
     */
    public function createCollage(array $imagePaths, string $outputPath, int $cols = 2): void
    {
        $thumbSize = 400;
        $padding   = 10;
        $rows      = (int) ceil(count($imagePaths) / $cols);

        $totalWidth  = $cols * $thumbSize + ($cols + 1) * $padding;
        $totalHeight = $rows * $thumbSize + ($rows + 1) * $padding;

        $canvas = new Imagick();
        $canvas->newImage($totalWidth, $totalHeight, new ImagickPixel('white'));
        $canvas->setImageFormat('jpg');

        foreach ($imagePaths as $index => $path) {
            $thumb = new Imagick($path);
            $thumb->cropThumbnailImage($thumbSize, $thumbSize);

            $col = $index % $cols;
            $row = (int) floor($index / $cols);
            $x   = $col * ($thumbSize + $padding) + $padding;
            $y   = $row * ($thumbSize + $padding) + $padding;

            $canvas->compositeImage($thumb, Imagick::COMPOSITE_OVER, $x, $y);
            $thumb->clear();
        }

        $canvas->writeImage($outputPath);
        $canvas->clear();
    }

    /**
     * Convert PDF to images
     */
    public function pdfToImages(string $pdfPath, string $outputDir, int $resolution = 150): array
    {
        $imagick = new Imagick();
        $imagick->setResolution($resolution, $resolution);
        $imagick->readImage($pdfPath);

        $files = [];
        foreach ($imagick as $i => $page) {
            $page->setImageFormat('jpeg');
            $page->setImageCompressionQuality(90);

            $outputPath = $outputDir . "/page_{$i}.jpg";
            $page->writeImage($outputPath);
            $files[] = $outputPath;
        }

        return $files;
    }
}
```

---

## ขั้นตอนที่ 2083: PDF Generation กับ DomPDF

```bash
composer require barryvdh/laravel-dompdf
```

```php
<?php

declare(strict_types=1);

// app/Services/PdfService.php
namespace App\Services;

use Barryvdh\DomPDF\Facade\Pdf;
use Illuminate\Http\Response;

class PdfService
{
    public function generateInvoice(\App\Models\Order $order): Response
    {
        $data = [
            'order'     => $order->load(['items.product', 'customer', 'shippingAddress']),
            'company'   => config('company'),
            'issued_at' => now()->format('d/m/Y'),
            'due_at'    => now()->addDays(30)->format('d/m/Y'),
        ];

        $pdf = Pdf::loadView('pdfs.invoice', $data);

        $pdf->setPaper('A4', 'portrait');
        $pdf->setOptions([
            'isHtml5ParserEnabled' => true,
            'isRemoteEnabled'      => true,
            'defaultFont'          => 'DejaVu Sans', // รองรับ Thai
            'dpi'                  => 150,
        ]);

        return $pdf->download("invoice-{$order->order_number}.pdf");
    }

    public function generateReport(array $data, string $title): \Illuminate\Http\Response
    {
        $pdf = Pdf::loadView('pdfs.report', compact('data', 'title'));
        $pdf->setPaper('A4', 'landscape');

        return $pdf->stream("{$title}.pdf");
    }
}
```

```html
{{-- resources/views/pdfs/invoice.blade.php --}}
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <style>
        body {
            font-family: 'DejaVu Sans', sans-serif;
            font-size: 12px;
            color: #333;
        }
        .header { display: flex; justify-content: space-between; margin-bottom: 30px; }
        .invoice-title { font-size: 28px; font-weight: bold; color: #2c3e50; }
        table { width: 100%; border-collapse: collapse; }
        th, td { padding: 8px 12px; text-align: left; border-bottom: 1px solid #ddd; }
        th { background: #2c3e50; color: white; }
        .total-row { font-weight: bold; font-size: 14px; }
    </style>
</head>
<body>
    <div class="header">
        <div>
            <div class="invoice-title">INVOICE</div>
            <div>{{ $company['name'] }}</div>
            <div>{{ $company['address'] }}</div>
        </div>
        <div>
            <div><strong>Invoice No:</strong> {{ $order->order_number }}</div>
            <div><strong>Date:</strong> {{ $issued_at }}</div>
            <div><strong>Due:</strong> {{ $due_at }}</div>
        </div>
    </div>

    <table>
        <thead>
            <tr>
                <th>รายการ</th>
                <th>จำนวน</th>
                <th>ราคา/หน่วย</th>
                <th>รวม</th>
            </tr>
        </thead>
        <tbody>
            @foreach($order->items as $item)
            <tr>
                <td>{{ $item->product->name }}</td>
                <td>{{ $item->quantity }}</td>
                <td>฿{{ number_format($item->unit_price, 2) }}</td>
                <td>฿{{ number_format($item->total, 2) }}</td>
            </tr>
            @endforeach
        </tbody>
        <tfoot>
            <tr class="total-row">
                <td colspan="3">ยอดรวม</td>
                <td>฿{{ number_format($order->total, 2) }}</td>
            </tr>
        </tfoot>
    </table>
</body>
</html>
```

---

## ขั้นตอนที่ 2084: QR Code Generation

```bash
composer require simplesoftwareio/simple-qrcode
```

```php
<?php

declare(strict_types=1);

// app/Services/QrCodeService.php
namespace App\Services;

use SimpleSoftwareIO\QrCode\Facades\QrCode;

class QrCodeService
{
    public function generateProductQr(int $productId): string
    {
        $url = route('products.show', $productId);

        return QrCode::format('png')
            ->size(300)
            ->errorCorrection('H')  // High error correction
            ->margin(2)
            ->color(44, 62, 80)     // Dark blue
            ->backgroundColor(255, 255, 255)
            ->encoding('UTF-8')
            ->generate($url);
    }

    public function generateWithLogo(string $data, string $logoPath): string
    {
        return QrCode::format('png')
            ->size(400)
            ->errorCorrection('H')
            ->merge($logoPath, 0.3)  // logo 30% of QR size
            ->generate($data);
    }

    public function generateVCard(array $contact): string
    {
        $vcard = QrCode::format('svg')
            ->size(250)
            ->contact(
                name:  $contact['name'],
                email: $contact['email'],
                phone: $contact['phone'],
            )
            ->generate();

        return base64_encode($vcard);
    }

    public function saveToFile(string $data, string $filePath): void
    {
        QrCode::format('png')
            ->size(300)
            ->generate($data, $filePath);
    }
}
```

---

## ขั้นตอนที่ 2085: PhpSpreadsheet

```bash
composer require phpoffice/phpspreadsheet
```

```php
<?php

declare(strict_types=1);

// app/Services/ExcelService.php
namespace App\Services;

use PhpOffice\PhpSpreadsheet\Spreadsheet;
use PhpOffice\PhpSpreadsheet\Writer\Xlsx;
use PhpOffice\PhpSpreadsheet\Style\Alignment;
use PhpOffice\PhpSpreadsheet\Style\Border;
use PhpOffice\PhpSpreadsheet\Style\Fill;
use PhpOffice\PhpSpreadsheet\Chart\Chart;
use PhpOffice\PhpSpreadsheet\Chart\DataSeries;
use PhpOffice\PhpSpreadsheet\Chart\PlotArea;
use PhpOffice\PhpSpreadsheet\Chart\DataSeriesValues;
use Symfony\Component\HttpFoundation\StreamedResponse;

class ExcelService
{
    public function generateSalesReport(array $data): StreamedResponse
    {
        $spreadsheet = new Spreadsheet();
        $sheet       = $spreadsheet->getActiveSheet();
        $sheet->setTitle('Sales Report');

        // Header styling
        $headerStyle = [
            'font'      => ['bold' => true, 'color' => ['rgb' => 'FFFFFF']],
            'fill'      => ['fillType' => Fill::FILL_SOLID, 'startColor' => ['rgb' => '2C3E50']],
            'alignment' => ['horizontal' => Alignment::HORIZONTAL_CENTER],
            'borders'   => ['allBorders' => ['borderStyle' => Border::BORDER_THIN]],
        ];

        // Headers
        $headers = ['วันที่', 'สินค้า', 'จำนวน', 'ราคา/หน่วย', 'รวม'];
        foreach ($headers as $col => $header) {
            $cell = chr(65 + $col) . '1';
            $sheet->setCellValue($cell, $header);
            $sheet->getStyle($cell)->applyFromArray($headerStyle);
            $sheet->getColumnDimension(chr(65 + $col))->setAutoSize(true);
        }

        // Data rows
        $row      = 2;
        $totalSum = 0;

        foreach ($data as $item) {
            $sheet->setCellValue("A{$row}", $item['date']);
            $sheet->setCellValue("B{$row}", $item['product']);
            $sheet->setCellValue("C{$row}", $item['quantity']);
            $sheet->setCellValue("D{$row}", $item['unit_price']);
            $sheet->setCellValue("E{$row}", "=C{$row}*D{$row}");

            // Number format
            $sheet->getStyle("D{$row}")->getNumberFormat()->setFormatCode('#,##0.00');
            $sheet->getStyle("E{$row}")->getNumberFormat()->setFormatCode('#,##0.00');

            $totalSum += $item['quantity'] * $item['unit_price'];
            $row++;
        }

        // Total row
        $sheet->setCellValue("D{$row}", 'รวมทั้งสิ้น');
        $sheet->setCellValue("E{$row}", "=SUM(E2:E" . ($row - 1) . ")");
        $sheet->getStyle("D{$row}:E{$row}")->getFont()->setBold(true);

        // Freeze panes
        $sheet->freezePane('A2');

        // Auto filter
        $sheet->setAutoFilter("A1:E1");

        return new StreamedResponse(function () use ($spreadsheet) {
            $writer = new Xlsx($spreadsheet);
            $writer->save('php://output');
        }, 200, [
            'Content-Type'        => 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
            'Content-Disposition' => 'attachment; filename="sales-report.xlsx"',
        ]);
    }

    public function importOrders(string $filePath): array
    {
        $reader      = new \PhpOffice\PhpSpreadsheet\Reader\Xlsx();
        $spreadsheet = $reader->load($filePath);
        $sheet       = $spreadsheet->getActiveSheet();

        $orders = [];
        $lastRow = $sheet->getHighestRow();

        for ($row = 2; $row <= $lastRow; $row++) {
            $orders[] = [
                'order_number'  => $sheet->getCell("A{$row}")->getValue(),
                'product_name'  => $sheet->getCell("B{$row}")->getValue(),
                'quantity'      => (int) $sheet->getCell("C{$row}")->getValue(),
                'price'         => (float) $sheet->getCell("D{$row}")->getValue(),
            ];
        }

        return array_filter($orders, fn ($o) => !empty($o['order_number']));
    }
}
```

---

## ขั้นตอนที่ 2086: PHPMailer

```bash
composer require phpmailer/phpmailer
```

```php
<?php

declare(strict_types=1);

// app/Services/DirectMailService.php
namespace App\Services;

use PHPMailer\PHPMailer\PHPMailer;
use PHPMailer\PHPMailer\SMTP;
use PHPMailer\PHPMailer\Exception;

class DirectMailService
{
    private PHPMailer $mailer;

    public function __construct()
    {
        $this->mailer = new PHPMailer(exceptions: true);

        $this->mailer->isSMTP();
        $this->mailer->Host        = config('mail.mailers.smtp.host');
        $this->mailer->SMTPAuth    = true;
        $this->mailer->Username    = config('mail.mailers.smtp.username');
        $this->mailer->Password    = config('mail.mailers.smtp.password');
        $this->mailer->SMTPSecure  = PHPMailer::ENCRYPTION_STARTTLS;
        $this->mailer->Port        = config('mail.mailers.smtp.port', 587);
        $this->mailer->CharSet     = 'UTF-8';
        $this->mailer->Encoding    = 'base64';

        // From
        $this->mailer->setFrom(
            config('mail.from.address'),
            config('mail.from.name')
        );
    }

    public function sendOrderConfirmation(\App\Models\Order $order): bool
    {
        try {
            $this->mailer->clearAddresses();
            $this->mailer->addAddress($order->customer->email, $order->customer->name);
            $this->mailer->Subject = "ยืนยันคำสั่งซื้อ #{$order->order_number}";
            $this->mailer->isHTML(true);
            $this->mailer->Body    = view('emails.order-confirmation', compact('order'))->render();
            $this->mailer->AltBody = strip_tags($this->mailer->Body);

            // Attach PDF invoice
            $pdfPath = storage_path("invoices/{$order->order_number}.pdf");
            if (file_exists($pdfPath)) {
                $this->mailer->addAttachment($pdfPath, "Invoice-{$order->order_number}.pdf");
            }

            return $this->mailer->send();

        } catch (Exception $e) {
            \Log::error("Email send failed: {$e->getMessage()}");
            return false;
        }
    }
}
```

---

## สรุป Part 74

| Extension/Library | ใช้งาน | Package |
|------------------|--------|---------|
| GD | Basic image resize, watermark | Built-in |
| Imagick | Advanced processing, PDF | ext-imagick |
| DomPDF | HTML to PDF | barryvdh/laravel-dompdf |
| mPDF | PDF with Thai support | mpdf/mpdf |
| QR Code | Generate QR codes | simplesoftwareio/simple-qrcode |
| PhpSpreadsheet | Excel read/write | phpoffice/phpspreadsheet |
| PHPMailer | Direct SMTP | phpmailer/phpmailer |
| intl | Internationalization | Built-in ext-intl |

ถัดไป → Part 75: Architecture Patterns
