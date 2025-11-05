# Scheduled Messaging

This document describes the complete implementation for **deferred/scheduled SMS sending** in Text Commander, including database models, Actions, Jobs, API endpoints, and UI components.

---

## Overview

Users can schedule SMS messages to be sent at a future date/time instead of sending immediately. Scheduled messages are stored in the database and processed by a scheduled command that runs every minute.

### Key Features
- ✅ Schedule messages to individuals, contacts, or groups
- ✅ View scheduled messages in Logs screen
- ✅ Edit or cancel scheduled messages before they're sent
- ✅ Automatic processing via Laravel scheduler
- ✅ Status tracking (pending → sending → sent)

---

## Database Schema

### Migration: `create_scheduled_messages_table`

**Location:** `database/migrations/xxxx_create_scheduled_messages_table.php`

```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('scheduled_messages', function (Blueprint $table) {
            $table->id();
            
            // Message content
            $table->text('message');
            $table->string('sender_id')->default('TXTCMDR');
            
            // Recipients (polymorphic)
            $table->string('recipient_type'); // 'numbers', 'group', 'mixed'
            $table->json('recipient_data');   // Store numbers, group IDs, or both
            
            // Scheduling
            $table->timestamp('scheduled_at');
            $table->timestamp('sent_at')->nullable();
            
            // Status tracking
            $table->enum('status', ['pending', 'processing', 'sent', 'failed', 'cancelled'])
                ->default('pending');
            
            // Metadata
            $table->integer('total_recipients')->default(0);
            $table->integer('sent_count')->default(0);
            $table->integer('failed_count')->default(0);
            $table->json('errors')->nullable();
            
            $table->timestamps();
            
            // Indexes
            $table->index(['status', 'scheduled_at']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('scheduled_messages');
    }
};
```

### Model: `ScheduledMessage`

**Location:** `app/Models/ScheduledMessage.php`

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Casts\Attribute;

class ScheduledMessage extends Model
{
    protected $fillable = [
        'message',
        'sender_id',
        'recipient_type',
        'recipient_data',
        'scheduled_at',
        'sent_at',
        'status',
        'total_recipients',
        'sent_count',
        'failed_count',
        'errors',
    ];

    protected $casts = [
        'recipient_data' => 'array',
        'errors' => 'array',
        'scheduled_at' => 'datetime',
        'sent_at' => 'datetime',
    ];

    // Scopes
    public function scopePending($query)
    {
        return $query->where('status', 'pending');
    }

    public function scopeReady($query)
    {
        return $query->where('status', 'pending')
            ->where('scheduled_at', '<=', now());
    }

    // Accessors
    public function isPending(): bool
    {
        return $this->status === 'pending';
    }

    public function isCancellable(): bool
    {
        return in_array($this->status, ['pending', 'processing']);
    }

    public function isEditable(): bool
    {
        return $this->status === 'pending' 
            && $this->scheduled_at->isFuture();
    }

    // Helper to get recipient summary
    public function recipientSummary(): Attribute
    {
        return Attribute::make(
            get: function () {
                return match($this->recipient_type) {
                    'numbers' => count($this->recipient_data['numbers']) . ' number(s)',
                    'group' => $this->recipient_data['group_name'] . ' (' . $this->total_recipients . ')',
                    'mixed' => $this->total_recipients . ' recipient(s)',
                    default => 'Unknown'
                };
            }
        );
    }
}
```

---

## Backend Implementation

### Action: `ScheduleMessage`

**Location:** `app/Actions/ScheduleMessage.php`

**Endpoint:** `POST /api/send/schedule`

```php
namespace App\Actions;

use App\Models\Contact;
use App\Models\Group;
use App\Models\ScheduledMessage;
use Carbon\Carbon;
use Illuminate\Http\JsonResponse;
use Lorisleiva\Actions\ActionRequest;
use Lorisleiva\Actions\Concerns\AsAction;
use Propaganistas\LaravelPhone\PhoneNumber;

class ScheduleMessage
{
    use AsAction;

    /**
     * Schedule a message for future delivery
     * 
     * @param array|string $recipients Phone numbers, contact names, or group names
     * @param string $message Message content
     * @param string|Carbon $scheduledAt When to send
     * @param string|null $senderId Sender ID
     */
    public function handle(
        array|string $recipients,
        string $message,
        string|Carbon $scheduledAt,
        ?string $senderId = null
    ): ScheduledMessage {
        $senderId = $senderId ?? config('sms.default_sender_id', 'TXTCMDR');
        $scheduledAt = $scheduledAt instanceof Carbon 
            ? $scheduledAt 
            : Carbon::parse($scheduledAt);

        // Normalize recipients
        $recipientArray = is_string($recipients) 
            ? array_map('trim', explode(',', $recipients))
            : $recipients;

        // Parse recipients (can be phone numbers, contacts, or groups)
        $parsedRecipients = $this->parseRecipients($recipientArray);

        return ScheduledMessage::create([
            'message' => $message,
            'sender_id' => $senderId,
            'recipient_type' => $parsedRecipients['type'],
            'recipient_data' => $parsedRecipients['data'],
            'scheduled_at' => $scheduledAt,
            'total_recipients' => $parsedRecipients['count'],
            'status' => 'pending',
        ]);
    }

    public function rules(): array
    {
        return [
            'recipients' => 'required',
            'message' => 'required|string|max:1600',
            'scheduled_at' => 'required|date|after:now',
            'sender_id' => 'nullable|string|max:11',
        ];
    }

    public function asController(ActionRequest $request): JsonResponse
    {
        $scheduledMessage = $this->handle(
            $request->recipients,
            $request->message,
            $request->scheduled_at,
            $request->sender_id ?? null
        );

        return response()->json([
            'id' => $scheduledMessage->id,
            'status' => 'scheduled',
            'scheduled_at' => $scheduledMessage->scheduled_at->toIso8601String(),
            'recipient_count' => $scheduledMessage->total_recipients,
            'message' => 'Message scheduled successfully',
        ], 201);
    }

    /**
     * Parse recipients into phone numbers and groups
     */
    protected function parseRecipients(array $recipients): array
    {
        $numbers = [];
        $groups = [];
        $totalCount = 0;

        foreach ($recipients as $recipient) {
            // Try to parse as phone number
            try {
                $phone = new PhoneNumber($recipient, 'PH');
                $contact = Contact::fromPhoneNumber($phone);
                $numbers[] = $contact->e164_mobile;
                $totalCount++;
                continue;
            } catch (\Exception $e) {
                // Not a valid phone number
            }

            // Try to find as contact by name
            $contact = Contact::whereRaw("extra_attributes->>'name' = ?", [$recipient])->first();
            if ($contact) {
                $numbers[] = $contact->e164_mobile;
                $totalCount++;
                continue;
            }

            // Try to find as group
            $group = Group::where('name', $recipient)->first();
            if ($group) {
                $groupContactCount = $group->contacts()->count();
                $groups[] = [
                    'id' => $group->id,
                    'name' => $group->name,
                    'count' => $groupContactCount,
                ];
                $totalCount += $groupContactCount;
            }
        }

        // Determine recipient type
        $type = 'mixed';
        if (empty($groups)) {
            $type = 'numbers';
        } elseif (empty($numbers) && count($groups) === 1) {
            $type = 'group';
        }

        return [
            'type' => $type,
            'data' => [
                'numbers' => $numbers,
                'groups' => $groups,
            ],
            'count' => $totalCount,
        ];
    }
}
```

**Routes:**
```php
// routes/api.php
use App\Actions\ScheduleMessage;

Route::post('/send/schedule', ScheduleMessage::class);
```

---

### Action: `UpdateScheduledMessage`

**Location:** `app/Actions/UpdateScheduledMessage.php`

**Endpoint:** `PUT /api/scheduled-messages/{id}`

```php
namespace App\Actions;

use App\Models\ScheduledMessage;
use Carbon\Carbon;
use Illuminate\Http\JsonResponse;
use Lorisleiva\Actions\ActionRequest;
use Lorisleiva\Actions\Concerns\AsAction;

class UpdateScheduledMessage
{
    use AsAction;

    public function handle(
        int $id,
        ?string $message = null,
        ?string $scheduledAt = null,
        ?string $senderId = null
    ): ScheduledMessage {
        $scheduledMessage = ScheduledMessage::findOrFail($id);

        // Only allow editing if status is 'pending' and scheduled_at is in the future
        if (!$scheduledMessage->isEditable()) {
            abort(422, 'Cannot edit this scheduled message');
        }

        $updates = [];

        if ($message !== null) {
            $updates['message'] = $message;
        }

        if ($scheduledAt !== null) {
            $updates['scheduled_at'] = Carbon::parse($scheduledAt);
        }

        if ($senderId !== null) {
            $updates['sender_id'] = $senderId;
        }

        $scheduledMessage->update($updates);

        return $scheduledMessage->fresh();
    }

    public function rules(): array
    {
        return [
            'message' => 'sometimes|string|max:1600',
            'scheduled_at' => 'sometimes|date|after:now',
            'sender_id' => 'sometimes|string|max:11',
        ];
    }

    public function asController(ActionRequest $request, int $id): JsonResponse
    {
        $scheduledMessage = $this->handle(
            $id,
            $request->message ?? null,
            $request->scheduled_at ?? null,
            $request->sender_id ?? null
        );

        return response()->json($scheduledMessage, 200);
    }
}
```

---

### Action: `CancelScheduledMessage`

**Location:** `app/Actions/CancelScheduledMessage.php`

**Endpoint:** `POST /api/scheduled-messages/{id}/cancel`

```php
namespace App\Actions;

use App\Models\ScheduledMessage;
use Illuminate\Http\JsonResponse;
use Lorisleiva\Actions\Concerns\AsAction;

class CancelScheduledMessage
{
    use AsAction;

    public function handle(int $id): ScheduledMessage
    {
        $scheduledMessage = ScheduledMessage::findOrFail($id);

        if (!$scheduledMessage->isCancellable()) {
            abort(422, 'Cannot cancel this message');
        }

        $scheduledMessage->update(['status' => 'cancelled']);

        return $scheduledMessage;
    }

    public function asController(int $id): JsonResponse
    {
        $scheduledMessage = $this->handle($id);

        return response()->json([
            'message' => 'Scheduled message cancelled',
            'scheduled_message' => $scheduledMessage,
        ], 200);
    }
}
```

---

### Action: `ListScheduledMessages`

**Location:** `app/Actions/ListScheduledMessages.php`

**Endpoint:** `GET /api/scheduled-messages`

```php
namespace App\Actions;

use App\Models\ScheduledMessage;
use Illuminate\Http\JsonResponse;
use Lorisleiva\Actions\ActionRequest;
use Lorisleiva\Actions\Concerns\AsAction;

class ListScheduledMessages
{
    use AsAction;

    public function handle(string $status = 'all')
    {
        $query = ScheduledMessage::query()->orderBy('scheduled_at', 'desc');

        if ($status !== 'all') {
            $query->where('status', $status);
        }

        return $query->paginate(20);
    }

    public function asController(ActionRequest $request): JsonResponse
    {
        $status = $request->query('status', 'all');
        $messages = $this->handle($status);

        return response()->json($messages, 200);
    }
}
```

---

### Job: `ProcessScheduledMessage`

**Location:** `app/Jobs/ProcessScheduledMessage.php`

**Description:** Dispatched by console command to process a single scheduled message.

```php
namespace App\Jobs;

use App\Jobs\SendSMSJob;
use App\Models\Contact;
use App\Models\Group;
use App\Models\ScheduledMessage;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class ProcessScheduledMessage implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(
        public int $scheduledMessageId
    ) {}

    public function handle(): void
    {
        $scheduledMessage = ScheduledMessage::find($this->scheduledMessageId);

        if (!$scheduledMessage || $scheduledMessage->status !== 'pending') {
            return;
        }

        // Update status to processing
        $scheduledMessage->update(['status' => 'processing']);

        $recipientData = $scheduledMessage->recipient_data;
        $numbers = [];

        // Collect all numbers
        if (!empty($recipientData['numbers'])) {
            $numbers = array_merge($numbers, $recipientData['numbers']);
        }

        // Collect numbers from groups
        if (!empty($recipientData['groups'])) {
            foreach ($recipientData['groups'] as $groupData) {
                $group = Group::find($groupData['id']);
                if ($group) {
                    $groupNumbers = $group->contacts()
                        ->pluck('mobile')
                        ->map(function ($mobile) {
                            $contact = Contact::where('mobile', $mobile)->first();
                            return $contact?->e164_mobile;
                        })
                        ->filter()
                        ->toArray();
                    
                    $numbers = array_merge($numbers, $groupNumbers);
                }
            }
        }

        // Remove duplicates
        $numbers = array_unique($numbers);

        // Dispatch individual SMS jobs
        $sentCount = 0;
        $failedCount = 0;
        $errors = [];

        foreach ($numbers as $number) {
            try {
                SendSMSJob::dispatch(
                    $number,
                    $scheduledMessage->message,
                    $scheduledMessage->sender_id
                );
                $sentCount++;
            } catch (\Exception $e) {
                $failedCount++;
                $errors[] = [
                    'number' => $number,
                    'error' => $e->getMessage(),
                ];
            }
        }

        // Update scheduled message
        $scheduledMessage->update([
            'status' => 'sent',
            'sent_at' => now(),
            'sent_count' => $sentCount,
            'failed_count' => $failedCount,
            'errors' => $errors,
        ]);
    }
}
```

---

### Console Command: `ProcessScheduledMessages`

**Location:** `app/Console/Commands/ProcessScheduledMessages.php`

**Schedule:** Runs every minute

```php
namespace App\Console\Commands;

use App\Jobs\ProcessScheduledMessage;
use App\Models\ScheduledMessage;
use Illuminate\Console\Command;

class ProcessScheduledMessages extends Command
{
    protected $signature = 'messages:process-scheduled';
    protected $description = 'Process scheduled messages that are ready to send';

    public function handle(): int
    {
        $messages = ScheduledMessage::ready()->get();

        if ($messages->isEmpty()) {
            $this->info('No scheduled messages ready to send');
            return 0;
        }

        foreach ($messages as $message) {
            ProcessScheduledMessage::dispatch($message->id);
        }

        $this->info("Dispatched {$messages->count()} scheduled message(s)");

        return 0;
    }
}
```

**Register in `app/Console/Kernel.php`:**
```php
protected function schedule(Schedule $schedule)
{
    $schedule->command('messages:process-scheduled')->everyMinute();
}
```

---

## API Endpoints Summary

```php
// routes/api.php

use App\Actions\{
    ScheduleMessage,
    UpdateScheduledMessage,
    CancelScheduledMessage,
    ListScheduledMessages
};

// Schedule a message
Route::post('/send/schedule', ScheduleMessage::class);

// Manage scheduled messages
Route::get('/scheduled-messages', ListScheduledMessages::class);
Route::put('/scheduled-messages/{id}', UpdateScheduledMessage::class);
Route::post('/scheduled-messages/{id}/cancel', CancelScheduledMessage::class);
```

---

## Frontend Implementation (Vue + Inertia.js)

### Component: `ScheduleModal.vue`

**Location:** `resources/js/Components/ScheduleModal.vue`

```vue
<template>
  <TransitionRoot :show="show" as="template">
    <Dialog as="div" class="relative z-50" @close="close">
      <TransitionChild
        enter="ease-out duration-300"
        enter-from="opacity-0"
        enter-to="opacity-100"
        leave="ease-in duration-200"
        leave-from="opacity-100"
        leave-to="opacity-0"
      >
        <div class="fixed inset-0 bg-gray-500 bg-opacity-75 transition-opacity" />
      </TransitionChild>

      <div class="fixed inset-0 z-10 overflow-y-auto">
        <div class="flex min-h-full items-center justify-center p-4">
          <TransitionChild
            enter="ease-out duration-300"
            enter-from="opacity-0 translate-y-4"
            enter-to="opacity-100 translate-y-0"
            leave="ease-in duration-200"
            leave-from="opacity-100 translate-y-0"
            leave-to="opacity-0 translate-y-4"
          >
            <DialogPanel class="relative w-full max-w-md transform overflow-hidden rounded-lg bg-white p-6 shadow-xl transition-all">
              <DialogTitle class="text-lg font-semibold text-gray-900 mb-4">
                Schedule Message
              </DialogTitle>

              <form @submit.prevent="submit">
                <!-- Date Picker -->
                <div class="mb-4">
                  <label class="block text-sm font-medium text-gray-700 mb-2">
                    Date
                  </label>
                  <input
                    v-model="form.date"
                    type="date"
                    :min="minDate"
                    required
                    class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
                  />
                </div>

                <!-- Time Picker -->
                <div class="mb-4">
                  <label class="block text-sm font-medium text-gray-700 mb-2">
                    Time
                  </label>
                  <input
                    v-model="form.time"
                    type="time"
                    required
                    class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
                  />
                </div>

                <!-- Summary -->
                <div class="mb-6 p-3 bg-blue-50 rounded-lg">
                  <p class="text-sm text-gray-700">
                    <span class="font-medium">Send on:</span>
                    {{ formattedDateTime }}
                  </p>
                  <p class="text-sm text-gray-700 mt-1">
                    <span class="font-medium">Recipients:</span>
                    {{ recipientCount }}
                  </p>
                </div>

                <!-- Actions -->
                <div class="flex gap-3">
                  <button
                    type="button"
                    @click="close"
                    class="flex-1 px-4 py-2 border border-gray-300 text-gray-700 rounded-lg hover:bg-gray-50"
                  >
                    Cancel
                  </button>
                  <button
                    type="submit"
                    :disabled="processing"
                    class="flex-1 px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 disabled:opacity-50"
                  >
                    {{ processing ? 'Scheduling...' : 'Schedule' }}
                  </button>
                </div>
              </form>
            </DialogPanel>
          </TransitionChild>
        </div>
      </div>
    </Dialog>
  </TransitionRoot>
</template>

<script setup>
import { ref, computed } from 'vue'
import { Dialog, DialogPanel, DialogTitle, TransitionRoot, TransitionChild } from '@headlessui/vue'
import { useForm } from '@inertiajs/vue3'
import dayjs from 'dayjs'

const props = defineProps({
  show: Boolean,
  recipients: [String, Array],
  message: String,
  senderId: String,
  recipientCount: Number,
})

const emit = defineEmits(['close', 'success'])

const minDate = computed(() => dayjs().format('YYYY-MM-DD'))

const form = useForm({
  date: dayjs().format('YYYY-MM-DD'),
  time: dayjs().add(1, 'hour').format('HH:mm'),
})

const formattedDateTime = computed(() => {
  if (!form.date || !form.time) return '—'
  return dayjs(`${form.date} ${form.time}`).format('MMM D, YYYY [at] h:mm A')
})

const processing = ref(false)

const submit = async () => {
  processing.value = true

  try {
    await axios.post('/api/send/schedule', {
      recipients: props.recipients,
      message: props.message,
      sender_id: props.senderId,
      scheduled_at: `${form.date} ${form.time}`,
    })

    emit('success')
    close()
  } catch (error) {
    console.error('Failed to schedule message:', error)
    alert('Failed to schedule message. Please try again.')
  } finally {
    processing.value = false
  }
}

const close = () => {
  emit('close')
}
</script>
```

---

### Page: `Dashboard.vue` (Updated)

**Location:** `resources/js/Pages/Dashboard.vue`

**Changes:**
- Add "Schedule" button next to "Send Now"
- Open `ScheduleModal` on click
- Pass recipients and message to modal

```vue
<template>
  <AppLayout>
    <div class="max-w-4xl mx-auto p-6">
      <h1 class="text-2xl font-bold text-gray-900 mb-6">📤 Send SMS</h1>

      <form @submit.prevent="sendNow">
        <!-- To: Recipients -->
        <div class="mb-4">
          <label class="block text-sm font-medium text-gray-700 mb-2">To:</label>
          <RecipientInput v-model="form.recipients" />
        </div>

        <!-- Message -->
        <div class="mb-4">
          <MessageTextarea v-model="form.message" />
        </div>

        <!-- Sender ID -->
        <div class="mb-6 flex items-center justify-between">
          <div class="text-sm text-gray-600">
            {{ characterCount }}/160 ({{ smsCount }} SMS)
          </div>
          <SenderSelect v-model="form.senderId" />
        </div>

        <!-- Actions -->
        <div class="flex gap-3">
          <button
            type="submit"
            :disabled="!canSend"
            class="flex-1 px-6 py-3 bg-blue-600 text-white font-semibold rounded-lg hover:bg-blue-700 disabled:opacity-50"
          >
            Send Now
          </button>
          <button
            type="button"
            @click="openScheduleModal"
            :disabled="!canSend"
            class="flex-1 px-6 py-3 border-2 border-blue-600 text-blue-600 font-semibold rounded-lg hover:bg-blue-50 disabled:opacity-50"
          >
            Schedule
          </button>
        </div>
      </form>
    </div>

    <!-- Schedule Modal -->
    <ScheduleModal
      :show="showScheduleModal"
      :recipients="form.recipients"
      :message="form.message"
      :sender-id="form.senderId"
      :recipient-count="recipientCount"
      @close="showScheduleModal = false"
      @success="handleScheduleSuccess"
    />
  </AppLayout>
</template>

<script setup>
import { ref, computed } from 'vue'
import { useForm } from '@inertiajs/vue3'
import AppLayout from '@/Layouts/AppLayout.vue'
import RecipientInput from '@/Components/SMSComposer/RecipientInput.vue'
import MessageTextarea from '@/Components/SMSComposer/MessageTextarea.vue'
import SenderSelect from '@/Components/SMSComposer/SenderSelect.vue'
import ScheduleModal from '@/Components/ScheduleModal.vue'

const form = useForm({
  recipients: [],
  message: '',
  senderId: 'TXTCMDR',
})

const showScheduleModal = ref(false)

const characterCount = computed(() => form.message.length)
const smsCount = computed(() => Math.ceil(characterCount.value / 160))
const recipientCount = computed(() => {
  // Calculate based on recipients (simplified)
  return Array.isArray(form.recipients) ? form.recipients.length : 1
})

const canSend = computed(() => {
  return form.recipients.length > 0 && form.message.trim().length > 0
})

const sendNow = () => {
  form.post('/api/send', {
    onSuccess: () => {
      form.reset()
      alert('✓ Message sent!')
    },
  })
}

const openScheduleModal = () => {
  showScheduleModal.value = true
}

const handleScheduleSuccess = () => {
  form.reset()
  alert('✓ Message scheduled!')
}
</script>
```

---

### Page: `Logs/Index.vue` (Updated)

**Location:** `resources/js/Pages/Logs/Index.vue`

**Changes:**
- Add filter for scheduled messages
- Show scheduled messages with edit/cancel buttons
- Display scheduled_at timestamp

```vue
<template>
  <AppLayout>
    <div class="max-w-6xl mx-auto p-6">
      <div class="flex items-center justify-between mb-6">
        <h1 class="text-2xl font-bold text-gray-900">Message Logs</h1>
        
        <div class="flex gap-2">
          <select
            v-model="statusFilter"
            class="px-4 py-2 border border-gray-300 rounded-lg"
          >
            <option value="all">All</option>
            <option value="pending">Scheduled</option>
            <option value="sent">Sent</option>
            <option value="failed">Failed</option>
            <option value="cancelled">Cancelled</option>
          </select>
        </div>
      </div>

      <div class="space-y-4">
        <!-- Scheduled Messages -->
        <div
          v-for="message in scheduledMessages.data"
          :key="message.id"
          class="bg-white border border-gray-200 rounded-lg p-4"
        >
          <div class="flex items-start justify-between">
            <div class="flex-1">
              <div class="flex items-center gap-2 mb-2">
                <span
                  :class="statusBadgeClass(message.status)"
                  class="px-2 py-1 text-xs font-medium rounded"
                >
                  {{ statusLabel(message.status) }}
                </span>
                <span class="text-sm text-gray-600">
                  {{ message.recipient_summary }}
                </span>
              </div>
              
              <p class="text-gray-900 mb-2">{{ message.message }}</p>
              
              <div class="text-sm text-gray-600">
                <span v-if="message.status === 'pending'">
                  ⏱️ Scheduled for {{ formatDateTime(message.scheduled_at) }}
                </span>
                <span v-else-if="message.sent_at">
                  ✓ Sent {{ formatDateTime(message.sent_at) }}
                </span>
                <span class="ml-3">From: {{ message.sender_id }}</span>
              </div>
            </div>

            <!-- Actions for scheduled messages -->
            <div v-if="message.status === 'pending'" class="flex gap-2">
              <button
                @click="editScheduledMessage(message.id)"
                class="px-3 py-1 text-sm text-blue-600 border border-blue-600 rounded hover:bg-blue-50"
              >
                Edit
              </button>
              <button
                @click="cancelScheduledMessage(message.id)"
                class="px-3 py-1 text-sm text-red-600 border border-red-600 rounded hover:bg-red-50"
              >
                Cancel
              </button>
            </div>
          </div>
        </div>
      </div>

      <!-- Pagination -->
      <div v-if="scheduledMessages.links" class="mt-6">
        <!-- Pagination component -->
      </div>
    </div>
  </AppLayout>
</template>

<script setup>
import { ref, watch } from 'vue'
import { router } from '@inertiajs/vue3'
import AppLayout from '@/Layouts/AppLayout.vue'
import dayjs from 'dayjs'

const props = defineProps({
  scheduledMessages: Object,
})

const statusFilter = ref('all')

watch(statusFilter, (value) => {
  router.get('/logs', { status: value }, { preserveState: true })
})

const statusBadgeClass = (status) => {
  const classes = {
    pending: 'bg-yellow-100 text-yellow-800',
    processing: 'bg-blue-100 text-blue-800',
    sent: 'bg-green-100 text-green-800',
    failed: 'bg-red-100 text-red-800',
    cancelled: 'bg-gray-100 text-gray-800',
  }
  return classes[status] || 'bg-gray-100 text-gray-800'
}

const statusLabel = (status) => {
  const labels = {
    pending: '⏱️ Scheduled',
    processing: '📤 Sending',
    sent: '✓ Sent',
    failed: '❌ Failed',
    cancelled: '🚫 Cancelled',
  }
  return labels[status] || status
}

const formatDateTime = (dateTime) => {
  return dayjs(dateTime).format('MMM D, YYYY [at] h:mm A')
}

const cancelScheduledMessage = async (id) => {
  if (!confirm('Cancel this scheduled message?')) return

  try {
    await axios.post(`/api/scheduled-messages/${id}/cancel`)
    router.reload()
  } catch (error) {
    alert('Failed to cancel message')
  }
}

const editScheduledMessage = (id) => {
  // Open edit modal or navigate to edit page
  router.visit(`/scheduled-messages/${id}/edit`)
}
</script>
```

---

## Testing

### Unit Test: `ScheduleMessageTest`

**Location:** `tests/Feature/ScheduleMessageTest.php`

```php
namespace Tests\Feature;

use App\Models\Contact;
use App\Models\Group;
use App\Models\ScheduledMessage;
use Carbon\Carbon;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class ScheduleMessageTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function it_can_schedule_a_message_to_phone_numbers()
    {
        $response = $this->postJson('/api/send/schedule', [
            'recipients' => '+639171234567,+639189876543',
            'message' => 'Test message',
            'scheduled_at' => Carbon::now()->addHour()->toIso8601String(),
            'sender_id' => 'TXTCMDR',
        ]);

        $response->assertStatus(201);
        $this->assertDatabaseHas('scheduled_messages', [
            'message' => 'Test message',
            'status' => 'pending',
        ]);
    }

    /** @test */
    public function it_can_cancel_a_scheduled_message()
    {
        $scheduled = ScheduledMessage::factory()->create([
            'status' => 'pending',
            'scheduled_at' => Carbon::now()->addHour(),
        ]);

        $response = $this->postJson("/api/scheduled-messages/{$scheduled->id}/cancel");

        $response->assertStatus(200);
        $this->assertEquals('cancelled', $scheduled->fresh()->status);
    }

    /** @test */
    public function it_cannot_edit_a_past_scheduled_message()
    {
        $scheduled = ScheduledMessage::factory()->create([
            'status' => 'pending',
            'scheduled_at' => Carbon::now()->subHour(),
        ]);

        $response = $this->putJson("/api/scheduled-messages/{$scheduled->id}", [
            'message' => 'Updated message',
        ]);

        $response->assertStatus(422);
    }
}
```

---

## Related Documentation

- [API Documentation](api-documentation.md) - All API endpoints
- [Backend Services](backend-services.md) - Actions and Jobs
- [UI/UX Design](ui-ux-design.md) - User interface specifications
- [Development Plan](development-plan.md) - Implementation roadmap
