# Jugnu

**A personal safety companion for women on the move.**

Built for **Vihaan X** by **404 Name Not Found**.

Jugnu is a mobile-first web app for planning a walk, finding nearby help, and sharing an SOS with a trusted contact. The hackathon demo starts at DTU in Delhi. It combines live walking directions with advisory route comparisons, a microphone-based distress trigger, and a call to a configured contact.

### The AI assessment backbone

<a href="https://maritime.sh/"><img src="docs/sponsors/maritime.png" alt="Maritime logo" width="44" height="44"></a>
&nbsp;
<a href="https://www.withprune.com/"><img src="docs/sponsors/prune.png" alt="Prune logo" width="44" height="44"></a>

**Maritime and Prune power Jugnu's incident assessment.** Maritime runs the private Python agent. Prune connects that agent to the configured vision model and can return a request receipt. [Explore the Maritime and Prune integration](#maritime-and-prune).

**Vihaan X presentation** · [View all 14 slides on GitHub](docs/jugnu-presentation.md) · [Download the PDF](Jugnu%20ppt%20%281%29.pdf)

![Jugnu desktop home screen with route planning and safety tools](docs/screenshots/home-desktop.webp)

## Try the demo

1. Open **Routes** and choose **Samaypur Badli Metro**, **Rithala Metro**, or **Japanese Park**. Compare the walking options Google Maps returns. Switch between daytime and nighttime, or ask AI to explain the suggested route.
2. Sign in, add a trusted contact, and set a four-digit SOS passcode. From a planned route, select **Start monitoring** to bring the map into an armed safety session.
3. With the demo account and Voice configured, **Start SOS** or a detected scream immediately requests a call to the saved demo contact. Audible mode starts the buzzer. A scream also opens an eight-second countdown for review. The passcode can dismiss the alert or silence the alarm after the call request. The police call appears as a labeled simulation.
4. Open **Community** to see the fictional DTU posts and submit a report. Use **Nearby help** to look up hospitals and police stations around a searched area or your current location.
5. With **Maritime and Prune** configured, open a saved incident, select **AI assessment**, and choose **Request assessment**. You can share one frame from a local recording by opting in. Jugnu displays the advisory summary and indicates whether Prune returned a receipt.

| Walking routes | Safety session setup | Community feed |
| --- | --- | --- |
| ![Phone layout showing DTU walking routes and advisory scores](docs/screenshots/route-planner.webp) | ![Phone layout showing safety session setup with alert and recording choices](docs/screenshots/safety-session.webp) | ![Phone layout showing the sample DTU community reports](docs/screenshots/community.webp) |

| Nearby help | SOS demo |
| --- | --- |
| ![Nearby hospitals and police stations on the desktop map](docs/screenshots/nearby-help.webp) | <img src="docs/screenshots/sos-demo.webp" width="340" alt="Phone layout showing the buzzer, a sample ringing status, and the simulated police call"> |

The SOS screenshot uses a simulated provider status. The live Twilio trial call was separately verified by the recipient. Community demo posts use fictional profiles and are labeled as examples.

## System architecture

![Jugnu system architecture showing browser detection, Flask services and immediate Twilio calling](docs/jugnu-system-architecture.png)

```mermaid
flowchart LR
    subgraph Device["Browser on phone or desktop"]
        UI["Next.js and React UI"]
        Audio["Microphone → AudioWorklet → YAMNet"]
        Clip["MediaRecorder → IndexedDB"]
        Audio --> UI
        UI --> Clip
    end
    UI -->|"Account, safety, and incident API"| Proxy["Next.js same-origin API proxy"]
    Proxy --> Flask["Flask"]
    Flask --> DB[("SQLite · accounts, contacts, incidents")]
    UI --> Maps["Google Maps · walking routes"]
    UI --> Weather["Next.js weather endpoint"]
    Weather --> Forecast["Open-Meteo or OpenWeather"]
    Flask --> Places["Google Places · nearby services"]
    Flask --> Router["OmniRoute or Gemini · text advice and voice intent"]
    Flask --> Voice["Twilio Voice · demo contact"]
    Flask --> Agent["Maritime · scene assessment agent"]
    Agent --> Prune["Prune · model gateway"]
    Prune --> Vision["Configured vision model"]
```

## SOS flow

```mermaid
flowchart TD
    Sound["Armed microphone recognizes a possible scream"] --> Countdown["Eight-second countdown"]
    Countdown --> Check{"Passcode or timely benign demo assessment?"}
    Check -->|"Yes"| Cancel["Close alert and stop buzzer"]
    Check -->|"No, uncertain, or camera unavailable"| Active["Sync active SOS"]
    Manual["Manual SOS or Get help now"] --> Active
    Countdown --> Evidence["Opt-in recording saved on this device"]
    Active --> Evidence
    Sound --> Demo{"Signed in as configured demo account?"}
    Manual --> Demo
    Demo -->|"Yes"| Contact["Call saved contact through Twilio"]
    Active --> Display{"Demo account?"}
    Display -->|"Yes"| Police["Show simulated police call and draft"]
    Display -->|"No"| Options["Show manual dialing and sharing options"]
    Contact --> Status["Show queued, ringing, connected, or outcome"]
```

### Demo call sequence

```mermaid
sequenceDiagram
    participant U as Jugnu client
    participant F as Flask
    participant T as Twilio Voice
    participant P as Saved contact
    U->>F: Sync manual SOS or detected scream
    U->>F: Request call for incident
    F->>F: Check account, saved number, and previous attempt
    F->>T: Create call
    T-->>F: Return call ID and status
    T->>P: Ring the saved number
    U->>F: Poll call status
    F->>T: Fetch call status
    F-->>U: Return provider status
```

The local browser plays the alarm and displays the police simulation. The server uses the saved demo destination and records one call request per incident. Trial mode uses Twilio’s hosted voice template. A standard account can play Jugnu’s own spoken message. [Twilio Voice trial settings](https://www.twilio.com/docs/usage/trials/try-out-voice) · [Twilio voice rules for India](https://www.twilio.com/en-us/guidelines/in/voice)

## Route comparison

```mermaid
flowchart LR
    Start["Choose origin and destination"] --> Maps["Google Maps walking routes"]
    Maps --> Factors["Compare walking time, detour distance, and mapped obstacles"]
    Conditions["Day or night and available weather"] --> Factors
    Factors --> Score["Advisory score per option"]
    Score --> Color["Red below 50 · yellow 50–70 · green above 70"]
    Score --> Explain["Optional AI explanation"]
    Color --> Pick["Carry chosen route into the safety session"]
```

The score compares known route features. Street lighting, pedestrian activity, and personal safety remain unverified.

## Maritime and Prune

Maritime and Prune form the backbone of Jugnu's optional AI assessment workflow. They help turn observations from an incident into a short summary for the user to review.

- **Maritime runs the assessment agent.** Jugnu's Flask API sends authenticated incident context and, with consent, camera frames to the private Python agent. [Maritime custom agents](https://maritime.sh/docs/frameworks/custom) · [Jugnu agent source](backend/maritime-agent/agent.py)
- **Prune is the model gateway.** The agent uses Prune to access the configured vision model, keep the provider key in Prune's vault, and track requests under the `jugnu-incident-assessment` feature ID. When Prune returns receipt metadata, Jugnu reports receipt presence in the incident view. [Prune integration docs](https://www.withprune.com/docs)

During a pending audio alert in the demo, the assessment can use two usable frames. A timely, clearly benign result can close the pending alert. Dark, missing, or inconclusive footage keeps the countdown active. Twilio requests the saved contact call as soon as the incident syncs, while the assessment runs separately.

The assessment path was verified with mocked services. To run it live, deploy the Maritime agent and configure a Prune app key and compatible vision provider. [Live setup guide](docs/jugnu-live-setup.md#sponsor-credentials)

## Tech stack

![Jugnu technology stack across the browser, backend, maps, voice and AI](docs/jugnu-tech-stack-pipeline.png)

| Layer | Technologies |
| --- | --- |
| Web app | Next.js 15, React 19, TypeScript, Tailwind CSS |
| Maps and context | Google Maps Routes and Places, Open-Meteo or OpenWeather |
| On-device audio | Web Audio API, AudioWorklet, MediaPipe Audio Tasks, YAMNet |
| Evidence | MediaRecorder, IndexedDB |
| Backend and auth | Python 3.12, Flask, SQLAlchemy, SQLite, hashed passwords and SOS passcodes |
| Text AI | OmniRoute with Groq GPT-OSS 20B and 120B fallback, or Gemini |
| Scene assessment | Maritime agent, Prune gateway, configurable vision model |
| Notifications | Twilio Voice for the demo contact, optional Twilio SMS |

## Run locally

Use Node.js 20 or later and Python 3.12. Copy [backend/.env.example](backend/.env.example) to `backend/.env` and [frontend/.env.example](frontend/.env.example) to `frontend/.env.local`.

```bash
cd backend
python -m venv venv
source venv/bin/activate
python -m pip install -r requirements.txt
python -m flask --app run run --host 127.0.0.1 --port 8000
```

On Windows PowerShell, activate the virtual environment with `.\venv\Scripts\Activate.ps1`.

```bash
cd frontend
npm ci --legacy-peer-deps
npm run dev
```

Open [localhost:3000](http://localhost:3000). Enable camera, microphone, and location from localhost or HTTPS. Follow the [live setup guide](docs/jugnu-live-setup.md) for Maps, OmniRoute or Gemini, Twilio, Maritime, and Prune. Use your own demo account and approved recipient. Keep credentials in local environment files.

