# 11 -- Email API

> Internal email sending service for PGEPilot ecosystem.
> Last updated: 2026-04-04

---

## Endpoint

```
POST https://service.pgepilot.cz/sendEmail
```

Internal (from Docker network):
```
POST http://localhost:8400/sendEmail
```

---

## Authentication

None. The endpoint is behind a firewall, accessible only from the internal network or via NPM proxy.
If needed in the future, add `X-Api-Key` header.

---

## Request

```http
POST /sendEmail HTTP/1.1
Content-Type: application/json

{
  "to": "customer@example.cz",
  "toName": "Jan Novak",
  "subject": "Email subject",
  "body": "<h3>HTML content</h3><p>Text...</p>",
  "profile": "info"
}
```

### Parameters

| Parameter | Required | Type | Description |
|-----------|----------|------|-------------|
| `to` | Yes | string | Recipient email (must be valid) |
| `toName` | No | string | Recipient name |
| `subject` | Yes | string | Email subject |
| `body` | Yes | string | HTML email body |
| `profile` | No | string | Sender profile (default: `servis`) |

### Sender Profiles

| Profile | FROM address | Display name |
|---------|-------------|--------------|
| `servis` | servis@profi-green-energy.cz | PGE Servis |
| `automat` | servis@profi-green-energy.cz | PGE Automat |
| `technika` | servis@profi-green-energy.cz | PGE Technika |
| `obchod` | servis@profi-green-energy.cz | PGE Obchod |
| `info` | info@profi-green-energy.cz | Profi green energy |

The `info` profile uses a shared mailbox (Send As permission). All other profiles send from `servis@` with different display names.

---

## Response

### Success (200)
```json
{
  "status": "success",
  "message": "Email sent successfully"
}
```

### Error (400 / 500)
```json
{
  "status": "error",
  "message": "Invalid or missing \"to\" email address"
}
```

---

## Code Examples

### curl
```bash
curl -X POST https://service.pgepilot.cz/sendEmail \
  -H "Content-Type: application/json" \
  -d '{
    "to": "recipient@example.cz",
    "subject": "FVE Report",
    "body": "<h3>Monthly report</h3><p>Production: 500 kWh</p>",
    "profile": "info"
  }'
```

### JavaScript (fetch)
```javascript
const response = await fetch('https://service.pgepilot.cz/sendEmail', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    to: 'customer@example.cz',
    subject: 'Alert',
    body: '<p>Your PV system has an issue.</p>',
    profile: 'servis'
  })
});
const result = await response.json();
```

### PHP (from another container)
```php
$ch = curl_init('http://pgepilot.cz:8400/sendEmail');
curl_setopt_array($ch, [
    CURLOPT_POST => true,
    CURLOPT_POSTFIELDS => json_encode([
        'to' => 'tech@example.cz',
        'subject' => 'Alert',
        'body' => '<p>Inverter offline</p>',
        'profile' => 'technika'
    ]),
    CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
    CURLOPT_RETURNTRANSFER => true,
]);
$result = json_decode(curl_exec($ch), true);
curl_close($ch);
```

### Python
```python
import requests

r = requests.post('https://service.pgepilot.cz/sendEmail', json={
    'to': 'recipient@example.cz',
    'subject': 'Test',
    'body': '<p>Test email from Python</p>',
    'profile': 'info'
})
print(r.json())
```

---

## SMTP Configuration

| Property | Value |
|----------|-------|
| SMTP server | smtp.office365.com |
| Port | 587 (STARTTLS) |
| Auth account | servis@profi-green-energy.cz |
| Password | [REDACTED] |
| Shared mailbox | info@profi-green-energy.cz (Send As permission) |

---

## Implementation Details

| Item | Location |
|------|----------|
| EmailSender class | `src/Specific/Utils/EmailSender.php` |
| Route | `app/routes.php` (`POST /sendEmail`) |
| Logs | `/var/log/pgepilot/email_sender.log` (in pgepilot_service container) |
| Container | pgepilot_service (port 8400) |
