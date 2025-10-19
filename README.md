# CapLieu
# arduino.h
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <WiFi.h>
#include <WebServer.h>

// === LCD1602 I2C ===
LiquidCrystal_I2C lcd(0x27, 16, 2); // Địa chỉ LCD

// === Rotary Encoder (HW-040) ===
#define CLK 21
#define DT  19
#define SW  18

// === Relay Output ===
#define RELAY 25

// === WiFi Access Point ===
const char* ssid_ap = "HIEU_CAPLIEU";
const char* password_ap = "12345678";

WebServer server(80);

// === Feeder State ===
int grams = 100;
bool feeding = false;
bool buttonPressed = false;
int lastCLK = HIGH;
bool longPressDetected = false;
unsigned long buttonPressTime = 0;
String lastFeedTime = "Không có";

// === Menu State ===
int menuIndex = 0;
const int MENU_COUNT = 2;
String menus[MENU_COUNT] = {"Manual Feed", "Web Control"};
bool inMenu = true;

// ========== Setup ========== //
void setup() {
  Serial.begin(115200);
  Wire.begin(23, 22); // SDA = GPIO 23, SCL = GPIO 22
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("HIEU.HELLO");
  delay(1000);

  pinMode(CLK, INPUT);
  pinMode(DT, INPUT);
  pinMode(SW, INPUT_PULLUP);
  lastCLK = digitalRead(CLK);

  pinMode(RELAY, OUTPUT);
  digitalWrite(RELAY, LOW);

  showMenu();
}

// ========== Loop ========== //
void loop() {
  handleEncoder();
  handleButton();
  if (!inMenu) {
    server.handleClient();
  }
}

void handleEncoder() {
  int currentCLK = digitalRead(CLK);
  if (currentCLK != lastCLK) {
    if (digitalRead(DT) != currentCLK) {
      if (inMenu) menuIndex = (menuIndex + 1) % MENU_COUNT;
      else grams += 10;
    } else {
      if (inMenu) menuIndex = (menuIndex - 1 + MENU_COUNT) % MENU_COUNT;
      else grams -= 10;
    }
    if (grams < 10) grams = 10;
    if (grams > 1000) grams = 1000;

    if (inMenu) showMenu();
    else showGrams();
  }
  lastCLK = currentCLK;
}

void handleButton() {
  if (digitalRead(SW) == LOW && !buttonPressed) {
    buttonPressed = true;
    buttonPressTime = millis();
    delay(50);
  }

  if (digitalRead(SW) == LOW && buttonPressed) {
    if (!longPressDetected && millis() - buttonPressTime > 1000) {
      longPressDetected = true;
      inMenu = true;
      showMenu();
    }
  }

  if (digitalRead(SW) == HIGH && buttonPressed) {
    if (!longPressDetected) {
      if (inMenu) {
        if (menuIndex == 0) {
          inMenu = false;
          showGrams();
        } else if (menuIndex == 1) {
          inMenu = false;
          startWiFiServer();
          showWiFiInfo();
        }
      } else {
        startFeeding();
      }
    }
    buttonPressed = false;
    longPressDetected = false;
  }
}

void showMenu() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Chon che do:");
  lcd.setCursor(0, 1);
  lcd.print("> ");
  lcd.print(menus[menuIndex]);
}

void showGrams() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Chon luong: ");
  lcd.setCursor(0, 1);
  lcd.print(grams);
  lcd.print("g (Nhan = cap)");
}

void startFeeding() {
  if (feeding) return;
  feeding = true;
  int duration = grams / 10;

  digitalWrite(RELAY, HIGH);
  for (int i = duration; i > 0; i--) {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Dang cap lieu...");
    lcd.setCursor(0, 1);
    lcd.print(grams);
    lcd.print("g con ");
    lcd.print(i);
    lcd.print("s");
    delay(1000);
  }
  digitalWrite(RELAY, LOW);

  feeding = false;
  lastFeedTime = String(grams) + "g lúc " + String(millis() / 1000) + "s";
  showGrams();
}

void startWiFiServer() {
  WiFi.mode(WIFI_AP);
  WiFi.softAP(ssid_ap, password_ap);
  IPAddress IP = WiFi.softAPIP();
  Serial.print("AP IP: ");
  Serial.println(IP);

  server.on("/", HTTP_GET, []() {
    String html = "<html><head><meta charset='utf-8'><title>ESP32 Feeder</title><style>body{background:#000;color:#fff;font-family:sans-serif;text-align:center;padding:20px;}button{padding:30px 60px;font-size:32px;background:#28a745;color:#fff;border:none;border-radius:10px;}input[type=number]{padding:20px;font-size:28px;width:200px;}p{font-size:24px;margin:20px 0;}h1{font-size:40px;}</style></head><body>";
    html += "<h1>🍚 HIẾU CẤP LIỆU /h1>";
    html += "<form action='/feed' method='GET'><input type='number' name='g' value='" + String(grams) + "' min='10' max='1000'> g<br><br><button type='submit'>BẮT ĐẦU CẤP LIỆU</button></form><br>";
    html += "<p><b>Khối lượng hiện tại:</b> " + String(grams) + "g</p>";
    html += "<p><b>Trạng thái:</b> " + String(feeding ? "ĐANG CẤP LIỆU" : "SẴN SÀNG") + "</p>";
    html += "<p><b>Lịch sử:</b> " + lastFeedTime + "</p>";
    html += "</body></html>";
    server.send(200, "text/html", html);
  });

  server.on("/feed", HTTP_GET, []() {
    if (server.hasArg("g")) {
      grams = server.arg("g").toInt();
      startFeeding();
    }
    server.sendHeader("Location", "/");
    server.send(303);
  });

  server.begin();
  Serial.println("Web server started");
}

void showWiFiInfo() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("SSID: HIEU_CAP_LIEU");
  lcd.setCursor(0, 1);
  lcd.print("IP: 192.168.4.1");
}
