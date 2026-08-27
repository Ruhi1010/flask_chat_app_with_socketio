
# Flask Chat App with Socket.IO

A lightweight, real-time multi-room chat application built with **Flask** and **Flask-SocketIO**. Users are automatically assigned a guest username, can join predefined chat rooms, exchange room-wide messages, send private messages, and see a live list of active users.

---

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Setup & Installation](#setup--installation)
- [Configuration](#configuration)
- [Running the App](#running-the-app)
- [Application Flow](#application-flow)
- [Socket.IO Event Reference](#socketio-event-reference)
  - [Client → Server](#client--server-events)
  - [Server → Client](#server--client-events)
- [Frontend](#frontend)
  - [`templates/index.html`](#templatesindexhtml)
  - [`static/chat.js`](#staticchatjs)
  - [`static/styles.css`](#staticstylescss)
- [Data Model](#data-model)
- [Logging](#logging)
- [Deployment Notes](#deployment-notes)
- [Known Limitations](#known-limitations)

---

## Features

- 🔒 Automatic guest session creation (no login required)
- 💬 Multiple predefined chat rooms
- 📢 Real-time room broadcast messaging
- ✉️ Private (direct) messaging between users
- 👥 Live active-user list, updated on connect/disconnect
- 🚪 Join/leave room notifications
- 🛡️ Reverse-proxy aware (via `ProxyFix`) for correct host/protocol handling behind load balancers

---

## Tech Stack

| Component        | Library                | Version   |
|-------------------|------------------------|-----------|
| Web framework      | Flask                  | 3.1.3     |
| Real-time layer    | Flask-SocketIO         | 5.6.1     |
| WebSocket engine    | python-socketio / python-engineio | 5.16.4 / 4.13.5 |
| WSGI utilities      | Werkzeug               | 3.1.8     |
| Templating          | Jinja2                 | 3.1.6     |
| WebSocket transport | simple-websocket, wsproto | 1.1.0 / 1.3.2 |

Full pinned dependency list is in [`requirements.txt`](./requirements.txt).

---

## Project Structure

```
.
├── main.py              # Flask app, Socket.IO event handlers, entry point
├── requirements.txt      # Python dependencies
├── README.md              # Project name/description
├── templates/
│   └── index.html         # Chat UI markup (Jinja2 template)
├── static/
│   ├── chat.js             # Client-side Socket.IO logic & DOM rendering
│   └── styles.css          # Chat UI styling
├── .gitattributes
└── .gitignore
```

> **Note:** `main.py` prints diagnostic information about the template
> folder on startup (working directory, template path, whether
> `index.html` exists). `index.html` references `styles.css` and
> `chat.js` via `url_for('static', filename=...)`, so both files must
> live in a `static/` directory alongside `templates/` for Flask's
> default static file handling to serve them correctly.

---

## Setup & Installation

### Prerequisites
- Python 3.9+ (recommended)
- `pip`

### Steps

```bash
# 1. Clone the repository
git clone <repo-url>
cd flask_chat_app_with_socketio

# 2. Create and activate a virtual environment
python -m venv venv
source venv/bin/activate      # Windows: venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Run the app
python main.py
```

The app will start on `http://0.0.0.0:5000` by default.

---

## Configuration

Configuration is managed via the `Config` class in `main.py`, populated from environment variables:

| Environment Variable | Default            | Description |
|------------------------|---------------------|--------------|
| `SECRET_KEY`             | random 24-byte value | Flask session signing key. **Set explicitly in production** so sessions survive app restarts. |
| `FLASK_DEBUG`            | `False`               | Enables Flask debug mode and the auto-reloader when set to `true`/`1`/`t`. |
| `CORS_ORIGINS`           | `*`                    | Allowed CORS origins for Socket.IO connections. Restrict this in production. |
| `PORT`                   | `5000`                 | Port the server listens on. |

### Chat Rooms

Available rooms are currently hardcoded in `Config.CHAT_ROOMS`:

```python
CHAT_ROOMS = [
    'General',
    'Zero to Knowing',
    'Code with Ruhi',
    'The Coding Train',
]
```

To add or remove rooms, edit this list. A comment in the code notes this could be moved to a database in the future.

---

## Running the App

### Development
```bash
FLASK_DEBUG=true python main.py
```

### Production
The `__main__` block explicitly recommends **not** using the built-in Werkzeug dev server in production. Use a production-grade server instead, e.g.:

```bash
gunicorn --worker-class eventlet -w 1 main:app
```
or
```bash
uwsgi --http :5000 --gevent 1000 --http-websockets --module main:app
```

> Because Socket.IO requires a single worker process (or sticky sessions / a
> shared message queue like Redis for multiple workers), scale with care.
> The in-memory `active_users` dictionary also assumes a single process —
> see [Known Limitations](#known-limitations).

---

## Application Flow

1. **User visits `/`**
   - If no `username` exists in the session, a guest username is generated (e.g. `Guest14329981`) and stored in the Flask session.
   - `index.html` is rendered with the username and the list of available rooms.

2. **Client connects via Socket.IO**
   - The server registers the user's session ID (`sid`) in the `active_users` dictionary.
   - The updated list of active usernames is broadcast to all connected clients.

3. **User joins a room**
   - Client emits `join` with a room name.
   - Server validates the room against `CHAT_ROOMS`, adds the socket to that room, and broadcasts a join notification.

4. **Messaging**
   - **Room messages:** broadcast to everyone in the room.
   - **Private messages:** delivered only to the target user's socket ID, looked up by username in `active_users`.

5. **User leaves a room / disconnects**
   - `leave` removes the socket from the room and notifies remaining members.
   - `disconnect` removes the user from `active_users` and re-broadcasts the active user list.

---

## Socket.IO Event Reference

### Client → Server Events

| Event | Payload | Description |
|--------|----------|--------------|
| `connect` | *(none — implicit)* | Fired automatically by Socket.IO on connection. Registers the user. |
| `disconnect` | *(none — implicit)* | Fired automatically on disconnection. Removes the user. |
| `join` | `{ "room": "<room name>" }` | Joins the specified room (must be one of `CHAT_ROOMS`). |
| `leave` | `{ "room": "<room name>" }` | Leaves the specified room. |
| `message` | `{ "room": "<room name>", "type": "message" \| "private", "msg": "<text>", "target": "<username>" (private only) }` | Sends a room message or, if `type` is `"private"`, a direct message to `target`. |

### Server → Client Events

| Event | Payload | Description |
|--------|----------|--------------|
| `active_users` | `{ "users": ["<username>", ...] }` | Broadcast whenever a user connects or disconnects. |
| `status` | `{ "msg": "<text>", "type": "join" \| "leave", "timestamp": "<ISO 8601>" }` | Sent to a room when a user joins or leaves. |
| `message` | `{ "msg": "<text>", "username": "<sender>", "room": "<room>", "timestamp": "<ISO 8601>" }` | Broadcast to a room when a regular message is sent. |
| `private_message` | `{ "msg": "<text>", "from": "<sender>", "to": "<recipient>", "timestamp": "<ISO 8601>" }` | Sent only to the recipient's socket. |

---

## Frontend

The UI is server-rendered (Jinja2) with a vanilla-JS Socket.IO client — no frontend framework or build step is used. Socket.IO's client library is loaded from a CDN.

### `templates/index.html`

- Loads the Socket.IO client from `cdnjs` (v4.0.1) and links `styles.css` / `chat.js` via Flask's `url_for('static', ...)`.
- Renders the guest `username` (from the Flask session) into a `<span id="username">`.
- Renders the room list (`{% for room in rooms %}`) as clickable `<div class="room-item">` elements, each calling `joinRoom('<room>')` on click.
- Contains the chat log container (`#chat`), the online-users panel (`#active-users`), and the message input (`#message`) with a **Send** button.

### `static/chat.js`

Handles all client-side real-time behavior:

| Responsibility | Details |
|---|---|
| **Connection** | On `connect`, automatically joins the `General` room and highlights it in the sidebar. |
| **Incoming messages** | Listens for `message`, `private_message`, `status`, and `active_users` server events and renders them into the chat log via `addMessage()`. |
| **Room switching** | `joinRoom(room)` emits `leave` for the current room, emits `join` for the new one, clears the chat log, and replays that room's cached history from the in-memory `roomMessages` object. |
| **Sending messages** | `sendMessage()` reads the `#message` input. If the text starts with `@username`, it's parsed and sent as a `private` message (`{ type: 'private', target, msg }`); otherwise it's sent as a normal room message (`{ room, msg }`). |
| **Private-message shortcut** | Clicking a name in the online-users list calls `insertPrivateMessage(user)`, which pre-fills the input with `@username `. |
| **Keyboard shortcut** | `handleKeyPress()` sends the message on `Enter` (without `Shift`). |
| **Client-side message cache** | `roomMessages` is a plain object keyed by room name, holding each room's messages **only for the current browser tab/session** — it resets on page reload and isn't synced with the server. |

> ⚠️ **Known bug:** the `DOMContentLoaded` handler calls `new ChatApp()`,
> but no `ChatApp` class is defined anywhere in `chat.js` (or elsewhere in
> the provided files). This will throw a `ReferenceError` in the browser
> console on page load. Since all of the chat functions (`joinRoom`,
> `sendMessage`, etc.) are defined as free-standing functions rather than
> methods on `ChatApp`, this call appears to be leftover/dead code and can
> likely be removed, or the app logic was intended to be wrapped in a
> `ChatApp` class that was never implemented. The rest of the script
> still runs, since the error only affects that one handler.

### `static/styles.css`

- Defines a CSS custom-property palette (`--primary`, `--accent`, `--success`, `--danger`, etc.) for consistent theming.
- Two-column layout: a fixed-width `.sidebar` (rooms + online users) and a flexible `.main-chat` panel, laid out with flexbox inside `.container`.
- Message bubble styling differentiated by type: `.message.own` (sent by current user), `.message.other` (from others), `.message.system` (join/leave notices), and `.message.private` (direct messages) — each with distinct background colors and alignment.
- Subtle UX polish: hover states with `translateX`/`translateY` transforms, a `fadeIn` keyframe animation on new messages, and custom scrollbar styling.
- Responsive breakpoint at `768px` that stacks the sidebar above the chat panel and widens message bubbles for smaller screens.

---

## Data Model

### `active_users` (in-memory dict)

```python
active_users: Dict[str, dict] = {
    "<socket_id>": {
        "username": "Guest14329981",
        "connected_at": "2026-08-28T10:15:00.000000",
        "room": "General"          # present only after joining a room
    },
    ...
}
```

Keyed by Socket.IO session ID (`request.sid`). This is **not persisted** — it resets on every server restart and is not shared across multiple processes/workers.

---

## Logging

Configured via Python's standard `logging` module:

```python
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
```

Logged events include: session creation, connect/disconnect, room joins/leaves, message sends, private message delivery/failures, and errors caught in each event handler's `try/except` block.

Socket.IO's own internal logging is also enabled (`logger=True`, `engineio_logger=True`), which is useful for debugging but verbose — consider disabling in production.

---

## Deployment Notes

- **Secret key:** Set `SECRET_KEY` explicitly via environment variable in production; the random fallback invalidates all sessions on every restart.
- **CORS:** `CORS_ORIGINS` defaults to `*` (allow all). Restrict this to your actual frontend origin(s) in production.
- **Reverse proxy:** `ProxyFix` is configured for `x_proto` and `x_host` headers — ensure your proxy (e.g. Nginx) sets `X-Forwarded-Proto` and `X-Forwarded-Host` accordingly.
- **Scaling:** For multiple server instances, use a Socket.IO message queue backend (e.g. Redis) so broadcasts and room state stay consistent across processes; the current in-memory `active_users` store won't work correctly in a multi-process/multi-node setup.
- **WSGI server:** Do not use `socketio.run()` in production, as the code itself notes — use `gunicorn`/`eventlet` or `uwsgi`/`gevent`.

---

## Known Limitations

- No persistence — messages and active user data are lost on restart.
- No authentication — usernames are auto-generated guest identities; there's no login or identity verification.
- Single-process only — `active_users` is a plain Python dict, unsuitable for multi-worker/multi-instance deployments without an external store.
- Chat room list is hardcoded in source rather than configurable at runtime.
- No rate limiting or spam/abuse protection on message sending.
- No message history — users only see messages sent after they join for the current browser session (client-side `roomMessages` cache resets on page reload; there's no server-side message store either).
- `chat.js` calls `new ChatApp()` on page load, but no `ChatApp` class exists — this throws a console error on every page load (see [Frontend](#staticchatjs)).
- No input sanitization is visible on the client for rendered usernames/messages beyond using `textContent` in `addMessage()`, which does mitigate basic HTML injection, but message content isn't validated/escaped server-side either.
