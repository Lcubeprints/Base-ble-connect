# Chain Pulse

On-chain activity dashboard for Base + Ethereum, with live BLE alerts to an ESP32/OLED device. Works in browser and inside the Base App (Farcaster Mini App).

---

## 1 — Setup

### Prerequisites
- Node.js 18+
- A free Etherscan account for the API key
- Chrome on desktop or Android (for BLE)

### Install
```bash
cd miniapp
npm install
```

### Environment variables

Copy `.env.local` and fill in your keys:

```bash
# Required — get from https://portal.cdp.coinbase.com/
NEXT_PUBLIC_ONCHAINKIT_API_KEY=your_key_here

# Required — used for both Base mainnet (8453) and Ethereum (1) via Etherscan v2
ETHERSCAN_API_KEY=your_key_here

# Your deployed URL (leave as localhost for dev)
NEXT_PUBLIC_URL=http://localhost:3000
```

**Getting an Etherscan API key:**
1. Go to [etherscan.io](https://etherscan.io) → sign up / log in
2. Account → API Keys → Add → copy the key
3. Paste into `ETHERSCAN_API_KEY=` in `.env.local`

The same key covers Ethereum mainnet, Base mainnet, and Base Sepolia — no separate Basescan key needed.

### Run
```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000) in Chrome.

---

## 2 — Connecting Your Wallet

1. The landing page shows a **Connect** screen
2. Click **MetaMask** or **Coinbase Wallet** — approve the connection in your wallet extension
3. The dashboard loads automatically once connected

> If you open the app inside the **Base App** on mobile, your wallet connects automatically via the Farcaster Mini App connector — no button press needed.

---

## 3 — Dashboard

Once connected, the dashboard shows your transaction history across **Base mainnet** and **Ethereum mainnet**.

| Section | What it shows |
|---|---|
| **Stats cards** | Total txs, ETH sent, total gas paid, first tx date |
| **Heatmap** | 365-day activity grid (darker blue = more txs that day) |
| **Monthly bar chart** | Tx count per month for the last 12 months |
| **Pie chart** | Outgoing ETH split into value buckets |
| **Live feed** | Last 20 transactions, newest first |

### Mainnet vs Testnet

The header has a **Testnet** button (flask icon, top-right):

- **Off (default):** Shows Base mainnet + Ethereum mainnet transactions
- **On (amber):** Switches to **Base Sepolia** testnet transactions

Use testnet mode to see test transactions you've made on Sepolia without mixing them with mainnet history.

---

## 4 — BLE / ESP32 Setup

The BLE panel at the bottom of the dashboard lets you connect an ESP32 device. When a new transaction is detected, it automatically sends a compact JSON payload to the ESP32 via Bluetooth.

### Requirements

- **Browser:** Chrome on desktop or Android only (iOS and Firefox do not support Web Bluetooth)
- **Hardware:** ESP32 + SSD1306 OLED (128×64, I2C)
- **Wiring:**

  | OLED | ESP32 |
  |------|-------|
  | VCC  | 3.3V  |
  | GND  | GND   |
  | SDA  | GPIO 21 |
  | SCL  | GPIO 22 |

### Flash the ESP32

1. Open the Arduino sketch from the project (see `esp32/ChainPulse.ino` or copy from the docs)
2. Install these libraries via Arduino Library Manager:
   - `Adafruit SSD1306`
   - `Adafruit GFX Library`
   - `ArduinoJson` (by Benoit Blanchon)
3. Select board: **ESP32 Dev Module**
4. Upload — OLED should show `Waiting...`

### Connect from the app

1. Scroll to the **ESP32 BLE** card at the bottom of the dashboard
2. Click **Connect ESP32**
3. Chrome opens a device picker — select **ChainPulse**
4. OLED shows `Listening...` — you're connected

### OLED animations per transaction

| Transaction type | OLED animation | Final display |
|---|---|---|
| Outgoing (any chain) | Arrows sweep right `> >> >>>` | `>> BASE` / `0.05 ETH` |
| Incoming (any chain) | Arrows sweep left `< << <<<` | `<< ETH` / `+0.05 ETH` |
| Self-transfer | Spinner `\| / - \` | `SELF TX` / `0 ETH` |

The result screen holds for 6 seconds, then returns to `Listening...`.

### Testing BLE without a real transaction

In the dashboard, temporarily add a test button to fire a fake send instantly:

```tsx
<button onClick={() => blePanelRef.current?.sendTx({
  hash: '0xtest', from: address, to: '0x0',
  valueEth: 0.042, gasCostEth: 0,
  timestamp: Date.now() / 1000, date: new Date().toISOString().split('T')[0],
  chain: 'base', direction: 'out', isError: false, blockNumber: 0,
})}>
  Test BLE
</button>
```

---

## 5 — How Live Updates Work

The app polls for new transactions every **15 seconds** using the last known block number as a starting point (no duplicate fetches).

When a new transaction is detected:
1. The **Live feed** adds it to the top with a green `+N new` counter in the header
2. If an ESP32 is connected via BLE, the payload is sent immediately and the animation plays

---

## 6 — Deploy to Vercel

```bash
# From the miniapp directory
npx vercel
```

After deploying:
1. Update `NEXT_PUBLIC_URL` in your Vercel environment variables to your production URL
2. To preview inside the Base App: go to [base.dev/preview](https://base.dev/preview), enter your URL, and scan the QR code

---

## Troubleshooting

| Problem | Fix |
|---|---|
| No transactions showing | Check `ETHERSCAN_API_KEY` is set in `.env.local` and restart `npm run dev` |
| Only ETH txs, no Base txs | Same — the Etherscan key covers Base via `chainid=8453` |
| Connect button missing / not working | Use Chrome; MetaMask or Coinbase Wallet extension must be installed |
| BLE device picker empty | ESP32 not advertising — power cycle it and try again |
| OLED blank after flash | Try `OLED_ADDR 0x3D` instead of `0x3C` in the sketch |
| BLE not available | iOS and Firefox are not supported; use Chrome on desktop or Android |