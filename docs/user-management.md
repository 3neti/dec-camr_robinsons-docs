# User Management & Authentication

## 🔐 Overview

Session-based authentication with role-based access control (RBAC).

**Controllers:** `CAMRUserAuthController.php`, `UserController.php`

## 📊 User Roles

### Access Levels

| Access | Sites | Edit | Reports |
|--------|-------|------|--------|
| **ALL** | All | Yes | Yes |
| **Selected** | Assigned | No | Yes |

## 🔑 Authentication

### Login
```php
$user = User::where('user_name', $request->user_name)->first();
if(Hash::check($request->InputPassword, $user->user_password)) {
    $request->session()->put('loginID', $user->user_id);
    return redirect('site');
}
```

### Session Variables
- `loginID` - User ID
- `site_current_tab` - Dashboard state

## 📝 User Management

### Create User
**Required:**
- Real Name (unique)
- Username (unique)
- Email (unique, optional)
- Password (6-20 chars)
- User Type
- User Access (ALL/Selected)

**Password:** bcrypt hashed
**Email:** Credentials sent to new users

### Site Access Control

**ALL Access:** No restrictions
**Selected Access:** `user_access_group` table

```sql
-- Assign site
INSERT INTO user_access_group (user_idx, site_idx)
VALUES (?, ?);
```

## 🔐 Security

- **Hashing:** bcrypt
- **Session:** 2-hour timeout
- **CSRF:** Laravel protection
- **Middleware:** `isLoggedIn`

## Related
- [Installation](installation.md)
- [Configuration](configuration/environment.md)
