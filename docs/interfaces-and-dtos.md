# Interfaces and DTOs

## SendMessageDTO

```ts
{
  recipient: string; // E.164
  message: string;
}
```

## GroupBroadcastDTO

```ts
{
  group: string;
  message: string;
}
```

## ContactImportDTO

```ts
{
  file: File;
  group: string;
}
```
