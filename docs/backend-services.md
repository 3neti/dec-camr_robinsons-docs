# Backend Services

## SMS Integration Layer

Text Commander uses a two-layer architecture for SMS delivery:

### Core Packages

1. **lbhurtado/sms** - SMS abstraction layer with driver support
2. **lbhurtado/engagespark** - engageSPARK platform integration

For detailed architecture documentation, see [Package Architecture](package-architecture.md).

## SMSBroadcastService

Main service class for sending SMS messages via the SMS facade.

**Location:** `app/Services/SMSBroadcastService.php`

**Key Methods:**
- `sendToContact(Contact $contact, string $message, string $senderId)` - Send to single recipient
- `broadcastToCampaign(Campaign $campaign)` - Send to all campaign contacts
- `sendBulk(array $recipients, string $message, string $senderId)` - Batch send

**Usage:**
```php
use App\Services\SMSBroadcastService;
use LBHurtado\SMS\Facades\SMS;

SMS::channel('engagespark')
    ->from('TXTCMDR')
    ->to('+639171234567')
    ->content('Your message')
    ->send();
```

## Queue Jobs

### SendSMSJob
Queued job for sending individual SMS messages.

**Location:** `app/Jobs/SendSMSJob.php`

**Properties:**
- `$mobile` - Recipient phone number
- `$message` - SMS content
- `$senderId` - Branded sender ID

**Usage:**
```php
SendSMSJob::dispatch(
    mobile: '+639171234567',
    message: 'Your message',
    senderId: 'TXTCMDR'
);
```

### BroadcastCampaignJob
Queued job for broadcasting to campaign contacts.

**Location:** `app/Jobs/BroadcastCampaignJob.php`

**Properties:**
- `$campaign` - Campaign model instance

## ContactImportJob
Parses and stores uploaded contact files (CSV, Excel).

**Location:** `app/Jobs/ContactImportJob.php`

**Supported Formats:**
- CSV (`,` or `;` delimiter)
- Excel (.xlsx, .xls)

**Expected Columns:**
- `mobile` (required) - Phone number in +639XXXXXXXXX format
- `name` (optional) - Contact name
- `tags` (optional) - Comma-separated tags

## ScheduledBroadcast
Cron job to handle scheduled or batch-delayed sends.

**Location:** `app/Console/Commands/SendScheduledBroadcasts.php`

**Schedule:** Runs every minute

**Logic:**
1. Query campaigns with `scheduled_at <= now()` and `status = 'pending'`
2. Dispatch `BroadcastCampaignJob` for each
3. Update campaign status to `sending`

## WebhookHandler
Handles status callbacks from engageSPARK (delivery reports, replies).

**Location:** `app/Http/Controllers/WebhookController.php`

**Endpoints:**
- `POST /webhooks/engagespark/sms` - SMS delivery status
- `POST /webhooks/engagespark/airtime` - Airtime transfer status

**Payload Example:**
```json
{
  "organizationId": "12345",
  "recipientPhoneNumber": "+639171234567",
  "status": "delivered",
  "messageId": "msg_abc123",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

**Status Values:**
- `queued` - Message queued for delivery
- `sent` - Message sent to carrier
- `delivered` - Confirmed delivery
- `failed` - Delivery failed

## Event Listeners

### LogSMSSent
Logs SMS sending events for analytics and debugging.

**Location:** `app/Listeners/LogSMSSent.php`

**Listens To:** `LBHurtado\EngageSpark\Events\MessageSent`

### UpdateDeliveryStatus
Updates message delivery status in database.

**Location:** `app/Listeners/UpdateDeliveryStatus.php`

**Listens To:** Webhook events

## Integration with engageSPARK

engageSPARK provides:
- **Branded Sender IDs** - Pre-approved sender names (e.g., "QUEZON_CITY")
- **Airtime Topups** - Incentivize recipients with mobile load
- **Delivery Reports** - Real-time status via webhooks
- **Two-way SMS** - Handle incoming replies

**Configuration:**
```dotenv
ENGAGESPARK_API_KEY=your_api_key_here
ENGAGESPARK_ORGANIZATION_ID=your_org_id
ENGAGESPARK_SENDER_ID=TXTCMDR
ENGAGESPARK_SMS_WEBHOOK=https://yourapp.com/webhooks/engagespark/sms
```

## Driver Architecture

The SMS facade supports multiple drivers:

- **engagespark** - Primary driver (branded sender IDs)
- **nexmo** - Fallback driver (Vonage/Nexmo)
- **null** - Testing driver (no actual sending)

Switch drivers at runtime:
```php
// Use engageSPARK
SMS::channel('engagespark')->from('TXTCMDR')->to($mobile)->send();

// Use Nexmo as fallback
SMS::channel('nexmo')->from('+639171111111')->to($mobile)->send();
```

## Testing

Use the `null` driver for testing without sending real SMS:

```php
// In tests
config(['sms.default' => 'null']);

SMS::channel('null')
    ->from('TXTCMDR')
    ->to('+639171234567')
    ->content('Test message')
    ->send(); // No SMS sent
```

## Related Documentation

- [Package Architecture](package-architecture.md) - Detailed package structure
- [API Documentation](api-documentation.md) - HTTP API endpoints
- [Development Plan](development-plan.md) - Implementation roadmap
