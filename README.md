BLYNK code:
#define BLYNK_TEMPLATE_ID "TMPL3Wf6qvz_I" 
#define BLYNK_TEMPLATE_NAME "trial"
#define BLYNK_AUTH_TOKEN "eRUBhYVSUVF69FRV8DpYXoA4QvC8lAcZ" 
#include <WiFi.h> 
#include <WiFiClient.h> 
#include <HTTPClient.h> 
#include <BlynkSimpleEsp32.h> 
#include <TimeLib.h> 
#include <WidgetRTC.h> 
WidgetRTC rtc; 
#define RELAY1_PIN 14 
#define RELAY2_PIN 27 
#define TOUCH1_PIN 26 // Touch sensor for Relay 1 
#define TOUCH2_PIN 25 // Touch sensor for Relay 2 
const int voltagePin1 = 34; 
const int currentPin1 = 35; 
const int voltagePin2 = 32; 
const int currentPin2 = 33; 
const float VREF = 3.3; 
const int ADC_RESOLUTION = 4095; 
float voltageCalibration = 493.89; 
float currentCalibration = 1.26; 
char ssid[] = "Pixel 7"; 
char pass[] = "88888888"; 
int relay1State = LOW; 
int relay2State = LOW; 
int startTime = -1; 
int endTime = -1; 
bool timerActive = false; 
bool interrupted = false; 
const char* server = "http://api.thingspeak.com/update"; 
String writeAPIKey = "PO1GMYKGC2BCPVRE"; 
unsigned long lastUpdate = 0; 
const unsigned long uploadInterval = 10000; 
bool lastTouch1 = LOW; 
bool lastTouch2 = LOW; 
void updateRelay1(int state) { 
relay1State = state; 
digitalWrite(RELAY1_PIN, state); 
Blynk.virtualWrite(V0, state); 
Serial.println(">> Relay 1: " + String(state == HIGH ? "ON" : "OFF")); }
void updateRelay2(int state) { 
relay2State = state; 
digitalWrite(RELAY2_PIN, state); 
Blynk.virtualWrite(V1, state); 
Serial.println(">> Relay 2: " + String(state == HIGH ? "ON" : "OFF")); } 
BLYNK_WRITE(V0) { 
updateRelay1(param.asInt()); 
} 
BLYNK_WRITE(V1) { 
if (timerActive && relay2State != param.asInt()) { 
interrupted = true; 
timerActive = false; 
Serial.println(">> Manual override during timer session."); 
} 
updateRelay2(param.asInt()); 
} 
BLYNK_WRITE(V2) { 
TimeInputParam t(param); 
if (t.hasStartTime() && t.hasStopTime()) { 
startTime = t.getStartHour() * 60 + t.getStartMinute(); 
endTime = t.getStopHour() * 60 + t.getStopMinute(); 
timerActive = true; 
interrupted = false; 
Serial.print(">> New schedule: "); 
Serial.print(t.getStartHour()); 
Serial.print(":"); 
Serial.print(t.getStartMinute()); 
Serial.print(" to "); 
Serial.print(t.getStopHour()); 
Serial.print(":"); 
Serial.println(t.getStopMinute()); 
} 
} 
void checkSchedule() { 
if (!timerActive || year() == 1970) return; 
int nowTime = hour() * 60 + minute(); 
if (nowTime == startTime) { 
updateRelay2(HIGH); 
interrupted = false; 
Serial.println(">> Schedule started. Relay 2 ON."); 
} 
if (nowTime == endTime && !interrupted) { 
updateRelay2(LOW);
Serial.println(">> Schedule ended. Relay 2 OFF."); 
timerActive = false; 
} 
} 
float readACVoltage(int pin) { 
const int samples = 1000; 
float offset = 0.0, sum = 0.0; 
for (int i = 0; i < samples; i++) { 
offset += (analogRead(pin) * VREF) / ADC_RESOLUTION; 
delayMicroseconds(100); 
} 
offset /= samples; 
for (int i = 0; i < samples; i++) { 
float v = (analogRead(pin) * VREF) / ADC_RESOLUTION - offset; sum += v * v; 
delayMicroseconds(100); 
} 
return sqrt(sum / samples) * voltageCalibration; 
} 
float readACCurrent(int pin) { 
const int samples = 1000; 
float offset = 0.0, sum = 0.0; 
for (int i = 0; i < samples; i++) { 
offset += (analogRead(pin) * VREF) / ADC_RESOLUTION; 
delayMicroseconds(100); 
} 
offset /= samples; 
for (int i = 0; i < samples; i++) { 
float v = (analogRead(pin) * VREF) / ADC_RESOLUTION - offset; sum += v * v; 
delayMicroseconds(100); 
} 
return sqrt(sum / samples) / currentCalibration; 
} 
void sendToThingSpeak(float voltage1, float current1, float voltage2, float current2) { if (WiFi.status() == WL_CONNECTED) { 
HTTPClient http; 
String url = server; 
url += "?api_key=" + writeAPIKey; 
url += "&field1=" + String(voltage1, 2); 
url += "&field2=" + String(current1, 3); 
url += "&field3=" + String(voltage2, 2); 
url += "&field4=" + String(current2, 3);
http.begin(url); 
int httpCode = http.GET(); 
if (httpCode > 0) { 
Serial.println(">> ThingSpeak update sent."); 
} else { 
Serial.println(">> ThingSpeak update failed."); 
} 
http.end(); 
} 
} 
void setup() { 
Serial.begin(115200); 
pinMode(RELAY1_PIN, OUTPUT); 
pinMode(RELAY2_PIN, OUTPUT); 
pinMode(TOUCH1_PIN, INPUT); 
pinMode(TOUCH2_PIN, INPUT); 
updateRelay1(LOW); 
updateRelay2(LOW); 
Blynk.begin(BLYNK_AUTH_TOKEN, ssid, pass); 
rtc.begin(); 
} 
void loop() { 
Blynk.run(); 
checkSchedule(); 
// Touch Sensor Logic 
bool currentTouch1 = digitalRead(TOUCH1_PIN); 
if (currentTouch1 == HIGH && lastTouch1 == LOW) { 
updateRelay1(!relay1State); 
Serial.println(">> Touch 1 triggered."); 
} 
lastTouch1 = currentTouch1; 
bool currentTouch2 = digitalRead(TOUCH2_PIN); 
if (currentTouch2 == HIGH && lastTouch2 == LOW) { 
updateRelay2(!relay2State); 
Serial.println(">> Touch 2 triggered."); 
} 
lastTouch2 = currentTouch2; 
float voltage1 = readACVoltage(voltagePin1); 
float current1 = readACCurrent(currentPin1); 
float voltage2 = readACVoltage(voltagePin2); 
float current2 = readACCurrent(currentPin2); 
Serial.print("Load 1 - Voltage: "); Serial.print(voltage1, 2); 
Serial.print(" V, Current: "); Serial.print(current1, 3); Serial.println(" A");

Serial.print("Load 2 - Voltage: "); Serial.print(voltage2, 2); 
Serial.print(" V, Current: "); Serial.print(current2, 3); Serial.println(" A"); 
if (millis() - lastUpdate >= uploadInterval) { 
sendToThingSpeak(voltage1, current1, voltage2, current2); 
lastUpdate = millis(); 
} 
if (Serial.available()) { 
String input = Serial.readStringUntil('\n'); 
input.trim(); 
if (input == "1") updateRelay1(!relay1State); 
else if (input == "2") { 
if (timerActive) { 
interrupted = true; 
timerActive = false; 
Serial.println(">> Serial interrupted schedule."); 
} 
updateRelay2(!relay2State); 
} else { 
Serial.println(">> Invalid input. Use 1 or 2."); 
} 
} 
} 
