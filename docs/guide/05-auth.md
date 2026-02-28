# Auth — Identity Platform + JWT

Uwierzytelnianie to jeden z trudniejszych kawałków tej architektury — wiele komponentów musi być zsynchronizowanych: Identity Platform, Firebase SDK, API Gateway, CORS, authorized_domains. Ten rozdział opisuje cały przepływ.

---

## Przepływ uwierzytelniania end-to-end

```
Użytkownik klika "Zaloguj przez Google"
    │
    ▼
Firebase Auth SDK → signInWithPopup(auth, GoogleAuthProvider)
    │ OAuth 2.0 redirect
    ▼
Google OAuth consent screen
    │ user grants permission
    ▼
Identity Platform wystawia JWT (ID Token)
    Payload: {
      sub: "uid123",         ← unikalny user ID
      email: "user@gmail.com",
      name: "Jan Kowalski",
      iss: "https://securetoken.google.com/PROJECT_ID",
      aud: "PROJECT_ID",
      exp: 1735000000        ← ważny 1h
    }
    │
    ▼
Frontend: Authorization: Bearer <JWT>
    │
    ▼
API Gateway weryfikuje JWT
    ✓ podpis (klucze publiczne securetoken.google.com)
    ✓ issuer == securetoken.google.com/PROJECT_ID
    ✓ audience == PROJECT_ID
    ✓ exp > now
    │ (jeśli wszystko OK)
    ▼
Cloud Run — decode_jwt(token) → claims
    claims["sub"] → uid (klucz dokumentu Firestore)
    claims["email"] → dane do zapisu
```

---

## Terraform — warstwa `tf/auth/`

### Identity Platform config

```hcl
resource "google_identity_platform_config" "default" {
  project = var.project_id

  # Wszystkie domeny, z których Firebase Auth może być użyty
  authorized_domains = [
    "localhost",                                       # dev lokalnie
    "storage.googleapis.com",                          # legacy GCS URL
    "kamilos.xyz",
    "app.kamilos.xyz",                                 # prod
    "staging.kamilos.xyz",                             # staging
    "gcp-prototype-1-20260224.firebaseapp.com",        # (1)
  ]
}
```

1. Subdomena `firebaseapp.com` jest wymagana jako `authDomain` w Firebase SDK. Tworzy się **automatycznie** po rejestracji projektu GCP w Firebase Console (`console.firebase.google.com`). Bez rejestracji → subdomena nie istnieje → `signInWithPopup` zwróci błąd sieciowy.

### Google SSO provider

```hcl
resource "google_identity_platform_default_supported_idp_config" "google" {
  idp_id        = "google.com"
  client_id     = var.google_oauth_client_id
  client_secret = var.google_oauth_client_secret
  enabled       = true
}
```

`client_id` i `client_secret` tworzysz ręcznie: GCP Console → APIs & Services → Credentials → Create credentials → OAuth client ID → Web application. Następnie dodaj `Authorized redirect URI`:
```
https://gcp-prototype-1-20260224.firebaseapp.com/__/auth/handler
```

### OAuth credentials w Secret Manager

```hcl
resource "google_secret_manager_secret" "oauth_client_id" {
  secret_id = "identity-platform-oauth-client-id"
  replication { auto {} }
}

resource "google_secret_manager_secret_version" "oauth_client_id" {
  secret      = google_secret_manager_secret.oauth_client_id.id
  secret_data = var.google_oauth_client_id  # (1)
}
```

1. `secret_data` jest sensitive — Terraform nie wyświetli wartości w planie ani outputcie. Przechowujemy w Secret Manager dla audytowalności dostępu (kto, kiedy, z jakiego SA czytał secret).

---

## Frontend — Firebase SDK

```javascript
// frontend/app.js

const FIREBASE_CONFIG = {
  // apiKey jest PUBLICZNY z założenia architektury Firebase.
  // Zabezpieczenie: authorized_domains w Identity Platform ogranicza,
  // z jakich domen można używać tego klucza.
  apiKey:     'AIzaSyDB9HH...',
  authDomain: 'gcp-prototype-1-20260224.firebaseapp.com',
  projectId:  'gcp-prototype-1-20260224',
};

const app      = initializeApp(FIREBASE_CONFIG);
const auth     = getAuth(app);
const provider = new GoogleAuthProvider();

// Logowanie — popup OAuth
loginBtn.addEventListener('click', () => signInWithPopup(auth, provider));

// Wylogowanie
logoutBtn.addEventListener('click', () => signOut(auth));

// Reakcja na zmianę stanu auth (odświeżenie strony, wygaśnięcie tokenu)
onAuthStateChanged(auth, async (user) => {
  if (user) {
    const token = await auth.currentUser.getIdToken(); // (1)
    // Użyj tokenu do API calls
    await fetch(`${API_URL}/api`, {
      headers: { Authorization: `Bearer ${token}` }
    });
  }
});
```

1. `getIdToken()` automatycznie odświeża token jeśli zbliża się expiry. Wywołuj przed każdym API call — nie cachuj tokenu lokalnie (localStorage, sessionStorage).

---

## Pułapki

!!! danger "Identity Platform wymaga ręcznej aktywacji w konsoli"
    `terraform apply` zwróci `Error 400: CONFIGURATION_NOT_FOUND` jeśli Identity Platform nie jest aktywowana ręcznie. Kroki:
    1. GCP Console → Identity Platform → Enable
    2. Po aktywacji zasób istnieje poza Terraform — trzeba zaimportować:
    ```bash
    terraform import google_identity_platform_config.default \
      projects/PROJECT_ID/config
    ```
    W `deploy.yml` krok importu jest wbudowany z `|| true` (idempotentny).

!!! danger "Firebase Console — projekt musi być zarejestrowany"
    `authDomain: PROJECT_ID.firebaseapp.com` istnieje tylko gdy projekt GCP jest dodany do Firebase Console. Bez tego subdomena nie istnieje → `signInWithPopup` rzuci błąd sieciowy bez czytelnego komunikatu.

!!! warning "authorized_domains — każda nowa domena"
    Dodajesz nowe środowisko? Musisz dodać domenę do `authorized_domains`. Brak → Firebase Auth zwraca `auth/unauthorized-domain` bez szczegółów.

!!! tip "apiKey w Firebase to nie sekret"
    Firebase API Key (Web API Key) nie jest sekretem — jest publiczny w kodzie frontendowym. Nie wrzucaj go do Secret Manager. Zabezpieczenie: `authorized_domains` + IAM w Cloud Run. Atakujący z obcej domeny nie może użyć klucza do logowania.
