# Phone Number Normalization

Text Commander uses [propaganistas/laravel-phone](https://github.com/Propaganistas/Laravel-Phone) for robust phone number handling, powered by Google's libphonenumber library.

## Installation

```bash
composer require propaganistas/laravel-phone
```

The Service Provider is auto-discovered by Laravel.

## Configuration

Add translation in `resources/lang/en/validation.php`:

```php
'phone' => 'The :attribute field must be a valid phone number.',
```

## PhoneNormalizationService

**Location:** `app/Services/PhoneNormalizationService.php`

This service provides centralized phone number normalization and validation for Text Commander.

### Implementation

```php
namespace App\Services;

use Propaganistas\LaravelPhone\PhoneNumber;
use Propaganistas\LaravelPhone\Exceptions\NumberParseException;

class PhoneNormalizationService
{
    /**
     * Default country code for Philippine numbers
     */
    private const DEFAULT_COUNTRY = 'PH';

    /**
     * Normalize phone number to E.164 format (+63XXXXXXXXXX)
     *
     * @param string $mobile Raw phone number input
     * @param string $country ISO 3166-1 alpha-2 country code
     * @return string Normalized phone number in E.164 format
     * @throws NumberParseException
     */
    public function normalize(string $mobile, string $country = self::DEFAULT_COUNTRY): string
    {
        try {
            $phone = new PhoneNumber($mobile, $country);
            return $phone->formatE164();
        } catch (NumberParseException $e) {
            throw new NumberParseException(
                "Invalid phone number: {$mobile}. " . $e->getMessage()
            );
        }
    }

    /**
     * Normalize to E.164 format, returns null on failure
     *
     * @param string $mobile Raw phone number input
     * @param string $country ISO 3166-1 alpha-2 country code
     * @return string|null Normalized phone number or null
     */
    public function normalizeOrNull(string $mobile, string $country = self::DEFAULT_COUNTRY): ?string
    {
        try {
            return $this->normalize($mobile, $country);
        } catch (NumberParseException $e) {
            return null;
        }
    }

    /**
     * Normalize array of phone numbers
     *
     * @param array $mobiles Array of raw phone numbers
     * @param string $country ISO 3166-1 alpha-2 country code
     * @return array Array of normalized phone numbers
     */
    public function normalizeMany(array $mobiles, string $country = self::DEFAULT_COUNTRY): array
    {
        return array_map(
            fn($mobile) => $this->normalize($mobile, $country),
            $mobiles
        );
    }

    /**
     * Normalize array of phone numbers, filtering out invalid ones
     *
     * @param array $mobiles Array of raw phone numbers
     * @param string $country ISO 3166-1 alpha-2 country code
     * @return array Array of valid normalized phone numbers
     */
    public function normalizeManyOrFilter(array $mobiles, string $country = self::DEFAULT_COUNTRY): array
    {
        return array_filter(
            array_map(
                fn($mobile) => $this->normalizeOrNull($mobile, $country),
                $mobiles
            )
        );
    }

    /**
     * Validate if phone number is valid
     *
     * @param string $mobile Raw phone number input
     * @param string $country ISO 3166-1 alpha-2 country code
     * @return bool True if valid, false otherwise
     */
    public function isValid(string $mobile, string $country = self::DEFAULT_COUNTRY): bool
    {
        try {
            new PhoneNumber($mobile, $country);
            return true;
        } catch (NumberParseException $e) {
            return false;
        }
    }

    /**
     * Check if phone number is mobile type
     *
     * @param string $mobile Raw phone number input
     * @param string $country ISO 3166-1 alpha-2 country code
     * @return bool True if mobile, false otherwise
     */
    public function isMobile(string $mobile, string $country = self::DEFAULT_COUNTRY): bool
    {
        try {
            $phone = new PhoneNumber($mobile, $country);
            return $phone->isOfType('mobile');
        } catch (NumberParseException $e) {
            return false;
        }
    }

    /**
     * Get phone number information
     *
     * @param string $mobile Raw phone number input
     * @param string $country ISO 3166-1 alpha-2 country code
     * @return array Phone number details
     */
    public function getInfo(string $mobile, string $country = self::DEFAULT_COUNTRY): array
    {
        try {
            $phone = new PhoneNumber($mobile, $country);

            return [
                'e164' => $phone->formatE164(),
                'international' => $phone->formatInternational(),
                'national' => $phone->formatNational(),
                'rfc3966' => $phone->formatRFC3966(),
                'country' => $phone->getCountry(),
                'type' => $phone->getType(),
                'is_mobile' => $phone->isOfType('mobile'),
                'is_fixed_line' => $phone->isOfType('fixed_line'),
            ];
        } catch (NumberParseException $e) {
            return [];
        }
    }

    /**
     * Format phone number for display
     *
     * @param string $mobile Phone number (preferably E.164)
     * @param string $format Format type: 'international', 'national', 'e164', 'rfc3966'
     * @return string Formatted phone number
     */
    public function format(string $mobile, string $format = 'international'): string
    {
        try {
            $phone = phone($mobile);

            return match ($format) {
                'international' => $phone->formatInternational(),
                'national' => $phone->formatNational(),
                'e164' => $phone->formatE164(),
                'rfc3966' => $phone->formatRFC3966(),
                default => $phone->formatInternational(),
            };
        } catch (NumberParseException $e) {
            return $mobile; // Return as-is if invalid
        }
    }
}
```

## Usage Examples

### Basic Normalization

```php
use App\Services\PhoneNormalizationService;

$service = app(PhoneNormalizationService::class);

// Philippine numbers
$service->normalize('0917 123 4567');          // +639171234567
$service->normalize('917-123-4567');           // +639171234567
$service->normalize('09171234567');            // +639171234567
$service->normalize('+63 917 123 4567');       // +639171234567

// Already normalized
$service->normalize('+639171234567');          // +639171234567

// International numbers
$service->normalize('(202) 555-0123', 'US');   // +12025550123
```

### Safe Normalization

```php
// Returns null instead of throwing exception
$result = $service->normalizeOrNull('invalid');  // null
$result = $service->normalizeOrNull('0917 123 4567'); // +639171234567
```

### Batch Normalization

```php
$mobiles = ['0917 123 4567', '0918 765 4321', '0919 111 2222'];

// Normalize all (throws on invalid)
$normalized = $service->normalizeMany($mobiles);
// ['+639171234567', '+639187654321', '+639191112222']

// Filter out invalid numbers
$mixedInput = ['0917 123 4567', 'invalid', '0918 765 4321'];
$normalized = $service->normalizeManyOrFilter($mixedInput);
// ['+639171234567', '+639187654321']
```

### Validation

```php
$service->isValid('0917 123 4567');      // true
$service->isValid('invalid');            // false
$service->isValid('+63 917 123 4567');   // true

$service->isMobile('0917 123 4567');     // true
$service->isMobile('02 8123 4567');      // false (landline)
```

### Phone Information

```php
$info = $service->getInfo('0917 123 4567');

/*
[
    'e164' => '+639171234567',
    'international' => '+63 917 123 4567',
    'national' => '0917 123 4567',
    'rfc3966' => 'tel:+63-917-123-4567',
    'country' => 'PH',
    'type' => 'MOBILE',
    'is_mobile' => true,
    'is_fixed_line' => false,
]
*/
```

### Formatting

```php
$normalized = '+639171234567';

$service->format($normalized, 'international'); // +63 917 123 4567
$service->format($normalized, 'national');      // 0917 123 4567
$service->format($normalized, 'e164');          // +639171234567
$service->format($normalized, 'rfc3966');       // tel:+63-917-123-4567
```

## Integration with Actions

### SendToMultipleRecipients Action

Update the action to normalize phone numbers before dispatching:

```php
namespace App\Actions;

use App\Jobs\SendSMSJob;
use App\Services\PhoneNormalizationService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Lorisleiva\Actions\Concerns\AsAction;

class SendToMultipleRecipients
{
    use AsAction;

    public function __construct(
        private PhoneNormalizationService $phoneService
    ) {}

    public function handle(array|string $recipients, string $message, ?string $senderId = null): array
    {
        $senderId = $senderId ?? config('sms.default_sender_id', 'TXTCMDR');
        
        // Normalize to array
        $recipientArray = is_string($recipients) 
            ? explode(',', $recipients) 
            : $recipients;
        
        // Trim whitespace
        $recipientArray = array_map('trim', $recipientArray);
        
        // Normalize phone numbers and filter invalid ones
        $normalizedRecipients = $this->phoneService->normalizeManyOrFilter($recipientArray);
        
        $dispatchedCount = 0;
        
        foreach ($normalizedRecipients as $mobile) {
            SendSMSJob::dispatch($mobile, $message, $senderId);
            $dispatchedCount++;
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

### AddContactToGroup Action

```php
namespace App\Actions\Contacts;

use App\Models\Contact;
use App\Models\Group;
use App\Services\PhoneNormalizationService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Lorisleiva\Actions\Concerns\AsAction;

class AddContactToGroup
{
    use AsAction;

    public function __construct(
        private PhoneNormalizationService $phoneService
    ) {}

    public function handle(int $groupId, string $mobile, ?string $name = null): Contact
    {
        $group = Group::findOrFail($groupId);
        
        // Normalize phone number
        $normalizedMobile = $this->phoneService->normalize($mobile);
        
        // Find or create contact
        $contact = Contact::firstOrCreate(
            ['mobile' => $normalizedMobile],
            ['name' => $name]
        );
        
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
        ]);

        $contact = $this->handle(
            $id,
            $validated['mobile'],
            $validated['name'] ?? null
        );

        return response()->json($contact, 201);
    }
}
```

## Laravel Validation Rules

Use the built-in `phone` validation rule from the package:

```php
// Validate Philippine mobile numbers
'mobile' => 'required|phone:PH',

// Validate mobile type only
'mobile' => 'required|phone:PH,mobile',

// Validate any international number
'mobile' => 'required|phone:INTERNATIONAL',

// Using Rule class
use Propaganistas\LaravelPhone\Rules\Phone;

'mobile' => ['required', (new Phone)->country('PH')->type('mobile')],
```

## Model Attribute Casting

Use automatic casting in Eloquent models:

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Propaganistas\LaravelPhone\Casts\E164PhoneNumberCast;

class Contact extends Model
{
    protected $casts = [
        'mobile' => E164PhoneNumberCast::class . ':PH',
    ];

    // Now $contact->mobile is always a PhoneNumber object
}
```

**Usage:**

```php
$contact = new Contact();
$contact->mobile = '0917 123 4567';  // Stored as +639171234567

echo $contact->mobile->formatInternational(); // +63 917 123 4567
echo $contact->mobile->formatNational();      // 0917 123 4567
```

## Helper Function

The package provides a global `phone()` helper:

```php
// Create PhoneNumber object
$phone = phone('0917 123 4567', 'PH');

// Format directly
$formatted = phone('0917 123 4567', 'PH')->formatE164();        // +639171234567
$formatted = phone('0917 123 4567', 'PH')->formatInternational(); // +63 917 123 4567

// Pass format as third parameter
$formatted = phone('0917 123 4567', 'PH', \libphonenumber\PhoneNumberFormat::E164);
```

## Database Schema Recommendations

### Approach 1: E.164 Only (Simplest)

Store normalized E.164 format:

```php
Schema::create('contacts', function (Blueprint $table) {
    $table->id();
    $table->string('mobile')->unique(); // +63XXXXXXXXXX
    $table->string('name')->nullable();
    $table->timestamps();
});
```

### Approach 2: Raw Input + Country (Best UX)

Preserve user input:

```php
Schema::create('contacts', function (Blueprint $table) {
    $table->id();
    $table->string('mobile');          // User's original input
    $table->string('mobile_country', 2)->default('PH');
    $table->string('name')->nullable();
    $table->timestamps();
});
```

### Approach 3: Multi-format (Best Search)

Support advanced searching:

```php
Schema::create('contacts', function (Blueprint $table) {
    $table->id();
    $table->string('mobile');                    // Original input
    $table->string('mobile_country', 2)->default('PH');
    $table->string('mobile_normalized');         // Digits only: 9171234567
    $table->string('mobile_national');           // National format digits: 09171234567
    $table->string('mobile_e164')->unique();     // E.164: +639171234567
    $table->string('name')->nullable();
    $table->timestamps();
    
    $table->index('mobile_normalized');
    $table->index('mobile_national');
});
```

**Model Observer:**

```php
namespace App\Observers;

use App\Models\Contact;
use App\Services\PhoneNormalizationService;

class ContactObserver
{
    public function __construct(
        private PhoneNormalizationService $phoneService
    ) {}

    public function saving(Contact $contact)
    {
        if ($contact->isDirty('mobile') && $contact->mobile) {
            $phone = phone($contact->mobile, $contact->mobile_country ?? 'PH');
            
            $contact->mobile_normalized = preg_replace('/[^0-9]/', '', $contact->mobile);
            $contact->mobile_national = preg_replace('/[^0-9]/', '', $phone->formatNational());
            $contact->mobile_e164 = $phone->formatE164();
        }
    }
}
```

## Testing

```php
namespace Tests\Unit\Services;

use Tests\TestCase;
use App\Services\PhoneNormalizationService;
use Propaganistas\LaravelPhone\Exceptions\NumberParseException;

class PhoneNormalizationServiceTest extends TestCase
{
    private PhoneNormalizationService $service;

    protected function setUp(): void
    {
        parent::setUp();
        $this->service = new PhoneNormalizationService();
    }

    public function test_normalizes_philippine_mobile_numbers()
    {
        $this->assertEquals('+639171234567', $this->service->normalize('0917 123 4567'));
        $this->assertEquals('+639171234567', $this->service->normalize('917-123-4567'));
        $this->assertEquals('+639171234567', $this->service->normalize('09171234567'));
        $this->assertEquals('+639171234567', $this->service->normalize('+63 917 123 4567'));
    }

    public function test_throws_exception_for_invalid_number()
    {
        $this->expectException(NumberParseException::class);
        $this->service->normalize('invalid');
    }

    public function test_returns_null_for_invalid_number_when_safe()
    {
        $this->assertNull($this->service->normalizeOrNull('invalid'));
    }

    public function test_validates_phone_numbers()
    {
        $this->assertTrue($this->service->isValid('0917 123 4567'));
        $this->assertFalse($this->service->isValid('invalid'));
    }

    public function test_checks_if_mobile_type()
    {
        $this->assertTrue($this->service->isMobile('0917 123 4567'));
        $this->assertFalse($this->service->isMobile('02 8123 4567')); // Landline
    }

    public function test_normalizes_batch_of_numbers()
    {
        $input = ['0917 123 4567', '0918 765 4321'];
        $expected = ['+639171234567', '+639187654321'];
        
        $this->assertEquals($expected, $this->service->normalizeMany($input));
    }

    public function test_filters_invalid_numbers_in_batch()
    {
        $input = ['0917 123 4567', 'invalid', '0918 765 4321'];
        $result = $this->service->normalizeManyOrFilter($input);
        
        $this->assertCount(2, $result);
        $this->assertContains('+639171234567', $result);
        $this->assertContains('+639187654321', $result);
    }
}
```

## Related Documentation

- [Backend Services](backend-services.md) - Actions and Controllers
- [SMS Integration](sms-integration.md) - SMS packages and drivers
- [Interfaces & DTOs](interfaces-and-dtos.md) - Data structures and validation
- [API Documentation](api-documentation.md) - HTTP API endpoints
