# Plan: Media Share Service dla TeleIRC

## Cel
Utworzenie prostego mikroserwisu do hostowania plików (video, audio, obrazy) z wbudowanym playerem HTML5, który zastąpi/uzupełni Imgur w TeleIRC.

## Architektura

### Decyzja: Osobny mikroserwis vs zintegrowany
**Wybór: Osobny mikroserwis** w katalogu `cmd/mediashare/`

Uzasadnienie:
- Może być deployowany niezależnie
- Jeden binary, zero zależności
- Możliwość użycia na innym serwerze niż TeleIRC
- Prostsza konfiguracja i skalowanie

## Struktura plików

```
cmd/
  mediashare/
    main.go           # Entry point serwisu
internal/
  mediashare/
    server.go         # HTTP server i handlery
    storage.go        # Zapis/odczyt plików
    templates.go      # HTML templates (embedded)
  config.go           # + nowe MediaShareSettings
```

## Funkcjonalności mikroserwisu

### Endpointy HTTP

1. **POST /upload**
   - Auth: `X-API-Key` header
   - Body: multipart/form-data z plikiem
   - Response: `{"url": "https://host/v/abc123", "delete_url": "https://host/d/abc123/token"}`

2. **GET /v/{id}**
   - Strona HTML z playerem (video/audio) lub obrazkiem
   - Auto-detect typu pliku
   - Responsive design

3. **GET /r/{id}**
   - Raw file (direct download/embed)

4. **DELETE /d/{id}/{token}**
   - Usunięcie pliku (opcjonalne)

### Konfiguracja (env)

```env
MEDIASHARE_PORT=8080
MEDIASHARE_API_KEY=secret123
MEDIASHARE_BASE_URL=https://media.example.com
MEDIASHARE_STORAGE_PATH=./uploads
MEDIASHARE_MAX_FILE_SIZE=50MB
MEDIASHARE_RETENTION_DAYS=30
```

### HTML Player

```html
<!DOCTYPE html>
<html>
<head>
    <title>{{.Filename}}</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <style>
        body { margin: 0; background: #1a1a1a; display: flex;
               justify-content: center; align-items: center;
               min-height: 100vh; }
        video, audio, img { max-width: 100%; max-height: 100vh; }
    </style>
</head>
<body>
    {{if .IsVideo}}<video controls autoplay><source src="/r/{{.ID}}"></video>{{end}}
    {{if .IsAudio}}<audio controls autoplay><source src="/r/{{.ID}}"></audio>{{end}}
    {{if .IsImage}}<img src="/r/{{.ID}}">{{end}}
</body>
</html>
```

## Integracja z TeleIRC

### Nowa konfiguracja w config.go

```go
type MediaShareSettings struct {
    Enabled  bool   `env:"MEDIASHARE_ENABLED" envDefault:"false"`
    Endpoint string `env:"MEDIASHARE_ENDPOINT" envDefault:""`
    APIKey   string `env:"MEDIASHARE_API_KEY" envDefault:""`
}
```

### Nowy uploader w internal/handlers/telegram/

```go
// upload.go - generyczny uploader
func uploadFile(tg *Client, fileID string) string {
    // 1. Pobierz URL pliku z Telegram API
    // 2. Pobierz plik
    // 3. Wyślij POST do MediaShare
    // 4. Zwróć URL
}
```

### Zmiany w handlerach

- `photoHandler` - użyje nowego uploadera (fallback na Imgur)
- `videoHandler` - użyje nowego uploadera
- `voiceHandler` - użyje nowego uploadera
- `documentHandler` - opcjonalnie użyje uploadera

## Plan implementacji

### Faza 1: Mikroserwis MediaShare
1. Utworzenie struktury katalogów
2. Implementacja `cmd/mediashare/main.go`
3. Implementacja `internal/mediashare/server.go`
4. Implementacja `internal/mediashare/storage.go`
5. Embedded HTML templates
6. Testy

### Faza 2: Integracja z TeleIRC
1. Dodanie `MediaShareSettings` do config.go
2. Utworzenie `internal/handlers/telegram/upload.go`
3. Modyfikacja handlerów (video, voice, photo)
4. Aktualizacja env.example
5. Testy integracyjne

### Faza 3: Dokumentacja i deployment
1. Aktualizacja README
2. Dockerfile dla mediashare
3. Przykładowa konfiguracja docker-compose

## Szczegóły techniczne

### Przechowywanie plików
- Katalog: `{STORAGE_PATH}/{YYYY}/{MM}/{DD}/{random_id}.{ext}`
- Metadane: `{id}.json` (opcjonalnie dla delete token)

### Generowanie ID
- 8 znaków alphanumerycznych (base62)
- Sprawdzenie kolizji przed zapisem

### Obsługiwane typy
- Video: mp4, webm, mov, avi
- Audio: mp3, ogg, wav, m4a
- Image: jpg, png, gif, webp

### Bezpieczeństwo
- API key dla uploadu
- Rate limiting (opcjonalnie)
- Max file size
- Sanityzacja nazw plików
