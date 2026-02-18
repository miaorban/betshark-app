# BetShark — Fejlesztői dokumentáció #2: Node.js Backend

> **Verzió:** 1.0
> **Utoljára frissítve:** 2026-02-18
> **Célközönség:** Backend fejlesztő

---

## 1. Áttekintés

A Node.js backend a BetShark rendszer API rétege. Feladatai:

1. **Felhasználókezelés:** regisztráció, bejelentkezés (email/jelszó, Google OAuth, Apple Sign-In), jelszó-visszaállítás, fiók törlés.
2. **JWT-alapú autentikáció:** minden védett végpont Bearer tokent vár.
3. **Előfizetés kezelés:** Apple IAP és Google Play Billing receipt validálás, 7 napos trial, visszaélés-megelőzés (hashed payment identifier).
4. **Sports adatok kiszolgálása:** esemény-, piac- és kimenetel-listák olvasása a PostgreSQL adatbázisból (amelyeket a Python poller tölt fel).
5. **Prémium Toplist:** csak aktív előfizetéssel rendelkező felhasználóknak elérhető.
6. **REST API:** a Flutter alkalmazás fogyasztja.

A Node.js backend **kizárólag olvassa** a sports adatokat (events, markets, outcomes, leagues táblák). Írni csak a felhasználói táblákat írja.

---

## 2. Tech stack

| Eszköz | Verzió | Leírás |
|---|---|---|
| Node.js | 20 LTS | Futtatókörnyezet |
| Express | 4.19.2 | HTTP framework |
| `pg` | 8.11.5 | PostgreSQL kliens |
| `jsonwebtoken` | 9.0.2 | JWT token generálás és validálás |
| `bcryptjs` | 2.4.3 | Jelszó hashelés |
| `google-auth-library` | 9.10.0 | Google ID token validálás |
| `apple-signin-auth` | 1.7.1 | Apple identity token validálás |
| `nodemailer` | 6.9.13 | Email küldés (jelszó-visszaállítás) |
| `dotenv` | 16.4.5 | .env betöltés |
| `joi` | 17.13.1 | Request validáció |
| `helmet` | 7.1.0 | HTTP security headerek |
| `cors` | 2.8.5 | CORS kezelés |
| `express-rate-limit` | 7.3.1 | Rate limiting |
| `crypto` (beépített) | — | Token és hash generálás |
| `winston` | 3.13.0 | Naplózás |

Telepítés:

```bash
npm install express@4.19.2 pg@8.11.5 jsonwebtoken@9.0.2 bcryptjs@2.4.3 \
  google-auth-library@9.10.0 apple-signin-auth@1.7.1 \
  nodemailer@6.9.13 dotenv@16.4.5 joi@17.13.1 \
  helmet@7.1.0 cors@2.8.5 express-rate-limit@7.3.1 winston@3.13.0
```

---

## 3. Projektstruktúra

```
betshark_backend/
├── .env                        # Környezeti változók (nem kerül verziókezelőbe)
├── .env.example                # Sablon
├── package.json
├── package-lock.json
├── src/
│   ├── app.js                  # Express app összeállítása
│   ├── server.js               # HTTP szerver indítása
│   ├── config/
│   │   └── index.js            # Konfiguráció betöltése .env-ből
│   ├── db/
│   │   └── pool.js             # pg Pool singleton
│   ├── middleware/
│   │   ├── auth.js             # JWT verifikáció middleware
│   │   ├── requirePremium.js   # Prémium előfizetés ellenőrzés
│   │   └── validate.js         # Joi validáció wrapper
│   ├── routes/
│   │   ├── auth.routes.js      # POST /api/auth/*
│   │   ├── events.routes.js    # GET /api/events/*
│   │   ├── outcomes.routes.js  # GET /api/outcomes/:id
│   │   ├── toplist.routes.js   # GET /api/toplist
│   │   └── user.routes.js      # GET,PUT,POST,DELETE /api/user,subscription
│   ├── controllers/
│   │   ├── auth.controller.js
│   │   ├── events.controller.js
│   │   ├── outcomes.controller.js
│   │   ├── toplist.controller.js
│   │   └── user.controller.js
│   ├── services/
│   │   ├── auth.service.js     # Regisztráció, login, OAuth logika
│   │   ├── email.service.js    # Nodemailer wrapper
│   │   ├── subscription.service.js  # Trial, IAP validálás
│   │   └── token.service.js    # JWT generálás/verifikálás
│   └── utils/
│       ├── errors.js           # Egyedi hibaosztályok
│       └── logger.js           # Winston logger
```

---

## 4. Környezeti változók

```dotenv
# Szerver
PORT=3000
NODE_ENV=production

# PostgreSQL
DATABASE_URL=postgresql://betshark_user:password@localhost:5432/betshark_db

# JWT
JWT_SECRET=minimum_32_karakteres_véletlen_string
JWT_EXPIRES_IN=30d

# Google OAuth
GOOGLE_CLIENT_ID=your_google_client_id.apps.googleusercontent.com

# Apple Sign-In
APPLE_CLIENT_ID=com.yourcompany.betshark
APPLE_TEAM_ID=XXXXXXXXXX
APPLE_KEY_ID=XXXXXXXXXX
APPLE_PRIVATE_KEY_PATH=/secrets/apple_auth_key.p8

# Email (pl. SendGrid SMTP, vagy Resend)
SMTP_HOST=smtp.sendgrid.net
SMTP_PORT=587
SMTP_USER=apikey
SMTP_PASS=your_sendgrid_api_key
EMAIL_FROM=noreply@betshark.app

# Apple IAP
APPLE_IAP_SHARED_SECRET=your_apple_shared_secret
APPLE_IAP_VERIFY_URL=https://buy.itunes.apple.com/verifyReceipt
APPLE_IAP_VERIFY_URL_SANDBOX=https://sandbox.itunes.apple.com/verifyReceipt

# Google Play Billing
GOOGLE_PLAY_PACKAGE_NAME=com.yourcompany.betshark
GOOGLE_PLAY_SERVICE_ACCOUNT_JSON=/secrets/google_play_service_account.json

# Trial
TRIAL_DAYS=7

# Rate limiting
RATE_LIMIT_WINDOW_MS=900000
RATE_LIMIT_MAX=100
```

---

## 5. Adatbázis beállítása

A teljes séma a Python poller dokumentációban szerepel (DEV_01). A Node.js backend szempontjából a **felhasználói táblákat** kell kiemelni:

```sql
-- Ha még nem futott le: lásd a teljes sémát a DEV_01 dokumentumban.
-- A Node.js backend ezeket a táblákat OLVASSA:
--   leagues, events, markets, outcomes
-- Ezeket KEZELI (olvas és ír):
--   users, subscriptions, used_trials, password_reset_tokens, legal_acceptances
```

A DB user (`betshark_user`) jogosultságai:

```sql
-- Csak olvasás a sport adatokhoz
GRANT SELECT ON leagues, events, markets, outcomes TO betshark_user;
-- Teljes hozzáférés a felhasználói adatokhoz
GRANT SELECT, INSERT, UPDATE, DELETE ON users, subscriptions, used_trials,
  password_reset_tokens, legal_acceptances TO betshark_user;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO betshark_user;
```

Hasznos indexek teljesítményhez:

```sql
CREATE INDEX idx_events_kickoff ON events(kickoff_time DESC);
CREATE INDEX idx_outcomes_overall ON outcomes(overall_score DESC);
CREATE INDEX idx_subscriptions_user ON subscriptions(user_id);
CREATE INDEX idx_password_reset_token ON password_reset_tokens(token);
```

---

## 6. Lépésről lépésre implementáció

### 6.1 lépés: Projekt inicializálása

```bash
mkdir betshark_backend && cd betshark_backend
npm init -y
mkdir -p src/{config,db,middleware,routes,controllers,services,utils}
```

Adj hozzá a `package.json`-ba:

```json
{
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js"
  },
  "engines": {
    "node": ">=20.0.0"
  }
}
```

---

### 6.2 lépés: Konfiguráció (`src/config/index.js`)

**Mit csináljon:** Betölti a `.env` fájlt és exportálja a konfigurációs értékeket. Ha egy kötelező érték hiányzik, azonnal kilép.

```javascript
// src/config/index.js
require('dotenv').config();

const required = [
  'DATABASE_URL', 'JWT_SECRET', 'GOOGLE_CLIENT_ID',
  'APPLE_CLIENT_ID', 'SMTP_HOST', 'SMTP_PASS'
];
for (const key of required) {
  if (!process.env[key]) {
    console.error(`Missing required environment variable: ${key}`);
    process.exit(1);
  }
}

module.exports = {
  port: parseInt(process.env.PORT || '3000'),
  nodeEnv: process.env.NODE_ENV || 'development',
  databaseUrl: process.env.DATABASE_URL,
  jwt: {
    secret: process.env.JWT_SECRET,
    expiresIn: process.env.JWT_EXPIRES_IN || '30d'
  },
  google: {
    clientId: process.env.GOOGLE_CLIENT_ID
  },
  apple: {
    clientId: process.env.APPLE_CLIENT_ID,
    teamId: process.env.APPLE_TEAM_ID,
    keyId: process.env.APPLE_KEY_ID,
    privateKeyPath: process.env.APPLE_PRIVATE_KEY_PATH
  },
  email: {
    host: process.env.SMTP_HOST,
    port: parseInt(process.env.SMTP_PORT || '587'),
    user: process.env.SMTP_USER,
    pass: process.env.SMTP_PASS,
    from: process.env.EMAIL_FROM || 'noreply@betshark.app'
  },
  apple_iap: {
    sharedSecret: process.env.APPLE_IAP_SHARED_SECRET,
    verifyUrl: process.env.APPLE_IAP_VERIFY_URL,
    sandboxUrl: process.env.APPLE_IAP_VERIFY_URL_SANDBOX
  },
  google_play: {
    packageName: process.env.GOOGLE_PLAY_PACKAGE_NAME,
    serviceAccountJson: process.env.GOOGLE_PLAY_SERVICE_ACCOUNT_JSON
  },
  trialDays: parseInt(process.env.TRIAL_DAYS || '7'),
  rateLimit: {
    windowMs: parseInt(process.env.RATE_LIMIT_WINDOW_MS || '900000'),
    max: parseInt(process.env.RATE_LIMIT_MAX || '100')
  }
};
```

---

### 6.3 lépés: Adatbázis connection pool (`src/db/pool.js`)

**Mit csináljon:** Egy `pg.Pool` singleton-t exportál, amit az összes service/controller importál.

```javascript
// src/db/pool.js
const { Pool } = require('pg');
const config = require('../config');

const pool = new Pool({
  connectionString: config.databaseUrl,
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000
});

pool.on('error', (err) => {
  console.error('Unexpected error on idle DB client', err);
});

module.exports = pool;
```

---

### 6.4 lépés: Naplózás (`src/utils/logger.js`)

```javascript
// src/utils/logger.js
const winston = require('winston');
const config = require('../config');

const logger = winston.createLogger({
  level: config.nodeEnv === 'production' ? 'info' : 'debug',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    config.nodeEnv === 'production'
      ? winston.format.json()
      : winston.format.simple()
  ),
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
    new winston.transports.File({ filename: 'logs/combined.log' })
  ]
});

module.exports = logger;
```

---

### 6.5 lépés: Egyedi hibaosztályok (`src/utils/errors.js`)

```javascript
// src/utils/errors.js
class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true;
  }
}

class NotFoundError extends AppError {
  constructor(message = 'Not found') { super(message, 404); }
}

class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') { super(message, 401); }
}

class ForbiddenError extends AppError {
  constructor(message = 'Forbidden') { super(message, 403); }
}

class ValidationError extends AppError {
  constructor(message = 'Validation failed') { super(message, 400); }
}

class ConflictError extends AppError {
  constructor(message = 'Conflict') { super(message, 409); }
}

module.exports = { AppError, NotFoundError, UnauthorizedError, ForbiddenError, ValidationError, ConflictError };
```

---

### 6.6 lépés: Token service (`src/services/token.service.js`)

**Mit csináljon:** JWT generálása és verifikálása. A token payload tartalmazza a `userId`-t és a `subscriptionStatus`-t.

```javascript
// src/services/token.service.js
const jwt = require('jsonwebtoken');
const config = require('../config');

function generateToken(user, subscription) {
  return jwt.sign(
    {
      userId: user.id,
      email: user.email,
      subscriptionStatus: subscription?.status || 'none'
    },
    config.jwt.secret,
    { expiresIn: config.jwt.expiresIn }
  );
}

function verifyToken(token) {
  return jwt.verify(token, config.jwt.secret);
}

module.exports = { generateToken, verifyToken };
```

---

### 6.7 lépés: Auth middleware (`src/middleware/auth.js`)

**Mit csináljon:** Kicsomagolja a Bearer tokent az `Authorization` headerből, verifikálja, és a `req.user` objektumra teszi a payload-ot. Ha nincs token, vagy érvénytelen, 401-et ad vissza.

```javascript
// src/middleware/auth.js
const { verifyToken } = require('../services/token.service');
const { UnauthorizedError } = require('../utils/errors');

function authenticate(req, res, next) {
  const authHeader = req.headers['authorization'];
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return next(new UnauthorizedError('No token provided'));
  }
  const token = authHeader.split(' ')[1];
  try {
    req.user = verifyToken(token);
    next();
  } catch (err) {
    next(new UnauthorizedError('Invalid or expired token'));
  }
}

module.exports = { authenticate };
```

---

### 6.8 lépés: Prémium middleware (`src/middleware/requirePremium.js`)

**Mit csináljon:** Az `authenticate` middleware után fut. Ellenőrzi, hogy a felhasználónak aktív előfizetése van-e a JWT payload alapján. Ha nem, 401-et ad vissza.

```javascript
// src/middleware/requirePremium.js
const { UnauthorizedError } = require('../utils/errors');

function requirePremium(req, res, next) {
  const status = req.user?.subscriptionStatus;
  const active = status === 'active' || status === 'trial';
  if (!active) {
    return next(new UnauthorizedError('Premium subscription required'));
  }
  next();
}

module.exports = { requirePremium };
```

**Megjegyzés:** A JWT tokenben lévő `subscriptionStatus` a bejelentkezéskor kerül bele, és 30 napig él. Előfizetés változásakor (pl. lejárat, lemondás) a következő bejelentkezéskor frissül. Ha valós idejű ellenőrzés kell, a middleware-t ki kell egészíteni DB-lekérdezéssel.

---

### 6.9 lépés: Joi validáció middleware (`src/middleware/validate.js`)

```javascript
// src/middleware/validate.js
const { ValidationError } = require('../utils/errors');

function validate(schema, property = 'body') {
  return (req, res, next) => {
    const { error } = schema.validate(req[property], { abortEarly: false });
    if (error) {
      const message = error.details.map(d => d.message).join('; ');
      return next(new ValidationError(message));
    }
    next();
  };
}

module.exports = { validate };
```

---

### 6.10 lépés: Email service (`src/services/email.service.js`)

**Mit csináljon:** Nodemailer-rel küld emaileket. Egyelőre csak a jelszó-visszaállítási emailt küldi.

```javascript
// src/services/email.service.js
const nodemailer = require('nodemailer');
const config = require('../config');

const transporter = nodemailer.createTransport({
  host: config.email.host,
  port: config.email.port,
  secure: config.email.port === 465,
  auth: {
    user: config.email.user,
    pass: config.email.pass
  }
});

async function sendPasswordResetEmail(toEmail, resetToken) {
  const resetUrl = `https://betshark.app/reset-password?token=${resetToken}`;
  await transporter.sendMail({
    from: config.email.from,
    to: toEmail,
    subject: 'BetShark – Jelszó visszaállítás',
    html: `
      <p>Kaptunk egy jelszó-visszaállítási kérelmet ehhez a fiókhoz.</p>
      <p>Kattints az alábbi linkre a jelszó megváltoztatásához (1 óráig érvényes):</p>
      <a href="${resetUrl}">${resetUrl}</a>
      <p>Ha nem te kérted, hagyd figyelmen kívül ezt az emailt.</p>
    `
  });
}

module.exports = { sendPasswordResetEmail };
```

---

### 6.11 lépés: Auth service (`src/services/auth.service.js`)

**Mit hozz létre:** `src/services/auth.service.js`

**Mit csináljon:** Az összes autentikációs logikát tartalmazza. Összesen 6 funkció:

1. **`registerWithEmail(email, password)`** — bcrypt-tel hashelje a jelszót, szúrja be a `users` táblába, 409-et dobjon, ha az email már létezik.
2. **`loginWithEmail(email, password)`** — lekéri a usert, bcrypt-tel összehasonlítja a jelszót, JWT-t generál.
3. **`loginWithGoogle(idToken)`** — Google id_token-t verifikál a `google-auth-library`-vel, upsert a `users` táblába.
4. **`loginWithApple(identityToken, userData)`** — Apple identity_token-t verifikál, upsert a `users` táblába.
5. **`requestPasswordReset(email)`** — 1 órás random tokent generál, `password_reset_tokens` táblába menti, emailt küld.
6. **`resetPassword(token, newPassword)`** — validálja a tokent (van-e, lejárt-e, használták-e), bcrypt-tel hashelje az új jelszót, frissíti a usert, tokent used=true-ra állítja.

```javascript
// src/services/auth.service.js
const bcrypt = require('bcryptjs');
const crypto = require('crypto');
const { OAuth2Client } = require('google-auth-library');
const appleSignin = require('apple-signin-auth');
const pool = require('../db/pool');
const { generateToken } = require('./token.service');
const { sendPasswordResetEmail } = require('./email.service');
const config = require('../config');
const { ConflictError, UnauthorizedError, NotFoundError, ValidationError } = require('../utils/errors');

const googleClient = new OAuth2Client(config.google.clientId);

// Segédfüggvény: aktív előfizetés lekérdezése user_id alapján
async function getActiveSubscription(userId) {
  const result = await pool.query(
    `SELECT * FROM subscriptions
     WHERE user_id = $1
       AND (status = 'active' OR (status = 'trial' AND trial_end > NOW()))
     ORDER BY created_at DESC LIMIT 1`,
    [userId]
  );
  return result.rows[0] || null;
}

async function registerWithEmail(email, password) {
  const existing = await pool.query('SELECT id FROM users WHERE email = $1', [email]);
  if (existing.rows.length > 0) throw new ConflictError('Email already in use');

  const hash = await bcrypt.hash(password, 12);
  const result = await pool.query(
    `INSERT INTO users (email, password_hash, auth_provider)
     VALUES ($1, $2, 'email') RETURNING *`,
    [email, hash]
  );
  const user = result.rows[0];
  const subscription = null;
  const token = generateToken(user, subscription);
  return { token, user: { id: user.id, email: user.email, language: user.language } };
}

async function loginWithEmail(email, password) {
  const result = await pool.query('SELECT * FROM users WHERE email = $1', [email]);
  const user = result.rows[0];
  if (!user) throw new UnauthorizedError('Invalid email or password');

  const valid = await bcrypt.compare(password, user.password_hash);
  if (!valid) throw new UnauthorizedError('Invalid email or password');

  const subscription = await getActiveSubscription(user.id);
  const token = generateToken(user, subscription);
  return { token, user: { id: user.id, email: user.email, language: user.language } };
}

async function loginWithGoogle(idToken) {
  const ticket = await googleClient.verifyIdToken({
    idToken,
    audience: config.google.clientId
  });
  const payload = ticket.getPayload();
  const { sub: googleId, email } = payload;

  let result = await pool.query(
    'SELECT * FROM users WHERE auth_provider = $1 AND auth_provider_id = $2',
    ['google', googleId]
  );
  let user = result.rows[0];

  if (!user) {
    const insert = await pool.query(
      `INSERT INTO users (email, auth_provider, auth_provider_id)
       VALUES ($1, 'google', $2) RETURNING *`,
      [email, googleId]
    );
    user = insert.rows[0];
  }

  const subscription = await getActiveSubscription(user.id);
  const token = generateToken(user, subscription);
  return { token, user: { id: user.id, email: user.email, language: user.language } };
}

async function loginWithApple(identityToken, userData) {
  const appleUser = await appleSignin.verifyIdToken(identityToken, {
    audience: config.apple.clientId,
    ignoreExpiration: false
  });
  const appleId = appleUser.sub;
  const email = appleUser.email || userData?.email;

  let result = await pool.query(
    'SELECT * FROM users WHERE auth_provider = $1 AND auth_provider_id = $2',
    ['apple', appleId]
  );
  let user = result.rows[0];

  if (!user) {
    const insert = await pool.query(
      `INSERT INTO users (email, auth_provider, auth_provider_id)
       VALUES ($1, 'apple', $2) RETURNING *`,
      [email, appleId]
    );
    user = insert.rows[0];
  }

  const subscription = await getActiveSubscription(user.id);
  const token = generateToken(user, subscription);
  return { token, user: { id: user.id, email: user.email, language: user.language } };
}

async function requestPasswordReset(email) {
  const result = await pool.query('SELECT id FROM users WHERE email = $1', [email]);
  if (result.rows.length === 0) {
    // Ne áruljuk el, hogy nincs ilyen email (biztonsági okokból)
    return;
  }
  const userId = result.rows[0].id;
  const token = crypto.randomBytes(32).toString('hex');
  const expiresAt = new Date(Date.now() + 60 * 60 * 1000); // 1 óra

  await pool.query(
    `INSERT INTO password_reset_tokens (user_id, token, expires_at)
     VALUES ($1, $2, $3)`,
    [userId, token, expiresAt]
  );
  await sendPasswordResetEmail(email, token);
}

async function resetPassword(token, newPassword) {
  const result = await pool.query(
    `SELECT * FROM password_reset_tokens
     WHERE token = $1 AND used = FALSE AND expires_at > NOW()`,
    [token]
  );
  if (result.rows.length === 0) throw new ValidationError('Invalid or expired reset token');

  const { id: tokenId, user_id: userId } = result.rows[0];
  const hash = await bcrypt.hash(newPassword, 12);

  await pool.query('UPDATE users SET password_hash = $1 WHERE id = $2', [hash, userId]);
  await pool.query('UPDATE password_reset_tokens SET used = TRUE WHERE id = $1', [tokenId]);
}

module.exports = {
  registerWithEmail, loginWithEmail, loginWithGoogle,
  loginWithApple, requestPasswordReset, resetPassword
};
```

---

### 6.12 lépés: Auth controller és route (`src/controllers/auth.controller.js`, `src/routes/auth.routes.js`)

**Auth controller:**

```javascript
// src/controllers/auth.controller.js
const authService = require('../services/auth.service');

const register = async (req, res, next) => {
  try {
    const result = await authService.registerWithEmail(req.body.email, req.body.password);
    res.status(201).json(result);
  } catch (err) { next(err); }
};

const login = async (req, res, next) => {
  try {
    const result = await authService.loginWithEmail(req.body.email, req.body.password);
    res.json(result);
  } catch (err) { next(err); }
};

const googleLogin = async (req, res, next) => {
  try {
    const result = await authService.loginWithGoogle(req.body.id_token);
    res.json(result);
  } catch (err) { next(err); }
};

const appleLogin = async (req, res, next) => {
  try {
    const result = await authService.loginWithApple(req.body.identity_token, req.body.user);
    res.json(result);
  } catch (err) { next(err); }
};

const forgotPassword = async (req, res, next) => {
  try {
    await authService.requestPasswordReset(req.body.email);
    res.json({ message: 'Ha az email cím regisztrált, küldtünk egy visszaállító linket.' });
  } catch (err) { next(err); }
};

const resetPassword = async (req, res, next) => {
  try {
    await authService.resetPassword(req.body.token, req.body.new_password);
    res.json({ message: 'A jelszó sikeresen megváltoztatva.' });
  } catch (err) { next(err); }
};

module.exports = { register, login, googleLogin, appleLogin, forgotPassword, resetPassword };
```

**Auth route (Joi validációval):**

```javascript
// src/routes/auth.routes.js
const express = require('express');
const router = express.Router();
const Joi = require('joi');
const { validate } = require('../middleware/validate');
const ctrl = require('../controllers/auth.controller');

const registerSchema = Joi.object({
  email: Joi.string().email().required(),
  password: Joi.string().min(8).required()
});

const loginSchema = Joi.object({
  email: Joi.string().email().required(),
  password: Joi.string().required()
});

const forgotSchema = Joi.object({ email: Joi.string().email().required() });

const resetSchema = Joi.object({
  token: Joi.string().required(),
  new_password: Joi.string().min(8).required()
});

router.post('/register',        validate(registerSchema), ctrl.register);
router.post('/login',           validate(loginSchema),    ctrl.login);
router.post('/google',          ctrl.googleLogin);
router.post('/apple',           ctrl.appleLogin);
router.post('/forgot-password', validate(forgotSchema),   ctrl.forgotPassword);
router.post('/reset-password',  validate(resetSchema),    ctrl.resetPassword);

module.exports = router;
```

---

### 6.13 lépés: Events controller és route

**Controller:**

```javascript
// src/controllers/events.controller.js
const pool = require('../db/pool');
const { NotFoundError } = require('../utils/errors');

const getEvents = async (req, res, next) => {
  try {
    const page = Math.max(1, parseInt(req.query.page) || 1);
    const limit = Math.min(50, Math.max(1, parseInt(req.query.limit) || 20));
    const offset = (page - 1) * limit;

    const countResult = await pool.query(
      `SELECT COUNT(*) FROM events WHERE kickoff_time > NOW() AND status != 'cancelled'`
    );
    const total = parseInt(countResult.rows[0].count);

    const result = await pool.query(
      `SELECT e.id, e.home_team, e.away_team, e.kickoff_time, e.status,
              l.name AS league_name, l.country
       FROM events e
       JOIN leagues l ON e.league_id = l.id
       WHERE e.kickoff_time > NOW() AND e.status != 'cancelled'
       ORDER BY e.kickoff_time ASC
       LIMIT $1 OFFSET $2`,
      [limit, offset]
    );

    res.json({ events: result.rows, total, page });
  } catch (err) { next(err); }
};

const getEventMarkets = async (req, res, next) => {
  try {
    const eventId = parseInt(req.params.id);

    const eventResult = await pool.query(
      `SELECT e.id, e.home_team, e.away_team, e.kickoff_time,
              l.name AS league_name
       FROM events e JOIN leagues l ON e.league_id = l.id
       WHERE e.id = $1`,
      [eventId]
    );
    if (!eventResult.rows.length) throw new NotFoundError('Event not found');

    const marketsResult = await pool.query(
      `SELECT m.id, m.name_hu, m.name_en,
              o.id AS outcome_id, o.name_hu AS outcome_name_hu,
              o.name_en AS outcome_name_en, o.best_odds, o.overall_score, o.value_label
       FROM markets m
       JOIN outcomes o ON o.market_id = m.id
       WHERE m.event_id = $1
       ORDER BY m.id, o.overall_score DESC`,
      [eventId]
    );

    // Csoportosítás piacok szerint
    const marketsMap = {};
    for (const row of marketsResult.rows) {
      if (!marketsMap[row.id]) {
        marketsMap[row.id] = {
          id: row.id,
          name_hu: row.name_hu,
          name_en: row.name_en,
          outcomes: []
        };
      }
      marketsMap[row.id].outcomes.push({
        id: row.outcome_id,
        name_hu: row.outcome_name_hu,
        name_en: row.outcome_name_en,
        best_odds: parseFloat(row.best_odds),
        overall_score: parseFloat(row.overall_score),
        value_label: row.value_label
      });
    }

    res.json({ event: eventResult.rows[0], markets: Object.values(marketsMap) });
  } catch (err) { next(err); }
};

module.exports = { getEvents, getEventMarkets };
```

**Route:**

```javascript
// src/routes/events.routes.js
const express = require('express');
const router = express.Router();
const ctrl = require('../controllers/events.controller');

router.get('/', ctrl.getEvents);
router.get('/:id/markets', ctrl.getEventMarkets);

module.exports = router;
```

---

### 6.14 lépés: Outcomes controller és route

```javascript
// src/controllers/outcomes.controller.js
const pool = require('../db/pool');
const { NotFoundError } = require('../utils/errors');

const getOutcome = async (req, res, next) => {
  try {
    const outcomeId = parseInt(req.params.id);
    const result = await pool.query(
      `SELECT o.*, m.name_hu AS market_name_hu, m.name_en AS market_name_en,
              e.home_team, e.away_team, e.kickoff_time,
              l.name AS league_name
       FROM outcomes o
       JOIN markets m ON o.market_id = m.id
       JOIN events e ON m.event_id = e.id
       JOIN leagues l ON e.league_id = l.id
       WHERE o.id = $1`,
      [outcomeId]
    );
    if (!result.rows.length) throw new NotFoundError('Outcome not found');

    const row = result.rows[0];
    res.json({
      id: row.id,
      name_hu: row.name_hu,
      name_en: row.name_en,
      likelihood_score: parseFloat(row.likelihood_score),
      odds_score: parseFloat(row.odds_score),
      overall_score: parseFloat(row.overall_score),
      best_odds: parseFloat(row.best_odds),
      value_label: row.value_label,
      analysis_hu: row.analysis_hu,
      analysis_en: row.analysis_en,
      market: { name_hu: row.market_name_hu, name_en: row.market_name_en },
      event: {
        home_team: row.home_team,
        away_team: row.away_team,
        kickoff_time: row.kickoff_time,
        league_name: row.league_name
      }
    });
  } catch (err) { next(err); }
};

module.exports = { getOutcome };
```

```javascript
// src/routes/outcomes.routes.js
const express = require('express');
const router = express.Router();
const ctrl = require('../controllers/outcomes.controller');
router.get('/:id', ctrl.getOutcome);
module.exports = router;
```

---

### 6.15 lépés: Toplist controller és route

```javascript
// src/controllers/toplist.controller.js
const pool = require('../db/pool');

const getToplist = async (req, res, next) => {
  try {
    const result = await pool.query(
      `SELECT o.id, o.name_hu, o.name_en, o.overall_score, o.best_odds, o.value_label,
              m.name_hu AS market_name_hu, m.name_en AS market_name_en,
              e.home_team, e.away_team, e.kickoff_time,
              l.name AS league_name
       FROM outcomes o
       JOIN markets m ON o.market_id = m.id
       JOIN events e ON m.event_id = e.id
       JOIN leagues l ON e.league_id = l.id
       WHERE o.overall_score >= 7.0
         AND o.best_odds >= 1.5
         AND e.kickoff_time > NOW()
       ORDER BY o.overall_score DESC
       LIMIT 50`
    );
    res.json({ outcomes: result.rows });
  } catch (err) { next(err); }
};

module.exports = { getToplist };
```

```javascript
// src/routes/toplist.routes.js
const express = require('express');
const router = express.Router();
const { authenticate } = require('../middleware/auth');
const { requirePremium } = require('../middleware/requirePremium');
const ctrl = require('../controllers/toplist.controller');

router.get('/', authenticate, requirePremium, ctrl.getToplist);

module.exports = router;
```

---

### 6.16 lépés: Subscription service (`src/services/subscription.service.js`)

**Mit csináljon:**
1. **`startTrial(userId, paymentIdentifier)`** — ellenőrzi, hogy a hashed payment identifier szerepel-e a `used_trials` táblában. Ha igen, megtagadja. Ha nem, létrehozza a trialt és menti a hash-t.
2. **`validateAppleReceipt(userId, receipt)`** — Apple IAP receipt-et validál az Apple szerverein, és frissíti az előfizetést.
3. **`validateGoogleReceipt(userId, receipt, productId)`** — Google Play purchase token-t validál a Google API-val.

```javascript
// src/services/subscription.service.js
const crypto = require('crypto');
const https = require('https');
const pool = require('../db/pool');
const config = require('../config');
const { ValidationError, ConflictError } = require('../utils/errors');

function hashPaymentIdentifier(identifier) {
  return crypto.createHash('sha256').update(identifier).digest('hex');
}

async function startTrial(userId, paymentIdentifier) {
  const hash = hashPaymentIdentifier(paymentIdentifier);

  // Ellenőrzés: használták-e már ezt az azonosítót?
  const existing = await pool.query(
    'SELECT id FROM used_trials WHERE payment_identifier_hash = $1',
    [hash]
  );
  if (existing.rows.length > 0) {
    throw new ConflictError('Trial has already been used for this payment method');
  }

  // Van-e már előfizetése?
  const subCheck = await pool.query(
    'SELECT id FROM subscriptions WHERE user_id = $1', [userId]
  );
  if (subCheck.rows.length > 0) {
    throw new ConflictError('User already has a subscription');
  }

  const now = new Date();
  const trialEnd = new Date(now.getTime() + config.trialDays * 24 * 60 * 60 * 1000);

  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    await client.query(
      `INSERT INTO subscriptions (user_id, status, platform, trial_start, trial_end)
       VALUES ($1, 'trial', 'internal', $2, $3)`,
      [userId, now, trialEnd]
    );
    await client.query(
      'INSERT INTO used_trials (payment_identifier_hash) VALUES ($1)',
      [hash]
    );
    await client.query('COMMIT');
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }

  return { status: 'trial', trial_start: now, trial_end: trialEnd };
}

async function validateAppleReceipt(userId, receiptData) {
  // Apple receipt validálás
  const body = JSON.stringify({
    'receipt-data': receiptData,
    'password': config.apple_iap.sharedSecret,
    'exclude-old-transactions': true
  });

  const verifyWithUrl = (url) => new Promise((resolve, reject) => {
    const req = https.request(url, { method: 'POST' }, (res) => {
      let data = '';
      res.on('data', chunk => data += chunk);
      res.on('end', () => resolve(JSON.parse(data)));
    });
    req.on('error', reject);
    req.write(body);
    req.end();
  });

  let appleResponse = await verifyWithUrl(config.apple_iap.verifyUrl);

  // 21007 = sandbox receipt produktív szerverrel küldve
  if (appleResponse.status === 21007) {
    appleResponse = await verifyWithUrl(config.apple_iap.sandboxUrl);
  }

  if (appleResponse.status !== 0) {
    throw new ValidationError(`Apple receipt validation failed: status ${appleResponse.status}`);
  }

  const latestReceipt = appleResponse.latest_receipt_info?.[0];
  if (!latestReceipt) throw new ValidationError('No valid receipt info from Apple');

  const expiresDate = new Date(parseInt(latestReceipt.expires_date_ms));
  const isActive = expiresDate > new Date();

  await pool.query(
    `INSERT INTO subscriptions (user_id, status, platform, subscription_start, subscription_end, platform_receipt)
     VALUES ($1, $2, 'apple', $3, $4, $5)
     ON CONFLICT (user_id) DO UPDATE
       SET status = EXCLUDED.status,
           subscription_start = EXCLUDED.subscription_start,
           subscription_end = EXCLUDED.subscription_end,
           platform_receipt = EXCLUDED.platform_receipt`,
    [userId, isActive ? 'active' : 'expired', new Date(), expiresDate, receiptData]
  );

  return { status: isActive ? 'active' : 'expired', subscription_end: expiresDate };
}

module.exports = { startTrial, validateAppleReceipt };
```

**Megjegyzés a Google Play validáláshoz:** A Google Play Billing validálás Google API client library-t igényel (`googleapis` npm csomag). A `purchases.subscriptions.get` endpoint-ot kell hívni a service account credential-lel. Ez hasonló logikájú az Apple-hez, de a Google REST API-n keresztül fut. Implementálási lépései azonosak a fentiekkel, csak az API hívás különböző.

---

### 6.17 lépés: User controller (`src/controllers/user.controller.js`)

```javascript
// src/controllers/user.controller.js
const pool = require('../db/pool');
const subService = require('../services/subscription.service');
const { NotFoundError } = require('../utils/errors');

const getProfile = async (req, res, next) => {
  try {
    const userResult = await pool.query(
      'SELECT id, email, auth_provider, language, created_at FROM users WHERE id = $1',
      [req.user.userId]
    );
    if (!userResult.rows.length) throw new NotFoundError('User not found');

    const subResult = await pool.query(
      `SELECT status, platform, trial_start, trial_end,
              subscription_start, subscription_end
       FROM subscriptions WHERE user_id = $1
       ORDER BY created_at DESC LIMIT 1`,
      [req.user.userId]
    );

    res.json({ user: userResult.rows[0], subscription: subResult.rows[0] || null });
  } catch (err) { next(err); }
};

const updateLanguage = async (req, res, next) => {
  try {
    const { language } = req.body;
    const result = await pool.query(
      'UPDATE users SET language = $1 WHERE id = $2 RETURNING id, email, language',
      [language, req.user.userId]
    );
    res.json({ user: result.rows[0] });
  } catch (err) { next(err); }
};

const acceptLegal = async (req, res, next) => {
  try {
    const { device_id, version } = req.body;
    await pool.query(
      `INSERT INTO legal_acceptances (user_id, device_id, version)
       VALUES ($1, $2, $3)`,
      [req.user?.userId || null, device_id, version]
    );
    res.json({ ok: true });
  } catch (err) { next(err); }
};

const startTrial = async (req, res, next) => {
  try {
    const subscription = await subService.startTrial(
      req.user.userId,
      req.body.payment_identifier
    );
    res.json({ subscription });
  } catch (err) { next(err); }
};

const validateReceipt = async (req, res, next) => {
  try {
    const { platform, receipt } = req.body;
    let subscription;
    if (platform === 'apple') {
      subscription = await subService.validateAppleReceipt(req.user.userId, receipt);
    } else if (platform === 'google') {
      // Google Play validálás - ld. subscription.service.js kommentjét
      throw new Error('Google Play validation not yet implemented');
    }
    res.json({ subscription });
  } catch (err) { next(err); }
};

const deleteAccount = async (req, res, next) => {
  try {
    // A CASCADE törlések automatikusan törölik a subscriptions, password_reset_tokens, stb. rekordokat.
    // A used_trials rekordok MEGMARADNAK (visszaélés-megelőzés!)
    await pool.query('DELETE FROM users WHERE id = $1', [req.user.userId]);
    res.json({ ok: true });
  } catch (err) { next(err); }
};

module.exports = { getProfile, updateLanguage, acceptLegal, startTrial, validateReceipt, deleteAccount };
```

**Megjegyzés:** A fiók törlésénél a `used_trials` tábla rekordjai szándékosan maradnak meg. Ez biztosítja, hogy a törölt fiókhoz tartozó payment identifier hash alapján ne lehessen újra trialt igényelni.

---

### 6.18 lépés: User route

```javascript
// src/routes/user.routes.js
const express = require('express');
const router = express.Router();
const Joi = require('joi');
const { authenticate } = require('../middleware/auth');
const { validate } = require('../middleware/validate');
const ctrl = require('../controllers/user.controller');

const langSchema = Joi.object({ language: Joi.string().valid('hu', 'en').required() });
const legalSchema = Joi.object({
  device_id: Joi.string().required(),
  version: Joi.string().required()
});
const trialSchema = Joi.object({ payment_identifier: Joi.string().required() });
const receiptSchema = Joi.object({
  platform: Joi.string().valid('apple', 'google').required(),
  receipt: Joi.string().required()
});

// Profil és fiókkezelés
router.get('/profile',                  authenticate, ctrl.getProfile);
router.put('/language',                 authenticate, validate(langSchema), ctrl.updateLanguage);
router.post('/accept-legal',            validate(legalSchema), ctrl.acceptLegal); // auth opcionális (guest is elfogadhat)
router.delete('/',                      authenticate, ctrl.deleteAccount);

// Előfizetés
router.post('/subscription/start-trial',      authenticate, validate(trialSchema), ctrl.startTrial);
router.post('/subscription/validate-receipt', authenticate, validate(receiptSchema), ctrl.validateReceipt);

module.exports = router;
```

---

### 6.19 lépés: Express app összeállítása (`src/app.js`)

```javascript
// src/app.js
const express = require('express');
const helmet = require('helmet');
const cors = require('cors');
const rateLimit = require('express-rate-limit');
const config = require('./config');
const logger = require('./utils/logger');
const { AppError } = require('./utils/errors');

// Routes
const authRoutes = require('./routes/auth.routes');
const eventsRoutes = require('./routes/events.routes');
const outcomesRoutes = require('./routes/outcomes.routes');
const toplistRoutes = require('./routes/toplist.routes');
const userRoutes = require('./routes/user.routes');

const app = express();

// Security middleware
app.use(helmet());
app.use(cors({ origin: '*' })); // Flutter-ből natív hívás, CORS kevésbé kritikus, de jó szokás
app.use(express.json({ limit: '1mb' }));

// Rate limiting
const limiter = rateLimit({
  windowMs: config.rateLimit.windowMs,
  max: config.rateLimit.max,
  standardHeaders: true,
  legacyHeaders: false
});
app.use('/api/', limiter);

// Request naplózás
app.use((req, res, next) => {
  logger.debug(`${req.method} ${req.path}`);
  next();
});

// Routes
app.use('/api/auth',        authRoutes);
app.use('/api/events',      eventsRoutes);
app.use('/api/outcomes',    outcomesRoutes);
app.use('/api/toplist',     toplistRoutes);
app.use('/api/user',        userRoutes);
app.use('/api/subscription', userRoutes); // Ugyanaz a router kezeli

// Health check
app.get('/health', (req, res) => res.json({ status: 'ok', timestamp: new Date().toISOString() }));

// 404 kezelés
app.use((req, res) => res.status(404).json({ error: 'Not found' }));

// Globális hibakezelő
app.use((err, req, res, next) => {
  if (err.isOperational) {
    logger.warn(`Operational error: ${err.message}`, { statusCode: err.statusCode });
    return res.status(err.statusCode).json({ error: err.message });
  }
  logger.error('Unexpected error', err);
  res.status(500).json({ error: 'Internal server error' });
});

module.exports = app;
```

---

### 6.20 lépés: HTTP szerver indítása (`src/server.js`)

```javascript
// src/server.js
const app = require('./app');
const config = require('./config');
const logger = require('./utils/logger');
const pool = require('./db/pool');

const server = app.listen(config.port, () => {
  logger.info(`BetShark Backend running on port ${config.port} [${config.nodeEnv}]`);
});

// Graceful shutdown
process.on('SIGTERM', () => {
  logger.info('SIGTERM received. Shutting down gracefully...');
  server.close(async () => {
    await pool.end();
    logger.info('DB pool closed. Process exiting.');
    process.exit(0);
  });
});
```

---

## 7. Futtatás és deployment

### Lokális futtatás

```bash
cd betshark_backend
npm install
cp .env.example .env
# Töltsd ki a .env fájlt!
node src/server.js
```

Fejlesztési módban (automatikus újraindítással):

```bash
npm install -D nodemon
npm run dev
```

### API tesztelés (curl példák)

```bash
# Regisztráció
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"Jelszo123"}'

# Bejelentkezés
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"Jelszo123"}'

# Események listázása (publikus)
curl http://localhost:3000/api/events?page=1&limit=10

# Toplist (tokennel)
curl http://localhost:3000/api/toplist \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

### Produktív deployment

**PM2 (ajánlott Node.js process manager):**

```bash
npm install -g pm2
pm2 start src/server.js --name betshark-backend -i 2
pm2 save
pm2 startup
```

**Docker:**

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY src ./src
EXPOSE 3000
CMD ["node", "src/server.js"]
```

```bash
docker build -t betshark-backend .
docker run -d -p 3000:3000 --env-file .env --name betshark-backend betshark-backend
```

**Nginx reverse proxy (HTTPS):**

```nginx
server {
    listen 443 ssl;
    server_name api.betshark.app;

    ssl_certificate     /etc/letsencrypt/live/api.betshark.app/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.betshark.app/privkey.pem;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Fontos megfontolások

- **JWT secret:** Minimum 32 karakteres, véletlenszerű string. Soha ne kerüljön verziókezelőbe.
- **Jelszó-visszaállítás URL:** A `sendPasswordResetEmail`-ben lévő URL-t állítsd be a valós domain-re.
- **Apple Sign-In:** Az `apple-signin-auth` csomag a `.p8` privát kulcsfájlt igényli. Ezt a fájlt soha ne commitold, csak szerver oldali secret-ként tárold.
- **Google Play:** A Google service account JSON-t szintén secret-ként kezeld.
- **Subscription conflict:** Az `ON CONFLICT` záradék a `subscriptions` táblában azt feltételezi, hogy minden usernek legfeljebb egy aktív előfizetési rekordja van. Ha több párhuzamos platform support kell (pl. egyszerre Apple + Google), a logika bővítésre szorul.
- **Trial abuse:** A `used_trials` hash összehasonlítás SHA-256 alapú. Az Apple esetén a payment identifier a `originalTransactionId`, Google esetén a `purchaseToken` hash-elhető.
