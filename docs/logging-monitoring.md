# Logging & Monitoring

**Status:** 🟡 Partial - See [TODO-SECTIONS.md](TODO-SECTIONS.md)

Centralized strategy for logs, application monitoring, performance metrics, and error tracking.

## Logging Strategy

### Conventions
- Use structured logging with context arrays
- Prefer `info` for normal operations, `warning` for recoverable issues, `error` for failures
- Log job lifecycle events and API errors

```php
Log::info('SMS sent', [
    'recipient' => $recipient,
    'sender_id' => $senderId,
    'message_id' => $messageId ?? null,
]);

Log::warning('SMS blocked - blacklisted', [
    'recipient' => $recipient,
]);

Log::error('SMS failed', [
    'recipient' => $recipient,
    'error' => $e->getMessage(),
]);
```

### Storage
- Default: `storage/logs/laravel.log`
- Rotate logs daily in production (see `config/logging.php`)

---

## Laravel Telescope (TODO)

Developer tool for debugging, including requests, exceptions, queries, jobs, etc.

```bash
composer require laravel/telescope --dev
php artisan telescope:install
php artisan migrate
```

```php
// app/Providers/TelescopeServiceProvider.php
Telescope::filter(function (IncomingEntry $entry) {
    return app()->environment('local') ||
        $entry->isRequest() || $entry->isJob() || $entry->isException();
});
```

**Access:** `/telescope` (local only)

---

## Error Tracking (TODO)

Integrate an external service (Sentry/Bugsnag) for errors in production.

### Sentry

```bash
composer require sentry/sentry-laravel
php artisan vendor:publish --provider="Sentry\Laravel\ServiceProvider"
```

```env
SENTRY_LARAVEL_DSN={{SENTRY_DSN}}
SENTRY_TRACE_SAMPLE_RATE=0.1
```

### Bugsnag

```bash
composer require bugsnag/bugsnag-laravel
```

```env
BUGSNAG_API_KEY={{BUGSNAG_API_KEY}}
```

---

## Performance Monitoring (TODO)

- Slow query log (enable and review)
- Queue time and job duration metrics
- Cache hit/miss metrics

---

## Uptime & Alerts (TODO)

- External uptime monitoring (e.g., UptimeRobot)
- Alerting via email/SMS on downtime
- Health check endpoint (`/health`)

---

## Related Documentation

- [Jobs & Commands](jobs-commands.md)
- [Notifications](notifications.md)
- [Deployment](deployment.md)
- [API Documentation](api-documentation.md)
