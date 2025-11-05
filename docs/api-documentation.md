# API Documentation

## POST /api/send
Send a single message

```json
{
  "recipient": "+639171234567",
  "message": "Hello from Quezon City!"
}
```

## POST /api/groups/send
Send to a named group

```json
{
  "group": "barangay-leaders",
  "message": "Please attend the emergency meeting."
}
```

## POST /api/contacts
Add a contact

## POST /api/groups
Create a group with contacts
