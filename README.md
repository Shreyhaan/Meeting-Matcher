# Meeting Matcher

Meeting Matcher is a lightweight, single-page web application that helps small groups rapidly converge on a mutually agreeable meeting time. The interface lets each participant tap through proposed 5-minute slots, voting "Yes" or "No" while the leaderboard updates in real time. All state lives in Firebase Firestore and rooms are automatically deleted when they expire.

## Features

- **Auto-deleting rooms** – owners create a room with a shareable link, which disappears automatically after the configured TTL in order to save space.
- **Five-minute slots** – generate a randomized list of candidate times inside a custom availability window.
- **Anonymous, real-time collaboration** – anonymous Firebase Auth plus Firestore listeners keep everyone in sync.
- **Immutable, auditable votes** – every vote is stored once per user per slot; the client aggregates tallies on the fly.
- **Two recommendation modes** – strict mode only surfaces untouched slots, while score mode prioritizes the highest `Yes − λ × No` score.

## Getting started

1. Create a Firebase project with Firestore and Anonymous Auth enabled.
2. Enable [Firestore TTL](https://firebase.google.com/docs/firestore/ttl) on the `rooms` collection for the `expiresAt` field (see the detailed steps below).
3. Download the repository and open [`index.html`](./index.html) in your editor.
4. Copy the entire `const firebaseConfig = { … }` object from the Firebase console's CDN snippet and paste it over the placeholder in [`index.html`](./index.html). Bring every key Firebase includes (for example `storageBucket`, `appId`, `measurementId`) and keep the existing `<script type="module">` imports—skip the extra analytics helper function that snippet shows by default.
5. Deploy the file to any static host (GitHub Pages, Firebase Hosting, Netlify, etc.).

If you see `Firebase: Error (auth/api-key-not-valid...)` after clicking **Create room**, double-check that the `firebaseConfig` block no longer contains placeholder values. Re-copy the snippet from **Project settings → General → Your apps → Web** and ensure `apiKey`, `authDomain`, `projectId`, and `appId` match your Firebase project.

### Copying the `firebaseConfig`

Follow these exact steps to grab the correct credentials:

1. In the Firebase console, open **Project settings → General**.
2. Scroll to **Your apps**. If you have not registered a web app yet, click the `</>` icon to add one (you can skip hosting and analytics during this wizard).
3. In the Web app card, click **Config**. Firebase will display a snippet that includes `const firebaseConfig = { … }`.
4. Copy the entire object—every property between the braces—and paste it into [`index.html`](./index.html), replacing the placeholder values.
5. If your snippet does **not** include `measurementId`, delete that line from the placeholder block so no `YOUR_MEASUREMENT_ID` text remains.

The app checks for any leftover `YOUR_…` placeholders and shows a detailed alert before Firebase initializes. If you still see `auth/api-key-not-valid`, repeat the steps above to ensure no key was truncated or mis-typed.

When you visit the deployed page:

1. Click **Create room** and provide optional start/end boundaries and a step size.
2. Share the generated link with participants.
3. Each participant votes on suggested slots. The leaderboard shows live tallies, and the best slot emerges automatically.

## Firestore security rules

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{db}/documents {

    function authed() {
      return request.auth != null;
    }

    match /rooms/{roomId} {
      allow read: if authed();
      allow create: if authed()
        && request.resource.data.keys().hasOnly([
          'ownerUid','createdAt','expiresAt','seed',
          'windowStart','windowEnd','stepMinutes'
        ])
        && request.resource.data.ownerUid == request.auth.uid;
      allow update, delete: if authed() && resource.data.ownerUid == request.auth.uid;

      match /slots/{slotId} {
        allow read: if authed();
        allow create: if authed() && get(/databases/$(db)/documents/rooms/$(roomId)).data.ownerUid == request.auth.uid
          && request.resource.data.keys().hasOnly(['ts']);
        allow update, delete: if false;
      }

      match /votes/{voteId} {
        allow read: if authed();
        allow create: if authed()
          && request.resource.data.keys().hasOnly(['slotId','uid','value','at'])
          && request.resource.data.uid == request.auth.uid
          && request.resource.data.value in ['yes','no']
          && voteId == (request.auth.uid + '__' + request.resource.data.slotId)
          && !exists(/databases/$(db)/documents/rooms/$(roomId)/votes/$(voteId));
        allow update, delete: if false;
      }
    }
  }
}
```

## Enabling Firestore TTL step by step

If the TTL screen is not obvious in the console, follow these exact steps:

1. In the Firebase console, open **Build → Firestore Database**.
2. At the top of the Firestore view, click the **Data** tab (this is the default tab where you browse collections).
3. In the left sidebar, select the collection you want to manage—choose `rooms`.
4. With the collection selected, click **Add TTL policy** in the right-side panel (or from the overflow menu) to start the policy wizard.
5. For **Field name**, enter `expiresAt`. The field must use the *Date and time* type.
6. Leave the default condition (“Delete document when TTL field time has passed”) enabled and save the policy.

Firestore will now automatically delete each room document (and its subcollections) once the `expiresAt` timestamp is in the past. If the **Add TTL policy** action is unavailable, make sure your project is on the Blaze plan (TTL is not offered on the Spark plan) and that you have already created at least one document in the `rooms` collection so the field can be detected.

## License

MIT
