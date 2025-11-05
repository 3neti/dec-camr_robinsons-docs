# Third-Party Packages

Text Commander leverages several third-party packages to accelerate development and provide robust functionality.

## Package Overview

| Package | Version | Purpose | Documentation |
|---------|---------|---------|---------------|
| **lbhurtado/sms** | ^4.0 | SMS facade and integration | [SMS Integration](sms-integration.md) |
| **lbhurtado/engagespark** | ^1.0 | EngageSpark API driver | [SMS Integration](sms-integration.md) |
| **lbhurtado/contact** | ^2.0 | Contact model with phone normalization | [Contact Package](contact-package.md) |
| **propaganistas/laravel-phone** | ^5.0 | Phone number validation/formatting | [Phone Normalization](phone-normalization.md) |
| **lorisleiva/laravel-actions** | ^2.8 | Action-based architecture | [Backend Services](backend-services.md) |
| **inertiajs/inertia-laravel** | ^1.0 | Server-side adapter for Inertia.js | [Frontend Scaffolding](frontend-scaffolding.md) |
| **laravel/sanctum** | ^4.0 | API authentication | [API Documentation](api-documentation.md) |
| **spatie/laravel-medialibrary** | ^11.0 | Media/file handling | - |
| **bavix/laravel-wallet** | ^10.0 | Wallet system (from Commerce package) | - |

## SMS & Communication

### lbhurtado/sms

**Installation:**
```bash
composer require lbhurtado/sms
php artisan vendor:publish --provider="Lbhurtado\Sms\SmsServiceProvider"
```

**Configuration (`config/sms.php`):**
```php
'default' => env('SMS_DRIVER', 'engagespark'),

'drivers' => [
    'engagespark' => [
        'api_key' => env('ENGAGESPARK_API_KEY'),
        'organization_id' => env('ENGAGESPARK_ORG_ID'),
    ],
],
```

**Usage:**
```php
use SMS;

SMS::send('+639171234567', 'Hello from Text Commander!', 'Quezon City');
```

**See:** [SMS Integration](sms-integration.md)

---

### lbhurtado/engagespark

**Installation:**
```bash
composer require lbhurtado/engagespark
```

**Features:**
- EngageSpark REST API integration
- Webhook handling
- Delivery status tracking

**See:** [SMS Integration](sms-integration.md)

---

## Contact Management

### lbhurtado/contact

**Installation:**
```bash
composer require lbhurtado/contact
php artisan migrate
```

**Features:**
- Contact model with phone normalization
- Eloquent relationships
- Phone formatting

**Model Extension:**
```php
namespace App\Models;

use Lbhurtado\Contact\Models\Contact as BaseContact;

class Contact extends BaseContact
{
    // Extend with custom functionality
}
```

**See:** [Contact Package](contact-package.md)

---

### propaganistas/laravel-phone

**Installation:**
```bash
composer require propaganistas/laravel-phone
```

**Usage:**
```php
use Propaganistas\LaravelPhone\PhoneNumber;

$phone = PhoneNumber::make('+639171234567', 'PH');
$formatted = $phone->formatE164(); // +639171234567
$formatted = $phone->formatNational(); // 0917 123 4567
```

**Validation:**
```php
$request->validate([
    'mobile' => 'required|phone:PH',
]);
```

**See:** [Phone Normalization](phone-normalization.md)

---

## Action Architecture

### lorisleiva/laravel-actions

**Installation:**
```bash
composer require lorisleiva/laravel-actions
```

**Features:**
- Single-purpose action classes
- Can be used as controllers, jobs, listeners, commands
- Validation built-in with ActionRequest
- Dependency injection support

**Example Action:**
```php
namespace App\Actions;

use Lorisleiva\Actions\Concerns\AsAction;

class SendToMultipleRecipients
{
    use AsAction;

    public function handle(array $recipients, string $message, string $senderId)
    {
        foreach ($recipients as $recipient) {
            SendSMSJob::dispatch($recipient, $message, $senderId);
        }

        return count($recipients);
    }

    public function asController(ActionRequest $request)
    {
        $sent = $this->handle(
            $request->validated('recipients'),
            $request->validated('message'),
            $request->validated('sender_id')
        );

        return response()->json(['sent' => $sent]);
    }

    public function rules()
    {
        return [
            'recipients' => 'required|array',
            'message' => 'required|string|max:160',
            'sender_id' => 'required|string',
        ];
    }
}
```

**See:** [Backend Services](backend-services.md)

---

## Frontend

### inertiajs/inertia-laravel

**Installation:**
```bash
composer require inertiajs/inertia-laravel
npm install @inertiajs/vue3
```

**Features:**
- SPA experience without API
- Server-side routing
- Vue 3 integration
- Seamless Laravel authentication

**Root View (`resources/views/app.blade.php`):**
```blade
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    @vite('resources/js/app.ts')
    @inertiaHead
</head>
<body>
    @inertia
</body>
</html>
```

**See:** [Frontend Scaffolding](frontend-scaffolding.md)

---

## Authentication

### laravel/sanctum

**Installation:**
```bash
composer require laravel/sanctum
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"
php artisan migrate
```

**Features:**
- API token authentication
- SPA authentication
- Mobile app authentication

**Usage:**
```php
// Generate token
$token = $user->createToken('api-token')->plainTextToken;

// Protect routes
Route::middleware('auth:sanctum')->group(function () {
    // Protected API routes
});

// Access user in controller
$user = $request->user();
```

**See:** [API Documentation](api-documentation.md)

---

## Media Handling

### spatie/laravel-medialibrary

**Installation:**
```bash
composer require spatie/laravel-medialibrary
php artisan vendor:publish --provider="Spatie\MediaLibrary\MediaLibraryServiceProvider"
php artisan migrate
```

**Features:**
- File uploads
- Image manipulation
- Media collections
- Responsive images

**Status:** 🟡 Planned for document attachments

---

## Wallet System

### bavix/laravel-wallet

**Installation:**
```bash
composer require bavix/laravel-wallet
php artisan migrate
```

**Features:**
- User wallets
- Transactions
- Deposits/withdrawals
- Balance tracking

**Usage:**
```php
$user->deposit(1000);
$user->withdraw(100);
$balance = $user->balance;
```

**Status:** 🟡 Available from Commerce package, not yet integrated

---

## Development Tools

### pestphp/pest

**Installation:**
```bash
composer require pestphp/pest --dev
php artisan pest:install
```

**Features:**
- Modern testing framework
- Laravel plugin
- Expressive syntax

**See:** [Test Scaffolding](test-scaffolding.md)

---

## Related Documentation

- [Package Architecture](package-architecture.md)
- [Backend Services](backend-services.md)
- [Frontend Scaffolding](frontend-scaffolding.md)
- [Test Scaffolding](test-scaffolding.md)
