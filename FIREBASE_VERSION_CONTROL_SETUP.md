# Firebase Realtime Version Control Setup Guide (Pure `.env` Architecture)

> **Purpose**: A complete, reusable blueprint for integrating Firebase Realtime Database app version control into Flutter projects without hardcoding credentials or relying solely on platform-specific configuration files (`google-services.json` / `GoogleService-Info.plist`).
>
> **Target Audience**: Developers setting up version control, forced update locks, and Firebase connectivity across mobile/web Flutter projects.

---

## Table of Contents
1. [Architecture Overview](#1-architecture-overview)
2. [Firebase Console Configuration](#2-firebase-console-configuration)
3. [Realtime Database Schema](#3-realtime-database-schema)
4. [Setting Up `.env` Configuration](#4-setting-up-env-configuration)
5. [Code Implementation Components](#5-code-implementation-components)
6. [Update Behavior & UX Rules](#6-update-behavior--ux-rules)
7. [Testing & Verification Guide](#7-testing--verification-guide)
8. [5-Minute Quick-Start for Upcoming Projects](#8-5-minute-quick-start-for-upcoming-projects)

---

## 1. Architecture Overview

```
                      ┌────────────────────────────────────────┐
                      │ Firebase Realtime Database             │
                      │ Path: /AETRAM_ERP                      │
                      │  - version: "1.0.1"                    │
                      │  - isMandatoryUpdate: true/false       │
                      │  - url: "https://store-or-apk-link"    │
                      └──────────────────┬─────────────────────┘
                                         │ Realtime Stream
                                         ▼
                      ┌────────────────────────────────────────┐
                      │ VersionControlService                  │
                      │ (Initializes Firebase from .env)       │
                      │ (Fallback: Realtime REST API)          │
                      └──────────────────┬─────────────────────┘
                                         │ Emits AppVersionData
                                         ▼
                      ┌────────────────────────────────────────┐
                      │ VersionControlNotifier (Riverpod)      │
                      │ Compares: AppConstants.appVersion vs   │
                      │           remoteData.version           │
                      └──────────────────┬─────────────────────┘
                                         │ state.shouldShowPopup
                                         ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ VersionControlGate (MaterialApp.builder Stack)                               │
│                                                                              │
│  ├── Optional Update:                                                        │
│  │    - Shows AppUpdateDialog                                                │
│  │    - "Update Now" launches store URL via url_launcher                     │
│  │    - "Remind Me Later" dismisses dialog for current session               │
│  │                                                                           │
│  └── Mandatory Update (isMandatoryUpdate == true):                           │
│       - Shows Critical Update Dialog with red security badge                 │
│       - PopScope(canPop: false) locks system back button & gestures          │
│       - Full-screen barrier blocks all UI interactions                       │
│       - No dismiss button — user CANNOT proceed until app is updated         │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Firebase Console Configuration

### Step 2.1: Create or Open a Firebase Project
1. Navigate to the [Firebase Console](https://console.firebase.google.com/).
2. Click **Add project** (or select your existing project).
3. Name your project (e.g., `Aetram-Assist` or `My-Project`).
4. Google Analytics can be toggled on or off as needed.

### Step 2.2: Create a Realtime Database
1. In the Firebase Console left menu, navigate to **Build** ➜ **Realtime Database**.
2. Click **Create Database**.
3. Select your database location (e.g., `United States (us-central1)` or `Singapore (asia-southeast1)`).
4. For security rules, select **Start in test mode** for initial setup, then lock it down as shown below.

### Step 2.3: Configure Database Security Rules
To allow your app clients to read the version info without authentication while preventing unauthorized writes:

```json
{
  "rules": {
    "AETRAM_ERP": {
      ".read": true,
      ".write": "auth != null && auth.token.admin === true"
    }
  }
}
```
*(If you do not use Firebase Auth for admin updates, keep `.write: false` and update values directly from the Firebase Console).*

### Step 2.4: Obtain Firebase Web / App Credentials
You do **not** need to download `google-services.json` if configuring dynamically via `.env`:
1. In Firebase Console, click the **Settings Gear** ⚙️ ➜ **Project settings**.
2. Under the **General** tab, scroll down to **Your apps**.
3. Click the **Web** icon `</>` (or your platform app) to register an app named e.g. `version-controller`.
4. Copy the `firebaseConfig` object keys:
   - `apiKey`
   - `appId`
   - `messagingSenderId`
   - `projectId`
   - `storageBucket`
   - `databaseURL` (Found under **Realtime Database** header, e.g. `https://<project-id>-default-rtdb.firebaseio.com/`).

---

## 3. Realtime Database Schema

In the Firebase Realtime Database Data tab, create a parent node matching your configured path (default: `AETRAM_ERP`).

### Exact Structure:

```json
{
  "AETRAM_ERP": {
    "isMandatoryUpdate": false,
    "url": "https://play.google.com/store/apps/details?id=com.aetram.assist",
    "version": "1.0.0",
    "title": "New Update Available",
    "releaseNotes": "Bug fixes and performance improvements."
  }
}
```

### Node Field Specifications:
| Field | Type | Description |
| :--- | :--- | :--- |
| `isMandatoryUpdate` | `boolean` | When `true`, completely locks the app with an un-dismissible dialog until updated. |
| `url` | `string` | The Play Store, App Store, TestFlight, or APK direct download link. |
| `version` | `string` | The target release version in semantic format (e.g., `"1.0.1"`, `"1.1.0"`). |
| `title` | `string` *(optional)* | Custom heading for the modal. Defaults to *"Critical Update Required"* or *"New Update Available"*. |
| `releaseNotes` | `string` *(optional)* | Custom description shown to the user. |

---

## 4. Setting Up `.env` Configuration

Add the following variables to your `.env` file at the root of your Flutter project:

```env
# ==============================================================================
# FIREBASE REALTIME DATABASE & VERSION CONTROL
# ==============================================================================
FIREBASE_API_KEY=AIzaSyAxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
FIREBASE_APP_ID=1:1234567890:android:abcdef123456
FIREBASE_PROJECT_ID=your-firebase-project-id
FIREBASE_MESSAGING_SENDER_ID=1234567890
FIREBASE_DATABASE_URL=https://your-firebase-project-id-default-rtdb.firebaseio.com
FIREBASE_STORAGE_BUCKET=your-firebase-project-id.appspot.com
FIREBASE_IOS_BUNDLE_ID=com.aetram.assist
FIREBASE_ANDROID_PACKAGE_NAME=com.aetram.assist
FIREBASE_VERSION_CONTROL_PATH=AETRAM_ERP
```

> [!IMPORTANT]
> - Never commit `.env` containing production API keys to public repositories.
> - Ensure `.env` is listed under `assets:` in `pubspec.yaml`:
>   ```yaml
>   flutter:
>     assets:
>       - .env
>   ```

---

## 5. Code Implementation Components

### 5.1 Single Source of Truth: `AppConstants`
Located in `lib/core/constants/app_constants.dart`:
```dart
class AppConstants {
  // App Version (Update here whenever releasing a new version)
  static const String appVersion = '1.0.0';
  static const int appBuildNumber = 1;

  /// Compares [installedVersion] vs [remoteVersion].
  /// Returns `true` if installed is strictly smaller than remote.
  static bool isVersionSmaller(String installedVersion, String remoteVersion) {
    ...
  }
}
```

### 5.2 Environment Loader: `Env`
Located in `lib/constant/env.dart`:
```dart
class Env {
  static String get firebaseApiKey => _v('FIREBASE_API_KEY');
  static String get firebaseAppId => _v('FIREBASE_APP_ID');
  static String get firebaseProjectId => _v('FIREBASE_PROJECT_ID');
  static String get firebaseMessagingSenderId => _v('FIREBASE_MESSAGING_SENDER_ID');
  static String get firebaseDatabaseUrl => _v('FIREBASE_DATABASE_URL');
  static String get firebaseVersionControlPath => _v('FIREBASE_VERSION_CONTROL_PATH', 'AETRAM_ERP');

  static bool get isFirebaseConfigured =>
      firebaseApiKey.isNotEmpty && firebaseAppId.isNotEmpty && firebaseProjectId.isNotEmpty;
}
```

### 5.3 Hybrid Service with REST Fallback
Located in `lib/core/version_control/services/version_control_service.dart`:
- Initializes `Firebase.initializeApp(options: FirebaseOptions(...))` dynamically from `.env`.
- Binds to `FirebaseDatabase.instance.ref(path).onValue` for instant socket updates.
- If native plugins or credentials are not present, automatically falls back to HTTP REST polling (`${databaseURL}/${path}.json`).

### 5.4 Reactive Riverpod Provider
Located in `lib/core/version_control/providers/version_control_provider.dart`:
```dart
final versionControlNotifierProvider =
    NotifierProvider<VersionControlNotifier, VersionControlState>(
  VersionControlNotifier.new,
);
```

### 5.5 Design System Compliant Dialog & Gate
- **`AppUpdateDialog`**: Built to the exact specifications of `UI_DESIGN_SYSTEM.md` (colors, typography, spacing, `HrmsToneIcon`, `HrmsButton`).
- **`VersionControlGate`**: Placed inside `MaterialApp.builder` in `lib/app.dart`. Works globally across all screens without inserting dialog triggers on each route.

---

## 6. Update Behavior & UX Rules

### Case 1: App Version is Up to Date
- Condition: `installedVersion >= remoteVersion`
- Behavior: Gate remains completely hidden (`SizedBox.shrink()`). User experiences zero friction.

### Case 2: Optional Update
- Condition: `installedVersion < remoteVersion` and `isMandatoryUpdate == false`
- Behavior:
  - Clean popup appears with primary Peoplebase Indigo theme badge.
  - Shows version change indicator: `v1.0.0` ➜ `v1.0.1`.
  - User can tap **Update Now** (opens store link via `url_launcher`).
  - User can tap **Remind Me Later** or tap outside the scrim to dismiss for the current app session.

### Case 3: Mandatory / Critical Update
- Condition: `installedVersion < remoteVersion` and `isMandatoryUpdate == true`
- Behavior:
  - Header turns to `HrmsTone.danger` with security shield icon.
  - Prominent red warning banner: *"All tasks are paused until the update is completed."*
  - **Remind Me Later button is completely hidden.**
  - Background scrim ignores outside taps.
  - Android system back button & iOS swipe gestures are locked via `PopScope(canPop: false)`.
  - The user **cannot bypass or perform any tasks** until they update the app to a version `>=` remote version.

---

## 7. Testing & Verification Guide

### Automated Tests
Run the comprehensive test suite verifying SemVer logic, model parsing, dismissal behavior, and widget lock states:

```bash
flutter test test/core/version_control_test.dart test/core/version_control_widget_test.dart
```

Expected result:
```
00:00 +0: AppConstants.isVersionSmaller standard patch comparison
...
00:01 +9: All tests passed!
```

### Live Firebase Realtime Test
1. Set `FIREBASE_DATABASE_URL` and credentials in `.env`.
2. Run the application (`flutter run`).
3. In the Firebase Realtime Database web console, set `version: "1.0.1"` and `isMandatoryUpdate: false`.
   - **Observation**: Within milliseconds, the update modal animates onto the screen in real-time.
4. Tap "Remind Me Later" ➜ Modal dismisses.
5. In Firebase console, change `isMandatoryUpdate: true`.
   - **Observation**: Modal immediately re-appears in real-time, dismiss button disappears, and back-button lock engages.
6. Change `version: "1.0.0"` in Firebase console.
   - **Observation**: Modal disappears automatically as the app version is now up to date.

---

## 8. 5-Minute Quick-Start for Upcoming Projects

To replicate this exact feature in another Flutter project:

1. **Add Dependencies** in `pubspec.yaml`:
   ```yaml
   dependencies:
     flutter_riverpod: ^3.0.0
     flutter_dotenv: ^6.0.1
     firebase_core: ^5.2.0
     firebase_database: ^13.1.0
     url_launcher: ^6.3.2
   ```

2. **Copy Files**:
   Copy the `lib/core/version_control/` folder:
   - `models/app_version_data.dart`
   - `services/version_control_service.dart`
   - `providers/version_control_provider.dart`
   - `widgets/app_update_dialog.dart`
   - `widgets/version_control_gate.dart`

3. **Declare App Version** in your constants file:
   - Copy `appVersion` and `isVersionSmaller()` to your `AppConstants`.

4. **Add Variables to `.env`**:
   - Paste the Firebase block from Section 4 into your project's `.env`.

5. **Mount `VersionControlGate`** in `MaterialApp.builder`:
   ```dart
   MaterialApp(
     builder: (context, child) => Stack(
       children: [
         child!,
         const VersionControlGate(),
       ],
     ),
   );
   ```

You are all set! Your new project now features enterprise-grade, real-time, dynamic version control.
