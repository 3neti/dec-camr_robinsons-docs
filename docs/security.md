# Security & Compliance

**Status:** 🟡 Partial - See [TODO-SECTIONS.md](TODO-SECTIONS.md)

This document covers security best practices, compliance features, and data protection in Text Commander.

## Implemented Security Features

### 1. Blacklist System ✅

Prevents SMS delivery to opted-out numbers and compliance violations.

- Job middleware filtering
- Public opt-out endpoint
- Reason tracking (opt-out, complaint, invalid)

**See:** [Blacklist Feature](blacklist-feature.md)

---

### 2. API Authentication ✅

Laravel Sanctum token-based authentication for API endpoints.

```php
// Generate token
$token = $user->createToken('api-token')->plainTextToken;

// Use in requests
Authorization: Bearer {token}
```

**See:** [API Documentation - Authentication](api-documentation.md)

---

### 3. Input Validation ✅

ActionRequest validation on all Actions.

```php
public function rules()
{
    return [
        'recipients' => 'required|array',
        'message' => 'required|string|max:160',
        'sender_id' => 'required|string|exists:sender_ids,sender_id',
    ];
}
```

**See:** [Backend Services](backend-services.md)

---

### 4. CSRF Protection ✅

Laravel default CSRF protection on all web routes.

```blade
@csrf
```

---

## Planned Security Enhancements

### Rate Limiting (TODO)

**Purpose:** Prevent API abuse

```php
RateLimiter::for('sms', function (Request $request) {
    return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
});
```

**Priority:** High

---

### SQL Injection Prevention (TODO)

**Current:** Eloquent ORM provides automatic protection  
**Enhancement needed:** Document best practices

---

### XSS Prevention (TODO)

**Current:** Blade templates auto-escape output  
**Enhancement needed:** Document sanitization for user input

---

### Authorization Policies (TODO)

**Purpose:** Role-based access control

```php
Gate::define('send-sms', function (User $user) {
    return $user->hasPermission('send-sms');
});
```

**Priority:** Medium

---

### API Security Headers (TODO)

**Recommended headers:**
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`
- `X-XSS-Protection: 1; mode=block`
- `Strict-Transport-Security: max-age=31536000`

**Priority:** Medium

---

## Data Protection

### Phone Number Storage

- Stored in E.164 format (+639171234567)
- Indexed for fast lookups
- Blacklist checking before processing

**See:** [Phone Normalization](phone-normalization.md)

---

### Password Hashing

Laravel uses bcrypt by default:

```php
'password' => 'hashed', // in User model $casts
```

---

### API Token Storage

Sanctum tokens are hashed in database:

```php
$token = $user->createToken('api-token')->plainTextToken;
// Plain text shown once, hash stored in database
```

---

## Compliance Features

### Opt-Out Mechanism ✅

Public endpoint for recipients to opt out:

```bash
POST /api/optout
Content-Type: application/json

{
  "mobile": "+639171234567"
}
```

**See:** [API Documentation - Opt-Out](api-documentation.md)

---

### Audit Logging

Log all SMS sends:

```php
Log::info('SMS sent', [
    'recipient' => $recipient,
    'sender_id' => $senderId,
    'timestamp' => now(),
]);
```

**Location:** `storage/logs/laravel.log`

---

## Security Checklist

- [x] API authentication (Sanctum)
- [x] CSRF protection (Laravel default)
- [x] Input validation (ActionRequest)
- [x] Blacklist system
- [x] Public opt-out endpoint
- [x] Password hashing
- [ ] Rate limiting
- [ ] Authorization policies
- [ ] Security headers
- [ ] SQL injection documentation
- [ ] XSS prevention documentation

---

## Related Documentation

- [Blacklist Feature](blacklist-feature.md)
- [API Documentation](api-documentation.md)
- [Middleware](middleware.md)
- [TODO-SECTIONS.md](TODO-SECTIONS.md)
