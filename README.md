#include <WiFiClientSecure.h>
#include <esp_crt_bundle.h>
#include <WiFi.h>
#include <ESPAsyncWebServer.h>
#include <ESP32Servo.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <UniversalTelegramBot.h>

#define ANEMOMETER_PIN 15
#define LDR_PIN 34
#define SERVO1_PIN 13
#define SERVO2_PIN 12
#define HUJAN_PIN 4

const char* ssid = "bumiterbang";
const char* password = "12345678";

// Telegram Bot settings
#define BOT_TOKEN "7746298129:AAHxmsFETPXfO1Gj_4JcfUMbdP4_0f02OyQ"
#define CHAT_ID "6441600532"

volatile unsigned int pulseCount = 0;
unsigned long lastSensorUpdate = 0;
float kecepatanAngin = 0;
bool hujan = false;
bool hujanSebelumnya = false;
String statusJemuran = "Keluar";
String mode = "Auto";

unsigned long lcdLastSwitch = 0;
int lcdState = 0;

Servo servo1, servo2;
AsyncWebServer server(80);
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Telegram
WiFiClientSecure client;
UniversalTelegramBot bot(BOT_TOKEN, client);

void IRAM_ATTR countPulse() {
  pulseCount++;
}

void kirimPesanTelegram(String alasan) {
  String pesan = "📢 *Status Jemuran Update!*\n";
  pesan += "Status: *" + statusJemuran + "*\n";
  pesan += "Mode: " + mode + "\n";
  pesan += "Hujan: " + String(hujan ? "Ya" : "Tidak") + "\n";
  pesan += "Angin: " + String(kecepatanAngin) + " m/s\n";
  pesan += "Cahaya: " + String(cahayaPersen()) + "%\n";
  pesan += "Alasan: " + alasan;
  bot.sendMessage(CHAT_ID, pesan, "Markdown");
}

void bukaJemuran() {
  if (statusJemuran != "Keluar") {
    delay(3000);
    servo1.write(180);
    servo2.write(180);
    statusJemuran = "Keluar";
    kirimPesanTelegram(mode == "Auto" ? "Jemuran dibuka (cuaca membaik)" : "Jemuran dibuka (manual)");
  }
}

void tutupJemuran() {
  if (statusJemuran != "Masuk") {
    delay(3000);
    servo1.write(0);
    servo2.write(0);
    statusJemuran = "Masuk";
    kirimPesanTelegram(mode == "Auto" ? "Jemuran ditutup (cuaca buruk)" : "Jemuran ditutup (manual)");
  }
}

int bacaCahayaRata2() {
  long total = 0;
  for (int i = 0; i < 10; i++) {
    total += analogRead(LDR_PIN);
    delay(5);
  }
  return total / 10;
}

int cahayaPersen() {
  int cahayaRaw = bacaCahayaRata2();
  cahayaRaw = constrain(cahayaRaw, 0, 4095);
  return map(cahayaRaw, 0, 4095, 100, 0); // 100% terang, 0% gelap
}

void setup() {
  Serial.begin(115200);
  Wire.begin(21, 22);
  lcd.init();
  lcd.backlight();

  pinMode(ANEMOMETER_PIN, INPUT_PULLUP);
  pinMode(HUJAN_PIN, INPUT);
  pinMode(LDR_PIN, INPUT);
  attachInterrupt(digitalPinToInterrupt(ANEMOMETER_PIN), countPulse, FALLING);

  servo1.attach(SERVO1_PIN);
  servo2.attach(SERVO2_PIN);
  bukaJemuran();

  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  client.setCACert(TELEGRAM_CERTIFICATE_ROOT);

  lcd.setCursor(0, 0);
  lcd.print("IP Web:");
  lcd.setCursor(0, 1);
  lcd.print(WiFi.localIP());

  server.on("/", HTTP_GET, [](AsyncWebServerRequest *request){
    String html = R"rawliteral(
    <!DOCTYPE html><html><head><meta name='viewport' content='width=device-width, initial-scale=1'>
    <style>
      body { font-family: Arial; padding: 20px; max-width: 600px; margin: auto; background: #f0f0f0; }
      .card { background: white; padding: 15px; border-radius: 10px; box-shadow: 0 2px 5px rgba(0,0,0,0.1); }
      button { width: 100%; padding: 10px; font-size: 16px; background: #007BFF; color: white; border: none; border-radius: 5px; margin-top: 10px; }
      button:hover { background: #0056b3; }
    </style>
    <script>
      setInterval(()=>{ 
        fetch('/data').then(r=>r.json()).then(d=>{
          document.getElementById('angin').innerText=d.angin;
          document.getElementById('cahaya').innerText=d.cahaya + "%";
          document.getElementById('hujan').innerText=d.hujan;
          document.getElementById('status').innerText=d.status;
          document.getElementById('mode').innerText=d.mode;
        });
      },1000);
    </script></head><body>
    <div class='card'>
      <h2>Status Jemuran: <span id='status'>)rawliteral" + statusJemuran + R"rawliteral(</span></h2>
      <p>Kecepatan Angin: <span id='angin'>)rawliteral" + String(kecepatanAngin) + R"rawliteral(</span> m/s</p>
      <p>Cahaya: <span id='cahaya'>)rawliteral" + String(cahayaPersen()) + R"rawliteral(</span></p>
      <p>Hujan: <span id='hujan'>)rawliteral" + String(hujan ? "Ya" : "Tidak") + R"rawliteral(</span></p>
      <p>Mode: <span id='mode'>)rawliteral" + mode + R"rawliteral(</span></p>
    </div>)rawliteral";

    if (mode == "Auto") {
      html += R"rawliteral(
        <a href="/buka"><button>Buka (Manual)</button></a>
        <a href="/tutup"><button>Tutup (Manual)</button></a>
      )rawliteral";
    } else {
      html += R"rawliteral(
        <a href="/buka"><button>Buka</button></a>
        <a href="/tutup"><button>Tutup</button></a>
        <a href="/auto"><button>Mode Auto</button></a>
      )rawliteral";
    }

    html += R"rawliteral(</body></html>)rawliteral";
    request->send(200, "text/html", html);
  });

  server.on("/buka", HTTP_GET, [](AsyncWebServerRequest *request){
    bukaJemuran();
    mode = "Manual";
    request->redirect("/");
  });

  server.on("/tutup", HTTP_GET, [](AsyncWebServerRequest *request){
    tutupJemuran();
    mode = "Manual";
    request->redirect("/");
  });

  server.on("/auto", HTTP_GET, [](AsyncWebServerRequest *request){
    mode = "Auto";
    request->redirect("/");
  });

  server.on("/data", HTTP_GET, [](AsyncWebServerRequest *request){
    String json = "{";
    json += "\"angin\":" + String(kecepatanAngin) + ",";
    json += "\"cahaya\":" + String(cahayaPersen()) + ",";
    json += "\"hujan\":\"" + String(hujan ? "Ya" : "Tidak") + "\",";
    json += "\"status\":\"" + statusJemuran + "\",";
    json += "\"mode\":\"" + mode + "\"}";
    request->send(200, "application/json", json);
  });

  server.begin();
  lastSensorUpdate = millis();
}

void loop() {
  unsigned long currentTime = millis();

  if (currentTime - lastSensorUpdate >= 1000) {
    detachInterrupt(digitalPinToInterrupt(ANEMOMETER_PIN));

    kecepatanAngin = pulseCount * 0.1; // sesuaikan dengan karakteristik sensor
    int cahaya = bacaCahayaRata2();
    hujan = digitalRead(HUJAN_PIN) == LOW;

    Serial.println("==== UPDATE SENSOR ====");
    Serial.printf("Angin: %.2f m/s, Cahaya: %d (%d%%), Hujan: %s\n",
      kecepatanAngin, cahaya, cahayaPersen(), hujan ? "YA" : "TIDAK");
    Serial.print("Mode: "); Serial.println(mode);

    if (mode == "Auto") {
      if (kecepatanAngin > 10.0) {
        tutupJemuran();
      } else if (cahayaPersen() < 20) {
        tutupJemuran();
      } else if (hujan && !hujanSebelumnya) {
        tutupJemuran();
      } else if (!hujan && statusJemuran == "Masuk") {
        bukaJemuran();
      }
    }

    hujanSebelumnya = hujan;
    pulseCount = 0;
    lastSensorUpdate = currentTime;
    attachInterrupt(digitalPinToInterrupt(ANEMOMETER_PIN), countPulse, FALLING);
  }

  if (millis() - lcdLastSwitch >= 3000) {
    lcd.clear();
    if (lcdState == 0) {
      lcd.setCursor(0, 0);
      lcd.print("IP:");
      lcd.setCursor(0, 1);
      lcd.print(WiFi.localIP());
    } else if (lcdState == 1) {
      lcd.setCursor(0, 0);
      lcd.print("Angin:" + String(kecepatanAngin));
      lcd.setCursor(0, 1);
      lcd.print("Cahaya:" + String(cahayaPersen()) + "%");
    } else {
      lcd.setCursor(0, 0);
      lcd.print("Jemuran:" + statusJemuran);
      lcd.setCursor(0, 1);
      lcd.print(hujan ? "Hujan" : "Tidak Hujan");
    }
    lcdState = (lcdState + 1) % 3;
    lcdLastSwitch = millis();
  }
}
