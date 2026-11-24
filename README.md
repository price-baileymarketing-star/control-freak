# Control Freak for Price Bailey events

Two-screen browser game. The big screen shows the game and a QR code. The player scans it and plays from their phone. The game can also be played on the big screen itself. Both routes show a details form first, and details go straight to HubSpot.

## Structure

```
index.html              big screen (game, QR code, big-screen form, end screen)
controller/index.html   phone controller (form, game pad, end screen)
lib/link.js             connection between the two screens (MQTT + BroadcastChannel)
lib/hubspot.js          HubSpot settings and submission, shared by both screens
lib/screen-kit.js       big-screen kit: QR code, details form, pause banner, end screen
lib/mqtt.min.js         bundled messaging library (mqtt.js 5.16.1)
lib/qrcode.js           bundled QR code library (qrcode-generator 1.5.2)
tailwind.config.js, src/  Tailwind rebuild files (see below)
assets/                 logo, fonts, compiled CSS
```

## How to play

- **Controls:** Phone: hold to walk, back, jump. At each building the pad changes to Good and Bad. Big screen: arrow keys, space, G and B (or 1 and 2).
- **Scoring:** +100 for a correct judgement, -50 for a wrong one, 10 per coin, 5 per second left as a bonus for finishing. Obstacles cost two coins. 90 second clock.
- The end screen shows the score, a message and **New game**, with no high score. It returns to a fresh QR code after 45 seconds.

## Settings to check before an event

- `lib/hubspot.js`: `EVENT_NAME` (must be set, it is recorded against every lead), `GAME_NAME` (currently `Control Freak`), `PRIVACY_URL`, consent wording.
- `lib/screen-kit.js` settings block: `FORM_IDLE_CLOSE_MS`, `END_AUTO_RESET_MS`, `CONTROLLER_TIMEOUT_MS`, `CONTROLLER_LOST_RESET_MS`.
- `index.html`: `PBScreen.init({...})` at the end of the game script holds the title copy, the phone HUD and the colour theme.
- `controller/index.html` settings block: send rate and button hold timings. Brand colours are CSS variables at the top.

## How the connection works

The big screen creates a four-character room code and shows a QR code to `controller/?room=CODE`. Both pages relay messages over `wss://broker.emqx.io:8084/mqtt` and `wss://broker.hivemq.com:8884/mqtt` together, de-duplicated by message id. Topics are `pricebailey/control-freak/ROOM/screen` and `.../controller`. BroadcastChannel is also used, so two tabs in one browser work as a desk test.

- Phone to screen: `hb`, `start`, `in` (held buttons every 150 ms plus cumulative tap counters, so repeats are harmless), `newgame`.
- Screen to phone: `hb`, `st` (state, score, pad name, HUD line, end title and message), `bye`.
- Heartbeat every 2 seconds, other side treated as gone after 6 seconds. The game pauses while the phone is away and is abandoned after 30 seconds.
- Only the first controller is accepted. Others see "Game in use".
- Add `?broker=ws://host:port` to either address to test against a local broker.

Both devices need internet. The public brokers have no uptime guarantee. For important events, move to a private broker or Firebase.

## Rebuilding the styles

Tailwind is compiled to `assets/tailwind.css` so the page loads nothing from a CDN. Fredoka and Bungee are bundled in `assets/fonts/`.

- After adding new utility classes to `index.html`, run `npx tailwindcss@3.4.17 -c tailwind.config.js -i src/tailwind.input.css -o assets/tailwind.css --minify` from the repository root.

## Testing

1. Open `index.html` in one tab, then the controller address shown under the QR code in a second tab (desk test).
2. Add `&debug` to the controller address to show HubSpot's replies on the phone.
3. Play one full game on a real phone and check the test contact in HubSpot.

## Data protection

Personal details are held in memory only. They are never written to the repository or browser storage, never shown on the big screen, and are cleared when the game starts and on New game. A form left open on the big screen closes and clears after 90 seconds.

## Notes

Sound on a phone-started game may stay silent until someone taps the big screen once.

## Blocking before a stand

- Event name set.
- Compliance approve the consent wording and privacy notice link, and `PRIVACY_URL` is filled in.
- A test contact appears in HubSpot with every field filled, including `game_form_consent` when ticked, and the game name `Control Freak` is accepted.
- `price-baileymarketing-star.github.io` stays registered as a trusted domain in HubSpot. Any other host must be added too.
