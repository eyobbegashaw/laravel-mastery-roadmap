### Detailed Conceptual Overview (70%)

File storage in modern web applications goes far beyond saving files to a directory. Laravel's filesystem abstraction provides a unified API for local storage, cloud services (Amazon S3, Google Cloud Storage, Azure Blob Storage), and FTP/SFTP servers. This abstraction allows you to write file handling code once and switch storage backends through configuration.

**The Filesystem Abstraction Layer**

Laravel's filesystem uses "disks"—named storage configurations defined in `config/filesystems.php`. Each disk specifies a driver and configuration. The `local` disk stores files in `storage/app`. The `public` disk stores files in `storage/app/public`, accessible via a symbolic link. The `s3` disk connects to Amazon S3 with credentials from environment variables.

The `Storage` facade provides a consistent API across all disks. `Storage::disk('s3')->put('avatars/user1.jpg', $contents)` works identically whether the disk is local or S3. This consistency means you can develop locally with the local disk and deploy to production with S3 without changing any file handling code.

**Public vs. Private Files**

Public files are accessible directly via URL—user avatars, product images, downloaded documents. They reside in `storage/app/public`, symlinked to `public/storage`. Private files require authentication—invoices, contracts, internal documents. They reside anywhere in `storage/app` and are served through controller methods that perform authorization checks.

The `public` disk is configured to use the `local` driver with root at `storage/app/public`. The `s3` disk can serve public files through S3's public URLs or CloudFront distributions. Private S3 files use temporary signed URLs that expire after a set time.

**File Upload Handling**

Laravel's `UploadedFile` class wraps PHP's file upload handling with a clean API. `$request->file('avatar')` returns an `UploadedFile` instance with methods for validation, storage, and manipulation. The `store()` method saves the file and returns the path. The `storeAs()` method specifies a custom filename.

File validation happens before storage. Rules like `image`, `mimes:jpeg,png`, `max:2048`, and `dimensions:min_width=100` ensure uploaded files meet requirements. Validation prevents malicious files, oversized uploads, and incorrect formats from reaching your storage.

**File Manipulation and Transformation**

Image manipulation integrates with Laravel through intervention/image package. Uploaded images can be resized, cropped, watermarked, and converted between formats before storage. A common pattern uploads the original image, then generates multiple sizes (thumbnail, medium, large) for different display contexts.

File streaming handles large files efficiently. Instead of loading an entire file into memory, `Storage::download()` streams the file to the browser. For S3, `Storage::temporaryUrl()` generates a time-limited direct download URL, offloading bandwidth from your server.

**Filesystem Events**

Laravel fires filesystem events when files are stored, retrieved, or deleted. `Illuminate\Filesystem\Events\FileStored` fires after a successful store. These events can trigger image optimization, virus scanning, audit logging, or cache invalidation.

### Production-Ready Code Snippets (30%)

**1. Comprehensive File Upload with Processing**
```php
<?php
// app/Http/Controllers/MediaController.php
namespace App\Http\Controllers;

use App\Models\Media;
use App\Services\ImageProcessor;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;

class MediaController extends Controller
{
    public function __construct(
        private ImageProcessor $imageProcessor
    ) {}

    /**
     * Upload and process an image with multiple sizes.
     */
    public function uploadImage(Request $request): \Illuminate\Http\JsonResponse
    {
        $request->validate([
            'image' => [
                'required',
                'image',
                'mimes:jpeg,png,webp',
                'max:10240', // 10MB
                'dimensions:min_width=200,min_height=200,max_width=8000,max_height=8000',
            ],
            'alt_text' => 'nullable|string|max:255',
            'folder' => 'nullable|string|max:100',
        ]);

        $file = $request->file('image');
        $disk = config('filesystems.default');
        $folder = $request->input('folder', 'general');
        
        // Generate unique filename
        $filename = sprintf(
            '%s-%s.%s',
            Str::uuid(),
            time(),
            $file->getClientOriginalExtension()
        );

        // Upload original
        $originalPath = $file->storeAs(
            "images/{$folder}/original",
            $filename,
            $disk
        );

        // Generate thumbnails
        $thumbnailPaths = $this->imageProcessor->generateSizes(
            $file,
            "images/{$folder}",
            $filename,
            [150, 300, 600, 1200], // Thumbnail sizes
            $disk
        );

        // Optimize the original
        $this->imageProcessor->optimize(
            Storage::disk($disk)->path($originalPath)
        );

        // Store record in database
        $media = Media::create([
            'user_id' => $request->user()->id,
            'filename' => $filename,
            'original_name' => $file->getClientOriginalName(),
            'mime_type' => $file->getMimeType(),
            'size' => $file->getSize(),
            'disk' => $disk,
            'path' => $originalPath,
            'folder' => $folder,
            'alt_text' => $request->input('alt_text'),
            'metadata' => [
                'dimensions' => getimagesize($file->path()),
                'thumbnails' => $thumbnailPaths,
            ],
        ]);

        return response()->json([
            'id' => $media->id,
            'url' => $media->getUrl(),
            'thumbnails' => $media->getThumbnailUrls(),
        ], 201);
    }

    /**
     * Serve private files with authorization.
     */
    public function downloadInvoice(Request $request, string $id)
    {
        $invoice = \App\Models\Invoice::findOrFail($id);

        // Authorization check
        if (! $request->user()->can('view', $invoice)) {
            abort(403, 'Access denied.');
        }

        // Stream the file
        return Storage::disk('private')
            ->download(
                $invoice->file_path,
                "invoice-{$invoice->number}.pdf",
                ['Content-Type' => 'application/pdf']
            );
    }

    /**
     * Generate temporary URL for S3 files.
     */
    public function getSecureUrl(Request $request, Media $media)
    {
        if (! $request->user()->can('download', $media)) {
            abort(403);
        }

        $url = Storage::disk($media->disk)
            ->temporaryUrl(
                $media->path,
                now()->addMinutes(5), // 5-minute expiration
                ['ResponseContentDisposition' => 'attachment']
            );

        return response()->json(['url' => $url]);
    }
}

// app/Services/ImageProcessor.php
namespace App\Services;

use Intervention\Image\Laravel\Facades\Image;
use Illuminate\Support\Facades\Storage;

class ImageProcessor
{
    /**
     * Generate multiple sizes for an uploaded image.
     */
    public function generateSizes(
        \Illuminate\Http\UploadedFile $file,
        string $folder,
        string $filename,
        array $sizes,
        string $disk
    ): array {
        $paths = [];

        foreach ($sizes as $size) {
            $image = Image::read($file);
            
            // Resize maintaining aspect ratio
            $image->resize($size, $size, function ($constraint) {
                $constraint->aspectRatio();
                $constraint->upsize(); // Don't enlarge smaller images
            });

            // Encode with optimization
            $optimizedContent = $image->encode(
                $file->getClientOriginalExtension(),
                quality: 85
            );

            $sizePath = "{$folder}/{$size}/{$filename}";
            
            Storage::disk($disk)->put($sizePath, $optimizedContent);
            
            $paths[$size] = $sizePath;
        }

        return $paths;
    }

    /**
     * Optimize an image file.
     */
    public function optimize(string $path): void
    {
        if (!file_exists($path)) {
            return;
        }

        $image = Image::read($path);
        
        // Strip metadata for privacy and size reduction
        $image->core()->image()->stripImage();
        
        // Optimize encoding
        $image->save($path, 85);
    }
}
```

**2. Filesystem Configuration for Multiple Disks**
```php
<?php
// config/filesystems.php
return [
    'default' => env('FILESYSTEM_DISK', 'local'),

    'disks' => [
        'local' => [
            'driver' => 'local',
            'root' => storage_path('app'),
            'throw' => false,
        ],

        'public' => [
            'driver' => 'local',
            'root' => storage_path('app/public'),
            'url' => env('APP_URL').'/storage',
            'visibility' => 'public',
            'throw' => false,
        ],

        'private' => [
            'driver' => 'local',
            'root' => storage_path('app/private'),
            'visibility' => 'private',
        ],

        's3' => [
            'driver' => 's3',
            'key' => env('AWS_ACCESS_KEY_ID'),
            'secret' => env('AWS_SECRET_ACCESS_KEY'),
            'region' => env('AWS_DEFAULT_REGION'),
            'bucket' => env('AWS_BUCKET'),
            'url' => env('AWS_URL'),
            'endpoint' => env('AWS_ENDPOINT'),
            'use_path_style_endpoint' => env('AWS_USE_PATH_STYLE_ENDPOINT', false),
            'throw' => false,
            'options' => [
                'CacheControl' => 'max-age=31536000, public',
            ],
        ],

        's3-private' => [
            'driver' => 's3',
            'key' => env('AWS_ACCESS_KEY_ID'),
            'secret' => env('AWS_SECRET_ACCESS_KEY'),
            'region' => env('AWS_DEFAULT_REGION'),
            'bucket' => env('AWS_PRIVATE_BUCKET'),
            'url' => env('AWS_URL'),
            'visibility' => 'private',
        ],

        'backups' => [
            'driver' => 's3',
            'key' => env('AWS_ACCESS_KEY_ID'),
            'secret' => env('AWS_SECRET_ACCESS_KEY'),
            'region' => env('AWS_DEFAULT_REGION'),
            'bucket' => env('AWS_BACKUP_BUCKET'),
            'throw' => false,
        ],
    ],

    'links' => [
        public_path('storage') => storage_path('app/public'),
    ],
];
```

**Best Practices (Staff Engineer Tips)**

1. **Disk Selection Strategy**: Use different disks for different purposes. Public user uploads go to a public S3 bucket with CloudFront CDN. Private documents go to a private S3 bucket with signed URLs. Backups go to a separate bucket with lifecycle policies for automatic archival.

2. **File Naming Strategy**: Never use user-provided filenames directly. Generate unique names using UUIDs or hashes. Store the original filename in the database for display purposes. Unique filenames prevent collisions, path traversal attacks, and information disclosure through predictable filenames.

3. **Image Optimization Pipeline**: Optimize images before storage, not on retrieval. Generate all needed sizes at upload time. Use WebP format with JPEG fallback for modern applications. Strip EXIF metadata that might contain location data, camera information, or other privacy-sensitive details.

4. **Large File Handling**: For very large files (>100MB), use multipart upload APIs. Stream files to disk rather than loading entirely into memory. Consider using direct-to-S3 upload with presigned URLs, bypassing your server entirely for uploads.

**Common Pitfalls and Solutions**

1. **Missing Storage Links**: In production, forgetting to run `php artisan storage:link` leaves no symbolic link from `public/storage` to `storage/app/public`. All public file URLs return 404. Include this command in deployment scripts.

2. **Local Files in Production**: Code that references `storage_path()` for files that users upload creates files on the local server. In load-balanced environments, files on one server aren't accessible from others. Always use a shared filesystem (S3) for user-uploaded content.

3. **Unvalidated File Types**: Checking file extension only is insufficient—attackers rename malicious files with safe extensions. Always validate both extension and MIME type. Use `mimes` and `mimetypes` validation rules. For images, use the `image` rule which validates actual file contents.

4. **Temporary URL Expiration**: S3 temporary URLs can expire while users are still downloading. For large downloads, use redirects to permanent URLs or implement resume support. Set expiration times appropriate for the expected download duration.

---
