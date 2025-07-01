# ESP32 WiFi Access Point & Web Server 🌐  

This project turns an **ESP32** into a **WiFi Access Point (AP)** with a built-in **web server**. Users can connect to the ESP32, access a webpage, and control an LED by sending simple **HTTP or TCP requests**.  

## ✨ Features  
- 📶 **Creates a WiFi hotspot** (no external router needed).  
- 🌍 **Hosts a simple web server** at `192.168.4.1`.
- 💡 **Control an LED** using:  
  - **Web browser**  
  - **Raw TCP commands** 

---

## 🛠️ Hardware Requirements  
- **ESP32 Dev Board**  
- **LED & Resistor** (or use the onboard LED at GPIO 2, can also be GPIO 5 or 22 depending on the board)  
- **Power source** (USB or battery)  

---

## 📜 Code Overview:  
#include <WiFi.h>
#include <NetworkClient.h>
#include <WiFiAP.h>

#ifndef LED_BUILTIN
#define LED_BUILTIN 2
#endif

// Set these to your desired credentials.
const char *ssid = "ESP32";
const char *password = "ESP32-pass";

NetworkServer server(80);

bool isBlinking = false;  // Track blinking state
unsigned long previousMillis = 0;  // For blinking LED
const long blinkInterval = 500;  // LED blink interval (ms)
const long serialDelay = 1000;  // Serial monitor delay (ms)
unsigned long previousSerialMillis = 0;

void setup() {
  pinMode(LED_BUILTIN, OUTPUT);

  Serial.begin(115200);
  Serial.println();
  Serial.println("Configuring access point...");

  // You can remove the password parameter if you want the AP to be open.
  // A valid password must have more than 7 characters
  if (!WiFi.softAP(ssid, password)) {
    log_e("Soft AP creation failed.");
    while (1);
  }
  IPAddress myIP = WiFi.softAPIP();
  Serial.print("AP IP address: ");
  Serial.println(myIP);
  server.begin();

  Serial.println("Server started");
}

// The IP is 192.168.4.1, NOT like station mode.

void loop() {
  NetworkClient client = server.accept();  // Listen for incoming clients

  if (client) {                     // If you get a client,
    Serial.println("New Client.");  // Print a message out the serial port
    String currentLine = "";        // Make a String to hold incoming data from the client
    while (client.connected()) {    // Loop while the client's connected
      if (client.available()) {     // If there's bytes to read from the client,
        char c = client.read();     // Read a byte, then
        Serial.write(c);            // Print it out the serial monitor
        if (c == '\n') {            // If the byte is a newline character

          // If the current line is blank, you got two newline characters in a row.
          // That's the end of the client HTTP request, so send a response:
          if (currentLine.length() == 0) {
            // HTTP headers always start with a response code (e.g. HTTP/1.1 200 OK)
            // and a content-type so the client knows what's coming, then a blank line:
            client.println("HTTP/1.1 200 OK");
            client.println("Content-type:text/html");
            client.println();

            // The content of the HTTP response follows the header:
            client.print("Click <a href=\"/H\">here</a> to turn on the LED on pin 2.<br>");
            client.print("Click <a href=\"/L\">here</a> to turn off the LED on pin 2.<br>");
            client.print("Click <a href=\"/B\">here</a> to make the LED blink continuously.<br>");

            // The HTTP response ends with another blank line:
            client.println();
            // Break out of the while loop:
            break;
          } else {  // If you got a newline, then clear currentLine:
            currentLine = "";
          }
        } else if (c != '\r') {  // If you got anything else but a carriage return character,
          currentLine += c;      // Add it to the end of the currentLine
        }

        // Check to see if the client request was "GET /H", "GET /L", or "GET /B":
        if (currentLine.endsWith("GET /H")) {
          digitalWrite(LED_BUILTIN, HIGH);  // GET /H turns the LED on
          isBlinking = false;               // Stop blinking
        }
        if (currentLine.endsWith("GET /L")) {
          digitalWrite(LED_BUILTIN, LOW);   // GET /L turns the LED off
          isBlinking = false;               // Stop blinking
        }
        if (currentLine.endsWith("GET /B")) {
          isBlinking = true;  // GET /B starts blinking
        }
      }
    }
    // Close the connection:
    client.stop();
    Serial.println("Client Disconnected.");
  }

  // Handle blinking if enabled
  if (isBlinking) {
    unsigned long currentMillis = millis();
    if (currentMillis - previousMillis >= blinkInterval) {
      previousMillis = currentMillis;
      digitalWrite(LED_BUILTIN, !digitalRead(LED_BUILTIN));  // Toggle LED
    }
  }

  // Separate delay for Serial Monitor messages
  unsigned long currentSerialMillis = millis();
  if (currentSerialMillis - previousSerialMillis >= serialDelay) {
    previousSerialMillis = currentSerialMillis;
    Serial.println("Server running...");
  }
}
