# Meeting Matcher

Meeting Matcher is a lightweight, single-page web application that helps small groups rapidly converge on a mutually agreeable meeting time. The interface lets each participant swipe or tap through proposed 5-minute slots, voting "Yes" or "No" while the leaderboard updates in real time. All state lives in Firebase Firestore and rooms are automatically deleted when they expire.

## Features

- **Ephemeral rooms** – owners create a room with a shareable link, which disappears automatically after the configured TTL.
- **Five-minute slots** – generate a randomized list of candidate times inside a custom availability window.
- **Anonymous, real-time collaboration** – anonymous Firebase Auth plus Firestore listeners keep everyone in sync.
- **Immutable, auditable votes** – every vote is stored once per user per slot; the client aggregates tallies on the fly.
- **Two recommendation modes** – strict mode only surfaces untouched slots, while score mode prioritizes the highest `Yes − λ × No` score.

## Getting started

1. Create a Firebase project with Firestore and Anonymous Auth enabled.
2. Enable [Firestore TTL](https://firebase.google.com/docs/firestore/ttl) on the `rooms` collection for the `expiresAt` field.
3. Download the repository and open [`index.html`](./index.html) in your editor.
4. Replace the `firebaseConfig` placeholder with your Firebase project's configuration snippet.
5. Deploy the file to any static host (GitHub Pages, Firebase Hosting, Netlify, etc.).

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

These rules align with the application logic: slot creation is restricted to the room owner, and each participant can only vote once per slot.

## License

MIT
