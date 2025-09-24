# API Endpoint Implementation Plan: Authentication Login

## 1. Przegląd punktu końcowego

Endpoint `/api/auth/login` służy do uwierzytelniania użytkowników w aplikacji IkigAI Chat App. Implementuje mocked system logowania z ustalonymi danymi uwierzytelniającymi, ustawia bezpieczne HttpOnly session cookies i zwraca kompletne informacje o użytkowniku, sesji oraz budżecie. Jest to kluczowy element systemu uwierzytelniania aplikacji MVP.

## 2. Szczegóły żądania

- **Metoda HTTP:** POST
- **Struktura URL:** `/api/auth/login`
- **Content-Type:** `application/json`
- **Parametry:**
  - **Wymagane:**
    - `email` (string) - adres email użytkownika
    - `password` (string) - hasło użytkownika
  - **Opcjonalne:** Brak
- **Request Body:**

```json
{
  "email": "string",
  "password": "string"
}
```

## 3. Wykorzystywane typy

### DTOs i Command Modele (src/types.ts):

```typescript
export interface LoginRequest {
  email: string;
  password: string;
}

export interface User {
  id: string;
  email: string;
  name: string;
}

export interface Session {
  id: string;
  issuedAt: string; // ISO-8601
  expiresAt: string; // ISO-8601
}

export interface Budget {
  limitUsd: number;
  usedUsd: number;
  remainingUsd: number;
  resetAt: string; // ISO-8601
}

export interface LoginResponse {
  user: User;
  session: Session;
  budget: Budget;
}

export interface ApiError {
  error: string;
  message: string;
  statusCode: number;
}
```

## 4. Szczegóły odpowiedzi

### Successful Response (200 OK):

```json
{
  "user": {
    "id": "user_123",
    "email": "demo@ikigai.app",
    "name": "Demo User"
  },
  "session": {
    "id": "session_456",
    "issuedAt": "2025-09-24T10:00:00.000Z",
    "expiresAt": "2025-09-25T10:00:00.000Z"
  },
  "budget": {
    "limitUsd": 0.5,
    "usedUsd": 0,
    "remainingUsd": 0.5,
    "resetAt": "2025-09-25T00:00:00.000Z"
  }
}
```

### Error Responses:

- **400 Bad Request:** Nieprawidłowe dane wejściowe
- **401 Unauthorized:** Nieprawidłowe dane uwierzytelniające
- **500 Internal Server Error:** Błędy serwera

**Cookies ustawiane:**

- `session-token` - HttpOnly, Secure, SameSite=Strict, expires w 24h

## 5. Przepływ danych

1. **Walidacja żądania:** Sprawdzenie formatu i obecności wymaganych pól
2. **Uwierzytelnianie:** Weryfikacja credentials przeciwko mocked danych
3. **Generowanie sesji:** Utworzenie unikalnego session ID i timestamps
4. **Inicjalizacja budżetu:** Ustawienie początkowych wartości budżetu
5. **Ustawienie cookie:** Bezpieczne HttpOnly session cookie
6. **Zwrócenie odpowiedzi:** Pełne dane użytkownika, sesji i budżetu

### Interakcje z usługami:

- **AuthService:** Walidacja credentials
- **SessionService:** Zarządzanie sesjami i cookies
- **BudgetService:** Inicjalizacja budżetu użytkownika

## 6. Względy bezpieczeństwa

### Uwierzytelnianie i autoryzacja:

- Mocked credentials: `demo@ikigai.app` / `demo123`
- Walidacja formatu email (regex)
- Minimum 6 znaków dla hasła

### Zabezpieczenia sesji:

- HttpOnly cookies (zapobieganie XSS)
- Secure flag (tylko HTTPS)
- SameSite=Strict (zapobieganie CSRF)
- Expiration time: 24 godziny

### Walidacja danych:

- Sanityzacja input danych
- Rate limiting (opcjonalnie)
- Consistent timing responses (zapobieganie timing attacks)

### Security Headers:

```typescript
'Content-Type': 'application/json',
'Cache-Control': 'no-store, no-cache, must-revalidate',
'Pragma': 'no-cache'
```

## 7. Obsługa błędów

### 400 Bad Request:

- Brak wymaganego pola `email`
- Brak wymaganego pola `password`
- Nieprawidłowy format email
- Hasło krótsze niż 6 znaków
- Nieprawidłowy Content-Type

### 401 Unauthorized:

- Nieprawidłowy email
- Nieprawidłowe hasło
- Kombinacja email/hasło nie istnieje w mocked danych

### 500 Internal Server Error:

- Błąd podczas generowania sesji
- Błąd podczas ustawiania cookies
- Nieprzewidziane błędy serwera

### Format odpowiedzi błędów:

```json
{
  "error": "BAD_REQUEST",
  "message": "Email is required",
  "statusCode": 400
}
```

## 8. Rozważania dotyczące wydajności

### Optymalizacje:

- Minimalny czas odpowiedzi (< 100ms dla mocked auth)
- Consistent response timing dla bezpieczeństwa
- Brak query do bazy danych - wszystko w pamięci
- Lightweight session generation

### Monitoring:

- Logowanie czasu wykonania
- Tracking liczby prób logowania
- Monitoring błędów uwierzytelniania

### Skalowanie:

- Rate limiting dla zapobiegania abuse
- Session storage w pamięci (MVP)
- Możliwość rozszerzenia o Redis w przyszłości

## 9. Etapy wdrożenia

### Krok 1: Przygotowanie struktury typów

- Utworzenie typów TypeScript w `src/types.ts`
- Definicja interfejsów dla request/response
- Utworzenie typów błędów

### Krok 2: Implementacja usług pomocniczych

- **AuthService** (`src/lib/auth-service.ts`):
  - Walidacja credentials
  - Mocked user data
  - Password verification
- **SessionService** (`src/lib/session-service.ts`):
  - Session generation
  - Cookie management
  - Session validation
- **BudgetService** (`src/lib/budget-service.ts`):
  - Budget initialization
  - Budget calculation utilities

### Krok 3: Implementacja walidacji

- Utworzenie utility functions w `src/lib/validation.ts`
- Email format validation
- Password strength validation
- Request body validation

### Krok 4: Implementacja Route Handler

- Utworzenie `src/app/api/auth/login/route.ts`
- Implementacja POST method handler
- Request/response processing
- Error handling

### Krok 5: Implementacja bezpieczeństwa

- Cookie security configuration
- Rate limiting middleware (opcjonalnie)
- Security headers
- Input sanitization

### Krok 6: Zustand Store dla uwierzytelniania

- Utworzenie `src/stores/auth-store.ts`
- State management dla user session
- Login/logout actions
- Authentication state persistence

### Krok 7: Obsługa błędów i logowanie

- Centralized error handling
- Logging utility functions
- Error response standardization

### Krok 8: Testowanie i dokumentacja

- Testowanie wszystkich scenariuszy błędów
- Weryfikacja security measures
- Dokumentacja API endpoint

### Krok 9: Integracja z frontend

- Login form component
- Authentication hooks
- Redirect logic po zalogowaniu
- Error handling w UI

### Krok 10: Finalizacja

- Code review
- Security audit
- Performance testing
- Deployment readiness check
