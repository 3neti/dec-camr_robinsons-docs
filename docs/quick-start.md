# Quick Start Guide

Get Text Commander's SMS integration running in 5 minutes.

## Prerequisites

- Laravel 8+ application
- PHP 8.0+
- Composer
- engageSPARK account with API credentials

## Installation

### 1. Install SMS Package

```bash
composer require lbhurtado/sms
```

This automatically installs `lbhurtado/engagespark` as a dependency.

### 2. Configure Environment

Add to your `.env` file:

```dotenv
# engageSPARK Credentials
ENGAGESPARK_API_KEY=your_api_key_here
ENGAGESPARK_ORGANIZATION_ID=your_org_id_here
ENGAGESPARK_SENDER_ID=TXTCMDR

# Webhooks (optional)
ENGAGESPARK_SMS_WEBHOOK=https://yourapp.com/webhooks/engagespark/sms
ENGAGESPARK_AIRTIME_WEBHOOK=https://yourapp.com/webhooks/engagespark/airtime

# SMS Driver Configuration
SMS_DRIVER=engagespark
```

### 3. Publish Configuration (Optional)

```bash
php artisan vendor:publish --provider="LBHurtado\EngageSpark\EngageSparkServiceProvider"
```

### 4. Set Up Notifications Table (If Using Notification Channel)

```bash
php artisan notifications:table
php artisan migrate
```

## Basic Usage

### Send Single SMS

```php
use LBHurtado\SMS\Facades\SMS;

SMS::channel('engagespark')
    ->from('TXTCMDR')
    ->to('+639171234567')
    ->content('Hello from Text Commander!')
    ->send();
```

### Send with Airtime Topup

```php
SMS::channel('engagespark')
    ->from('TXTCMDR')
    ->to('+639171234567')
    ->content('Thank you! Here\'s P25 load as a token of appreciation.')
    ->send()
    ->topup(25); // PHP 25 airtime
```

### Broadcast to Multiple Recipients

```php
$recipients = ['+639171234567', '+639181234567', '+639191234567'];
$message = 'Emergency: Flood warning in your area.';
$senderId = 'QUEZON_CITY';

foreach ($recipients as $mobile) {
    SMS::channel('engagespark')
        ->from($senderId)
        ->to($mobile)
        ->content($message)
        ->send();
}
```

### Queue SMS Sending

Create a job:

```php
namespace App\Jobs;

use LBHurtado\SMS\Facades\SMS;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;

class SendSMSJob implements ShouldQueue
{
    use InteractsWithQueue, Queueable;

    public function __construct(
        public string $mobile,
        public string $message,
        public string $senderId = 'TXTCMDR'
    ) {}

    public function handle()
    {
        SMS::channel('engagespark')
            ->from($this->senderId)
            ->to($this->mobile)
            ->content($this->message)
            ->send();
    }
}
```

Dispatch it:

```php
SendSMSJob::dispatch('+639171234567', 'Your message', 'TXTCMDR');
```

## Alternative: Laravel Notifications

If you prefer using Laravel's notification system:

### 1. Create a Notification

```php
namespace App\Notifications;

use Illuminate\Notifications\Notification;
use LBHurtado\EngageSpark\EngageSparkChannel;
use LBHurtado\EngageSpark\EngageSparkMessage;

class WelcomeSMS extends Notification
{
    public function __construct(
        private string $message
    ) {}

    public function via($notifiable)
    {
        return [EngageSparkChannel::class];
    }

    public function toEngageSpark($notifiable)
    {
        return (new EngageSparkMessage())
            ->content($this->message);
    }
}
```

### 2. Add Routing to Your Model

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use LBHurtado\EngageSpark\Traits\HasEngageSpark;

class Contact extends Model
{
    use HasEngageSpark;
    
    // Or manually:
    public function routeNotificationForEngageSpark()
    {
        return $this->mobile;
    }
}
```

### 3. Send Notification

```php
$contact->notify(new WelcomeSMS('Hello from Text Commander!'));
```

### 4. Use Built-in Notifications

```php
use LBHurtado\EngageSpark\Notifications\Adhoc;
use LBHurtado\EngageSpark\Notifications\Topup;

// Send ad-hoc SMS
$contact->notify(new Adhoc('Your custom message'));

// Send airtime
$contact->notify(new Topup(25)); // PHP 25 load
```

## Testing

### Unit Tests

Use the `null` driver to prevent sending real SMS in tests:

```php
namespace Tests\Feature;

use Tests\TestCase;
use LBHurtado\SMS\Facades\SMS;

class SMSTest extends TestCase
{
    protected function setUp(): void
    {
        parent::setUp();
        
        // Use null driver
        config(['sms.default' => 'null']);
    }

    public function test_sends_sms()
    {
        SMS::channel('null')
            ->from('TXTCMDR')
            ->to('+639171234567')
            ->content('Test message')
            ->send();

        // Assert something (e.g., database record created)
        $this->assertDatabaseHas('sms_logs', [
            'recipient' => '+639171234567',
        ]);
    }
}
```

### Integration Tests

Test with real engageSPARK API (use sparingly):

```php
namespace Tests\Integration;

use Tests\TestCase;
use LBHurtado\SMS\Facades\SMS;

class EngageSparkTest extends TestCase
{
    /** @test */
    public function it_sends_via_engagespark()
    {
        if (!config('engagespark.api_key')) {
            $this->markTestSkipped('engageSPARK not configured');
        }

        SMS::channel('engagespark')
            ->from('TXTCMDR')
            ->to(config('test.recipient_mobile'))
            ->content('Integration test message')
            ->send();

        $this->assertTrue(true);
    }
}
```

## Event Handling

Listen to SMS events for logging/analytics:

### 1. Create Listener

```php
namespace App\Listeners;

use LBHurtado\EngageSpark\Events\MessageSent;
use Illuminate\Support\Facades\Log;

class LogSMSSent
{
    public function handle(MessageSent $event)
    {
        Log::info('SMS sent via engageSPARK', [
            'recipient' => $event->recipient,
            'sender' => $event->sender,
            'message' => $event->message,
            'timestamp' => now(),
        ]);
    }
}
```

### 2. Register in EventServiceProvider

```php
use LBHurtado\EngageSpark\Events\{MessageSent, AirtimeTransferred};
use App\Listeners\{LogSMSSent, LogAirtimeTransferred};

protected $listen = [
    MessageSent::class => [
        LogSMSSent::class,
    ],
    AirtimeTransferred::class => [
        LogAirtimeTransferred::class,
    ],
];
```

## Webhook Handling

Handle delivery reports from engageSPARK:

### 1. Create Controller

```php
namespace App\Http\Controllers;

use Illuminate\Http\Request;

class EngageSparkWebhookController extends Controller
{
    public function sms(Request $request)
    {
        $payload = $request->all();
        
        // Update delivery status
        \DB::table('sms_logs')
            ->where('message_id', $payload['messageId'])
            ->update([
                'status' => $payload['status'],
                'delivered_at' => $payload['status'] === 'delivered' 
                    ? now() 
                    : null,
            ]);

        return response()->json(['success' => true]);
    }

    public function airtime(Request $request)
    {
        $payload = $request->all();
        
        // Log airtime transfer status
        \Log::info('Airtime transfer status', $payload);

        return response()->json(['success' => true]);
    }
}
```

### 2. Register Routes

```php
// routes/api.php
Route::post('/webhooks/engagespark/sms', [EngageSparkWebhookController::class, 'sms']);
Route::post('/webhooks/engagespark/airtime', [EngageSparkWebhookController::class, 'airtime']);
```

### 3. Update .env

```dotenv
ENGAGESPARK_SMS_WEBHOOK=https://yourapp.com/api/webhooks/engagespark/sms
ENGAGESPARK_AIRTIME_WEBHOOK=https://yourapp.com/api/webhooks/engagespark/airtime
```

## Common Patterns

### Service Class Pattern

```php
namespace App\Services;

use LBHurtado\SMS\Facades\SMS;
use App\Models\Campaign;

class BroadcastService
{
    public function sendCampaign(Campaign $campaign)
    {
        foreach ($campaign->contacts as $contact) {
            \App\Jobs\SendSMSJob::dispatch(
                $contact->mobile,
                $campaign->message,
                $campaign->sender_id
            );
        }
    }
}
```

### Repository Pattern

```php
namespace App\Repositories;

use App\Models\SMSLog;

class SMSLogRepository
{
    public function logSent(string $recipient, string $message, string $sender)
    {
        return SMSLog::create([
            'recipient' => $recipient,
            'message' => $message,
            'sender' => $sender,
            'status' => 'sent',
            'sent_at' => now(),
        ]);
    }

    public function updateStatus(string $messageId, string $status)
    {
        return SMSLog::where('message_id', $messageId)
            ->update(['status' => $status]);
    }
}
```

## Troubleshooting

### SMS Not Sending

1. **Check credentials:** Verify `ENGAGESPARK_API_KEY` and `ENGAGESPARK_ORGANIZATION_ID`
2. **Check balance:** Ensure engageSPARK account has sufficient credits
3. **Check logs:** Look for errors in `storage/logs/laravel.log`
4. **Test connection:** Use tinker to test manually

```php
php artisan tinker
>>> use LBHurtado\SMS\Facades\SMS;
>>> SMS::channel('engagespark')->from('TXTCMDR')->to('+639171234567')->content('Test')->send();
```

### Invalid Sender ID

- Sender IDs must be pre-approved by engageSPARK
- For testing, use default sender ID from your engageSPARK dashboard
- Apply for branded sender ID (e.g., "QUEZON_CITY") through engageSPARK

### Webhook Not Working

1. **Check URL:** Ensure webhook URL is publicly accessible (use ngrok for local testing)
2. **Verify signature:** engageSPARK may sign webhook requests
3. **Check logs:** Look for incoming requests in server logs
4. **Test manually:** Use Postman to simulate webhook payload

### Rate Limiting

engageSPARK has rate limits. If hitting limits:
- Queue messages with delays
- Use batch sending endpoints if available
- Contact engageSPARK support to increase limits

## Next Steps

- **Read full architecture:** [Package Architecture](package-architecture.md)
- **Explore backend services:** [Backend Services](backend-services.md)
- **Review API endpoints:** [API Documentation](api-documentation.md)
- **Review development plan:** [Development Plan](development-plan.md)

## Getting Help

- **engageSPARK Docs:** https://www.engagespark.com/docs/
- **Package Issues:** https://github.com/lbhurtado/sms/issues
- **Laravel Docs:** https://laravel.com/docs/notifications

## Credits

Packages created by [Lester Hurtado](https://github.com/lbhurtado):
- `lbhurtado/sms` - SMS abstraction layer
- `lbhurtado/engagespark` - engageSPARK integration
