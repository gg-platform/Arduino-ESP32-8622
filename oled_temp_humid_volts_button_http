#include <Wire.h>
#include <WiFi.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <DHT.h>
#include <math.h>

// --- OLED Setup ---
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
#define OLED_ADDR     0x3C   // change to 0x3D if needed
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// --- DHT11 Setup ---
#define DHTPIN 14
#define DHTTYPE DHT11
DHT dht(DHTPIN, DHTTYPE);

// --- Voltage Sensor Setup ---
const int voltagePin = 33;
const float Vref = 3.3;
const float voltageCalFactor = 5.0;

// --- Buttons and Buzzer ---
const int button1PinManualOverride = 26;
const int button2PinGeneratorRunning = 27;
const int buzzerPin = 25;

// --- Networking Setup ---
const char* ssid = "LAN down under";
const char* password = "01454412408";

// ---- Names / endpoints ----
const char* gennyName       = "bigGenny";
const char* deviceName      = "Office";       // temp/humidity device
const char* batteryDeviceName = "officeBattery"; // voltage device

const char* getGennyUrl     = "https://gotgreens.farm/_functions/getGeneratorState?gennyName=bigGenny"; 
const char* postGennyUrl    = "https://gotgreens.farm/_functions/setGeneratorState"; 
const char* postSensorUrl   = "https://gotgreens.farm/_functions/setSensorReading"; 

// ---- Auth tokens ----
const char* GENNY_POST_AUTH_TOKEN = "12v5vIIOOj0c9VizPRz0eoF7xwkta9F27slcKKx7GugbVc81o3bxUZPvQsFzLjlkebicIpfI9uBsRCoB9rhyChqKjGGGwyCUoEvG0bi6UEw03fxlgwSETg3yyBf9MX8iuHJbjJwTUkQC2ZjdWeQgiqElSkpIOvS1cU66HEhxsy684jzyGg3EBuBQjJzJwOmiIMCKh42Bs3O";                 // for generator endpoints 
const char* SENSOR_AUTH_TOKEN     = "12v5vARDSENSORwkta9F27slcKKx7GugbVc81o3bxUZPvQsFzLjlkebicIpfI9uBsRCoB9rhyChqKjGGGwyCUoEvG0bi6UEw03fxlgwSETg3yyBf9MX8iuHJbjJwTUkQC2ZjdWeQgiqElSkpIOvS1cU66HEhxsy684jzyGg3EBuBQjJzJwOmiIMCKh42Bs3O";  

// --- Voltage thresholds (hysteresis) ---
const float startVoltage = 12.2; // run when BELOW this
const float stopVoltage  = 13.4; // stop when ABOVE this

// --- State (backend truth) ---
bool manualOverride           = true; // from backend
bool backendGeneratorState    = false; // from backend
bool generatorRunning         = false; // our current/last posted state

bool wifiConnected = false;

volatile bool isPosting = false;
volatile bool isGetting = false;

unsigned long lastGennySync    = 0;             // last GET for backend state
unsigned long lastSensorPost   = 0;             // last POST for sensor readings
const unsigned long syncIntervalMs    = 30000;  // 30s for GET + auto-control decision
const unsigned long sensorPostEveryMs = 30000;  // 30s for sensor reading POSTs

// -----------------------------
// Sync indicator (GET/POST/OFF)
// -----------------------------
enum SyncDisp { SYNC_OFF=0, SYNC_WAIT=1, SYNC_GET=2, SYNC_POST=3 };
SyncDisp syncDispCurrent = SYNC_WAIT;
unsigned long syncLockUntil = 0;
unsigned long syncBlinkStart = 0;
const unsigned long SYNC_LOCK_MS  = 2500;
const unsigned long SYNC_BLINK_MS = 300;
const unsigned long SYNC_BLINK_WINDOW_MS = 1200;

inline int syncPriority(SyncDisp s) {
  switch (s) { case SYNC_POST: return 3; case SYNC_GET: return 2; case SYNC_WAIT: return 1; case SYNC_OFF: default: return 0; }
}
void syncEvent(SyncDisp s) {
  syncDispCurrent = s;
  unsigned long now = millis();
  syncLockUntil = now + SYNC_LOCK_MS;
  syncBlinkStart = now;
}
void syncUpdateIdle(bool wifiOK, bool activePost, bool activeGet) {
  SyncDisp desired = wifiOK ? (activePost ? SYNC_POST : (activeGet ? SYNC_GET : SYNC_WAIT)) : SYNC_OFF;
  if (syncPriority(desired) > syncPriority(syncDispCurrent)) { syncEvent(desired); return; }
  if (!wifiOK && syncDispCurrent != SYNC_OFF) { syncEvent(SYNC_OFF); return; }
  if (desired == SYNC_WAIT && millis() >= syncLockUntil && syncDispCurrent != SYNC_WAIT) {
    syncDispCurrent = SYNC_WAIT;
  }
}
bool syncShouldBlink() {
  unsigned long age = millis() - syncBlinkStart;
  if (age < SYNC_BLINK_WINDOW_MS) return ((millis() / SYNC_BLINK_MS) % 2) == 0;
  return false;
}
bool syncIsLarge() { return millis() < syncLockUntil; }

// -----------------------------
// Debounced buttons
// -----------------------------
struct DebouncedButton {
  uint8_t pin;
  bool stableState;
  bool lastReading;
  uint32_t lastChangeMs;
  const uint16_t debounceMs = 30;
  void begin(uint8_t p) { pin = p; pinMode(pin, INPUT_PULLUP); stableState = HIGH; lastReading = HIGH; lastChangeMs = 0; }
  bool fell() {
    bool reading = digitalRead(pin);
    uint32_t now = millis();
    if (reading != lastReading) { lastChangeMs = now; lastReading = reading; }
    if ((now - lastChangeMs) >= debounceMs && stableState != reading) { stableState = reading; if (stableState == LOW) return true; }
    return false;
  }
};
DebouncedButton btnOverride, btnGenny;

uint32_t lastRenderMs   = 0;   const uint16_t renderIntervalMs = 50;
uint32_t lastDhtMs      = 0;   const uint16_t dhtIntervalMs    = 1000;
uint32_t lastVoltMs     = 0;   const uint16_t voltIntervalMs   = 250;

// shared sensor cache
float tCache = NAN, hCache = NAN, vCache = NAN;

// activity / auto-dim
uint32_t lastActivityMs = 0;
const uint32_t DIM_AFTER_MS = 60000;
bool isDimmed = false;
const uint8_t BRIGHT_CONTRAST = 0x8F;
const uint8_t DIM_CONTRAST    = 0x02;

void setOLEDContrast(uint8_t val) {
  Wire.beginTransmission(OLED_ADDR); Wire.write(0x00); Wire.write(0x81); Wire.write(val); Wire.endTransmission();
}
void brightenIfNeeded() {
  lastActivityMs = millis();
  if (isDimmed) { setOLEDContrast(BRIGHT_CONTRAST); isDimmed = false; }
}

// -----------------------------
// Buzzers
// -----------------------------
void buzzAckPattern() { digitalWrite(buzzerPin, HIGH); delay(120); digitalWrite(buzzerPin, LOW); }
void buzzOnPattern()  { for (int i=0;i<2;i++){ digitalWrite(buzzerPin,HIGH); delay(60); digitalWrite(buzzerPin,LOW); delay(60);} }
void buzzOffPattern() { digitalWrite(buzzerPin, HIGH); delay(120); digitalWrite(buzzerPin, LOW); }
void buzzErrorPattern(){ for (int i=0;i<2;i++){ digitalWrite(buzzerPin,HIGH); delay(220); digitalWrite(buzzerPin,LOW); delay(90);} }

// -----------------------------
// Helpers
// -----------------------------
float readBatteryVoltage() {
  int raw = analogRead(voltagePin);
  float sensorV = (raw / 4095.0f) * Vref;
  return sensorV * voltageCalFactor;
}
int wifiBarsFromRSSI(int rssi) {
  if (rssi > -50) return 4;
  if (rssi > -60) return 3;
  if (rssi > -70) return 2;
  if (rssi > -80) return 1;
  return 0;
}
void drawWifiAt(int x, int y, int bars) {
  for (int i = 0; i < 4; i++) {
    int h = (i + 1) * 2;
    int yy = y + 7 - h;
    if (i < bars) display.fillRect(x + i*3, yy, 2, h, SSD1306_WHITE);
    else          display.drawRect(x + i*3, yy, 2, h, SSD1306_WHITE);
  }
}

// --- Sync indicator shapes (8x8 or 12x12) ---
void drawSyncIndicatorAt(int x, int y, SyncDisp st, bool large, bool blink) {
  int sz = large ? 12 : 8;
  display.fillRect(x, y, sz, sz, SSD1306_BLACK);
  if (blink) { display.drawRect(x, y, sz, sz, SSD1306_WHITE); return; }

  if (!large) {
    switch (st) {
      case SYNC_WAIT: display.drawCircle(x+4, y+4, 3, SSD1306_WHITE); break;
      case SYNC_GET:  display.fillTriangle(x, y+1, x+8, y+1, x+4, y+7, SSD1306_WHITE); break;
      case SYNC_POST: display.fillTriangle(x+4, y+1, x, y+7, x+8, y+7, SSD1306_WHITE); break;
      case SYNC_OFF:
      default:        display.fillRect(x+2, y+2, 4, 4, SSD1306_WHITE); break;
    }
  } else {
    switch (st) {
      case SYNC_WAIT: display.drawCircle(x+6, y+6, 5, SSD1306_WHITE); break;
      case SYNC_GET:  display.fillTriangle(x+1, y+2, x+11, y+2, x+6, y+10, SSD1306_WHITE); break;
      case SYNC_POST: display.fillTriangle(x+6, y+2, x+1, y+10, x+11, y+10, SSD1306_WHITE); break;
      case SYNC_OFF:
      default:        display.fillRect(x+3, y+3, 6, 6, SSD1306_WHITE); break;
    }
  }
}

// ---- UI bits ----
void drawMiniPill(int x, int y, int w, int h, bool on, const char* onTxt="ON", const char* offTxt="OFF") {
  display.drawRoundRect(x, y, w, h, 5, SSD1306_WHITE);
  if (on) {
    display.fillRoundRect(x+1, y+1, w-2, h-2, 4, SSD1306_WHITE);
    display.setTextSize(1); display.setTextColor(SSD1306_BLACK);
    int tx = x + w/2 - 6; int ty = y + (h/2 - 3);
    display.setCursor(tx, ty); display.print(onTxt);
    display.setTextColor(SSD1306_WHITE);
  } else {
    display.setTextSize(1);
    int tx = x + w/2 - 9; int ty = y + (h/2 - 3);
    display.setCursor(tx, ty); display.print(offTxt);
  }
}
void drawStatusColumn(int x, int y, int w, const char* label, bool state) {
  display.setTextSize(1); display.setTextColor(SSD1306_WHITE);
  display.setCursor(x, y); display.print(label);
  int pillY = y + 8; int pillH = 16;
  int pillX = x + 2; int pillW = w - 4;
  drawMiniPill(pillX, pillY, pillW, pillH, state);
}
void drawTopBandAndHeader(const char* name, int rssi, float volts) {
  display.setTextSize(1); display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 2); display.print(volts, 1); display.print("V");
  int bars = wifiBarsFromRSSI(rssi);
  drawWifiAt(88, 2, bars);
  bool large = syncIsLarge();
  bool blink = syncShouldBlink();
  int sx = large ? 112 : 116; int sy = 2;
  drawSyncIndicatorAt(sx, sy, syncDispCurrent, large, blink);
  display.drawFastHLine(0, 16, SCREEN_WIDTH, SSD1306_WHITE);
  display.setCursor(0, 18);
  String n = String(name); if (n.length() > 12) n = n.substring(0,12);
  display.print(n);
}
void renderUI(const char* name,
              float tC, float hPct, float volts,
              int rssi,
              bool overrideOn, bool gennyOn,
              unsigned long secsLeft) {
  display.clearDisplay();
  drawTopBandAndHeader(name, rssi, volts);

  const int colY  = 26;
  const int colW  = 60; const int gap = 8;
  const int leftX = 4; const int rightX= leftX + colW + gap;
  drawStatusColumn(leftX,  colY, colW, "OVR", overrideOn);
  drawStatusColumn(rightX, colY, colW, "RUN", gennyOn);
  display.drawFastVLine(leftX + colW + gap/2, colY-2, 24, SSD1306_WHITE);

  display.drawFastHLine(0, 50, SCREEN_WIDTH, SSD1306_WHITE);
  display.setTextSize(1); display.setTextColor(SSD1306_WHITE);
  display.setCursor(2, 52);
  display.print("T "); display.print(tC,1); display.print("C  ");
  display.print("H "); display.print(hPct,0); display.print("%");
  String s = String(secsLeft) + "s";
  int xRight = SCREEN_WIDTH - (s.length() * 6) - 2; if (xRight < 80) xRight = 80;
  display.setCursor(xRight, 52); display.print(s);
  display.display();
}

// -----------------------------
// HTTP helpers
// -----------------------------
void postGeneratorState(bool overrideState, bool genState) {
  if (!wifiConnected) return;
  isPosting = true; syncEvent(SYNC_POST); brightenIfNeeded();

  HTTPClient http;
  http.begin(postGennyUrl);
  http.addHeader("Content-Type", "application/json");
  http.addHeader("x-auth-token", GENNY_POST_AUTH_TOKEN);

  StaticJsonDocument<200> doc;
  doc["gennyName"] = gennyName;
  doc["generatorRunning"] = genState;
  doc["manualOverride"] = overrideState;

  String jsonStr; serializeJson(doc, jsonStr);
  int code = http.POST(jsonStr);
  Serial.printf("📡 POST setGeneratorState (%d): %s\n", code, jsonStr.c_str());
  http.end();

  isPosting = false; brightenIfNeeded();
}

void postSensorReading(float temp, float humidity) {
  if (!wifiConnected) return;
  isPosting = true; syncEvent(SYNC_POST); brightenIfNeeded();

  HTTPClient http;
  http.begin(postSensorUrl);
  http.addHeader("Content-Type", "application/json");
  http.addHeader("x-auth-token", SENSOR_AUTH_TOKEN);

  StaticJsonDocument<200> doc;
  doc["deviceName"] = deviceName;       // temp/humidity device
  if (!isnan(temp))     doc["temp"] = temp;         // optional
  if (!isnan(humidity)) doc["humidity"] = humidity; // optional

  String jsonStr; serializeJson(doc, jsonStr);
  int code = http.POST(jsonStr);
  Serial.printf("📡 POST setSensorReading (T/H) (%d): %s\n", code, jsonStr.c_str());
  http.end();

  isPosting = false; brightenIfNeeded();
}

// ★ NEW: dedicated voltage post using deviceName="officeBattery"
void postVoltageReading(float volts) {
  if (!wifiConnected || isnan(volts)) return;
  isPosting = true; syncEvent(SYNC_POST); brightenIfNeeded();

  HTTPClient http;
  http.begin(postSensorUrl);
  http.addHeader("Content-Type", "application/json");
  http.addHeader("x-auth-token", SENSOR_AUTH_TOKEN);

  StaticJsonDocument<200> doc;
  doc["deviceName"] = batteryDeviceName; // "officeBattery"
  doc["value"]    = volts;

  String jsonStr; serializeJson(doc, jsonStr);
  int code = http.POST(jsonStr);
  Serial.printf("📡 POST setSensorReading (V) (%d): %s\n", code, jsonStr.c_str());
  http.end();

  isPosting = false; brightenIfNeeded();
}

void getGeneratorStatus() {
  if (!wifiConnected) return;
  isGetting = true; syncEvent(SYNC_GET);

  HTTPClient http;
  http.begin(getGennyUrl);
  http.addHeader("x-auth-token", GENNY_POST_AUTH_TOKEN);

  int code = http.GET();
  if (code != 200) {
    Serial.printf("⚠️ GET genny state failed: %d\n", code);
    http.end(); isGetting = false; return;
  }

  String response = http.getString();
  StaticJsonDocument<256> doc;
  DeserializationError err = deserializeJson(doc, response);
  http.end();
  isGetting = false;

  if (err) { Serial.println("❌ JSON parse error"); return; }

  bool remoteOverride = doc["manualOverride"] | false;
  bool remoteGenny    = doc["generatorRunning"] | false;

  manualOverride = remoteOverride;
  backendGeneratorState = remoteGenny;
  // On first sync, align our local state to backend
  static bool firstSync = true;
  if (firstSync) { generatorRunning = backendGeneratorState; firstSync = false; }
}

// -----------------------------
// Setup / Loop
// -----------------------------
void setup() {
  Serial.begin(115200);
  delay(300);
  Wire.begin(4, 5);

  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println(F("SSD1306 allocation failed")); while (1);
  }

  display.clearDisplay();
  display.setTextSize(2); display.setTextColor(SSD1306_WHITE);
  display.setCursor(8, 18); display.print("GENNY");
  display.setTextSize(1); display.setCursor(10, 40); display.print("Monitor v1.0");
  display.display(); delay(800); display.clearDisplay();

  dht.begin();
  btnOverride.begin(button1PinManualOverride);
  btnGenny.begin(button2PinGeneratorRunning);
  pinMode(buzzerPin, OUTPUT); digitalWrite(buzzerPin, LOW);

  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) { delay(300); Serial.print("."); }
  wifiConnected = true;

  lastGennySync = millis();
  lastSensorPost = millis();
  lastActivityMs = millis();
  setOLEDContrast(BRIGHT_CONTRAST);
  syncEvent(wifiConnected ? SYNC_WAIT : SYNC_OFF);
}

void loop() {
  uint32_t now = millis();
  wifiConnected = (WiFi.status() == WL_CONNECTED);

  // Inputs
  if (btnOverride.fell()) {
    manualOverride = !manualOverride;
    buzzAckPattern(); brightenIfNeeded();
    // Post change immediately; keep generatorRunning as-is
    postGeneratorState(manualOverride, generatorRunning);
  }
  if (btnGenny.fell()) {
    if (manualOverride) {
      bool old = generatorRunning;
      generatorRunning = !generatorRunning;
      if (generatorRunning != old) { if (generatorRunning) buzzOnPattern(); else buzzOffPattern(); }
      brightenIfNeeded();
      postGeneratorState(manualOverride, generatorRunning);
    } else {
      buzzErrorPattern(); brightenIfNeeded();
    }
  }

  // Sensors on timers
  if (now - lastDhtMs >= dhtIntervalMs) { tCache = dht.readTemperature(); hCache = dht.readHumidity(); lastDhtMs = now; }
  if (now - lastVoltMs >= voltIntervalMs) { vCache = readBatteryVoltage(); lastVoltMs = now; }

  // Periodic GET of backend (every 30s)
  if (now - lastGennySync >= syncIntervalMs && !isPosting && !isGetting) {
    getGeneratorStatus();
    lastGennySync = now;

    // Decide desired generator state after GET:
    float volts = isnan(vCache) ? readBatteryVoltage() : vCache;
    bool desired = generatorRunning; // default to current for hysteresis band

    if (manualOverride) {
      // Respect manual override: do what backend wants
      desired = backendGeneratorState;
      Serial.printf("💪 Manual override active → backend wants: %s\n", desired ? "RUN" : "STOP");
    } else {
      if (volts < startVoltage) desired = true;
      else if (volts > stopVoltage) desired = false;
      // else keep current state (hysteresis)
      Serial.printf("🔋 Auto by V=%.2fV → desired: %s\n", volts, desired ? "RUN" : "STOP");
    }

    if (desired != generatorRunning) {
      generatorRunning = desired;
      if (generatorRunning) buzzOnPattern(); else buzzOffPattern();
      postGeneratorState(manualOverride, generatorRunning);
    }
  }

  // Periodic POST of sensor readings (every 30s) — independent of genny posts
  if ((now - lastSensorPost >= sensorPostEveryMs) && wifiConnected && !isPosting && !isGetting) {
    float t = isnan(tCache) ? dht.readTemperature() : tCache;
    float h = isnan(hCache) ? dht.readHumidity()    : hCache;
    float v = isnan(vCache) ? readBatteryVoltage()  : vCache;

    postSensorReading(t, h);   // existing temp/humidity post
    postVoltageReading(v);     // ★ NEW: voltage post to "officeBattery"

    lastSensorPost = now;
  }

  // Countdown to next GET for UI
  uint32_t elapsed = now - lastGennySync;
  uint32_t secsLeft = (elapsed >= syncIntervalMs) ? 0 : (syncIntervalMs - elapsed) / 1000;

  // Update sync indicator state machine
  syncUpdateIdle(wifiConnected, isPosting, isGetting);

  // Render ~20fps
  if (now - lastRenderMs >= renderIntervalMs) {
    int rssi = wifiConnected ? WiFi.RSSI() : -1000;
    float t = isnan(tCache) ? dht.readTemperature() : tCache;
    float h = isnan(hCache) ? dht.readHumidity()    : hCache;
    float v = isnan(vCache) ? readBatteryVoltage()  : vCache;
    renderUI(gennyName, t, h, v, rssi, manualOverride, generatorRunning, secsLeft);
    lastRenderMs = now;
  }

  // Safe dim
  if (!isDimmed && (now - lastActivityMs) >= DIM_AFTER_MS) { setOLEDContrast(DIM_CONTRAST); isDimmed = true; }
}
