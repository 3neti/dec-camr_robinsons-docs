# Contact Package Integration

Text Commander uses the **lbhurtado/contact** package for robust contact management with built-in phone number normalization and schemaless attributes support.

## Package Overview

**Package:** `lbhurtado/contact`  
**Author:** Lester Hurtado  
**Features:**
- Phone number normalization (via `propaganistas/laravel-phone`)
- Bank account management
- Schemaless attributes (via `spatie/laravel-schemaless-attributes`)
- Meta data storage
- Data Transfer Objects (via `spatie/laravel-data`)

## Installation

```bash
composer require lbhurtado/contact
```

Publish and run migrations:

```bash
php artisan vendor:publish --provider="LBHurtado\Contact\ContactServiceProvider"
php artisan migrate
```

## Configuration

**File:** `config/contact.php`

```php
return [
    'default' => [
        'country' => 'PH',
        'bank_code' => 'GXCHPHM2XXX'  // Default bank code
    ]
];
```

## Database Schema

The package provides a `contacts` table with:

```php
Schema::create('contacts', function (Blueprint $table) {
    $table->id();
    $table->string('mobile');
    $table->string('country');
    $table->string('bank_account')->nullable();
    $table->schemalessAttributes('extra_attributes'); // Via spatie/laravel-schemaless-attributes
    $table->timestamps();
});
```

## Contact Model

**Package Model:** `LBHurtado\Contact\Models\Contact`

### Key Features

1. **Automatic Phone Normalization** (via `HasMobile` trait)
2. **Bank Account Management** (via `HasBankAccount` trait)
3. **Schemaless Attributes** (via `HasAdditionalAttributes` trait)
4. **Meta Data** (via `HasMeta` trait)

### Model Structure

```php
namespace LBHurtado\Contact\Models;

use LBHurtado\Contact\Traits\{HasAdditionalAttributes, HasMeta, HasMobile, HasBankAccount};
use LBHurtado\Contact\Contracts\Bankable;
use Illuminate\Database\Eloquent\Model;

class Contact extends Model implements Bankable
{
    use HasAdditionalAttributes;
    use HasBankAccount;
    use HasMobile;
    use HasMeta;

    protected $fillable = [
        'mobile',
        'country',
        'bank_account',
    ];

    protected $appends = [
        'name'  // Pulled from meta or extra_attributes
    ];
}
```

## Usage Examples

### Creating Contacts

```php
use LBHurtado\Contact\Models\Contact;

// Basic creation
$contact = Contact::create([
    'mobile' => '0917 123 4567',
    'country' => 'PH',
]);

// Auto-formats to: 09171234567 (formatForMobileDialingInCountry)
echo $contact->mobile; // 09171234567

// With bank account
$contact = Contact::create([
    'mobile' => '+63 917 123 4567',
    'country' => 'PH',
    'bank_account' => 'BDO:1234567890',
]);

// Bank account auto-generated if not provided
// Format: {BANK_CODE}:{MOBILE}
// Example: GXCHPHM2XXX:09171234567
```

### Creating from PhoneNumber Object

```php
use Propaganistas\LaravelPhone\PhoneNumber;

$phone = new PhoneNumber('0917 123 4567', 'PH');

$contact = Contact::fromPhoneNumber($phone);
// Auto-extracts mobile and country, creates or finds existing contact
```

### Accessing Phone Formats

```php
$contact = Contact::find(1);

// Stored format (formatForMobileDialingInCountry)
echo $contact->mobile; // 09171234567

// Get different formats using propaganistas/laravel-phone
$phone = phone($contact->mobile, $contact->country);
echo $phone->formatE164();          // +639171234567
echo $phone->formatInternational(); // +63 917 123 4567
echo $phone->formatNational();      // 0917 123 4567
```

### Bank Account Management

```php
$contact = Contact::find(1);

// Get bank code
echo $contact->bank_code; // BDO or GXCHPHM2XXX

// Get account number
echo $contact->account_number; // 1234567890 or 09171234567

// Full bank account
echo $contact->bank_account; // BDO:1234567890
```

### Schemaless Attributes (Extra Attributes)

```php
// Set extra attributes
$contact->extra_attributes->set('tags', ['barangay-leader', 'health-worker']);
$contact->extra_attributes->set('address', 'Quezon City, Metro Manila');
$contact->extra_attributes->set('notes', 'Active volunteer');
$contact->save();

// Get extra attributes
$tags = $contact->extra_attributes->get('tags'); // ['barangay-leader', 'health-worker']
$address = $contact->extra_attributes->get('address'); // 'Quezon City, Metro Manila'

// Check if attribute exists
if ($contact->extra_attributes->has('notes')) {
    echo $contact->extra_attributes->get('notes');
}

// Get all extra attributes
$all = $contact->extra_attributes->all();
/*
[
    'tags' => ['barangay-leader', 'health-worker'],
    'address' => 'Quezon City, Metro Manila',
    'notes' => 'Active volunteer',
]
*/
```

### Meta Data (Name, etc.)

```php
// Set name via meta
$contact->setMeta('name', 'Juan Dela Cruz');
$contact->save();

// Get name (via appended attribute)
echo $contact->name; // Juan Dela Cruz

// Direct meta access
$contact->setMeta('email', 'juan@example.com');
$email = $contact->getMeta('email');

// Multiple meta values
$contact->setMeta('role', 'Volunteer');
$contact->setMeta('status', 'Active');
```

## ContactData DTO

The package includes a Data Transfer Object for type-safe contact data:

```php
use LBHurtado\Contact\Data\ContactData;
use LBHurtado\Contact\Models\Contact;

// Create DTO from model
$contact = Contact::find(1);
$data = ContactData::fromModel($contact);

echo $data->mobile;         // 09171234567
echo $data->country;        // PH
echo $data->bank_account;   // BDO:1234567890
echo $data->name;           // Juan Dela Cruz

// Use in API responses
return response()->json($data);
```

## Integration with Text Commander

### Extended Contact Model

Create an application-specific Contact model extending the package model:

**File:** `app/Models/Contact.php`

```php
namespace App\Models;

use LBHurtado\Contact\Models\Contact as BaseContact;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

class Contact extends BaseContact
{
    /**
     * Groups this contact belongs to
     */
    public function groups(): BelongsToMany
    {
        return $this->belongsToMany(Group::class, 'contact_group')
            ->withTimestamps();
    }

    /**
     * SMS logs for this contact
     */
    public function smsLogs()
    {
        return $this->hasMany(SMSLog::class, 'mobile', 'mobile');
    }

    /**
     * Get formatted mobile for SMS sending (E.164)
     */
    public function getE164MobileAttribute(): string
    {
        return phone($this->mobile, $this->country)->formatE164();
    }

    /**
     * Set tags helper
     */
    public function setTags(array $tags): self
    {
        $this->extra_attributes->set('tags', $tags);
        return $this;
    }

    /**
     * Get tags helper
     */
    public function getTags(): array
    {
        return $this->extra_attributes->get('tags', []);
    }

    /**
     * Check if contact has tag
     */
    public function hasTag(string $tag): bool
    {
        return in_array($tag, $this->getTags());
    }
}
```

### Update Actions to Use Package

**SendToMultipleRecipients Action:**

```php
namespace App\Actions;

use App\Jobs\SendSMSJob;
use App\Models\Contact;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
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

    public function asController(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'recipients' => 'required',
            'message' => 'required|string|max:1600',
            'sender_id' => 'nullable|string|max:11',
        ]);

        $result = $this->handle(
            $validated['recipients'],
            $validated['message'],
            $validated['sender_id'] ?? null
        );

        return response()->json($result, 200);
    }
}
```

**AddContactToGroup Action:**

```php
namespace App\Actions\Contacts;

use App\Models\Contact;
use App\Models\Group;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
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
        
        // Create PhoneNumber and Contact
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

    public function asController(Request $request, int $id): JsonResponse
    {
        $validated = $request->validate([
            'mobile' => 'required|phone:PH',
            'name' => 'nullable|string|max:255',
            'tags' => 'nullable|array',
            'tags.*' => 'string',
        ]);

        $contact = $this->handle(
            $id,
            $validated['mobile'],
            $validated['name'] ?? null,
            $validated['tags'] ?? []
        );

        return response()->json(
            ContactData::fromModel($contact),
            201
        );
    }
}
```

### Contact Resource

**File:** `app/Http/Resources/ContactResource.php`

```php
namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\JsonResource;
use LBHurtado\Contact\Data\ContactData;

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

## Advanced Features

### Querying by Tags

```php
// Find all contacts with specific tag
$contacts = Contact::all()->filter(function ($contact) {
    return $contact->hasTag('barangay-leader');
});

// Find contacts with any of multiple tags
$tags = ['barangay-leader', 'health-worker'];
$contacts = Contact::all()->filter(function ($contact) use ($tags) {
    return !empty(array_intersect($contact->getTags(), $tags));
});
```

### Bulk Contact Import

```php
use App\Models\Contact;
use Propaganistas\LaravelPhone\PhoneNumber;

class ImportContactsJob
{
    public function handle(array $data)
    {
        foreach ($data as $row) {
            try {
                $phone = new PhoneNumber($row['mobile'], $row['country'] ?? 'PH');
                $contact = Contact::fromPhoneNumber($phone);
                
                if (isset($row['name'])) {
                    $contact->setMeta('name', $row['name']);
                }
                
                if (isset($row['tags'])) {
                    $contact->setTags(
                        is_array($row['tags']) 
                            ? $row['tags'] 
                            : explode(',', $row['tags'])
                    );
                }
                
                if (isset($row['extra'])) {
                    foreach ($row['extra'] as $key => $value) {
                        $contact->extra_attributes->set($key, $value);
                    }
                }
                
                $contact->save();
            } catch (\Exception $e) {
                \Log::error('Contact import failed', [
                    'row' => $row,
                    'error' => $e->getMessage()
                ]);
            }
        }
    }
}
```

### Search Across Attributes

```php
// Search by mobile, name, or tags
$search = '0917';

$contacts = Contact::where('mobile', 'like', "%{$search}%")
    ->orWhere(function($query) use ($search) {
        // Search in extra_attributes for name
        $query->whereRaw("extra_attributes->>'name' LIKE ?", ["%{$search}%"]);
    })
    ->get();
```

## Benefits for Text Commander

1. **✅ Automatic Phone Normalization**
   - No manual normalization needed
   - Uses `formatForMobileDialingInCountry()` by default
   - Easy conversion to E.164 for SMS sending

2. **✅ Flexible Attributes**
   - Store tags, notes, addresses without schema changes
   - Perfect for varying contact information
   - JSON-based storage via `spatie/laravel-schemaless-attributes`

3. **✅ Bank Account Support**
   - Built-in for payment/incentive features
   - Auto-generates if not provided
   - Easy to parse and validate

4. **✅ Type-Safe DTOs**
   - `ContactData` for API responses
   - IDE autocompletion
   - Validation built-in

5. **✅ Country Support**
   - Multi-country contact management
   - Per-contact country tracking
   - Configurable defaults

## Testing

```php
namespace Tests\Feature;

use Tests\TestCase;
use App\Models\Contact;
use Propaganistas\LaravelPhone\PhoneNumber;

class ContactTest extends TestCase
{
    public function test_creates_contact_from_phone_number()
    {
        $phone = new PhoneNumber('0917 123 4567', 'PH');
        $contact = Contact::fromPhoneNumber($phone);

        $this->assertInstanceOf(Contact::class, $contact);
        $this->assertEquals('09171234567', $contact->mobile);
        $this->assertEquals('PH', $contact->country);
    }

    public function test_contact_has_e164_format()
    {
        $contact = Contact::create([
            'mobile' => '0917 123 4567',
            'country' => 'PH',
        ]);

        $this->assertEquals('+639171234567', $contact->e164_mobile);
    }

    public function test_can_set_and_get_tags()
    {
        $contact = Contact::factory()->create();
        $contact->setTags(['tag1', 'tag2']);
        $contact->save();

        $this->assertEquals(['tag1', 'tag2'], $contact->getTags());
        $this->assertTrue($contact->hasTag('tag1'));
        $this->assertFalse($contact->hasTag('tag3'));
    }

    public function test_auto_generates_bank_account()
    {
        $contact = Contact::create([
            'mobile' => '0917 123 4567',
            'country' => 'PH',
        ]);

        $this->assertEquals('GXCHPHM2XXX:09171234567', $contact->bank_account);
    }
}
```

## Migration from Simple Contact Model

If you already have a simple Contact model, here's how to migrate:

### 1. Backup Data

```php
$contacts = \App\Models\Contact::all()->map(function($contact) {
    return [
        'id' => $contact->id,
        'mobile' => $contact->mobile,
        'name' => $contact->name,
    ];
})->toArray();
```

### 2. Update Model

Change your Contact model to extend the package model:

```php
namespace App\Models;

use LBHurtado\Contact\Models\Contact as BaseContact;

class Contact extends BaseContact
{
    // Your existing relationships and methods
}
```

### 3. Migrate Data

```php
foreach ($contacts as $data) {
    $contact = Contact::find($data['id']);
    if ($contact && isset($data['name'])) {
        $contact->setMeta('name', $data['name']);
        $contact->save();
    }
}
```

## Related Documentation

- [Phone Normalization](phone-normalization.md) - Phone number handling
- [Backend Services](backend-services.md) - Actions and Controllers
- [SMS Integration](sms-integration.md) - SMS packages and drivers
- [Interfaces & DTOs](interfaces-and-dtos.md) - Data structures
- [API Documentation](api-documentation.md) - HTTP API endpoints
