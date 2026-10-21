# PhoneBridge — Direct Phone Data Transfer PWA

A production-oriented, mobile-first Progressive Web App (PWA) designed to transfer photos, videos, and documents from an old phone to a new phone over direct, end-to-end encrypted WebRTC DataChannels with zero cloud storage.

---

## 🌟 Key Features

- **Direct Peer-to-Peer Transfer:** Files travel directly between phones via WebRTC DataChannels over local Wi-Fi or peer networks. No files are ever uploaded to any cloud server.
- **Fast QR Pairing:** The new phone generates a secure pairing QR code; the old phone scans it with its camera to establish the peer session in seconds.
- **Manual 6-Character PIN Fallback:** If camera access is denied or the camera lens is broken, devices can pair instantly using a clean 6-character code (e.g. `ABC-123`).
- **Separated Media Categories:**
  - 📸 **Photos:** Image gallery grid, instant thumbnail generation, resolution badges, multi-select.
  - 🎬 **Videos:** Video preview thumbnails, duration extraction, file size tracking.
  - 📁 **Files & Documents:** Supports PDFs, DOCX, XLSX, ZIPs, APKs, and custom documents with category badges.
- **Industrial-Grade Transfer Engine:**
  - 64 KB binary chunking with flow control and `bufferedAmount` backpressure management to prevent memory exhaustion on mobile devices.
  - Hardware-accelerated SHA-256 checksum integrity verification via Web Crypto API.
  - Real-time metrics: transfer speed in MB/s, ETA countdown, file progress, and overall progress percentage.
  - Transfer pause, resume, and cancellation controls.
  - Failed file retry without restarting the entire transfer batch.
- **Smart File Saving:**
  - Modern Chromium **File System Access API** (`showDirectoryPicker`) for streaming direct-to-folder file saves on supported devices.
  - Automatic fallback to browser download triggers for universal mobile compatibility.
- **PWA & Offline Ready:**
  - Web manifest with app icons and standalone orientation configuration.
  - Workbox service worker caching app shell assets for instant launches.

---

## 🏗️ System Architecture

```
┌────────────────────────────────────────────────────────┐
│                   Signaling Server                     │
│               (Node.js + WebSocket :3001)              │
└───────────────▲────────────────────────▲───────────────┘
                │                        │
       1. QR Code / PIN         2. SDP Offer/Answer
          Session Pair              ICE Candidates
                │                        │
┌───────────────▼────────┐      ┌────────▼───────────────┐
│     New Phone          │      │     Old Phone          │
│   (Receive Mode)       │◄────►│    (Send Mode)         │
└────────────────────────┘      └────────────────────────┘
          ▲                                ▲
          └────── Direct WebRTC P2P ───────┘
                DataChannel (64KB Chunks)
                 (DTLS / Zero-Cloud Path)
```

1. **Signaling Server:** A lightweight WebSocket server (`server/server.ts`) creates temporary pairing sessions, handles rate limiting, and brokers SDP offers/answers and ICE candidates. **No file contents ever touch the signaling server.**
2. **QR Code:** Encodes the session identifier, connection endpoint, and high-entropy pairing secret.
3. **WebRTC DataChannel:** Handles direct binary chunk transfer (`ordered: true`, 64 KB chunks, backpressure threshold monitoring at 256 KB).

---

## 🚀 Quick Start Guide

### Prerequisites
- Node.js v18+ (tested on Node v26)
- npm v9+

### 1. Installation
Clone the repository and install dependencies:
```bash
git clone <repo-url>
cd "Data Transfer PWA"
npm install
```

### 2. Start Frontend and Signaling Server Together
Run both the Vite dev server and the WebSocket signaling server with a single command:
```bash
npm run dev:all
```

Alternatively, you can run them in separate terminals:
```bash
# Terminal 1: Frontend (PWA)
npm run dev

# Terminal 2: Signaling Server
npm run server
```

The terminal will output the local network addresses:
```
  ➜  Local:   http://localhost:5173/
  ➜  Network: http://192.168.x.x:5173/

PhoneBridge Signaling Server running on port 3001:
  - Localhost:   http://localhost:3001
  - Network IP:  http://192.168.x.x:3001
  - WebSocket:   ws://192.168.x.x:3001
```

---

## 📱 Testing with Two Physical Phones

Follow these simple steps to perform an actual transfer between two physical phones (e.g., two Android phones or an Android and iPhone):

### Step 1: Connect Phones to the Same Wi-Fi Network
Ensure both phones and your computer running the dev server are connected to the same Wi-Fi network (or mobile hotspot).

### Step 2: Open PhoneBridge on Both Phones
1. Note the `Network` IP shown in your terminal (e.g., `http://192.168.31.102:5173`).
2. Open the browser (Google Chrome recommended) on both phones and navigate to that URL.

> **PWA Installation (Optional):** On Android Chrome, tap the menu (⋮) and tap **"Install App"** or **"Add to Home Screen"** to install PhoneBridge as a standalone app.

### Step 3: Start Receive on New Phone
1. On the **New Phone**, tap **"Receive Data"**.
2. A high-contrast pairing QR code and a 6-character code (e.g. `ABC-123`) will be displayed.
3. *(Optional)* Tap **"Choose Save Folder"** if you want files automatically stored in a specific folder.

### Step 4: Scan and Pair on Old Phone
1. On the **Old Phone**, tap **"Send Data"**.
2. Allow camera permission when prompted.
3. Point the camera at the QR code displayed on the New Phone.
4. *Camera unavailable?* Tap **"Enter code manually"** and type the 6-character code shown on the New Phone.
5. Within 1-2 seconds, both phones will chime and show **"Connected!"**.

### Step 5: Select Data & Transfer
1. On the **Old Phone**, browse through:
   - **Photos:** Select gallery photos with instant thumbnail previews.
   - **Videos:** Select video clips (durations and thumbnails are calculated automatically).
   - **Files:** Choose PDFs, document files, or ZIP archives.
2. Review the total selected size in the bottom summary bar.
3. Tap **"Start Transfer"**.
4. Both devices will display real-time animated circular progress, live speed (MB/s), time remaining (ETA), and active file status.
5. Once complete, the New Phone verifies each file's SHA-256 hash and enables instant download or save.

---

## 🔒 Security & Privacy

- **DTLS Encryption:** WebRTC DataChannels are encrypted by default using Datagram Transport Layer Security (DTLS).
- **Zero Cloud Storage:** Transfer packets move directly peer-to-peer across the local network; file payloads are never routed through or stored on any server.
- **Short-Lived Secrets:** Pairing tokens are cryptographically generated (128-bit entropy) and automatically expire after 10 minutes.
- **Single-Use Sessions:** Sessions allow only one sender and one receiver, rejecting unauthorized connection attempts.
- **Data Integrity:** Each file is verified with SHA-256 upon reception before being marked as completed.

---

## ⚙️ Configuration & Environment Variables

Copy `.env.example` to `.env` to customize settings:

```ini
# Port for the WebSocket signaling server
SIGNALING_PORT=3001

# Optional production signaling URL (e.g. wss://signaling.yourdomain.com)
VITE_SIGNALING_URL=

# Optional TURN Relay Server (for strict symmetric NATs)
VITE_TURN_URL=turn:turn.yourdomain.com:3478
VITE_TURN_USER=phonebridge
VITE_TURN_PASS=your-turn-password
```

Users can also customize signaling URLs and STUN/TURN credentials directly inside the app by tapping the **Settings** icon in the header.

---

## 🌐 Production Deployment

### 1. Build the Production Bundle
```bash
npm run build
```
This generates the optimized web build and PWA service worker inside the `dist/` directory.

### 2. Host the Frontend
Serve the `dist/` directory using any HTTPS-capable static hosting service (e.g., Cloudflare Pages, Vercel, Netlify, or Nginx).
> **Note:** Camera APIs and PWA Service Workers require **HTTPS** (or `localhost`) in production browsers.

### 3. Deploy the Signaling Server
Deploy `server/server.ts` to any Node.js container or VPS (e.g. Railway, Render, Fly.io, or DigitalOcean) with WebSocket (`wss://`) support.

---

## 📁 Project Structure

```
Data Transfer PWA/
├── public/
│   ├── favicon.svg             # Brand vector logo
│   └── icons/                  # PWA application icons
├── server/
│   ├── server.ts               # WebSocket signaling server & local IP detection
│   └── types.ts                # Signaling message protocols
├── src/
│   ├── components/
│   │   ├── Header.tsx          # Brand header, P2P status badge, mute & settings
│   │   ├── HomeScreen.tsx      # Mode selector cards (Receive vs Send), 3-step guide
│   │   ├── ReceiveScreen.tsx   # QR code generation, 6-char PIN, waiting state
│   │   ├── SendScreen.tsx      # Camera QR scanner, rear-camera toggle, torch, manual input
│   │   ├── DataSelectionScreen.tsx # Photos, Videos, Files tabs with previews
│   │   ├── TransferScreen.tsx  # Circular gauge, live speed (MB/s), ETA, pause/resume
│   │   ├── CompletionScreen.tsx # Verified file summary, download & retry actions
│   │   ├── SettingsModal.tsx   # Signaling URL, detected IPs, STUN/TURN config
│   │   └── AlertBanner.tsx     # Non-blocking alert & error recovery banners
│   ├── lib/
│   │   ├── webrtc/
│   │   │   ├── WebRTCManager.ts# RTCPeerConnection lifecycle & DataChannel manager
│   │   │   └── config.ts       # STUN/TURN servers & 64KB chunk parameters
│   │   ├── transfer/
│   │   │   ├── FileSender.ts   # Flow control, backpressure, chunking, speed/ETA
│   │   │   ├── FileReceiver.ts # Chunk reassembly, SHA-256 verify, file saving
│   │   │   ├── checksum.ts     # Web Crypto hardware-accelerated SHA-256
│   │   │   └── types.ts        # Transfer state & control message definitions
│   │   ├── storage/
│   │   │   ├── fileSaver.ts    # File System Access API & browser download fallbacks
│   │   │   └── idb.ts          # IndexedDB session persistence & history
│   │   ├── qr/
│   │   │   ├── qrGenerator.ts  # QR pairing payload serialization & parser
│   │   │   └── qrScanner.ts    # html5-qrcode camera integration
│   │   ├── signaling/
│   │   │   └── SignalingClient.ts # WebSocket client with heartbeat & auto-reconnect
│   │   └── utils/
│   │       ├── formatters.ts   # formatBytes, formatSpeed, formatDuration
│   │       ├── fileTypes.ts    # Media classification & video thumbnailing
│   │       └── sound.ts        # Synthesized Web Audio chimes
│   ├── App.tsx                 # Main application state machine & screen router
│   ├── main.tsx                # React root entry point
│   └── index.css               # Tailwind CSS directives & custom micro-animations
├── vite.config.ts              # Vite + VitePWA + 0.0.0.0 network host configuration
├── tailwind.config.js          # Tailwind theme tokens & glowing shadow styles
└── package.json
```

---

## 📜 License
MIT License. Built for secure, privacy-first, zero-cloud phone migration.
