# Database Schema

**Database:** PostgreSQL  
**ORM:** Laravel Eloquent

## Tables Overview

Text Commander uses the following core tables:

| Table | Description | Documentation |
|-------|-------------|---------------|
| `users` | Admin users | [Controller Scaffolding](controller-scaffolding.md) |
| `contacts` | Contact records | [Contact Package](contact-package.md) |
| `groups` | Contact groups | [Backend Services](backend-services.md) |
| `contact_group` | Pivot table for contacts-groups | [Backend Services](backend-services.md) |
| `scheduled_messages` | Scheduled SMS | [Scheduled Messaging](scheduled-messaging.md) |
| `blacklisted_numbers` | No-send list | [Blacklist Feature](blacklist-feature.md) |
| `sender_ids` | Branded sender IDs | [Backend Services](backend-services.md) |

## Entity Relationship Diagram

```mermaid
erDiagram
    USERS {
        bigint id PK
        string name
        string email UK
        timestamp email_verified_at
        string password
        string remember_token
        timestamps
    }
    
    CONTACTS {
        bigint id PK
        string name
        string mobile UK
        string email
        json extra
        timestamps
    }
    
    GROUPS {
        bigint id PK
        string name UK
        text description
        bigint user_id FK
        timestamps
    }
    
    CONTACT_GROUP {
        bigint contact_id FK
        bigint group_id FK
        timestamps
    }
    
    SCHEDULED_MESSAGES {
        bigint id PK
        text message
        json recipients
        json group_ids
        string sender_id
        timestamp scheduled_at
        string status
        timestamps
    }
    
    BLACKLISTED_NUMBERS {
        bigint id PK
        string mobile UK
        string reason
        text notes
        string added_by
        timestamps
    }
    
    SENDER_IDS {
        bigint id PK
        string sender_id UK
        string name
        boolean is_active
        timestamps
    }
    
    USERS ||--o{ GROUPS : "owns"
    CONTACTS ||--o{ CONTACT_GROUP : "belongs to"
    GROUPS ||--o{ CONTACT_GROUP : "has"
```

## Table Details

### Users Table

**Purpose:** Store admin user accounts

```php
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->rememberToken();
    $table->timestamps();
});
```

**Model:** `App\Models\User`  
**Authenticatable:** Yes (Laravel Sanctum)

---

### Contacts Table

**Purpose:** Store contact information for SMS recipients

```php
Schema::create('contacts', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('mobile')->unique();
    $table->string('email')->nullable();
    $table->json('extra')->nullable();
    $table->timestamps();
});
```

**Model:** `App\Models\Contact` (extends `Lbhurtado\Contact\Models\Contact`)  
**Features:** Phone normalization, group relationships

**See:** [Contact Package](contact-package.md)

---

### Groups Table

**Purpose:** Organize contacts into logical groups

```php
Schema::create('groups', function (Blueprint $table) {
    $table->id();
    $table->string('name')->unique();
    $table->text('description')->nullable();
    $table->foreignId('user_id')->constrained()->onDelete('cascade');
    $table->timestamps();
});
```

**Model:** `App\Models\Group`  
**Relationships:** 
- `belongsTo(User::class)`
- `belongsToMany(Contact::class)`

**See:** [Backend Services](backend-services.md)

---

### Contact-Group Pivot Table

**Purpose:** Many-to-many relationship between contacts and groups

```php
Schema::create('contact_group', function (Blueprint $table) {
    $table->foreignId('contact_id')->constrained()->onDelete('cascade');
    $table->foreignId('group_id')->constrained()->onDelete('cascade');
    $table->timestamps();
    
    $table->primary(['contact_id', 'group_id']);
});
```

---

### Scheduled Messages Table

**Purpose:** Store scheduled SMS broadcasts

```php
Schema::create('scheduled_messages', function (Blueprint $table) {
    $table->id();
    $table->text('message');
    $table->json('recipients')->nullable();
    $table->json('group_ids')->nullable();
    $table->string('sender_id');
    $table->timestamp('scheduled_at');
    $table->enum('status', ['pending', 'processing', 'sent', 'failed'])->default('pending');
    $table->timestamps();
});
```

**Model:** `App\Models\ScheduledMessage`  
**Features:** Scheduled job processing, status tracking

**See:** [Scheduled Messaging](scheduled-messaging.md)

---

### Blacklisted Numbers Table

**Purpose:** Store no-send list to prevent SMS delivery to specific numbers

```php
Schema::create('blacklisted_numbers', function (Blueprint $table) {
    $table->id();
    $table->string('mobile')->unique();
    $table->enum('reason', ['opt-out', 'complaint', 'invalid', 'other']);
    $table->text('notes')->nullable();
    $table->string('added_by')->default('system');
    $table->timestamps();
});
```

**Model:** `App\Models\BlacklistedNumber`  
**Features:** Job middleware filtering, public opt-out endpoint

**See:** [Blacklist Feature](blacklist-feature.md)

---

### Sender IDs Table

**Purpose:** Store branded sender IDs for SMS broadcasts

```php
Schema::create('sender_ids', function (Blueprint $table) {
    $table->id();
    $table->string('sender_id')->unique();
    $table->string('name');
    $table->boolean('is_active')->default(true);
    $table->timestamps();
});
```

**Model:** `App\Models\SenderID`  
**Features:** Active/inactive status, pre-approved branding

---

## Indexes

Key indexes for performance:

- `contacts.mobile` - Unique index for contact lookup
- `contacts.email` - Index for email search
- `groups.name` - Unique index for group names
- `blacklisted_numbers.mobile` - Unique index for blacklist checks
- `scheduled_messages.scheduled_at` - Index for job processing
- `scheduled_messages.status` - Index for status filtering

## Related Documentation

- [Backend Services](backend-services.md)
- [Models](models.md)
- [API Documentation](api-documentation.md)
- [Test Scaffolding](test-scaffolding.md)
