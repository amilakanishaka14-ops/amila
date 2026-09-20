# Pen Tube — Next-Gen AI Creator OS

Pen Tube is a static, GitHub Pages-compatible frontend backed by Firebase Authentication, Firestore, and callable Cloud Functions. It includes three creator tools: Thumbnail Prompt Generator, SEO & Keywords Builder, and Script Outline Generator.

## What is included

- `index.html` — landing page and About section
- `app.html` — AI Engine with three independent tools
- `dashboard.html` — Creator Vault
- `auth.html` — email/password + Google sign-in + password reset UI
- `privacy.html` — privacy/data-processing page
- `admin.html` — claim-protected admin console
- `css/style.css` — responsive cyberpunk/glassmorphism design system
- `js/` — auth, cloud API wrapper, Firestore Vault, admin UI, app controller, utilities
- `firebase/` — web config template, Firestore rules, Storage rules
- `functions/` — secure server-side Gemini/YouTube integration and admin stats

## Important implementation notes

### Custom Uiverse components

The source prompt referenced five Uiverse components, but the supplied text contained placeholders rather than their actual HTML/CSS. This project therefore includes functional Pen Tube-native equivalents for:

- Download action buttons
- Sparkle Generate button
- Skeleton loader
- Login/register UI
- File upload dropzone

To preserve actual Uiverse source code exactly, replace only those specific component blocks after you have the original snippets.

### AI provider

The server currently uses `gemini-3.8-flash`, because current Gemini model documentation lists it as a stable Flash model. Do not hard-code an old retired model name such as `gemini-1.5-flash` into production code.

The browser never receives the Gemini API secret. The frontend calls the Firebase callable function `generateAI`; the function reads `GEMINI_API_KEY` from a server-side secret.

### “Free” usage

Pen Tube is designed around free-tier-compatible infrastructure, but external providers impose their own quotas/limits. The application therefore implements a 20-generation-per-user-per-day application cap and displays a “Free tier / limits apply” notice. This is not a promise of unlimited provider usage.

### YouTube research

`searchYouTube` uses the YouTube Data API search endpoint from a callable function, with a 30-minute Firestore cache per normalized query. Search results are research signals only. The UI does not claim to expose an official “trending keyword score”.

### Thumbnail preview

The “Generate Preview” action uses the public Pollinations image endpoint. Preview generation is optional and failure does not block Gemini generation.

## Step 1 — Create Firebase project

1. Open Firebase Console.
2. Create a project.
3. Add a Web App.
4. Copy the Web App configuration into `firebase/firebase-config.js`.
5. Keep the placeholder values until you have your real config. Never invent credentials.

## Step 2 — Enable Authentication

Firebase Console → Authentication → Sign-in method:

- Email/Password
- Google

Add your local/GitHub Pages domains under Authorized domains.

## Step 3 — Create Firestore

Create a Firestore database in production mode.

Deploy the rules in `firebase/firestore.rules`.

## Step 4 — Optional Storage

Enable Firebase Storage if you plan to add persistent creator asset uploads. The included Storage rules only allow authenticated users to access their own `users/{uid}/assets/...` images.

The current reference-image dropzone is deliberately local-preview-only; files are not automatically uploaded.

## Step 5 — Install Firebase CLI

Install the current Firebase CLI locally, then run from the project root:

```bash
firebase login
firebase use --add
cd functions
npm install
cd ..
```

## Step 6 — Configure server secrets

Set secrets using the Firebase CLI instead of putting keys in frontend code:

```bash
firebase functions:secrets:set GEMINI_API_KEY
firebase functions:secrets:set YOUTUBE_API_KEY
```

Deploy functions:

```bash
firebase deploy --only functions
```

## Step 7 — Deploy Firestore + Storage rules

```bash
firebase deploy --only firestore:rules,storage
```

## Step 8 — Admin setup

The default admin display name shown in the UI is “Amila Kanishka”. The display name is NOT authorization.

Create the admin account through Firebase Authentication. Copy its real Firebase Auth UID. Then, server-side, run:

```bash
cd functions/scripts
GOOGLE_APPLICATION_CREDENTIALS=/absolute/path/service-account.json node setAdminClaim.js REAL_FIREBASE_UID
```

Never place the service-account JSON in the GitHub repository or public hosting.

After the claim is set, the administrator should sign out and sign back in so the ID token is refreshed.

## Step 9 — Deploy GitHub Pages

For the static frontend:

1. Push the repository to GitHub.
2. Open Repository Settings → Pages.
3. Select the branch/folder containing the Pen Tube static files.
4. Deploy.

The Firebase callable functions do NOT run on GitHub Pages; they run on Firebase Cloud Functions. This split is intentional so private provider secrets stay server-side.

## Step 10 — Test checklist

- Firebase config inserted
- Email/password sign-in works
- Google sign-in works
- Password reset works
- Vault loads only for the signed-in owner
- Generate Thumbnail Prompt works through `generateAI`
- Generate SEO works through `generateAI`
- Generate Script works through `generateAI`
- AI output parsing rejects malformed JSON safely
- YouTube research returns cached/fresh results
- Pollinations preview can fail without crashing the page
- TXT and JSON downloads work
- Copy-to-clipboard shows success/error toast
- Vault save/delete works
- Admin route denies users without `admin=true`
- Admin statistics load only from the secure callable function
- Mobile navigation and forms work at 320px+

## Local testing

Because ES Modules, Firebase Auth, and module imports should run over HTTP, test with a local static server rather than opening HTML files with `file://`.

For example:

```bash
python -m http.server 8080
```

Then open:

`http://127.0.0.1:8080/`

## Security model

- Browser receives only public Firebase Web App configuration.
- Gemini and YouTube API secrets stay in Cloud Functions Secret Manager.
- AI and YouTube calls go through callable functions.
- Firestore rules enforce per-user vault ownership.
- Admin statistics require the `admin=true` Firebase custom claim on the request token.
- User-generated content is inserted into the DOM with `textContent`/escaped rendering patterns instead of trusting arbitrary HTML.
- Provider errors are translated into user-facing messages rather than exposing server secrets.

## Free-tier caveat

“Free for users” means Pen Tube does not require a paid subscription in this starter architecture. External services may change their free quotas, pricing, rate limits, model availability, or policies. Monitor provider limits before public launch.

## Production hardening before public launch

- Configure Firebase App Check.
- Restrict API keys by API/referrer where supported.
- Add abuse/rate-limit monitoring beyond the application daily cap.
- Add a proper account deletion workflow that deletes user vault records before deleting the Auth account.
- Consider adding a backend task queue if generation traffic grows substantially.
- Add automated tests and CI for rules/functions.
