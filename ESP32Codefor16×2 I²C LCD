/*
   ──────────────────────────────────────────────────────────────
    ESP32 • RC522 RFID Attendance • 16×2 I²C LCD
    Late-arrival detection (period = 50 min, 5 min grace)
   ──────────────────────────────────────────────────────────────
*/

#include <Wire.h>
#include <LiquidCrystal_PCF8574.h>
#include <SPI.h>
#include <MFRC522.h>
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <HTTPClient.h>
#include <time.h>

/* ▒▒▒ Wi-Fi ▒▒▒ */
const char *WIFI_SSID     = "";
const char *WIFI_PASSWORD = "";

/* ▒▒▒ Firebase ▒▒▒ */
const char *FB_HOST   = ".firebaseio.com";
const char *FB_SECRET = "****77";   // leave empty if public DB

/* ▒▒▒ Hardware ▒▒▒ */
#define LCD_ADDR 0x27
LiquidCrystal_PCF8574 lcd(LCD_ADDR);   // SDA=4, SCL=5

#define SS_PIN   21
#define RST_PIN  22
MFRC522 rfid(SS_PIN, RST_PIN);

#define BUZZER_PIN 27
inline void beep(uint16_t ms) {
  digitalWrite(BUZZER_PIN, HIGH);
  delay(ms);
  digitalWrite(BUZZER_PIN, LOW);
}

/* ▒▒▒ Admin UID ▒▒▒ */
const char *ADMIN_UID = "1CDD8A3F";

/* ▒▒▒ Class schedule ▒▒▒ */
const int GRACE_MIN = 5;
struct Period { uint8_t h, m; };
const Period periods[] = {
  {8,30},{9,25},{10,25},{10,40},{11,35},
  {13,30},{14,25},{15,20},{16,15}
};

/* ▒▒▒ Roster ▒▒▒ */
struct Student { const char *uid; const char *name; bool present; };
Student roster[60] = {
  {"71B54A3C","A S Pradeep",false},
  {"53A7BFD9","A S Pradeep",false},
  {"438217E2","A S Pradeep",false},
  {"EDC3963F","A S Pradeep",false},
  
  // add more students
};

/* ▒▒▒ Prototypes ▒▒▒ */
int     findStudentIndex(const String&);
size_t  countPresent();
String  isoTimestamp();
bool    isLate();
void    sendToCloud(const String&, const char*, bool, bool);

/* ───────── SETUP ───────── */
void setup() {
  Serial.begin(115200);

  Wire.begin(4,5);
  lcd.begin(16,2);
  lcd.setBacklight(255);

  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);

  SPI.begin();
  rfid.PCD_Init();

  lcd.print("Connecting...");
  lcd.setCursor(0,1);
  lcd.print("WiFi");

  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  while (WiFi.status() != WL_CONNECTED) {
    delay(300);
    Serial.print('.');
  }

  configTime(19800, 0, "pool.ntp.org", "time.nist.gov");
  while (time(nullptr) < 1609459200) delay(200);

  lcd.clear();
  lcd.print("CSE Attend");
  lcd.setCursor(0,1);
  lcd.print("Scan SNU ID");
}

/* ───────── LOOP ───────── */
void loop() {
  if (!rfid.PICC_IsNewCardPresent() || !rfid.PICC_ReadCardSerial()) return;

  String uid;
  for (byte i = 0; i < rfid.uid.size; i++) {
    if (rfid.uid.uidByte[i] < 0x10) uid += '0';
    uid += String(rfid.uid.uidByte[i], HEX);
  }
  uid.toUpperCase();
  Serial.println(uid);

  /* Admin Card */
  if (uid == ADMIN_UID) {
    lcd.clear();
    lcd.print("Present:");
    lcd.setCursor(0,1);
    lcd.print(countPresent());
    lcd.print("/");
    lcd.print(sizeof(roster)/sizeof(roster[0]));

    beep(150); delay(300); beep(150);
    delay(3000);

    lcd.clear();
    lcd.print("EEE 2A Attend");
    lcd.setCursor(0,1);
    lcd.print("Scan Reva ID");

    rfid.PICC_HaltA();
    rfid.PCD_StopCrypto1();
    return;
  }

  /* Student */
  int idx = findStudentIndex(uid);
  bool late = isLate();

  lcd.clear();
  if (idx != -1) {
    bool first = !roster[idx].present;
    roster[idx].present = true;

    if (first) {
      lcd.print(late ? "Late Entry!" : "Welcome");
      if (late) { beep(150); delay(100); beep(150); }
      else beep(80);
    } else {
      lcd.print("Already Marked");
      beep(30);
    }

    lcd.setCursor(0,1);
    lcd.print(String(roster[idx].name).substring(0,16));
    sendToCloud(uid, roster[idx].name, first, late);
  }
  else {
    lcd.print("Unknown Card");
    lcd.setCursor(0,1);
    lcd.print(uid.substring(0,16));
    beep(250);
    sendToCloud(uid, "UNKNOWN", false, false);
  }

  delay(2500);
  lcd.clear();
  lcd.print("EEE 2A Attend");
  lcd.setCursor(0,1);
  lcd.print("Scan Reva ID");

  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();
}

/* ───────── Helpers ───────── */
int findStudentIndex(const String& u) {
  for (size_t i = 0; i < sizeof(roster)/sizeof(roster[0]); i++)
    if (roster[i].uid && u == roster[i].uid) return i;
  return -1;
}

size_t countPresent() {
  size_t c = 0;
  for (auto &s : roster) if (s.present) c++;
  return c;
}

bool isLate() {
  struct tm tm;
  if (!getLocalTime(&tm)) return false;

  int nowMin = tm.tm_hour * 60 + tm.tm_min;
  for (auto &p : periods) {
    int start = p.h * 60 + p.m;
    if (nowMin >= start && nowMin <= start + 55)
      return nowMin > start + GRACE_MIN;
  }
  return false;
}

String isoTimestamp() {
  struct tm t;
  if (!getLocalTime(&t)) return "";
  char buf[10];
  strftime(buf, sizeof(buf), "%H:%M", &t);
  return String(buf);
}

void sendToCloud(const String& uid, const char* name, bool first, bool late) {
  if (WiFi.status() != WL_CONNECTED) return;

  String ts = isoTimestamp();
  if (ts == "") return;

  String url = String("https://") + FB_HOST + "/attendance.json";
  if (strlen(FB_SECRET)) url += "?auth=" + String(FB_SECRET);

  String payload =
    "{\"uid\":\"" + uid +
    "\",\"name\":\"" + String(name) +
    "\",\"first_mark\":" + (first ? "true" : "false") +
    ",\"late\":" + (late ? "true" : "false") +
    ",\"time\":\"" + ts + "\"}";

  WiFiClientSecure client;
  client.setInsecure();
  HTTPClient https;

  if (https.begin(client, url)) {
    https.addHeader("Content-Type", "application/json");
    https.POST(payload);
    https.end();
  }
}
