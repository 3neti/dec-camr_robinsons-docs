# Package Architecture

This document describes the two-layer SMS architecture powering Text Commander, consisting of a driver package (`lbhurtado/engagespark`) and an abstraction layer (`lbhurtado/sms`).

## Architecture Overview

```
┌─────────────────────────────────────────┐
│       Text Commander Application        │
│                                         │
└─────────────────┬───────────────────────┘
                  │
                  │ Uses SMS Facade
                  │
┌─────────────────▼───────────────────────┐
│       lbhurtado/sms Package            │
│       (SMS Abstraction Layer)           │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │     SMSManager                  │   │
│  │  (Driver Factory & Selector)    │   │
│  └──┬───────────────────────┬──────┘   │
│     │                       │           │
│  ┌──▼──────────┐     ┌─────▼────────┐  │
│  │EngageSpark  │     │ NexmoDriver  │  │
│  │   Driver    │     │  (Optional)  │  │
│  └──┬──────────┘     └──────────────┘  │
└─────┼────────────────────────────────────┘
      │
      │ Uses EngageSpark Service
      │
┌─────▼──────────────────────────────────┐
│    lbhurtado/engagespark Package       │
│    (engageSPARK Integration)           │
│                                        │
│  ┌────────────────────────────────┐   │
│  │      EngageSpark Service       │   │
│  │  (HTTP API Client)             │   │
│  └────────────────────────────────┘   │
│                                        │
│  ┌────────────────────────────────┐   │
│  │  Laravel Notification Channel  │   │
│  │  (Alternative Integration)     │   │
│  └────────────────────────────────┘   │
│                                        │
│  ┌────────────────────────────────┐   │
│  │       Jobs & Events            │   │
│  │  • SendMessage Job             │   │
│  │  • TransferAirtime Job         │   │
│  │  • MessageSent Event           │   │
│  │  • AirtimeTransferred Event    │   │
│  └────────────────────────────────┘   │
└───────────┬────────────────────────────┘
            │
            │ HTTP API Calls
            │
┌───────────▼────────────────────────────┐
│       engageSPARK Platform API         │
│       (SMS & Airtime Services)         │
└────────────────────────────────────────┘
```

## Layer 1: lbhurtado/engagespark

**Purpose:** Low-level driver for engageSPARK platform integration

**Repository:** `lbhurtado/engagespark`

**Package Type:** Laravel notification channel + service wrapper

### Core Components

#### 1. EngageSpark Service (`src/EngageSpark.php`)
The main service class that handles HTTP communication with engageSPARK API.

```php
use LBHurtado\EngageSpark\EngageSpark;

$engageSpark = app(EngageSpark::class);
$response = $engageSpark->send($params, ServiceMode::SMS);
```

**Key Methods:**
- `send($params, $mode)` - Send SMS or transfer airtime
- `getOrgId()` - Get configured organization ID
- `getSenderId()` - Get default sender ID
- `getEndPoint($mode)` - Get API endpoint for service mode

#### 2. Laravel Notification Channel (`src/EngageSparkChannel.php`)
Enables Laravel notification system integration.

```php
use LBHurtado\EngageSpark\EngageSparkChannel;
use LBHurtado\EngageSpark\EngageSparkMessage;

public function via($notifiable)
{
    return [EngageSparkChannel::class];
}

public function toEngageSpark($notifiable)
{
    return (new EngageSparkMessage())
        ->content('Your message here');
}
```

#### 3. Jobs
- **SendMessage** (`src/Jobs/SendMessage.php`) - Queue SMS sending
- **TransferAirtime** (`src/Jobs/TransferAirtime.php`) - Queue airtime topup

#### 4. Events
- **MessageSent** (`src/Events/MessageSent.php`) - Fired after message dispatch
- **AirtimeTransferred** (`src/Events/AirtimeTransferred.php`) - Fired after topup dispatch

#### 5. Built-in Notifications
- **Adhoc** (`src/Notifications/Adhoc.php`) - Send ad-hoc SMS
- **Topup** (`src/Notifications/Topup.php`) - Transfer airtime

```php
$user->notify(new Adhoc('Your message'));
$user->notify(new Topup(25)); // PHP 25 load
```

#### 6. Traits
- **HasEngageSpark** (`src/Traits/HasEngageSpark.php`) - Add to models for notification routing

```php
use LBHurtado\EngageSpark\Traits\HasEngageSpark;

class Contact extends Model 
{
    use HasEngageSpark;
}
```

### Configuration

**Environment Variables:**
```dotenv
ENGAGESPARK_API_KEY=your_api_key
ENGAGESPARK_ORGANIZATION_ID=your_org_id
ENGAGESPARK_SENDER_ID=TXTCMDR
ENGAGESPARK_SMS_WEBHOOK=https://yourapp.com/webhooks/sms
ENGAGESPARK_AIRTIME_WEBHOOK=https://yourapp.com/webhooks/airtime
```

**Service Modes:**
- `ServiceMode::SMS` - Text messaging
- `ServiceMode::AIRTIME` - Airtime transfer (load)

### Installation

```bash
composer require lbhurtado/engagespark
php artisan notifications:table
php artisan migrate

# Optional: publish config
php artisan vendor:publish --provider="LBHurtado\EngageSpark\EngageSparkServiceProvider"
```

---

## Layer 2: lbhurtado/sms

**Purpose:** Multi-driver SMS abstraction layer (inspired by Laravel's notification channels)

**Repository:** `lbhurtado/sms`

**Package Type:** Driver-based SMS manager

### Architecture Pattern

The package follows Laravel's Manager pattern, allowing you to:
1. Switch between different SMS providers without changing application code
2. Add custom drivers easily
3. Use a fluent, expressive API

### Core Components

#### 1. SMSManager (`src/SMSManager.php`)
Factory class that creates and manages driver instances.

```php
use LBHurtado\SMS\Facades\SMS;

SMS::channel('engagespark')  // Select driver
    ->from('TXTCMDR')        // Set sender ID
    ->to('+639171234567')    // Set recipient
    ->content('Message')     // Set content
    ->send();                // Dispatch
```

**Methods:**
- `channel($name)` - Select SMS driver
- `createEngageSparkDriver()` - Create engageSPARK driver instance
- `createNexmoDriver()` - Create Nexmo/Vonage driver instance
- `createNullDriver()` - Create null driver (for testing)

#### 2. Driver Interface (`src/Contracts/SMS.php`)
All drivers must implement this contract.

**Required Methods:**
- `from($sender)` - Set sender ID
- `to($recipient)` - Set recipient number
- `content($message)` - Set message body
- `send()` - Send the message

#### 3. EngageSparkDriver (`src/Drivers/EngageSparkDriver.php`)
Concrete implementation wrapping the engageSPARK service.

```php
use LBHurtado\SMS\Drivers\EngageSparkDriver;

$driver = new EngageSparkDriver(app(EngageSpark::class));
$driver->from('TXTCMDR')
    ->to('+639171234567')
    ->content('Hello!')
    ->send()
    ->topup(25);  // Bonus: airtime topup
```

**Extended Methods:**
- `topup($amount)` - Transfer airtime (engageSPARK-specific feature)
- `reference($ref)` - Set transaction reference
- `getOrgId()` - Get organization ID
- `client()` - Access underlying EngageSpark service

#### 4. Other Drivers
- **NexmoDriver** (`src/Drivers/NexmoDriver.php`) - Nexmo/Vonage integration
- **NullDriver** (`src/Drivers/NullDriver.php`) - No-op driver for testing

### Configuration

**Default Driver:**
Set in `config/sms.php` or via environment:
```dotenv
SMS_DRIVER=engagespark
```

### Usage Examples

#### Basic SMS
```php
use LBHurtado\SMS\Facades\SMS;

SMS::channel('engagespark')
    ->from('TXTCMDR')
    ->to('+639171234567')
    ->content('Welcome to Text Commander!')
    ->send();
```

#### SMS + Airtime Topup
```php
SMS::channel('engagespark')
    ->from('TXTCMDR')
    ->to('+639171234567')
    ->content('Thank you! Here\'s P25 load.')
    ->send()
    ->topup(25);
```

#### Broadcast to Multiple Recipients
```php
$recipients = ['+639171234567', '+639181234567', '+639191234567'];

foreach ($recipients as $mobile) {
    SMS::channel('engagespark')
        ->from('QUEZON_CITY')
        ->to($mobile)
        ->content('Emergency alert: Typhoon approaching.')
        ->send();
}
```

#### Switch Drivers
```php
// Use engageSPARK for branded sender ID
SMS::channel('engagespark')
    ->from('TXTCMDR')
    ->to($mobile)
    ->content($message)
    ->send();

// Fall back to Nexmo if needed
SMS::channel('nexmo')
    ->from('+639171111111')
    ->to($mobile)
    ->content($message)
    ->send();
```

### Installation

```bash
composer require lbhurtado/sms
```

**Note:** This automatically installs `lbhurtado/engagespark` as a dependency.

---

## Integration Strategy for Text Commander

### Recommended Approach: Use SMS Abstraction Layer

**Why?**
1. **Future-proof:** Easy to add other SMS providers (e.g., Twilio, Vonage, local carriers)
2. **Branded sender IDs:** engageSPARK supports pre-approved sender IDs like "Quezon City"
3. **Clean API:** Fluent, expressive syntax
4. **Testing:** Use `NullDriver` in tests without sending real SMS

### Implementation in Text Commander

#### 1. Service Layer (`app/Services/SMSBroadcastService.php`)

```php
namespace App\Services;

use LBHurtado\SMS\Facades\SMS;
use App\Models\Contact;
use App\Models\Campaign;

class SMSBroadcastService
{
    public function sendToContact(Contact $contact, string $message, string $senderId = null)
    {
        $senderId = $senderId ?? config('sms.default_sender_id');
        
        SMS::channel('engagespark')
            ->from($senderId)
            ->to($contact->mobile)
            ->content($message)
            ->send();
    }

    public function broadcastToCampaign(Campaign $campaign)
    {
        foreach ($campaign->contacts as $contact) {
            dispatch(function() use ($contact, $campaign) {
                $this->sendToContact(
                    $contact, 
                    $campaign->message,
                    $campaign->sender_id
                );
            });
        }
    }
}
```

#### 2. Queue Jobs (`app/Jobs/SendSMSJob.php`)

```php
namespace App\Jobs;

use LBHurtado\SMS\Facades\SMS;
use Illuminate\Bus\Queueable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Contracts\Queue\ShouldQueue;

class SendSMSJob implements ShouldQueue
{
    use InteractsWithQueue, Queueable;

    public function __construct(
        public string $mobile,
        public string $message,
        public string $senderId
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

#### 3. API Controllers (`app/Http/Controllers/API/BroadcastController.php`)

```php
namespace App\Http\Controllers\API;

use Illuminate\Http\Request;
use App\Services\SMSBroadcastService;
use App\Http\Controllers\Controller;

class BroadcastController extends Controller
{
    public function __construct(
        private SMSBroadcastService $smsService
    ) {}

    public function sendSingle(Request $request)
    {
        $request->validate([
            'mobile' => 'required|phone:PH',
            'message' => 'required|max:160',
            'sender_id' => 'nullable|string|max:11',
        ]);

        $this->smsService->sendToContact(
            mobile: $request->mobile,
            message: $request->message,
            senderId: $request->sender_id
        );

        return response()->json(['status' => 'sent']);
    }
}
```

### Testing Strategy

#### Unit Tests with NullDriver

```php
namespace Tests\Unit;

use Tests\TestCase;
use LBHurtado\SMS\Facades\SMS;

class SMSServiceTest extends TestCase
{
    protected function setUp(): void
    {
        parent::setUp();
        
        // Use null driver for testing
        config(['sms.default' => 'null']);
    }

    public function test_can_send_sms()
    {
        $result = SMS::channel('null')
            ->from('TXTCMDR')
            ->to('+639171234567')
            ->content('Test message')
            ->send();
        
        $this->assertNotNull($result);
    }
}
```

#### Integration Tests with engageSPARK

```php
namespace Tests\Integration;

use Tests\TestCase;
use LBHurtado\SMS\Facades\SMS;

class EngageSparkIntegrationTest extends TestCase
{
    /** @test */
    public function it_sends_sms_via_engagespark()
    {
        if (!config('engagespark.api_key')) {
            $this->markTestSkipped('engageSPARK credentials not configured');
        }

        SMS::channel('engagespark')
            ->from('TXTCMDR')
            ->to(config('test.sms.recipient'))
            ->content('Test from CI/CD pipeline')
            ->send();

        $this->assertTrue(true); // If no exception thrown, it worked
    }
}
```

---

## Advanced Features

### Airtime Incentives

```php
use LBHurtado\SMS\Facades\SMS;

// Send SMS + airtime reward
SMS::channel('engagespark')
    ->from('TXTCMDR')
    ->to('+639171234567')
    ->content('Thanks for voting! Here\'s P25 load.')
    ->send()
    ->topup(25);
```

### Custom Transaction References

```php
use LBHurtado\SMS\Facades\SMS;

SMS::channel('engagespark')
    ->reference('CAMP-2024-001')
    ->from('QUEZON_CITY')
    ->to('+639171234567')
    ->content('Campaign message')
    ->send();
```

### Event Listeners

Listen to engageSPARK events for logging/analytics:

```php
namespace App\Listeners;

use LBHurtado\EngageSpark\Events\MessageSent;

class LogSMSSent
{
    public function handle(MessageSent $event)
    {
        \Log::info('SMS sent', [
            'recipient' => $event->recipient,
            'sender' => $event->sender,
            'message' => $event->message,
        ]);
    }
}
```

Register in `EventServiceProvider`:

```php
protected $listen = [
    MessageSent::class => [
        LogSMSSent::class,
    ],
];
```

---

## Package Dependencies

### lbhurtado/engagespark
```json
{
  "require": {
    "php": "~7.1||^8.0.1",
    "illuminate/support": "^8.0|^9.0|^10.0|^11.0|^12.0",
    "lbhurtado/common": "^2.2.0",
    "eloquent/enumeration": "^6.0",
    "guzzlehttp/guzzle": "^7.0.1"
  }
}
```

### lbhurtado/sms
```json
{
  "require": {
    "php": "~7.1||^8.0.1",
    "illuminate/support": "^8.0|^9.0|^10.0|^11.0|^12.0",
    "lbhurtado/common": "^2.2.0",
    "lbhurtado/engagespark": "^3.2"
  }
}
```

---

## Comparison: Direct vs Abstraction Layer

| Feature | Direct EngageSpark | SMS Abstraction |
|---------|-------------------|-----------------|
| **API Style** | Notifications | Fluent/Facade |
| **Driver Switching** | ❌ No | ✅ Yes |
| **Testing** | Mock notifications | Use NullDriver |
| **Vendor Lock-in** | High | Low |
| **Feature Access** | Full (topups) | Full (preserved) |
| **Learning Curve** | Laravel notifications | Simple facade |
| **Recommended For** | Single-provider apps | Multi-provider apps |

**Text Commander Recommendation:** Use `lbhurtado/sms` for flexibility and future-proofing.

---

## Next Steps

1. **Install packages** in Text Commander Laravel app
2. **Configure** engageSPARK credentials
3. **Create service layer** using SMS facade
4. **Implement queue jobs** for broadcast sending
5. **Add webhook handlers** for delivery reports
6. **Test with NullDriver** before production
7. **Deploy with branded sender ID** (e.g., "QUEZON_CITY")

## References

- [engageSPARK API Documentation](https://www.engagespark.com/docs/)
- [Laravel Notifications](https://laravel.com/docs/notifications)
- [Laravel Manager Pattern](https://laravel.com/api/master/Illuminate/Support/Manager.html)
