# Appendices

**Status:** ❌ Not documented - See [TODO-SECTIONS.md](TODO-SECTIONS.md)

Reference materials, command cheatsheet, troubleshooting, and FAQs.

## A. Environment Variables (TODO)

### Complete .env.example

```env
# Application
APP_NAME="Text Commander"
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://localhost

# Database
DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=txtcmdr
DB_USERNAME=postgres
DB_PASSWORD=

# Cache & Queue
CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis

# Redis
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

# Mail
MAIL_MAILER=smtp
MAIL_HOST=smtp.mailtrap.io
MAIL_PORT=2525
MAIL_USERNAME=
MAIL_PASSWORD=
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS="hello@txtcmdr.local"
MAIL_FROM_NAME="${APP_NAME}"

# SMS Integration
ENGAGESPARK_API_KEY=
ENGAGESPARK_ORG_ID=

# Third-Party Services
SENTRY_LARAVEL_DSN=
BUGSNAG_API_KEY=
```

---

## B. Artisan Commands Cheatsheet (TODO)

### Development
```bash
php artisan serve                # Start dev server
php artisan migrate              # Run migrations
php artisan db:seed              # Seed database
php artisan queue:work           # Start queue worker
php artisan schedule:work        # Run scheduler (dev)
```

### Testing
```bash
php artisan test                 # Run all tests
php artisan test --parallel      # Run tests in parallel
```

### Optimization
```bash
php artisan optimize             # Optimize framework
php artisan config:cache         # Cache config
php artisan route:cache          # Cache routes
php artisan view:cache           # Cache views
php artisan optimize:clear       # Clear all caches
```

### Custom Commands
```bash
php artisan sms:process-scheduled  # Process scheduled messages
```

---

## C. API Testing (TODO)

### Postman Collection

Download: [Text Commander Postman Collection](#) (TODO)

### cURL Examples

**Send SMS:**
```bash
curl -X POST https://txtcmdr.example.com/api/sms/send \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "recipients": ["+639171234567"],
    "message": "Test message",
    "sender_id": "TestSender"
  }'
```

**Schedule SMS:**
```bash
curl -X POST https://txtcmdr.example.com/api/sms/schedule \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Scheduled message",
    "recipients": ["+639171234567"],
    "sender_id": "TestSender",
    "scheduled_at": "2024-12-31 12:00:00"
  }'
```

---

## D. Troubleshooting (TODO)

### Common Issues

**Queue workers not processing jobs**
```bash
# Check if workers are running
sudo supervisorctl status

# Restart workers
php artisan queue:restart
sudo supervisorctl restart txtcmdr-worker:*
```

**Scheduled tasks not running**
```bash
# Check cron entry
crontab -l

# Test scheduler manually
php artisan schedule:run
```

**SMS not sending**
```bash
# Check EngageSpark credentials
php artisan tinker
>>> config('services.engagespark.api_key')

# Check blacklist
php artisan tinker
>>> \App\Models\BlacklistedNumber::where('mobile', '+639171234567')->exists()
```

---

## E. FAQ (TODO)

**Q: How do I generate an API token?**  
A: Via Sanctum: `$user->createToken('api-token')->plainTextToken`

**Q: How do I add a number to the blacklist?**  
A: See [Blacklist Feature](blacklist-feature.md)

**Q: How do I schedule a message?**  
A: See [Scheduled Messaging](scheduled-messaging.md)

**Q: How do I change the sender ID?**  
A: Update `sender_ids` table and use in SMS send request

---

## F. Glossary (TODO)

- **Action** - Single-purpose class (lorisleiva/laravel-actions)
- **Blacklist** - List of numbers that should not receive SMS
- **EngageSpark** - SMS gateway provider
- **E.164** - International phone number format (+639171234567)
- **Inertia.js** - SPA framework without building an API
- **Job Middleware** - Middleware that runs before queued jobs
- **Sanctum** - Laravel's API authentication package
- **Sender ID** - Branded SMS sender name (e.g., "Quezon City")

---

## G. Admin Credentials (Staging) (TODO)

**Email:** admin@txtcmdr.local  
**Password:** (provided separately)

---

## Related Documentation

- [Quick Start](quick-start.md)
- [API Documentation](api-documentation.md)
- [Deployment](deployment.md)
- [TODO-SECTIONS.md](TODO-SECTIONS.md)
