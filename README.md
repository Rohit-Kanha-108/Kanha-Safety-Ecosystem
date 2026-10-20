# Kanha Safety Ecosystem

**Women Safety & Emergency Response App — Hack for Social Cause 2027**

Kanha is a mobile safety app that works as a digital first-responder for women. One tap sends an SOS to trusted contacts, shares live location, and starts recording audio/video evidence automatically, even if the app is sent to the background.

> Built with Flutter and Supabase. Android is the primary target (background recording and foreground services need a real device / APK).

---

## Table of Contents

1. [Problem & Approach](#problem--approach)
2. [Features](#features)
3. [Tech Stack](#tech-stack)
4. [Architecture](#architecture)
5. [Project Structure](#project-structure)
6. [Setup Guide](#setup-guide)
7. [Environment Variables](#environment-variables)
8. [Supabase Setup](#supabase-setup)
9. [Android Permissions](#android-permissions)
10. [Running & Building](#running--building)
11. [Demo Flow](#demo-flow)
12. [Known Limitations & Roadmap](#known-limitations--roadmap)
13. [Security Notes](#security-notes)
14. [License](#license)

---

## Problem & Approach

In an emergency, a woman may not be able to call, type, or even unlock her phone. Kanha reduces that to **one touch** and then works on its own: it alerts contacts, shares where she is, keeps checking on her during journeys, and secures evidence for later.

---

## Features

### 1. One-Touch Panic Alerts (SOS)
- Large SOS button on the home screen triggers an SOS event instantly.
- Alert is sent to the user's **enrolled emergency contacts** via SMS (Supabase Edge Function + Fast2SMS) and in-app realtime alerts.
- Contacts see an incoming-alert overlay with a live map when they have the app open.
- Contacts can reply with status updates / messages (`sos_alert_statuses`, `sos_contact_messages`).
- Offline-safe SOS queue (`sos_queue_service`) retries when connectivity returns.
- SOS runs inside an Android **foreground service**, so it continues when the app is in the background.

### 2. Live Location Tracking
- During an active SOS, the device GPS position is pushed to Supabase every **10 seconds** (`sos_location_updates`).
- Emergency contacts subscribe via Supabase Realtime and see the moving location on Google Maps.
- **Travel Tracker / Journey Timer:** the user sets a destination (Google Places search) and an expected duration. If the timer expires or the user misses check-ins, contacts are alerted (`send-journey-alert`).
- **Periodic safety check-ins:** during a journey the app asks "Are you safe?" at a chosen interval. A missed check-in triggers a retry; a second missed check-in **auto-triggers SOS**.

### 3. Automated Audio/Video Evidence Collection
- Recording starts automatically as soon as SOS is triggered.
- Recording is saved in **chunks** (configurable chunk size and max duration) so evidence is preserved even if the phone is lost or the app is killed.
- Files are uploaded to a **private** Supabase Storage bucket (`incident-recordings`), scoped per user with Row Level Security.
- **Evidence Vault** screen lists recordings and is protected by a PIN.

### 4. Stealth Mode
- App can disguise itself as a **Calculator** (launcher icon/label switch via Android activity-alias).
- Blackout screen overlay during SOS so an attacker cannot see that recording/alerting is in progress.
- The same stealth PIN protects the Evidence Vault.

### 5. Other
- Email/password and Google Sign-In authentication (Supabase Auth).
- Emergency contact enrollment with **QR code / SMS confirmation** so contacts explicitly opt in.
- Alerts timeline, notification preferences, onboarding with permission requests.

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| Frontend | Flutter (Dart ^3.9), `go_router`, `sizer`, `google_fonts` |
| Backend | Supabase — Postgres, Auth, Realtime, Storage, Edge Functions (Deno/TypeScript) |
| Maps & Places | Google Maps SDK (`google_maps_flutter`), Google Places API (`google_places_flutter`) |
| SMS | Fast2SMS (India) from Supabase Edge Functions; Twilio configured as an alternative |
| Location | `geolocator` |
| Background work | `flutter_foreground_task` (SOS + journey foreground services) |
| Notifications | `flutter_local_notifications` (check-in prompts, SOS alerts) |
| Evidence capture | `camera`, `record`, `path_provider` |
| Device sensors | `sensors_plus`, `battery_plus`, `vibration` |
| Permissions | `permission_handler` |
| Misc | `qr_flutter`, `shared_preferences`, `connectivity_plus`, `dio`, `intl` |

---

## Architecture

```
┌─────────────────────────────┐
│        Flutter App          │
│  Home · Contacts · Travel   │
│  Vault · Settings · Stealth │
└──────────────┬──────────────┘
               │ supabase_flutter
               ▼
┌──────────────────────────────────────────────┐
│                   Supabase                    │
│  Auth │ Postgres (RLS) │ Realtime │ Storage  │
│                                               │
│  Edge Functions                               │
│   ├─ send-sos-alert        → SMS to contacts  │
│   ├─ send-journey-alert    → SMS on overdue   │
│   └─ send-enrollment-sms   → contact opt-in   │
└──────────────┬───────────────────────────────┘
               │ HTTPS
               ▼
        Fast2SMS / Twilio  →  Emergency contacts' phones
```

**SOS flow:** SOS tap → create `sos_events` row → start foreground service → start chunked recording → push GPS every 10 s → call `send-sos-alert` → contacts receive SMS + realtime overlay → user cancels SOS → recording finalised and saved in the vault.

---

## Project Structure

```
kanha/
├── android/                         # Android config (manifest, foreground service, stealth aliases)
├── ios/                             # iOS config
├── web/                             # Web shell (preview only)
├── assets/images/                   # Static images / logos
├── env.json                         # Environment variables (use placeholders in git)
├── pubspec.yaml
├── lib/
│   ├── main.dart                    # Entry point
│   ├── core/app_export.dart         # Shared exports
│   ├── routes/app_routes.dart       # go_router configuration
│   ├── theme/app_theme.dart
│   ├── services/                    # Business logic
│   │   ├── supabase_service.dart        # DB / realtime / edge-function calls
│   │   ├── sos_location_service.dart    # 10 s GPS broadcast during SOS
│   │   ├── media_recording_service.dart # Chunked audio/video evidence
│   │   ├── sos_foreground_service*.dart # Background SOS (native + stub)
│   │   ├── journey_foreground_service*.dart
│   │   ├── periodic_checkin_service.dart
│   │   ├── stealth_mode_service.dart
│   │   ├── sos_queue_service.dart       # Offline SOS retry
│   │   └── permission_service.dart
│   ├── presentation/
│   │   ├── splash_screen/
│   │   ├── sign_up_login_screen/
│   │   ├── onboarding_screen/           # Permission onboarding
│   │   ├── home_screen/                 # SOS button, live map, activity
│   │   ├── emergency_contacts_screen/   # Contacts + QR/SMS enrollment
│   │   ├── enrollment_response_screen/  # Public opt-in page for contacts
│   │   ├── travel_tracker_screen/       # Journey timer + destination picker
│   │   ├── evidence_vault_screen/       # PIN-protected recordings
│   │   ├── incident_recording_screen/
│   │   ├── alerts_timeline_screen/
│   │   ├── notification_preferences_screen/
│   │   ├── settings_screen/
│   │   ├── calculator_screen/           # Stealth disguise
│   │   └── future_integrations_screen/
│   └── widgets/                     # Reusable UI components
└── supabase/
    ├── migrations/                  # SQL schema + RLS + storage bucket
    └── functions/                   # Edge Functions (Deno)
        ├── send-sos-alert/
        ├── send-journey-alert/
        └── send-enrollment-sms/
```

---

## Setup Guide

### Prerequisites
- Flutter SDK `^3.38` / Dart `^3.9`
- Android Studio (Android SDK) and a **physical Android phone** for full testing
- A [Supabase](https://supabase.com) project
- [Supabase CLI](https://supabase.com/docs/guides/cli) (for migrations and edge functions)
- Google Cloud project with Maps SDK for Android, Maps JavaScript API (web preview), and Places API enabled
- A [Fast2SMS](https://www.fast2sms.com) account (or Twilio) for SMS delivery

### Steps

1. **Clone the repository**
   ```bash
   git clone https://github.com/Rohit-Kanha-108/Kanha-Safety-Ecosystem.git
   cd Kanha-Safety-Ecosystem
   ```

2. **Install dependencies**
   ```bash
   flutter pub get
   ```

3. **Create your environment file.** Copy the template and fill in your own values (see [Environment Variables](#environment-variables)):
   ```bash
   cp env.example.json env.json
   ```
   > This project reads configuration from `env.json` via `--dart-define-from-file`. It does **not** use a `.env` file.

4. **Set up Supabase** — follow [Supabase Setup](#supabase-setup).

5. **Add your Google Maps key for Android.** In `android/app/src/main/AndroidManifest.xml`, replace the value of `com.google.android.geo.API_KEY` with your own restricted key.

6. **Run the app**
   ```bash
   flutter run --dart-define-from-file=env.json
   ```

---

## Environment Variables

Create `env.json` in the project root. **Never commit real keys** — commit only an `env.example.json` with placeholders.

```json
{
  "SUPABASE_URL": "https://YOUR-PROJECT-ref.supabase.co",
  "SUPABASE_ANON_KEY": "YOUR_SUPABASE_ANON_KEY",
  "GOOGLE_WEB_CLIENT_ID": "YOUR_GOOGLE_WEB_CLIENT_ID.apps.googleusercontent.com",
  "GOOGLE_MAPS_API_KEY": "YOUR_GOOGLE_MAPS_API_KEY",
  "GOOGLE_PLACES_API_KEY": "YOUR_GOOGLE_PLACES_API_KEY",
  "TWILIO_ACCOUNT_SID": "",
  "TWILIO_AUTH_TOKEN": "",
  "TWILIO_PHONE_NUMBER": ""
}
```

| Variable | Required | Used for |
|----------|----------|----------|
| `SUPABASE_URL` | Yes | Supabase project URL |
| `SUPABASE_ANON_KEY` | Yes | Supabase public anon key |
| `GOOGLE_WEB_CLIENT_ID` | For Google Sign-In | OAuth web client ID |
| `GOOGLE_MAPS_API_KEY` | Yes | Maps rendering |
| `GOOGLE_PLACES_API_KEY` | Yes | Destination search in Travel Tracker |
| `TWILIO_*` | Optional | Alternative SMS provider |

### Edge Function secrets (set in Supabase, not in `env.json`)

| Secret | Used by | Purpose |
|--------|---------|---------|
| `FAST2SMS_API_KEY` | all three functions | Sending SMS |
| `SUPABASE_SERVICE_ROLE_KEY` | all three functions | Server-side DB access (auto-provided by Supabase) |
| `FCM_SERVICE_ACCOUNT_JSON` | `send-sos-alert` | Optional; push notifications (currently disabled) |

```bash
supabase secrets set FAST2SMS_API_KEY=your_key_here
```

---

## Supabase Setup

1. **Create a project** at [supabase.com](https://supabase.com) and copy the Project URL and anon key into `env.json`.

2. **Link the CLI and apply migrations**
   ```bash
   supabase login
   supabase link --project-ref YOUR_PROJECT_REF
   supabase db push
   ```
   Migrations in `supabase/migrations/` create:

   | Table | Purpose |
   |-------|---------|
   | `user_profiles` | Profile, phone, settings |
   | `emergency_contacts` | Trusted contacts per user |
   | `enrollment_confirmations` | Contact opt-in tokens/status |
   | `sos_events` | Each SOS incident |
   | `sos_location_updates` | Live GPS pings during SOS |
   | `sos_alert_statuses` | Contact acknowledgements |
   | `sos_contact_messages` | Messages from contacts during SOS |
   | `sos_push_notifications` | Push notification queue |
   | `journey_timers` | Travel Tracker sessions |
   | `incident_recordings` | Evidence metadata |

   All tables use **Row Level Security**. The migration also creates the private storage bucket **`incident-recordings`**, restricted so each user can only access files in their own folder (`<user_id>/...`).

3. **Deploy Edge Functions**
   ```bash
   supabase functions deploy send-sos-alert
   supabase functions deploy send-journey-alert
   supabase functions deploy send-enrollment-sms
   ```

4. **Set secrets** (see above): `supabase secrets set FAST2SMS_API_KEY=...`

5. **Enable Realtime** for `sos_location_updates`, `sos_events`, `sos_alert_statuses`, `sos_contact_messages` and `journey_timers` (Dashboard → Database → Replication) if not already enabled by the migrations.

6. **Authentication:** enable Email provider; for Google Sign-In enable the Google provider and add your Web Client ID. Add your site URL under Auth → URL Configuration.

---

## Android Permissions

The app requests the following at onboarding. **All must be allowed** for the full SOS experience:

| Permission | Why |
|-----------|-----|
| Location (fine/coarse) | Live location sharing |
| Camera | Video evidence |
| Microphone | Audio evidence |
| Notifications (Android 13+) | Foreground-service and check-in notifications |
| Contacts | Pick emergency contacts |
| Ignore battery optimisation | Keeps SOS alive in background (recommended) |

The manifest declares foreground service types `dataSync | location | camera | microphone` so recording and tracking continue with the screen off.

> **Note:** Real camera/microphone recording, foreground services and background location work only on a **real Android device / APK**, not in a web browser preview.

---

## Running & Building

```bash
# Debug run
flutter run --dart-define-from-file=env.json

# Release APK
flutter build apk --release --dart-define-from-file=env.json
```

APK output: `build/app/outputs/flutter-apk/app-release.apk`

---

## Demo Flow

1. Sign up / log in and complete permission onboarding.
2. Add an emergency contact → contact receives an SMS/QR to confirm enrollment.
3. Press the **SOS** button → contacts receive an SMS and see live location; recording starts silently.
4. Cancel SOS → open **Evidence Vault** (PIN) → recording is listed.
5. Start a **Travel Tracker** journey with a destination and check-in interval → miss two check-ins to see automatic SOS.
6. Enable **Stealth Mode** in Settings → app appears as a Calculator.

---

## Known Limitations & Roadmap

Being transparent about the current state of the project:

- **Alerts go to personal emergency contacts**, not directly to police/authorities. Integration with emergency services (e.g. 112 / 181 helplines) is planned.
- **Live location sharing** is active during SOS events; continuous background sharing for the entire journey is being extended.
- **Push notifications (FCM)** are currently disabled; realtime alerts reach contacts who have the app open, and everyone else is reached by SMS.
- **Evidence encryption at rest** (client-side) and stricter vault access rules are planned. Today, files sit in a private bucket protected by RLS and the app-level PIN.
- **iOS** background recording is not yet tested; Android is the supported platform.
- Recording reliability depends on OEM battery policies; users should disable battery optimisation for the app.

---

## Security Notes

- Never commit `env.json` with real keys. Add it to `.gitignore` and share only `env.example.json`.
- Restrict the Google Maps/Places keys by **package name + SHA-1** (Android) and **HTTP referrer** (web).
- Keep `FAST2SMS_API_KEY` and the Supabase **service role** key only in Supabase secrets — never in the Flutter app.
- Rotate any key that has ever been shared publicly before making the repository public.

---

## Team

Built for **Hack for Social Cause 2027** by the Kanha team.

## License

This project is licensed under the MIT License — see the `LICENSE` file for details.
