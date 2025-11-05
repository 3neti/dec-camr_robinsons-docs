# Jobs, Events & Listeners

**Status:** ❌ Not yet documented - See [TODO-SECTIONS.md](TODO-SECTIONS.md)

This document will cover event-driven architecture, events, listeners, and subscribers in Text Commander.

## Current Architecture

Text Commander currently uses **Jobs** for asynchronous processing instead of event-driven architecture.

**See:** [Jobs & Commands](jobs-commands.md)

---

## Planned Events (TODO)

### SMS Events

```php
// When SMS is sent successfully
event(new SmsSent($recipient, $message, $senderId));

// When SMS fails
event(new SmsFailed($recipient, $message, $error));

// When number is blacklisted
event(new NumberBlacklisted($mobile, $reason));
```

---

### Contact Events

```php
// When contact is created
event(new ContactCreated($contact));

// When contact is added to group
event(new ContactAddedToGroup($contact, $group));
```

---

### Scheduled Message Events

```php
// When scheduled message is processed
event(new ScheduledMessageProcessed($scheduledMessage));

// When scheduled message fails
event(new ScheduledMessageFailed($scheduledMessage, $error));
```

---

## Planned Listeners (TODO)

### Logging Listener

```php
class LogSmsActivity
{
    public function handle(SmsSent $event)
    {
        Log::info('SMS sent successfully', [
            'recipient' => $event->recipient,
            'sender_id' => $event->senderId,
        ]);
    }
}
```

---

### Notification Listener

```php
class NotifyOnSmsFailed
{
    public function handle(SmsFailed $event)
    {
        // Notify admin of failure
        Mail::to(config('mail.admin'))
            ->send(new SmsFailedNotification($event));
    }
}
```

---

## Event Registration (TODO)

**File:** `app/Providers/EventServiceProvider.php`

```php
protected $listen = [
    SmsSent::class => [
        LogSmsActivity::class,
    ],
    SmsFailed::class => [
        NotifyOnSmsFailed::class,
    ],
    NumberBlacklisted::class => [
        LogBlacklistActivity::class,
    ],
];
```

---

## Testing Events (TODO)

```php
use Illuminate\Support\Facades\Event;

test('dispatches SmsSent event', function () {
    Event::fake();

    SendSMSJob::dispatch('+639171234567', 'Test', 'TestSender');

    Event::assertDispatched(SmsSent::class);
});
```

---

## Related Documentation

- [Jobs & Commands](jobs-commands.md)
- [Backend Services](backend-services.md)
- [Notifications](notifications.md)
- [TODO-SECTIONS.md](TODO-SECTIONS.md)
