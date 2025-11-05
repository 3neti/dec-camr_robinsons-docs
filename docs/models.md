# Models

**ORM:** Laravel Eloquent  
**Namespace:** `App\Models`

Text Commander uses Eloquent models with relationships, scopes, and static helper methods following Laravel best practices.

## Core Models

### User Model

**Path:** `app/Models/User.php`  
**Table:** `users`  
**Authenticatable:** Yes (Laravel Sanctum)

```php
namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens;

    protected $fillable = [
        'name',
        'email',
        'password',
    ];

    protected $hidden = [
        'password',
        'remember_token',
    ];

    protected $casts = [
        'email_verified_at' => 'datetime',
        'password' => 'hashed',
    ];

    // Relationships
    public function groups()
    {
        return $this->hasMany(Group::class);
    }
}
```

**See:** [Controller Scaffolding - Authentication](controller-scaffolding.md)

---

### Contact Model

**Path:** `app/Models/Contact.php`  
**Table:** `contacts`  
**Extends:** `Lbhurtado\Contact\Models\Contact`

```php
namespace App\Models;

use Lbhurtado\Contact\Models\Contact as BaseContact;

class Contact extends BaseContact
{
    protected $fillable = [
        'name',
        'mobile',
        'email',
        'extra',
    ];

    protected $casts = [
        'extra' => 'array',
    ];

    // Relationships
    public function groups()
    {
        return $this->belongsToMany(Group::class)
            ->withTimestamps();
    }

    // Scopes
    public function scopeSearch($query, $search)
    {
        return $query->where('name', 'like', "%{$search}%")
            ->orWhere('mobile', 'like', "%{$search}%");
    }
}
```

**Features:**
- Phone normalization (inherited from package)
- Group relationships
- Search scope
- JSON extra field for additional data

**See:** [Contact Package](contact-package.md)

---

### Group Model

**Path:** `app/Models/Group.php`  
**Table:** `groups`

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Group extends Model
{
    protected $fillable = [
        'name',
        'description',
        'user_id',
    ];

    // Relationships
    public function user()
    {
        return $this->belongsTo(User::class);
    }

    public function contacts()
    {
        return $this->belongsToMany(Contact::class)
            ->withTimestamps();
    }

    // Accessors
    public function getContactCountAttribute()
    {
        return $this->contacts()->count();
    }
}
```

**See:** [Backend Services - Group Management](backend-services.md)

---

### ScheduledMessage Model

**Path:** `app/Models/ScheduledMessage.php`  
**Table:** `scheduled_messages`

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Carbon\Carbon;

class ScheduledMessage extends Model
{
    protected $fillable = [
        'message',
        'recipients',
        'group_ids',
        'sender_id',
        'scheduled_at',
        'status',
    ];

    protected $casts = [
        'recipients' => 'array',
        'group_ids' => 'array',
        'scheduled_at' => 'datetime',
    ];

    // Scopes
    public function scopePending($query)
    {
        return $query->where('status', 'pending');
    }

    public function scopeDue($query)
    {
        return $query->where('scheduled_at', '<=', now())
            ->where('status', 'pending');
    }

    // Status helpers
    public function markAsProcessing()
    {
        $this->update(['status' => 'processing']);
    }

    public function markAsSent()
    {
        $this->update(['status' => 'sent']);
    }

    public function markAsFailed()
    {
        $this->update(['status' => 'failed']);
    }
}
```

**Features:**
- JSON recipients and group_ids
- Status management methods
- Query scopes for job processing

**See:** [Scheduled Messaging](scheduled-messaging.md)

---

### BlacklistedNumber Model

**Path:** `app/Models/BlacklistedNumber.php`  
**Table:** `blacklisted_numbers`

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class BlacklistedNumber extends Model
{
    protected $fillable = [
        'mobile',
        'reason',
        'notes',
        'added_by',
    ];

    // Static helpers
    public static function isBlacklisted(string $mobile): bool
    {
        return static::where('mobile', $mobile)->exists();
    }

    public static function add(string $mobile, string $reason, ?string $notes = null, string $addedBy = 'system'): self
    {
        return static::updateOrCreate(
            ['mobile' => $mobile],
            [
                'reason' => $reason,
                'notes' => $notes,
                'added_by' => $addedBy,
            ]
        );
    }

    public static function remove(string $mobile): bool
    {
        return static::where('mobile', $mobile)->delete() > 0;
    }

    // Scopes
    public function scopeByReason($query, string $reason)
    {
        return $query->where('reason', $reason);
    }

    public function scopeOptOuts($query)
    {
        return $query->where('reason', 'opt-out');
    }
}
```

**Features:**
- Static helper methods for blacklist operations
- Reason-based scopes
- Used by CheckBlacklist middleware

**See:** [Blacklist Feature](blacklist-feature.md)

---

### SenderID Model

**Path:** `app/Models/SenderID.php`  
**Table:** `sender_ids`

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class SenderID extends Model
{
    protected $table = 'sender_ids';

    protected $fillable = [
        'sender_id',
        'name',
        'is_active',
    ];

    protected $casts = [
        'is_active' => 'boolean',
    ];

    // Scopes
    public function scopeActive($query)
    {
        return $query->where('is_active', true);
    }

    // Static helpers
    public static function getActive()
    {
        return static::active()->pluck('name', 'sender_id');
    }
}
```

**Features:**
- Active/inactive status
- Pre-approved branded sender IDs

**See:** [Backend Services](backend-services.md)

---

## Model Conventions

### Naming
- Singular PascalCase (e.g., `Contact`, `Group`)
- Table names are plural snake_case (auto-inferred)

### Fillable vs Guarded
- Use `$fillable` for explicit mass assignment
- Avoid `$guarded` unless protecting specific fields

### Casts
- Use `$casts` for type conversion (datetime, array, boolean)
- JSON columns should be cast to `array`

### Relationships
- Define inverse relationships for Eloquent magic
- Use `withTimestamps()` on pivot tables
- Use `constrained()` and `onDelete('cascade')` in migrations

### Scopes
- Global scopes for model-wide filters
- Local scopes for reusable query constraints
- Prefix with `scope` (e.g., `scopeActive`)

### Static Helpers
- Use for common operations (e.g., `BlacklistedNumber::isBlacklisted()`)
- Keep business logic in Actions, not models

## Related Documentation

- [Database Schema](database-schema.md)
- [Backend Services](backend-services.md)
- [Test Scaffolding - Factories](test-scaffolding.md)
