# Fall-Detection-Band
This is a Wearable Device that can be used to track elder people or children and send alerts when they fall.
# Components Required
1. NodeMCU v3
2. GPS Module
3. Gyroscope (MPU6050/ADXL345)
4. Connecting Wires
5. Li-ion Batteries 5v
6. Battery Holder
7. Charging Module (TP4056)
8. Strap/Belt
9. Breadboard
10. Transparent Box
# Connections
1. ADXL345
   VCC - VCC
   GND - GND
   SDA - D2
   SCL - D1
   CS - 3v3
   SDO - GND
   
2. GPS MODULE
   VCC - VCC
   GND - GND
   Rx - D8
   Tx - D7

# Programming:
1. Make all connections according to above instructions.
2. Connect NodeMCU to the PC on arduiono IDE.
3. Download all required libraries.
4. Upload the given code to the firmware.
5. All Set.

# Setting UP
1. Connect the TP4056 to battery set in battery holder.
2. Now connect +ve terminal of power supply (5v Li-ion bsttery) to the Vin on NodeMCU.
3. Connect -ve to GND on Board.
4. Fix the system into a Transparent box and paste a strap on it to tie it onn elbow.

# Code:
//Abhinav Vats
//www.linkedin.co/in/abhinavvats06



#include <Wire.h>
#include <Adafruit_Sensor.h>
#include <Adafruit_ADXL345_U.h>
#include <TinyGPS++.h>
#include <SoftwareSerial.h>
#include <ESP8266WiFi.h>
#include <ESP8266WebServer.h>
#include <time.h>

#define WIFI_SSID "FallDetect"
#define WIFI_PASSWORD "45454545"

Adafruit_ADXL345_Unified accel = Adafruit_ADXL345_Unified(12345);
TinyGPSPlus gps;
SoftwareSerial gpsSerial(D7, D8);

ESP8266WebServer server(80);

bool fallSent = false;
String lastFallData = "No fall detected yet.";

void setup() {
  Serial.begin(115200);
  Wire.begin(D2, D1);

  if (!accel.begin()) {
    Serial.println("ADXL345 not found!");
    while (1);
  }
  accel.setRange(ADXL345_RANGE_16_G);

  gpsSerial.begin(9600);

  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi connected!");
  Serial.print("IP Address: "); Serial.println(WiFi.localIP());

  // Local web server routes
  server.on("/", handleRoot);
  server.begin();
  Serial.println("Web server started");

  // Initialize time for timestamp
  configTime(0, 0, "pool.ntp.org", "time.nist.gov");
}

void loop() {
  server.handleClient();

  sensors_event_t event;
  accel.getEvent(&event);
  float accel_mag = sqrt(event.acceleration.x*event.acceleration.x +
                         event.acceleration.y*event.acceleration.y +
                         event.acceleration.z*event.acceleration.z);

  while (gpsSerial.available() > 0) gps.encode(gpsSerial.read());

  // Fall detection
  if ((accel_mag > 25 || accel_mag < 3) && !fallSent) {
    fallSent = true;
    Serial.println("🚨 Fall detected!");

    String lat = gps.location.isValid() ? String(gps.location.lat(), 6) : "Unknown";
    String lon = gps.location.isValid() ? String(gps.location.lng(), 6) : "Unknown";

    // Timestamp
    struct tm timeinfo;
    getLocalTime(&timeinfo);
    char timestamp[25];
    strftime(timestamp, sizeof(timestamp), "%Y-%m-%d %H:%M:%S", &timeinfo);

    // Build fall data string
    lastFallData = "<h2>Fall Detected!</h2>";
    lastFallData += "<p>Timestamp: " + String(timestamp) + "</p>";
    lastFallData += "<p>Accel X: " + String(event.acceleration.x) + 
                    ", Y: " + String(event.acceleration.y) + 
                    ", Z: " + String(event.acceleration.z) + "</p>";
    lastFallData += "<p>Magnitude: " + String(accel_mag) + "</p>";
    lastFallData += "<p>Location: " + lat + ", " + lon + "</p>";

    if(lat != "Unknown") {
      lastFallData += "<iframe width='600' height='400' " 
                      "src='https://maps.google.com/maps?q=" + lat + "," + lon + "&hl=es;z=14&output=embed'></iframe>";
    } else {
      lastFallData += "<p>GPS Available</p>";
    }

    // Print on serial
    Serial.println(lastFallData);

    delay(30000); // reset after 30 seconds
    fallSent = false;
  }

  delay(1000);
}

void handleRoot() {
  String page = "<!DOCTYPE html><html><head><title>Fall Detection</title></head><body>";
  page += lastFallData;
  page += "</body></html>";
  server.send(200, "text/html", page);
}
