Automatically Provision AWS S3 Buckets for each tenant. It's an Extension for [stancl/tenancy](https://github.com/stancl/tenancy). For more details refer to [TenancyForLaravel](https://tenancyforlaravel.com/).

## Credits

This is a modified version of the vidwanco/tenant-buckets package, changed to run on Laravel Vapor. A simple fix applied for correctly inject the AWS credentials in-line with Vapor requirements.

Also, for larger files, Vapor must utilise presigned URLs and frontloading the files via the frontend, so a change has been made to enable the correct CORS policy on each bucket as it is created.

## Concept

The concept is simple. It is to automatically provison a new AWS S3 bucket for tenant on registration and update the same on the central database's tenant table & data coloumn under `tenant_bucket`.
Then using a bootstrapper updating the bucket in config `filesystems.disks.s3.bucket` during runtime when in Tenant's context and then reverting it back on central context.

### Roadmap

This repo may be discontinued if the changes provided are merged into the original package, at which point we recommend using the original package.

> **Note:** I have still not tested this package under ***production*** environment or with a real AWS S3 Bucket. I have only tested it under ***development*** environment using [MinIO](https://min.io/). I will update this after testing it on AWS S3 Bucket with an additional section on AWS IAM Policy Setup for creating the buckets using `aws-sdk-php`. Untill then, if you have tested, a PR is welcome.

## Installation

You can install the package via composer:

```bash
composer require lanos/vapor-tenant-buckets
```

## Usage

### 1. Filesystem Config Setup

Ensure your S3 configuration has all the Key/Value pairs, as below:

**File:** `config/filesystems.php` 
```php
        's3' => [
            'driver' => 's3',
            'key' => env('AWS_ACCESS_KEY_ID'),
            'secret' => env('AWS_SECRET_ACCESS_KEY'),
            'region' => env('AWS_DEFAULT_REGION'),
            'bucket' => env('AWS_BUCKET'),
            'url' => env('AWS_URL'),
            'endpoint' => env('AWS_ENDPOINT'),
            'use_path_style_endpoint' => env('AWS_USE_PATH_STYLE_ENDPOINT', false),
        ],

```
> Using Minio for development? Make sure to update your `.env` with `AWS_USE_PATH_STYLE_ENDPOINT=true`

### 2. Tenancy Config

There are two parts in tenancy config to take care of.
#### Part **a**.

Add the `TenantBucketBootstrapper::class` to the tenancy config file under `bootstrappers`.

```php
Vidwan\TenantBuckets\Bootstrappers\TenantBucketBootstrapper::class
```

**File:** `config/tenancy.php`
```php
    'bootstrappers' => [
        // Tenancy Bootstrappers
        Vidwan\TenantBuckets\Bootstrappers\TenantBucketBootstrapper::class,
    ],
```

#### Part **b**.

Make sure the `s3` is commented in `tenancy.filesystem.disks` config.

**File:** `config/tenancy.php`
```php
    'filesystem' => [
        'suffix_base' => 'tenant',
        'disks' => [
            'local',
            'public',
            // 's3', // Make sure this stays commented
        ],
    ],
```

### 3. Job Pipeline

Add `Vidwan\TenantBuckets\Jobs\CreateTenantBucket` in `JobPipeline::make()`

**File:** `app/Providers/TenancyServiceProviders.php`
```php
use Vidwan\TenantBuckets\Jobs\CreateTenantBucket;

...

    public function events()
    {
        return [
            // Tenant events
            ...
            Events\TenantCreated::class => [
                JobPipeline::make([
                    Jobs\CreateDatabase::class,
                    Jobs\MigrateDatabase::class,
                    Jobs\SeedDatabase::class,
                    // Your own jobs to prepare the tenant.
                    // Provision API keys, create S3 buckets, anything you want!
                    CreateTenantBucket::class, // <-- Place it Here
					
                ])->send(function (Events\TenantCreated $event) {
                    return $event->tenant;
                })->shouldBeQueued(false),
            ],
            ...
        ];
    }
```

Cheers! 🥳
## Testing

```bash
composer test
```

## Changelog

Please see [CHANGELOG](CHANGELOG.md) for more information on what has changed recently.

## Contributing

Please see [CONTRIBUTING](.github/CONTRIBUTING.md) for details.

## Security Vulnerabilities

Please review [our security policy](../../security/policy) on how to report security vulnerabilities.

## Credits

- [Shashwat Mishra](https://github.com/secrethash)
- [All Contributors](../../contributors)

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.
