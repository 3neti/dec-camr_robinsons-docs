# Notifications

**Status:** 🟡 Partial (SMS only) - See [TODO-SECTIONS.md](TODO-SECTIONS.md)

Text Commander supports SMS notifications via EngageSpark. Email and in-app notifications are planned.

## SMS Notifications ✅

### via EngageSpark

Text Commander's primary notification channel is SMS via the EngageSpark integration.

**See:** [SMS Integration](sms-integration.md)

### Send SMS

```php
use SMS;

SMS::send(
    recipient: '+639171234567',
    message: 'Your verification code is 123456',
    senderId: 'YourApp'
);
```

### Bulk SMS

```php
SendToMultipleRecipients::run(
    recipients: ['+639171234567', '+639181234567'],
    message: 'Important announcement',
    senderId: 'Quezon City'
);
```

**See:** [Backend Services](backend-services.md)

---

## Email Notifications (TODO)

### Planned Features

- Admin notifications for system events
- Failed SMS alerts
- Daily/weekly reports
- Low balance warnings

### Configuration

```php
// config/mail.php
'default' => env('MAIL_MAILER', 'smtp'),

'mailers' => [
    'smtp' => [
        'transport' => 'smtp',
        'host' => env('MAIL_HOST', 'smtp.mailtrap.io'),
        'port' => env('MAIL_PORT', 2525),
        // ...
    ],
],
```

### Example Notification

```php
namespace App\Notifications;

use Illuminate\Notifications\Notification;
use Illuminate\Notifications\Messages\MailMessage;

class SmsFailedNotification extends Notification
{
    public function via($notifiable)
    {
        return ['mail'];
    }

    public function toMail($notifiable)
    {
        return (new MailMessage)
            ->subject('SMS Delivery Failed')
            ->line('An SMS failed to send.')
            ->line('Recipient: ' . $this->recipient)
            ->action('View Logs', url('/logs'));
    }
}
```

**Priority:** Medium

---

## In-App Notifications (TODO)

### Planned Features

- Real-time notifications in UI
- Notification bell icon
- Read/unread status
- Notification preferences

### Database Table

```php
Schema::create('notifications', function (Blueprint $table) {
    $table->uuid('id')->primary();
    $table->string('type');
    $table->morphs('notifiable');
    $table->text('data');
    $table->timestamp('read_at')->nullable();
    $table->timestamps();
});
```

### Frontend Component

```vue
<template>
  <div class="notifications">
    <button @click="toggleDropdown">
      <BellIcon />
      <span v-if="unreadCount">{{ unreadCount }}</span>
    </button>
    <NotificationDropdown v-if="showDropdown" :notifications="notifications" />
  </div>
</template>
```

**Priority:** Low

---

## Push Notifications (Not Planned)

Mobile push notifications are not currently planned for Text Commander.

---

## Notification Preferences (TODO)

Allow users to configure notification preferences:

```php
// User preferences
$user->preferences = [
    'notifications' => [
        'sms_failed' => true,
        'low_balance' => true,
        'daily_report' => false,
    ],
];
```

---

## Related Documentation

- [SMS Integration](sms-integration.md)
- [Backend Services](backend-services.md)
- [Events & Listeners](events-listeners.md)
- [TODO-SECTIONS.md](TODO-SECTIONS.md)
