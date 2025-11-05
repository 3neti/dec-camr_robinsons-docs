# Backend Services

## SMSService
Handles sending to individuals or groups via a gateway (Semaphore, Twilio, etc.)

## ContactImportJob
Parses and stores uploaded contact files (CSV, Excel)

## ScheduledBroadcast
Cron job to handle scheduled or batch-delayed sends

## WebhookHandler
Handles status callbacks from SMS gateway (e.g., delivery reports)
