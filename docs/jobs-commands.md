# Jobs & Commands

**Queue Driver:** Redis (recommended)  
**Namespace:** `App\Jobs` and `App\Console\Commands`

## Jobs

Text Commander uses queued jobs for asynchronous SMS processing and background tasks.

### SendSMSJob

**Path:** `app/Jobs/SendSMSJob.php`  
**Purpose:** Send individual SMS messages  
**Queue:** `sms`

```php
namespace App\Jobs;

use App\Jobs\Middleware\CheckBlacklist;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class SendSMSJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(
        public string $recipient,
        public string $message,
        public string $senderId
    ) {}

    public function middleware()
    {
        return [new CheckBlacklist()];
    }

    public function handle()
    {
        SMS::sendMessage(
            $this->recipient,
            $this->message,
            $this->senderId
        );

        Log::info('SMS sent', [
            'recipient' => $this->recipient,
            'sender_id' => $this->senderId,
        ]);
    }
}
```

**Features:**
- CheckBlacklist middleware auto-filters blacklisted numbers
- Queued for async processing
- Logging for audit trail

**See:** [Backend Services](backend-services.md)

---

### BroadcastToGroupJob

**Path:** `app/Jobs/BroadcastToGroupJob.php`  
**Purpose:** Broadcast SMS to all contacts in a group  
**Queue:** `broadcasts`

**See:** [Backend Services](backend-services.md)

---

### ProcessScheduledMessage

**Path:** `app/Jobs/ProcessScheduledMessage.php`  
**Purpose:** Process scheduled messages when due  
**Queue:** `scheduled`

**See:** [Scheduled Messaging](scheduled-messaging.md)

---

### ContactImportJob

**Path:** `app/Jobs/ContactImportJob.php`  
**Purpose:** Import contacts from CSV/Excel  
**Queue:** `imports`

**See:** [Backend Services](backend-services.md)

---

## Console Commands

### ProcessScheduledMessages

**Signature:** `sms:process-scheduled`  
**Schedule:** Every minute via Laravel Scheduler

```php
namespace App\Console\Commands;

use App\Models\ScheduledMessage;
use App\Jobs\ProcessScheduledMessage;
use Illuminate\Console\Command;

class ProcessScheduledMessages extends Command
{
    protected $signature = 'sms:process-scheduled';
    protected $description = 'Process due scheduled messages';

    public function handle()
    {
        $messages = ScheduledMessage::due()->get();

        foreach ($messages as $message) {
            ProcessScheduledMessage::dispatch($message);
        }

        $this->info("Dispatched {$messages->count()} scheduled messages");
    }
}
```

**Scheduler Setup (app/Console/Kernel.php):**

```php
protected function schedule(Schedule $schedule)
{
    $schedule->command('sms:process-scheduled')->everyMinute();
}
```

**See:** [Scheduled Messaging](scheduled-messaging.md)

---

### SendScheduledBroadcasts

**Signature:** `sms:send-scheduled-broadcasts`  
**Schedule:** Every minute

**See:** [Backend Services](backend-services.md)

---

## Queue Configuration

### Queue Worker Setup

**Supervisor Config (`/etc/supervisor/conf.d/txtcmdr-worker.conf`):**

```ini
[program:txtcmdr-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /path/to/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=8
redirect_stderr=true
stdout_logfile=/path/to/storage/logs/worker.log
stopwaitsecs=3600
```

**Start workers:**

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start txtcmdr-worker:*
```

---

## Job Middleware

### CheckBlacklist

**Path:** `app/Jobs/Middleware/CheckBlacklist.php`  
**Purpose:** Intercept SMS jobs and filter blacklisted numbers

**See:** [Blacklist Feature - Job Middleware](blacklist-feature.md)

---

## Related Documentation

- [Backend Services](backend-services.md)
- [Scheduled Messaging](scheduled-messaging.md)
- [Blacklist Feature](blacklist-feature.md)
- [Middleware](middleware.md)
- [DevOps & Deployment](deployment.md)
