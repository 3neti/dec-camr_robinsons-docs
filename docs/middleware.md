# Middleware

Text Commander uses Laravel middleware for HTTP request filtering and job processing.

## HTTP Middleware

### HandleInertiaRequests

**Path:** `app/Http/Middleware/HandleInertiaRequests.php`  
**Type:** Inertia.js middleware  
**Purpose:** Share data with all Inertia responses

```php
namespace App\Http\Middleware;

use Illuminate\Http\Request;
use Inertia\Middleware;

class HandleInertiaRequests extends Middleware
{
    protected $rootView = 'app';

    public function share(Request $request): array
    {
        return array_merge(parent::share($request), [
            'auth' => [
                'user' => $request->user(),
            ],
            'flash' => [
                'success' => fn () => $request->session()->get('success'),
                'error' => fn () => $request->session()->get('error'),
            ],
        ]);
    }
}
```

**Features:**
- Share auth user with all Vue components
- Share flash messages for notifications
- Custom root view configuration

**See:** [Controller Scaffolding](controller-scaffolding.md)

---

## Job Middleware

### CheckBlacklist

**Path:** `app/Jobs/Middleware/CheckBlacklist.php`  
**Type:** Job middleware  
**Purpose:** Prevent SMS jobs from sending to blacklisted numbers

```php
namespace App\Jobs\Middleware;

use App\Models\BlacklistedNumber;
use Illuminate\Support\Facades\Log;

class CheckBlacklist
{
    public function handle($job, $next)
    {
        // Extract recipient from job
        $recipient = $job->recipient;

        // Check if blacklisted
        if (BlacklistedNumber::isBlacklisted($recipient)) {
            Log::warning('SMS blocked - recipient blacklisted', [
                'recipient' => $recipient,
                'job' => get_class($job),
            ]);

            // Release job without executing
            return;
        }

        // Continue job execution
        $next($job);
    }
}
```

**Features:**
- Automatic blacklist checking for all SMS jobs
- Silent failure (no exceptions thrown)
- Logging for audit trail
- Applied via `middleware()` method in jobs

**Usage:**

```php
class SendSMSJob implements ShouldQueue
{
    public function middleware()
    {
        return [new CheckBlacklist()];
    }
}
```

**See:** [Blacklist Feature](blacklist-feature.md)

---

## Route Middleware

### Sanctum Authentication (`auth:sanctum`)

**Type:** Laravel Sanctum middleware  
**Purpose:** API token authentication

**Usage in routes/api.php:**

```php
Route::middleware('auth:sanctum')->group(function () {
    Route::post('/sms/send', [SMSController::class, 'send']);
    Route::post('/sms/broadcast', [SMSController::class, 'broadcast']);
    // ... more protected routes
});

// Public opt-out endpoint (no auth)
Route::post('/optout', OptOutController::class);
```

**See:** [API Documentation](api-documentation.md)

---

### Web Middleware Group

**Default Laravel middleware:**
- `EncryptCookies`
- `AddQueuedCookiesToResponse`
- `StartSession`
- `ShareErrorsFromSession`
- `VerifyCsrfToken`
- `SubstituteBindings`
- `HandleInertiaRequests` (custom)

**Applied to all web routes** (routes/web.php)

---

## Custom Middleware Examples

### Rate Limiting (Not yet implemented)

**Purpose:** Prevent API abuse

```php
// config/routes.php example
Route::middleware(['throttle:sms'])->group(function () {
    Route::post('/sms/send', [SMSController::class, 'send']);
});

// app/Providers/RouteServiceProvider.php
RateLimiter::for('sms', function (Request $request) {
    return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
});
```

**Status:** 🟡 TODO - See [TODO-SECTIONS.md](TODO-SECTIONS.md)

---

## Related Documentation

- [Blacklist Feature - CheckBlacklist Middleware](blacklist-feature.md)
- [Controller Scaffolding](controller-scaffolding.md)
- [API Documentation](api-documentation.md)
- [Security Guide](security.md)
