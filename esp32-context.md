
## 1. ESP32 Arduino Sketch

Create a new Arduino sketch and paste this. It sets up a BLE GATT server with the exact UUIDs the web app expects, parses the JSON payload, and displays on SSD1306 OLED.

**Required libraries** (install via Arduino Library Manager):

-   `Adafruit SSD1306`
-   `Adafruit GFX Library`
-   `ArduinoJson`  (by Benoit Blanchon)
-   BLE is built into the ESP32 Arduino core


```c
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <ArduinoJson.h>

// ── OLED config ──────────────────────────────────────────────
#define SCREEN_WIDTH  128
#define SCREEN_HEIGHT  64
#define OLED_RESET     -1      // no reset pin
#define OLED_ADDRESS  0x3C     // common address (try 0x3D if blank)
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// ── BLE UUIDs — must match the web app exactly ───────────────
#define SERVICE_UUID  "12345678-1234-1234-1234-123456789012"
#define CHAR_UUID     "87654321-4321-4321-4321-210987654321"

BLEServer*         pServer   = nullptr;
BLECharacteristic* pChar     = nullptr;
bool               connected = false;

// ── Show two lines on OLED ────────────────────────────────────
void showTx(const String& line1, const String& line2) {
  display.clearDisplay();
  display.setTextSize(2);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 8);
  display.println(line1);
  display.setCursor(0, 36);
  display.println(line2);
  display.display();
}

void showStatus(const String& msg) {
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 28);
  display.println(msg);
  display.display();
}

// ── BLE server callbacks ──────────────────────────────────────
class ServerCallbacks : public BLEServerCallbacks {
  void onConnect(BLEServer*) override {
    connected = true;
    showStatus("Phone connected");
    Serial.println("[BLE] Connected");
  }
  void onDisconnect(BLEServer* s) override {
    connected = false;
    showStatus("Waiting...");
    Serial.println("[BLE] Disconnected — advertising again");
    s->startAdvertising();
  }
};

// ── Characteristic write callback — fires on each BLE packet ─
class CharCallbacks : public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic* pC) override {
    String raw = pC->getValue().c_str();
    Serial.print("[BLE] Received: "); Serial.println(raw);

    StaticJsonDocument<128> doc;
    if (deserializeJson(doc, raw) != DeserializationError::Ok) {
      Serial.println("[JSON] Parse error");
      showStatus("Bad payload");
      return;
    }

    const char* t = doc["t"] | "";
    if (strcmp(t, "tx") != 0) return;          // ignore non-tx packets

    const char* v = doc["v"] | "0";
    const char* c = doc["c"] | "?";            // "base" or "eth"
    const char* d = doc["d"] | "out";          // "in" | "out" | "self"

    String chain = String(c);
    chain.toUpperCase();                        // "BASE" or "ETH"

    String line1, line2;

    if (strcmp(d, "self") == 0) {
      line1 = "SELF TX";
      line2 = "0 ETH";
    } else if (strcmp(d, "in") == 0) {
      line1 = "<< " + chain;
      line2 = "+" + String(v) + " ETH";
    } else {                                    // "out"
      line1 = ">> " + chain;
      line2 = String(v) + " ETH";
    }

    showTx(line1, line2);

    // Clear back to idle after 8 seconds
    delay(8000);
    showStatus(connected ? "Listening..." : "Waiting...");
  }
};

// ── Setup ─────────────────────────────────────────────────────
void setup() {
  Serial.begin(115200);

  // Init OLED
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDRESS)) {
    Serial.println("[OLED] Init failed — check wiring");
    while (true) delay(1000);
  }
  display.clearDisplay();
  display.display();
  showStatus("Starting BLE...");

  // Init BLE
  BLEDevice::init("ChainPulse");              // device name shown in browser picker
  pServer = BLEDevice::createServer();
  pServer->setCallbacks(new ServerCallbacks());

  BLEService* pService = pServer->createService(SERVICE_UUID);
  pChar = pService->createCharacteristic(
    CHAR_UUID,
    BLECharacteristic::PROPERTY_WRITE_NR    // write without response = low latency
  );
  pChar->setCallbacks(new CharCallbacks());
  pService->start();

  BLEAdvertising* pAdv = BLEDevice::getAdvertising();
  pAdv->addServiceUUID(SERVICE_UUID);
  pAdv->setScanResponse(true);
  BLEDevice::startAdvertising();

  showStatus("Waiting...");
  Serial.println("[BLE] Advertising as 'ChainPulse'");
}

void loop() {
  delay(1000);
}
```


**Wiring (I2C OLED to ESP32):**

|  OLED PIN| ESP32 PIN |
|--|--|
| VCC | 3.3 V |
| GND | GND |
| SDA | GPIO 21 |
| SCL | GPIO 22 |


----------

## 2. Test Flow — Step by Step

### A. Flash & verify ESP32

1.  Flash the sketch via Arduino IDE (select board:  **ESP32 Dev Module**)
2.  Open Serial Monitor at  **115200 baud**
3.  OLED should show  `Waiting...`
4.  Serial should print  `[BLE] Advertising as 'ChainPulse'`

### B. Connect from the web app

1.  Open  `http://localhost:3000`  in  **Chrome on desktop or Android**  (not iOS, not Firefox — Web Bluetooth not supported there)
2.  Connect your wallet → dashboard loads
3.  Scroll to the  **ESP32 BLE**  panel at the bottom
4.  Click  **Connect ESP32**  → Chrome shows a device picker
5.  Select  **ChainPulse**  from the list
6.  OLED shows  `Phone connected`, Serial prints  `[BLE] Connected`

### C. Trigger a test transaction (two options)

**Option 1 — Real transaction (cleanest test)**

-   Send any amount of ETH/Base from your connected wallet
-   Wait up to 15 seconds (poll interval) — when the polling picks it up, it calls  `blePanel.sendTx(tx)`  automatically
-   OLED displays e.g.  `>> BASE`  /  `0.001 ETH`
-   BLE panel send log shows the entry

**Option 2 — Simulate without spending gas** 
Add this temporary button to `Dashboard.tsx` for instant testing:

```c
// Add near the top of Dashboard.tsx return, inside <main>
<button
  className="text-xs bg-purple-600 px-3 py-1.5 rounded-lg text-white"
  onClick={() => blePanelRef.current?.sendTx({
    hash: '0xtest', from: address, to: '0x0',
    valueEth: 0.042, gasCostEth: 0,
    timestamp: Date.now() / 1000, date: new Date().toISOString().split('T')[0],
    chain: 'base', direction: 'out', isError: false, blockNumber: 0,
  })}
>
  Test BLE Send
</button>
```
Click it → immediately sends `{"t":"tx","v":"0.042","c":"base","d":"out"}` to ESP32 → OLED shows `>> BASE` / `0.042 ETH`.


## 3. Troubleshooting

|Symptom  | Fix |
|--|--|
| Device picker is empty | ESP32 not advertising — check Serial for init errors; retry after power cycle |
|OLED blank|Try  `OLED_ADDRESS 0x3D`; check SDA/SCL wiring|
|`Bad payload`  on OLED|BLE MTU too small — increase  `StaticJsonDocument<128>`  to  `<256>`|
|Chrome says "Bluetooth unavailable"|Must use HTTPS in production;  `localhost`  is exempt|
|Transaction detected but BLE not firing|Wallet not connected to BLE panel — status badge must show "Connected"|
|`delay(8000)`  blocks BLE writes|Replace with a non-blocking timer using  `millis()`  if multiple rapid txs expected|


**The animation plan before writing code:**

| Direction | Chain | OLED Animation | Frames |
|--|--|--|--|
|`out`|base/eth|Arrows sweep right → then hold result|`>`  `>>`  `>>>`  →  `>> BASE`  /  `0.05 ETH`|
|`in`|base/eth|Arrows sweep left ← then hold result|`<`  `<<`  `<<<`  →  `<< BASE`  /  `+0.05 ETH`|
|`self`|any|Spinner then hold|`|`  `/`  `-`  `\`  × 3 →  `SELF TX`|

Here is the full ESP32 sketch with non-blocking animations using `millis()`:

```c
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <ArduinoJson.h>

// ── OLED ─────────────────────────────────────────────────────
#define SCREEN_WIDTH  128
#define SCREEN_HEIGHT  64
#define OLED_RESET     -1
#define OLED_ADDR     0x3C
Adafruit_SSD1306 oled(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// ── BLE ───────────────────────────────────────────────────────
#define SERVICE_UUID "12345678-1234-1234-1234-123456789012"
#define CHAR_UUID    "87654321-4321-4321-4321-210987654321"
BLEServer*         pServer = nullptr;
BLECharacteristic* pChar   = nullptr;
bool               bleConnected = false;

// ── Animation state machine ───────────────────────────────────
enum AnimType { ANIM_NONE, ANIM_OUT, ANIM_IN, ANIM_SELF };

struct TxAnim {
  AnimType  type      = ANIM_NONE;
  String    chain     = "";     // "BASE" or "ETH"
  String    value     = "";     // "0.042"
  String    direction = "";     // "out" or "in"

  uint8_t   frame     = 0;
  uint32_t  lastTick  = 0;
  bool      holding   = false;
  uint32_t  holdStart = 0;
};

TxAnim anim;
volatile bool newTxPending = false;
char pendingRaw[128];

// ── Small helpers ─────────────────────────────────────────────
void oledLine(const String& l1, const String& l2, uint8_t size1 = 2, uint8_t size2 = 2) {
  oled.clearDisplay();
  oled.setTextColor(SSD1306_WHITE);
  oled.setTextSize(size1);
  oled.setCursor(0, 4);
  oled.print(l1);
  oled.setTextSize(size2);
  oled.setCursor(0, 36);
  oled.print(l2);
  oled.display();
}

void oledStatus(const String& msg) {
  oled.clearDisplay();
  oled.setTextSize(1);
  oled.setTextColor(SSD1306_WHITE);
  oled.setCursor(0, 28);
  oled.print(msg);
  oled.display();
}

// ── Tick animation (called every loop) ───────────────────────
void tickAnim() {
  if (anim.type == ANIM_NONE) return;

  uint32_t now = millis();

  // ── Holding the result screen ──────────────────────────────
  if (anim.holding) {
    if (now - anim.holdStart >= 6000) {   // 6 s display
      anim.type = ANIM_NONE;
      anim.holding = false;
      oledStatus(bleConnected ? "Listening..." : "Waiting...");
    }
    return;
  }

  // ── OUT animation: arrows sweep right ─────────────────────
  if (anim.type == ANIM_OUT) {
    const uint16_t FRAME_MS = 120;
    const uint8_t  FRAMES   = 9;   // 3 build-up + 3 full + 3 fast exit
    if (now - anim.lastTick < FRAME_MS) return;
    anim.lastTick = now;

    oled.clearDisplay();
    oled.setTextColor(SSD1306_WHITE);

    if (anim.frame < 3) {
      // Build up: 1→2→3 arrows
      String arrows = "";
      for (uint8_t i = 0; i <= anim.frame; i++) arrows += ">";
      oled.setTextSize(3);
      oled.setCursor(0, 18);
      oled.print(arrows);
    } else if (anim.frame < 6) {
      // Slide across: arrows moving right
      uint8_t x = (anim.frame - 3) * 28;
      oled.setTextSize(3);
      oled.setCursor(x, 18);
      oled.print(">>>");
    } else {
      // Final: show result
      anim.holding   = true;
      anim.holdStart = now;
      oledLine(">>" + anim.chain, anim.value + " ETH");
      return;
    }
    oled.display();
    anim.frame++;
    if (anim.frame >= FRAMES) {
      anim.holding = true;
      anim.holdStart = now;
      oledLine(">>" + anim.chain, anim.value + " ETH");
    }
  }

  // ── IN animation: arrows sweep left ───────────────────────
  else if (anim.type == ANIM_IN) {
    const uint16_t FRAME_MS = 120;
    if (now - anim.lastTick < FRAME_MS) return;
    anim.lastTick = now;

    oled.clearDisplay();
    oled.setTextColor(SSD1306_WHITE);

    if (anim.frame < 3) {
      // Build up from right
      String arrows = "";
      for (uint8_t i = 0; i <= anim.frame; i++) arrows += "<";
      uint8_t x = SCREEN_WIDTH - 6 - (anim.frame + 1) * 22;
      oled.setTextSize(3);
      oled.setCursor(x, 18);
      oled.print(arrows);
    } else if (anim.frame < 6) {
      // Slide to left
      int x = 72 - (anim.frame - 3) * 28;
      oled.setTextSize(3);
      oled.setCursor(max(0, x), 18);
      oled.print("<<<");
    } else {
      anim.holding   = true;
      anim.holdStart = now;
      oledLine("<<" + anim.chain, "+" + anim.value + " ETH");
      return;
    }
    oled.display();
    anim.frame++;
    if (anim.frame >= 9) {
      anim.holding = true;
      anim.holdStart = now;
      oledLine("<<" + anim.chain, "+" + anim.value + " ETH");
    }
  }

  // ── SELF animation: spinner ───────────────────────────────
  else if (anim.type == ANIM_SELF) {
    const uint16_t FRAME_MS = 100;
    const char SPIN[] = {'|', '/', '-', '\\'};
    if (now - anim.lastTick < FRAME_MS) return;
    anim.lastTick = now;

    if (anim.frame < 16) {   // 16 spinner frames (~1.6 s)
      oled.clearDisplay();
      oled.setTextSize(3);
      oled.setTextColor(SSD1306_WHITE);
      oled.setCursor(52, 18);
      oled.print(SPIN[anim.frame % 4]);
      oled.display();
      anim.frame++;
    } else {
      anim.holding   = true;
      anim.holdStart = now;
      oledLine("SELF TX", "  0 ETH");
    }
  }
}

// ── BLE callbacks ─────────────────────────────────────────────
class ServerCB : public BLEServerCallbacks {
  void onConnect(BLEServer*) override {
    bleConnected = true;
    oledStatus("Listening...");
    Serial.println("[BLE] Connected");
  }
  void onDisconnect(BLEServer* s) override {
    bleConnected = false;
    anim.type = ANIM_NONE;
    oledStatus("Waiting...");
    Serial.println("[BLE] Disconnected");
    s->startAdvertising();
  }
};

class CharCB : public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic* pC) override {
    // Copy to global buffer and set flag — keep callback fast
    String v = pC->getValue().c_str();
    v.toCharArray(pendingRaw, sizeof(pendingRaw));
    newTxPending = true;
  }
};

// ── Parse payload and kick off animation ─────────────────────
void handlePending() {
  StaticJsonDocument<128> doc;
  if (deserializeJson(doc, pendingRaw) != DeserializationError::Ok) {
    Serial.println("[JSON] Parse error");
    return;
  }

  const char* t = doc["t"] | "";
  if (strcmp(t, "tx") != 0) return;

  const char* v = doc["v"] | "0";
  const char* c = doc["c"] | "?";
  const char* d = doc["d"] | "out";

  Serial.printf("[TX] chain=%s dir=%s val=%s\n", c, d, v);

  // Reset animation state
  anim.frame    = 0;
  anim.holding  = false;
  anim.lastTick = millis();
  anim.value    = String(v);
  anim.chain    = " " + String(c);   // space before "BASE"/"ETH" for layout
  anim.chain.toUpperCase();

  if (strcmp(d, "in")   == 0) anim.type = ANIM_IN;
  else if (strcmp(d, "self") == 0) anim.type = ANIM_SELF;
  else                              anim.type = ANIM_OUT;
}

// ── Setup ─────────────────────────────────────────────────────
void setup() {
  Serial.begin(115200);

  if (!oled.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("[OLED] Init failed");
    while (true);
  }
  oledStatus("Starting...");

  BLEDevice::init("ChainPulse");
  pServer = BLEDevice::createServer();
  pServer->setCallbacks(new ServerCB());

  BLEService* svc = pServer->createService(SERVICE_UUID);
  pChar = svc->createCharacteristic(CHAR_UUID, BLECharacteristic::PROPERTY_WRITE_NR);
  pChar->setCallbacks(new CharCB());
  svc->start();

  BLEAdvertising* adv = BLEDevice::getAdvertising();
  adv->addServiceUUID(SERVICE_UUID);
  adv->setScanResponse(true);
  BLEDevice::startAdvertising();

  oledStatus("Waiting...");
  Serial.println("[BLE] Ready — 'ChainPulse'");
}

// ── Loop ──────────────────────────────────────────────────────
void loop() {
  if (newTxPending) {
    newTxPending = false;
    handlePending();
  }
  tickAnim();
}
```