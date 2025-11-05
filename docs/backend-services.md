# Backend Services

This document outlines the Laravel Actions, Controllers, and Services that power the Text Commander API.

> **Architecture Note:** Text Commander uses [lorisleiva/laravel-actions](https://laravelactions.com/) to combine controllers and business logic in single Action classes. Each Action can be invoked as a controller (`asController`), job (`asJob`), listener (`asListener`), or command (`asCommand`).

## Table of Contents

- [SMS Broadcasting Actions](#sms-broadcasting-actions)
- [Group Management Actions](#group-management-actions)
- [Contact Management Actions](#contact-management-actions)
- [Supporting Services](#supporting-services)
- [Jobs](#jobs)
- [Console Commands](#console-commands)

---

## SMS Broadcasting Actions

### SendToMultipleRecipients

**Endpoint:** `POST /api/send`

**Location:** `app/Actions/SendToMultipleRecipients.php`

**Description:** Send SMS to one or more mobile numbers (comma-delimited or array).

**Request:**
```json
{
  "recipients": "+639171234567,+639189876543",
  "message": "Hello Quezon City!",
  "sender_id": "QUEZON_CITY"
}
```

**Implementation:**
```php
namespace App\Actions;

use App\Jobs\SendSMSJob;
use App\Models\Contact;
use Illuminate\Http\JsonResponse;
use Lorisleiva\Actions\ActionRequest;
use Lorisleiva\Actions\Concerns\AsAction;
use Propaganistas\LaravelPhone\PhoneNumber;

class SendToMultipleRecipients
{
    use AsAction;

    public function handle(array|string $recipients, string $message, ?string $senderId = null): array
    {
        $senderId = $senderId ?? config('sms.default_sender_id', 'TXTCMDR');
        
        // Normalize to array
        $recipientArray = is_string($recipients) 
            ? explode(',', $recipients) 
            : $recipients;
        
        $normalizedRecipients = [];
        $dispatchedCount = 0;
        
        foreach ($recipientArray as $mobile) {
            try {
                // Create PhoneNumber object and Contact
                $phone = new PhoneNumber(trim($mobile), 'PH');
                $contact = Contact::fromPhoneNumber($phone);
                
                // Get E.164 format for SMS sending
                $e164Mobile = $contact->e164_mobile;
                
                SendSMSJob::dispatch($e164Mobile, $message, $senderId);
                
                $normalizedRecipients[] = $e164Mobile;
                $dispatchedCount++;
            } catch (\Exception $e) {
                // Skip invalid numbers
                continue;
            }
        }
        
        return [
            'status' => 'queued',
            'count' => $dispatchedCount,
            'recipients' => $normalizedRecipients,
            'invalid_count' => count($recipientArray) - count($normalizedRecipients),
        ];
    }

    public function rules(): array
    {
        return [
            'recipients' => 'required',
            'message' => 'required|string|max:1600',
            'sender_id' => 'nullable|string|max:11',
        ];
    }

    public function asController(ActionRequest $request): JsonResponse
    {
        $result = $this->handle(
            $request->recipients,
            $request->message,
            $request->sender_id ?? null
        );

        return response()->json($result, 200);
    }
}
```

**Routes:**
```php
// routes/api.php
use App\Actions\SendToMultipleRecipients;

Route::post('/send', SendToMultipleRecipients::class);
```

---

### SendToMultipleGroups

**Endpoint:** `POST /api/groups/send`

**Location:** `app/Actions/SendToMultipleGroups.php`

**Description:** Send SMS to one or more groups (comma-delimited or array).

**Request:**
```json
{
  "groups": "barangay-leaders,health-workers",
  "message": "Please attend the emergency meeting.",
  "sender_id": "QUEZON_CITY"
}
```

**Implementation:**
```php
namespace App\Actions;

use App\Jobs\BroadcastToGroupJob;
use App\Models\Group;
use Illuminate\Http\JsonResponse;
use Lorisleiva\Actions\ActionRequest;
use Lorisleiva\Actions\Concerns\AsAction;

class SendToMultipleGroups
{
    use AsAction;

    public function handle(array|string $groups, string $message, ?string $senderId = null): array
    {
        $senderId = $senderId ?? config('sms.default_sender_id', 'TXTCMDR');
        
        // Normalize to array
        $groupArray = is_string($groups) 
            ? explode(',', $groups) 
            : $groups;
        
        // Trim and filter
        $groupArray = array_filter(
            array_map('trim', $groupArray)
        );
        
        $dispatchedGroups = [];
        $totalContacts = 0;
        
        foreach ($groupArray as $groupIdentifier) {
            $group = Group::where('name', $groupIdentifier)
                ->orWhere('id', $groupIdentifier)
                ->first();
            
            if ($group) {
                BroadcastToGroupJob::dispatch(
                    $group->id,
                    $message,
                    $senderId
                );
                
                $contactCount = $group->contacts()->count();
                $totalContacts += $contactCount;
                
                $dispatchedGroups[] = [
                    'id' => $group->id,
                    'name' => $group->name,
                    'contacts' => $contactCount,
                ];
            }
        }
        
        return [
            'status' => 'queued',
            'groups' => $dispatchedGroups,
            'total_contacts' => $totalContacts,
        ];
    }

    public function rules(): array
    {
        return [
            'groups' => 'required',
            'message' => 'required|string|max:1600',
            'sender_id' => 'nullable|string|max:11',
        ];
    }

    public function asController(ActionRequest $request): JsonResponse
    {
        $result = $this->handle(
            $request->groups,
            $request->message,
            $request->sender_id ?? null
        );

        return response()->json($result, 200);
    }
}
```

**Routes:**
```php
// routes/api.php
use App\Actions\SendToMultipleGroups;

Route::post('/groups/send', SendToMultipleGroups::class);
```

---

## Group Management Actions

### CreateGroup

**Endpoint:** `POST /api/groups`

**Location:** `app/Actions/Groups/CreateGroup.php`

**Request:**
```json
{
  "name": "barangay-leaders",
  "description": "Group of active barangay officials"
}
```

**Implementation:**
```php
namespace App\Actions\Groups;

use App\Models\Group;
use Illuminate\Http\JsonResponse;
use Lorisleiva\Actions\ActionRequest;
use Lorisleiva\Actions\Concerns\AsAction;

class CreateGroup
{
    use AsAction;

    public function handle(string $name, ?string $description = null): Group
    {
        return Group::create([
            'name' => $name,
            'description' => $description,
        ]);
    }

    public function rules(): array
    {
        return [
            'name' => 'required|string|max:255|unique:groups,name',
            'description' => 'nullable|string|max:500',
        ];
    }

    public function asController(ActionRequest $request): JsonResponse
    {
        $group = $this->handle(
            $request->name,
            $request->description ?? null
        );

        return response()->json($group, 201);
    }
}
```

---

### ListGroups

**Endpoint:** `GET /api/groups`

**Location:** `app/Actions/Groups/ListGroups.php`

**Implementation:**
```php
namespace App\Actions\Groups;

use App\Models\Group;
use Illuminate\Http\JsonResponse;
use Lorisleiva\Actions\ActionRequest;
use Lorisleiva\Actions\Concerns\AsAction;

class ListGroups
{
    use AsAction;

    public function handle()
    {
        return Group::withCount('contacts')
            ->orderBy('name')
            ->get();
    }

    public function asController(ActionRequest $request): JsonResponse
    {
        $groups = $this->handle();

        return response()->json($groups, 200);
    }
}
```

---

### GetGroup

**Endpoint:** `GET /api/groups/{id}`

**Location:** `app/Actions/Groups/GetGroup.php`

**Implementation:**
```php
namespace App\Actions\Groups;

use App\Models\Group;
use Illuminate\Http\JsonResponse;
use Lorisleiva\Actions\Concerns\AsAction;

class GetGroup
{
    use AsAction;

    public function handle(int $id): Group
    {
        return Group::with('contacts')
            ->withCount('contacts')
            ->findOrFail($id);
    }

    public function asController(int $id): JsonResponse
    {
        $group = $this->handle($id);

        return response()->json($group, 200);
    }
}
```

---

### UpdateGroup

**Endpoint:** `PUT /api/groups/{id}`

**Location:** `app/Actions/Groups/UpdateGroup.php`

**Request:**
```json
{
  "name": "barangay-leaders",
  "description": "Updated description"
}
```

**Implementation:**
```php
namespace App\Actions\Groups;

use App\Models\Group;
use Illuminate\Http\JsonResponse;
use Lorisleiva\Actions\ActionRequest;
use Lorisleiva\Actions\Concerns\AsAction;

class UpdateGroup
{
    use AsAction;

    public function handle(int $id, array $data): Group
    {
        $group = Group::findOrFail($id);
        $group->update($data);

        return $group->fresh();
    }

    public function rules(): array
    {
        return [
            'name' => 'sometimes|string|max:255|unique:groups,name,' . $this->route('id'),
            'description' => 'nullable|string|max:500',
        ];
    }

    public function asController(ActionRequest $request, int $id): JsonResponse
    {
        $group = $this->handle($id, $request->validated());

        return response()->json($group, 200);
    }
}
```

---

### DeleteGroup

**Endpoint:** `DELETE /api/groups/{id}`

**Location:** `app/Actions/Groups/DeleteGroup.php`

**Implementation:**
```php
namespace App\Actions\Groups;

use App\Models\Group;
use Illuminate\Http\JsonResponse;
use Lorisleiva\Actions\Concerns\AsAction;

class DeleteGroup
{
    use AsAction;

    public function handle(int $id): bool
    {
        $group = Group::findOrFail($id);
        
        return $group->delete();
    }

    public function asController(int $id): JsonResponse
    {
        $this->handle($id);

        return response()->json([
            'message' => 'Group deleted successfully',
        ], 200);
    }
}
```

---

## Contact Management Actions

### AddContactToGroup

**Endpoint:** `POST /api/groups/{id}/contacts`

**Location:** `app/Actions/Contacts/AddContactToGroup.php`

**Request:**
```json
{
  "mobile": "0917 123 4567",
  "name": "Juan Dela Cruz",
  "tags": ["barangay-leader", "volunteer"]
}
```

**Implementation:**
```php
namespace App\Actions\Contacts;

use App\Models\Contact;
use App\Models\Group;
use Illuminate\Http\JsonResponse;
use LBHurtado\Contact\Data\ContactData;
use Lorisleiva\Actions\ActionRequest;
use Lorisleiva\Actions\Concerns\AsAction;
use Propaganistas\LaravelPhone\PhoneNumber;

class AddContactToGroup
{
    use AsAction;

    public function handle(
        int $groupId, 
        string $mobile, 
        ?string $name = null,
        array $tags = []
    ): Contact {
        $group = Group::findOrFail($groupId);
        
        // Create PhoneNumber and Contact using package method
        $phone = new PhoneNumber($mobile, 'PH');
        $contact = Contact::fromPhoneNumber($phone);
        
        // Set name if provided
        if ($name) {
            $contact->setMeta('name', $name);
        }
        
        // Set tags if provided
        if (!empty($tags)) {
            $contact->setTags($tags);
        }
        
        $contact->save();
        
        // Attach to group if not already attached
        if (!$group->contacts->contains($contact)) {
            $group->contacts()->attach($contact->id);
        }
        
        return $contact;
    }

    public function rules(): array
    {
        return [
            'mobile' => 'required|phone:PH',
            'name' => 'nullable|string|max:255',
            'tags' => 'nullable|array',
            'tags.*' => 'string',
        ];
    }

    public function asController(ActionRequest $request, int $id): JsonResponse
    {
        $contact = $this->handle(
            $id,
            $request->mobile,
            $request->name ?? null,
            $request->tags ?? []
        );

        return response()->json(
            ContactData::fromModel($contact),
            201
        );
    }
}
```

---

### UpdateContactInGroup

**Endpoint:** `PUT /api/groups/{group_id}/contacts/{contact_id}`

**Location:** `app/Actions/Contacts/UpdateContactInGroup.php`

**Request:**
```json
{
  "mobile": "0918 765 4321",
  "name": "Juan Dela Cruz",
  "tags": ["barangay-leader", "health-worker"]
}
```

**Implementation:**
```php
namespace App\Actions\Contacts;

use App\Models\Contact;
use App\Models\Group;
use Illuminate\Http\JsonResponse;
use LBHurtado\Contact\Data\ContactData;
use Lorisleiva\Actions\ActionRequest;
use Lorisleiva\Actions\Concerns\AsAction;
use Propaganistas\LaravelPhone\PhoneNumber;

class UpdateContactInGroup
{
    use AsAction;

    public function handle(
        int $groupId, 
        int $contactId, 
        ?string $mobile = null,
        ?string $name = null,
        ?array $tags = null
    ): Contact {
        $group = Group::findOrFail($groupId);
        $contact = Contact::findOrFail($contactId);
        
        // Verify contact belongs to group
        if (!$group->contacts->contains($contact)) {
            abort(404, 'Contact not found in this group');
        }
        
        // Update mobile if provided
        if ($mobile) {
            $phone = new PhoneNumber($mobile, 'PH');
            $contact->mobile = $phone->formatForMobileDialingInCountry('PH');
            $contact->country = 'PH';
        }
        
        // Update name if provided
        if ($name !== null) {
            $contact->setMeta('name', $name);
        }
        
        // Update tags if provided
        if ($tags !== null) {
            $contact->setTags($tags);
        }
        
        $contact->save();

        return $contact->fresh();
    }

    public function rules(): array
    {
        return [
            'mobile' => 'sometimes|phone:PH',
            'name' => 'nullable|string|max:255',
            'tags' => 'nullable|array',
            'tags.*' => 'string',
        ];
    }

    public function asController(ActionRequest $request, int $groupId, int $contactId): JsonResponse
    {
        $contact = $this->handle(
            $groupId,
            $contactId,
            $request->mobile ?? null,
            $request->name ?? null,
            $request->tags ?? null
        );

        return response()->json(
            ContactData::fromModel($contact),
            200
        );
    }
}
```

---

### DeleteContactFromGroup

**Endpoint:** `DELETE /api/groups/{group_id}/contacts/{contact_id}`

**Location:** `app/Actions/Contacts/DeleteContactFromGroup.php`

**Implementation:**
```php
namespace App\Actions\Contacts;

use App\Models\Contact;
use App\Models\Group;
use Illuminate\Http\JsonResponse;
use Lorisleiva\Actions\Concerns\AsAction;

class DeleteContactFromGroup
{
    use AsAction;

    public function handle(int $groupId, int $contactId): bool
    {
        $group = Group::findOrFail($groupId);
        $contact = Contact::findOrFail($contactId);
        
        // Detach from group
        $group->contacts()->detach($contact->id);
        
        // Optionally delete contact if not in any other groups
        if ($contact->groups()->count() === 0) {
            $contact->delete();
        }
        
        return true;
    }

    public function asController(int $groupId, int $contactId): JsonResponse
    {
        $this->handle($groupId, $contactId);

        return response()->json([
            'message' => 'Contact removed from group',
        ], 200);
    }
}
```

---

### ListGroupContacts

**Endpoint:** `GET /api/groups/{id}/contacts`

**Location:** `app/Actions/Contacts/ListGroupContacts.php`

**Implementation:**
```php
namespace App\Actions\Contacts;

use App\Http\Resources\ContactResource;
use App\Models\Group;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;
use Lorisleiva\Actions\Concerns\AsAction;

class ListGroupContacts
{
    use AsAction;

    public function handle(int $groupId)
    {
        $group = Group::findOrFail($groupId);
        
        return $group->contacts()
            ->orderBy('mobile')
            ->get();
    }

    public function asController(int $id): AnonymousResourceCollection
    {
        $contacts = $this->handle($id);

        return ContactResource::collection($contacts);
    }
}
```

---

### ContactResource

**Location:** `app/Http/Resources/ContactResource.php`

**Description:** API resource for Contact model using lbhurtado/contact package features.

**Implementation:**
```php
namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\JsonResource;

class ContactResource extends JsonResource
{
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'mobile' => $this->mobile,
            'mobile_e164' => $this->e164_mobile,
            'country' => $this->country,
            'name' => $this->name,
            'bank_account' => $this->bank_account,
            'bank_code' => $this->bank_code,
            'account_number' => $this->account_number,
            'tags' => $this->getTags(),
            'extra_attributes' => $this->extra_attributes->all(),
            'created_at' => $this->created_at,
            'updated_at' => $this->updated_at,
        ];
    }
}
```

**Example Response:**
```json
{
  "id": 1,
  "mobile": "09171234567",
  "mobile_e164": "+639171234567",
  "country": "PH",
  "name": "Juan Dela Cruz",
  "bank_account": "GXCHPHM2XXX:09171234567",
  "bank_code": "GXCHPHM2XXX",
  "account_number": "09171234567",
  "tags": ["barangay-leader", "volunteer"],
  "extra_attributes": {
    "address": "Quezon City",
    "notes": "Active volunteer"
  },
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:30:00Z"
}
```

> **Note:** The Contact model extends `LBHurtado\Contact\Models\Contact`. See [Contact Package](contact-package.md) for complete documentation on package features including phone normalization, schemaless attributes, and bank account management.

---

## Supporting Services

### SMSService

**Location:** `app/Services/SMSService.php`

**Description:** Wrapper service for SMS facade with logging and error handling.

**Implementation:**
```php
namespace App\Services;

use LBHurtado\SMS\Facades\SMS;
use Illuminate\Support\Facades\Log;

class SMSService
{
    public function send(string $mobile, string $message, string $senderId): bool
    {
        try {
            SMS::channel('engagespark')
                ->from($senderId)
                ->to($mobile)
                ->content($message)
                ->send();
            
            Log::info('SMS sent', [
                'mobile' => $mobile,
                'sender_id' => $senderId,
            ]);
            
            return true;
        } catch (\Exception $e) {
            Log::error('SMS send failed', [
                'mobile' => $mobile,
                'error' => $e->getMessage(),
            ]);
            
            return false;
        }
    }
    
    public function sendWithTopup(string $mobile, string $message, string $senderId, int $amount): bool
    {
        try {
            SMS::channel('engagespark')
                ->from($senderId)
                ->to($mobile)
                ->content($message)
                ->send()
                ->topup($amount);
            
            Log::info('SMS with topup sent', [
                'mobile' => $mobile,
                'sender_id' => $senderId,
                'topup_amount' => $amount,
            ]);
            
            return true;
        } catch (\Exception $e) {
            Log::error('SMS with topup failed', [
                'mobile' => $mobile,
                'error' => $e->getMessage(),
            ]);
            
            return false;
        }
    }
}
```

---

### ContactNormalizationService

**Location:** `app/Services/ContactNormalizationService.php`

**Description:** Normalize phone numbers to +63 format.

**Implementation:**
```php
namespace App\Services;

class ContactNormalizationService
{
    public function normalize(string $mobile): string
    {
        // Remove all non-numeric characters
        $mobile = preg_replace('/[^0-9]/', '', $mobile);
        
        // Handle different formats
        if (str_starts_with($mobile, '63')) {
            return '+' . $mobile;
        }
        
        if (str_starts_with($mobile, '0')) {
            return '+63' . substr($mobile, 1);
        }
        
        // Assume 10-digit local number
        if (strlen($mobile) === 10) {
            return '+63' . $mobile;
        }
        
        return '+' . $mobile;
    }
    
    public function validate(string $mobile): bool
    {
        $normalized = $this->normalize($mobile);
        
        // Must be +63 followed by 10 digits
        return preg_match('/^\+63[0-9]{10}$/', $normalized) === 1;
    }
}
```

---

## Jobs

### SendSMSJob

**Location:** `app/Jobs/SendSMSJob.php`

**Description:** Queued job for sending individual SMS. See [SMS Integration](sms-integration.md) for full implementation.

---

### BroadcastToGroupJob

**Location:** `app/Jobs/BroadcastToGroupJob.php`

**Description:** Queued job for broadcasting to all contacts in a group. See [SMS Integration](sms-integration.md) for full implementation.

---

### ContactImportJob

**Location:** `app/Jobs/ContactImportJob.php`

**Description:** Parses and imports contacts from CSV/Excel files.

**Supported Formats:**
- CSV (`,` or `;` delimiter)
- Excel (.xlsx, .xls)

**Expected Columns:**
- `mobile` (required) - Phone number
- `name` (optional) - Contact name
- `tags` (optional) - Comma-separated tags

---

## Console Commands

### SendScheduledBroadcasts

**Location:** `app/Console/Commands/SendScheduledBroadcasts.php`

**Schedule:** Runs every minute

**Description:** Processes scheduled broadcasts.

**Logic:**
1. Query campaigns with `scheduled_at <= now()` and `status = 'pending'`
2. Dispatch `BroadcastToGroupJob` for each
3. Update campaign status to `sending`

**Implementation:**
```php
namespace App\Console\Commands;

use App\Jobs\BroadcastToGroupJob;
use App\Models\Campaign;
use Illuminate\Console\Command;

class SendScheduledBroadcasts extends Command
{
    protected $signature = 'broadcasts:send';
    protected $description = 'Send scheduled broadcasts';

    public function handle()
    {
        $campaigns = Campaign::where('status', 'pending')
            ->where('scheduled_at', '<=', now())
            ->get();

        foreach ($campaigns as $campaign) {
            foreach ($campaign->groups as $group) {
                BroadcastToGroupJob::dispatch(
                    $group->id,
                    $campaign->message,
                    $campaign->sender_id
                );
            }
            
            $campaign->update(['status' => 'sending']);
        }

        $this->info("Dispatched {$campaigns->count()} scheduled broadcasts");
    }
}
```

**Register in `app/Console/Kernel.php`:**
```php
protected function schedule(Schedule $schedule)
{
    $schedule->command('broadcasts:send')->everyMinute();
}
```

---

## Routes Summary

```php
// routes/api.php

use App\Actions\SendToMultipleRecipients;
use App\Actions\SendToMultipleGroups;
use App\Actions\Groups\{CreateGroup, ListGroups, GetGroup, UpdateGroup, DeleteGroup};
use App\Actions\Contacts\{AddContactToGroup, UpdateContactInGroup, DeleteContactFromGroup, ListGroupContacts};

// SMS Broadcasting
Route::post('/send', SendToMultipleRecipients::class);
Route::post('/groups/send', SendToMultipleGroups::class);

// Group Management
Route::get('/groups', ListGroups::class);
Route::post('/groups', CreateGroup::class);
Route::get('/groups/{id}', GetGroup::class);
Route::put('/groups/{id}', UpdateGroup::class);
Route::delete('/groups/{id}', DeleteGroup::class);

// Contact Management
Route::get('/groups/{id}/contacts', ListGroupContacts::class);
Route::post('/groups/{id}/contacts', AddContactToGroup::class);
Route::put('/groups/{group_id}/contacts/{contact_id}', UpdateContactInGroup::class);
Route::delete('/groups/{group_id}/contacts/{contact_id}', DeleteContactFromGroup::class);
```

---

## Related Documentation

- [API Documentation](api-documentation.md) - HTTP API endpoints
- [SMS Integration](sms-integration.md) - SMS packages and drivers
- [Package Architecture](package-architecture.md) - Detailed package structure
- [Development Plan](development-plan.md) - Implementation roadmap
