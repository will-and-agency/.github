---
  # Authentication Flow

  ## First Login (New Account)

  When a root creates a moderator or supermoderator, the backend generates a
  random 6-digit OTP code, stores it as the account password (hashed), and
  sends
  it to the mod's email. The account is created with `must_change_password:
  true`
  in its metadata.

  ---

  ### Step 1 — Login with OTP

  **POST** `/auth/login`

  ```json
  Request:
  {
    "email": "user@example.com",
    "password": "123456"  // the OTP received by email
  }

  Response 200:
  {
    "temp_token": "<jwt>",
    "requires_password_change": true
  }

  What the backend does:
  - Looks up the account by email (returns 401 if not found, never 404 —
  prevents email enumeration)
  - Checks if the account is locked (5 failed attempts locks for 5 minutes)
  - Checks if the account is active
  - Verifies the password against the stored bcrypt hash
  - Detects must_change_password: true in metadata
  - Issues a short-lived temp JWT with step = "pending_password_change" (5 min
   expiry)

  ---
  Step 2 — Change Initial Password

  POST /auth/change-initial-password
  Header: Authorization: Bearer <temp_token>

  Request:
  {
    "new_password": "MyNewSecurePassword1!"
  }

  Response 200:
  {
    "temp_token": "<new_jwt>",
    "setup_required": true
  }

  What the backend does:
  - Validates the token has step = "pending_password_change" — rejects full
  JWTs
  - Validates password strength
  - Hashes the new password and updates it in the DB
  - Removes must_change_password from the account metadata
  - Issues a new temp JWT with step = "pending_2fa" (5 min expiry)
  - Since TOTP is not yet set up, responds with setup_required: true

  ---
  Step 3 — Set Up TOTP

  POST /auth/setup-2fa
  Header: Authorization: Bearer <temp_token>
  No request body.

  Response 200:
  {
    "secret": "BASE32ENCODEDSECRET",
    "uri": "otpauth://totp/AppName:user@example.com?secret=...&issuer=AppName"
  }

  What the backend does:
  - Validates the token has step = "pending_2fa" — rejects full JWTs
  - Generates a new TOTP secret and otpauth:// URI (used for QR code display)
  - Encrypts the secret with AES-256-GCM before storing it in the DB
  - Stores the encrypted secret but keeps totp_enabled = false until confirmed
  - If called again before confirming, overwrites the old secret (allows
  retry)

  The login page would display the URI as a QR code or shows the raw secret for
  manual
  entry into an authenticator app (Google Authenticator, Authy, etc.).

  ---
  Step 4 — Confirm TOTP and Complete Login

  POST /auth/verify-2fa
  Header: Authorization: Bearer <temp_token>

  Request:
  {
    "code": "123456"  // 6-digit code from authenticator app (TOTP)
  }

  Response 200:
  {
    "full_token": "<jwt>"
  }

  What the backend does:
  - Validates the token has step = "pending_2fa"
  - Fetches the account and checks totp_secret exists (must have called
  setup-2fa first)
  - Checks replay attack: rejects if the same code was already used in the
  current
  30-second TOTP window (tracked via totp_last_used_at)
  - Decrypts the stored TOTP secret and verifies the code (±1 window tolerance
   for
  clock drift)
  - Since totp_enabled = false, sets totp_enabled = true in the DB
  (activation)
  - Updates totp_last_used_at to now
  - Issues a full JWT with no step claim (normal expiry, default 24h)

  The user is now fully logged in. All subsequent logins use the flow below.

  ---
  Subsequent Logins (TOTP Already Set Up)

  ---
  Step 1 — Login with Password

  POST /auth/login

  Request:
  {
    "email": "user@example.com",
    "password": "MyNewSecurePassword1!"
  }

  Response 200:
  {
    "temp_token": "<jwt>",
    "requires_2fa": true
  }

  What the backend does:
  - Same checks as before (lockout, active, password verification)
  - No must_change_password in metadata, so skips that branch
  - Detects totp_enabled = true on the account
  - Issues a temp JWT with step = "pending_2fa" (5 min expiry)
  - Responds with requires_2fa: true

  ---
  Step 2 — Verify TOTP and Complete Login

  POST /auth/verify-2fa
  Header: Authorization: Bearer <temp_token>

  Request:
  {
    "code": "123456"  // 6-digit code from authenticator app
  }

  Response 200:
  {
    "full_token": "<jwt>"
  }

  What the backend does:
  - Validates the token has step = "pending_2fa"
  - Fetches the account and checks totp_secret exists
  - Replay attack check via totp_last_used_at
  - Decrypts and verifies the TOTP code
  - Since totp_enabled = true, skips the activation step (no DB change to that
   field)
  - Updates totp_last_used_at to now
  - Issues a full JWT

  ---
  Token Types Summary

  ┌─────────┬────────────────────┬─────────────┬─────────────────────────┐
  │  Token  │     step claim     │   Expiry    │       Accepted by       │
  ├─────────┼────────────────────┼─────────────┼─────────────────────────┤
  │ Temp (p │                    │             │                         │
  │ assword │ pending_password_c │ 5 min       │ /auth/change-initial-pa │
  │         │ hange              │             │ ssword                  │
  │ change) │                    │             │                         │
  ├─────────┼────────────────────┼─────────────┼─────────────────────────┤
  │ Temp    │ pending_2fa        │ 5 min       │ /auth/setup-2fa,        │
  │ (2FA)   │                    │             │ /auth/verify-2fa        │
  ├─────────┼────────────────────┼─────────────┼─────────────────────────┤
  │ Full    │ none               │ 24h (config │ All protected routes    │
  │         │                    │ urable)     │                         │
  └─────────┴────────────────────┴─────────────┴─────────────────────────┘

  Any endpoint that receives the wrong token type returns 401 Unauthorized:
  Invalid token type.

  ---
  Error Reference

  ┌──────┬─────────────────────────────────┬──────────────────────────────┐
  │ Code │             Message             │            Cause             │
  ├──────┼─────────────────────────────────┼──────────────────────────────┤
  │ 401  │ Invalid credentials             │ Wrong password or account    │
  │      │                                 │ not found                    │
  ├──────┼─────────────────────────────────┼──────────────────────────────┤
  │ 401  │ Account is temporarily locked   │ 5 failed attempts, wait 5    │
  │      │                                 │ minutes                      │
  ├──────┼─────────────────────────────────┼──────────────────────────────┤
  │ 401  │ Account is inactive             │ Account suspended            │
  ├──────┼─────────────────────────────────┼──────────────────────────────┤
  │ 401  │ Invalid token type              │ Wrong step claim for this    │
  │      │                                 │ endpoint                     │
  ├──────┼─────────────────────────────────┼──────────────────────────────┤
  │ 401  │ TOTP code already used          │ Replay attack — same code in │
  │      │                                 │  same 30s window             │
  ├──────┼─────────────────────────────────┼──────────────────────────────┤
  │ 401  │ Invalid TOTP code               │ Wrong 6-digit code           │
  ├──────┼─────────────────────────────────┼──────────────────────────────┤
  │ 400  │ TOTP not configured. Use        │ Called verify-2fa without    │
  │      │ /auth/setup-2fa first.          │ setting up TOTP              │
  ├──────┼─────────────────────────────────┼──────────────────────────────┤
  │ 404  │ Account not found               │ Account deleted between      │
  │      │                                 │ token issue and request      │
  ├──────┼─────────────────────────────────┼──────────────────────────────┤
  │ 500  │ —                               │ DB or crypto failure         │
  ├──────┼─────────────────────────────────┼──────────────────────────────┤
  │ ```  │                                 │                              │
  └──────┴─────────────────────────────────┴──────────────────────────────┘
