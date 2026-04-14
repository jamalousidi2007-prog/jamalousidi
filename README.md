# AgriPulse Control (Next.js + Firebase + Arduino)

A fancy bilingual (EN/FR) web dashboard that:

- reads `temperature`, `humidity`, `rain`, `light`, and `pressure` from a locally hosted Arduino API
- shows live cards + chart + session stats
- supports Firebase auth (register/login/logout)
- includes role-based admin features:
  - ban/unban users
  - promote user to admin
  - send in-app notifications to users
  - edit key website texts in both English and French

## 1) Install and run

```bash
npm install
cp .env.example .env.local
npm run dev
```

Open: `http://localhost:3000`

## 2) Configure `.env.local`

Fill Firebase values from your Firebase project settings, then set your Arduino endpoint:

- `ARDUINO_API_URL=http://127.0.0.1:5000/sensors`

Expected Arduino JSON shape (flexible keys like `temp` are also accepted):

```json
{
  "temperature": 24.7,
  "humidity": 58.1,
  "rain": 15,
  "light": 830,
  "pressure": 1012.4,
  "timestamp": "2026-04-09T20:30:00.000Z"
}
```

## 3) Firestore collections used

- `users/{uid}`
  - `uid`, `email`, `displayName`, `role`, `banned`, `locale`, `createdAtMs`, `lastSeenAtMs`
- `notifications/{id}`
  - `userId`, `title`, `message`, `read`, `createdAtMs`, `createdByUid`
- `siteContent/main`
  - `heroTitleEn`, `heroTitleFr`, `heroSubtitleEn`, `heroSubtitleFr`, `statsHeadlineEn`, `statsHeadlineFr`, `footerTextEn`, `footerTextFr`

## 4) Suggested Firestore rules (starter)

Use this as a base and adapt to your security requirements:

```text
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function isSignedIn() {
      return request.auth != null;
    }

    function isAdmin() {
      return isSignedIn() &&
        get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == 'admin';
    }

    match /users/{userId} {
      allow read: if isSignedIn();
      allow create: if isSignedIn() && request.auth.uid == userId;
      allow update: if isAdmin() || (isSignedIn() && request.auth.uid == userId);
    }

    match /notifications/{id} {
      allow create: if isAdmin();
      allow read, update: if isSignedIn() &&
        resource.data.userId == request.auth.uid;
    }

    match /siteContent/{id} {
      allow read: if true;
      allow write: if isAdmin();
    }
  }
}
```

## 5) Admin bootstrap

To auto-promote the first admin account, set:

- `NEXT_PUBLIC_SEED_ADMIN_EMAIL=admin@example.com`

Then register with that email.

## 6) Notes

- Ban works by setting `users/{uid}.banned=true`; the user is signed out and blocked in the app UI.
- Notifications are in-app messages (not FCM push by default).

