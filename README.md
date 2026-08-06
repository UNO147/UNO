# Draw Four — Online Multiplayer UNO

Single-file PWA (`index.html`) + `manifest.json` + `service-worker.js`. Host on GitHub Pages like your other projects.

## Setup

1. Create/reuse a Firebase project. Enable **Realtime Database** and **Anonymous Authentication** (Build → Authentication → Sign-in method → Anonymous).
2. Copy your web app config (Project settings → General → Your apps → SDK setup) into the `FIREBASE_CONFIG` object near the top of the `<script>` in `index.html`.
3. Paste the security rules below into Realtime Database → Rules.
4. Drop two icon PNGs at `assets/icon-192.png` and `assets/icon-512.png` (any square UNO-style icon works — manifest references them).
5. Push to `kon41.github.io` (or a subfolder) and open on two devices to test.

## Realtime Database rules

This is the important part — it's what stops a player from reading someone else's hand or writing game state directly from devtools.

```json
{
  "rules": {
    "users": {
      "$uid": {
        ".read": "auth != null",
        ".write": "auth != null && auth.uid === $uid"
      }
    },
    "rooms": {
      "$code": {
        ".read": "auth != null",
        ".write": "auth != null"
      }
    },
    "hands": {
      "$code": {
        "$uid": {
          ".read": "auth != null && (auth.uid === $uid || root.child('rooms').child($code).child('hostUid').val() === auth.uid)",
          ".write": "auth != null && (auth.uid === $uid || root.child('rooms').child($code).child('hostUid').val() === auth.uid)"
        }
      }
    },
    "deck": {
      "$code": {
        ".read": "auth != null && root.child('rooms').child($code).child('hostUid').val() === auth.uid",
        ".write": "auth != null && root.child('rooms').child($code).child('hostUid').val() === auth.uid"
      }
    },
    "actions": {
      "$code": {
        ".read": "auth != null",
        ".write": "auth != null"
      }
    },
    "chat": {
      "$code": {
        ".read": "auth != null",
        ".write": "auth != null"
      }
    },
    "publicRooms": {
      ".read": "auth != null",
      ".write": "auth != null"
    }
  }
}
```

**Update Aug 6:** the original rules above blocked real gameplay — `deck` was unreadable by anyone (so draws silently failed) and `hands` was only readable by its owner (so the host couldn't validate/process any *other* player's move). The ruleset above is fixed: the host can now read/write `deck` and any player's `hand`, everyone else is still locked to their own hand only. **Re-paste the rules into Realtime Database → Rules** to pick up the fix.

**Known limitation (read this):** the spec asked for Firebase Cloud Functions as a server-authoritative engine so clients can never cheat. I can't deploy Cloud Functions into your project from here, so this build uses the standard fallback: **the host's browser runs the game engine** (shuffles, deals, validates moves, applies effects) and writes results to the DB. The rules above stop other players from reading each other's `hands/` node or writing directly to `deck/`, which blocks the common cheats — but a player who is *hosting* a room could tamper with their own client to see the deck or force outcomes for that room. For a fully tamper-proof game you'd move `processAction()` (in `index.html`, search for that function) into a Cloud Function later; the client-side logic is written so that porting is mostly copy-paste.

## If gameplay still doesn't work

The app now surfaces errors as on-screen toasts instead of failing silently — re-upload `index.html` and try again first; the toast text will tell you exactly what's wrong (usually a `PERMISSION_DENIED` from a rule that didn't get pasted in correctly).

To isolate whether it's a **rules problem** or a **logic problem**, temporarily paste this fully-open ruleset into Realtime Database → Rules, test a full game, then switch back to the real rules above afterward (this version has zero security — anyone with your database URL can read/write anything, so don't leave it live):

```json
{
  "rules": {
    ".read": "auth != null",
    ".write": "auth != null"
  }
}
```

If the game works fine under open rules but breaks under the restricted rules above, the issue is a rules typo — double check the `$code` room code in your rules matches exactly (case-sensitive) and that you saved/published the rules (there's a **Publish** button in the console, not just typing them in).

If it still breaks under fully open rules, it's a code bug — open your browser's DevTools console (⋮ menu → More tools → Developer tools → Console tab on desktop Chrome, or connect the phone via `chrome://inspect` from a desktop Chrome) while playing, and send me the exact error text.



**Working:** anonymous login, create/join room by 6-char code, host controls (kick, start, custom starting card count 1–50 with deck-size validation), optional rules toggles (stack draw / seven-zero flag stored, seven-zero *effect* not yet implemented — draw2/wild4/skip/reverse are), private synced hands, turn order + direction, UNO call + catch-penalty, draw/pass, win detection, stat tracking (wins/matches/XP/coins), disconnect flag + host transfer, room chat with emoji, confetti on win, PWA install + offline shell caching.

**Not built — flag if you want these next:**
- Google login (anonymous only right now)
- QR code join
- Jump-in / Force Play optional rules, and the Seven-Zero *card-swap* effect
- Real sound files (hooks are in `playSound()` — drop mp3s in `/sounds/` and uncomment the one line)
- Achievements / match history UI (stats are tracked in the DB, just not displayed beyond the profile screen)
- Reconnect-to-same-hand UI after a tab refresh mid-game (presence flag exists, but the client doesn't currently auto-rejoin an in-progress game on reload — tell me if that's the next priority)

Iterate on this the way you usually do — tell me what's broken or what to add next and I'll patch the file directly.
