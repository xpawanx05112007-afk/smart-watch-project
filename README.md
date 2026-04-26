# smart-watch-project
#include <WiFi.h>
#include <Wire.h>
#include <MAX30105.h>
#include <Adafruit_Sensor.h>
#include <Adafruit_ADXL345_U.h>
#include <TinyGPS++.h>

// ========== WIFI ==========
const char* ssid = "YOUR_WIFI_NAME";
const char* password = "YOUR_WIFI_PASSWORD";
WiFiServer server(80);

// ========== SENSORS ==========
MAX30105 particleSensor;
Adafruit_ADXL345_Unified accel = Adafruit_ADXL345_Unified();
TinyGPSPlus gps;

// ========== SERIAL ==========
HardwareSerial gsm(1);
HardwareSerial gpsSerial(0);

// ========== PINS ==========
#define BUZZER 2
#define VIBRATION 3
#define BUTTON 4

// ========== VARIABLES ==========
float heartRate = 0;
float spO2 = 0;

long lastBeat = 0;
const long beatThreshold = 50000;

bool fallDetected = false;
String fallStatus = "SAFE";

// High HR
bool highHRDetected = false;
unsigned long highHRStart = 0;
const int HR_THRESHOLD = 100;
const int HR_DURATION = 5000;

// GPS
float latitude = 12.9716;
float longitude = 77.5946;

// =================================
void setup() {
  Serial.begin(115200);

  pinMode(BUZZER, OUTPUT);
  pinMode(VIBRATION, OUTPUT);
  pinMode(BUTTON, INPUT_PULLUP);

  Wire.begin();

  particleSensor.begin();
  accel.begin();

  gsm.begin(9600, SERIAL_8N1, 20, 21);
  gpsSerial.begin(9600, SERIAL_8N1, 6, 7);

  // WiFi Setup
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);

  Serial.print("Connecting WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("\nConnected ✅");
  Serial.print("IP: ");
  Serial.println(WiFi.localIP());

  server.begin();
}

// =================================
void loop() {

  readHeartRateOxygen();
  detectFall();
  updateGPS();

  if (fallDetected || highHRDetected) {
    triggerAlert();
  }

  handleWeb();
}

// =================================
// ❤️ HEART + SPO2
void readHeartRateOxygen() {

  long irValue = particleSensor.getIR();
  long redValue = particleSensor.getRed();

  if (irValue < 5000) {
    heartRate = 0;
    spO2 = 0;
    highHRDetected = false;
    return;
  }

  if (irValue > beatThreshold) {
    long beatDuration = millis() - lastBeat;

    if (beatDuration > 300 && beatDuration < 2000) {
      heartRate = 60.0 / (beatDuration / 1000.0);
      lastBeat = millis();
    }
  }

  float ratio = (float)redValue / irValue;
  spO2 = 110.0 - (25.0 * ratio);
  spO2 = constrain(spO2, 90, 100);

  // High HR detection
  if (heartRate > HR_THRESHOLD) {
    if (highHRStart == 0) highHRStart = millis();

    if (millis() - highHRStart > HR_DURATION) {
      highHRDetected = true;
    }
  } else {
    highHRStart = 0;
    highHRDetected = false;
  }
}

// =================================
// ⚠️ FALL DETECTION
void detectFall() {
  sensors_event_t event;
  accel.getEvent(&event);

  float total = sqrt(
    event.acceleration.x * event.acceleration.x +
    event.acceleration.y * event.acceleration.y +
    event.acceleration.z * event.acceleration.z
  );

  if (total < 3) {
    fallDetected = true;
    fallStatus = "FALL DETECTED";
  } else {
    fallStatus = "SAFE";
  }
}

// =================================
// 📍 GPS UPDATE
void updateGPS() {
  while (gpsSerial.available()) {
    gps.encode(gpsSerial.read());
  }

  if (gps.location.isValid()) {
    latitude = gps.location.lat();
    longitude = gps.location.lng();
  }
}

// =================================
// 🚨 ALERT
void triggerAlert() {

  // Beep + vibrate
  for (int i = 0; i < 3; i++) {
    digitalWrite(BUZZER, HIGH);
    digitalWrite(VIBRATION, HIGH);
    delay(500);
    digitalWrite(BUZZER, LOW);
    digitalWrite(VIBRATION, LOW);
    delay(500);
  }

  unsigned long startTime = millis();

  while (millis() - startTime < 10000) {
    if (digitalRead(BUTTON) == LOW) {
      Serial.println("Cancelled");
      fallDetected = false;
      highHRDetected = false;
      fallStatus = "SAFE";
      return;
    }
  }

  sendSMS();

  fallDetected = false;
  highHRDetected = false;
}

// =================================
// 📩 SEND SMS
void sendSMS() {

  String msg = "Emergency Alert!\n";

  if (fallDetected) msg += "Fall Detected\n";
  if (highHRDetected) {
    msg += "High Heart Rate\n";
    msg += "HR: " + String(heartRate) + " BPM\n";
  }

  msg += "Location:\nhttps://maps.google.com/?q=";
  msg += String(latitude, 6) + "," + String(longitude, 6);

  gsm.println("AT+CMGF=1");
  delay(1000);
  gsm.println("AT+CMGS=\"+91XXXXXXXXXX\"");
  delay(1000);
  gsm.print(msg);
  delay(1000);
  gsm.write(26);

  Serial.println("SMS Sent");
}

// =================================
// 🌐 WEB DASHBOARD
void handleWeb() {

  WiFiClient client = server.available();
  if (!client) return;

  client.readStringUntil('\r');
  client.flush();

  client.println("HTTP/1.1 200 OK");
  client.println("Content-type:text/html\n");

  client.println("<!DOCTYPE html><html><head>");
  client.println("<meta http-equiv='refresh' content='2'>");
  client.println("<style>");
  client.println("body{background:#111;color:#fff;text-align:center;font-family:Arial;}");
  client.println(".card{background:#222;padding:20px;margin:20px;border-radius:15px;}");
  client.println("</style></head><body>");

  client.println("<h1>SMART WATCH</h1>");

  client.println("<div class='card'><h2>Heart Rate</h2><h3>" + String(heartRate) + " BPM</h3></div>");
  client.println("<div class='card'><h2>SpO2</h2><h3>" + String(spO2) + " %</h3></div>");
  client.println("<div class='card'><h2>Fall</h2><h3>" + fallStatus + "</h3></div>");

  client.println("<div class='card'><h2>Status</h2>");
  if (highHRDetected)
    client.println("<h3 style='color:red;'>HIGH HEART RATE!</h3>");
  else
    client.println("<h3 style='color:green;'>NORMAL</h3>");
  client.println("</div>");

  client.println("<div class='card'><h2>Location</h2>");
  client.println("<h3>Lat: " + String(latitude,6) + "</h3>");
  client.println("<h3>Lng: " + String(longitude,6) + "</h3>");
  client.println("<a href='https://maps.google.com/?q=" + String(latitude,6) + "," + String(longitude,6) + "' target='_blank'>Open Map</a>");
  client.println("</div>");

  client.println("</body></html>");

  client.stop();
}
