#define BLYNK_TEMPLATE_ID "TMPL3-92ZS3un"
#define BLYNK_TEMPLATE_NAME "Air quality monitor"
#define BLYNK_AUTH_TOKEN "Enter your token"

#define BLYNK_PRINT Serial

#include <ESP8266WiFi.h>
#include <BlynkSimpleEsp8266.h>

#include <DHT.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

byte degree_symbol[8] = {
  0b00111,
  0b00101,
  0b00111,
  0b00000,
  0b00000,
  0b00000,
  0b00000,
  0b00000
};

char auth[] = BLYNK_AUTH_TOKEN;

char ssid[] = "Enter Your Wifi user id";
char pass[] = "Enter your wifi password";

BlynkTimer timer;

int gas = A0;
int sensorThreshold = 100; // Not actively used in the current logic

#define DHTPIN 14
#define DHTTYPE DHT11
DHT dht(DHTPIN, DHTTYPE);

// Define the buzzer pin - Using D6 (GPIO12) as other pins are occupied
#define BUZZER_PIN 12

void sendSensor() {
  float h = dht.readHumidity();
  float t = dht.readTemperature();

  if (isnan(h) || isnan(t)) {
    Serial.println("Failed to read from DHT sensor!");
    return;
  }
  int analogSensor = analogRead(gas);
  Blynk.virtualWrite(V2, analogSensor);
  Serial.print("Gas Value: ");
  Serial.println(analogSensor);

  Blynk.virtualWrite(V0, t);
  Blynk.virtualWrite(V1, h);

  Serial.print("Temperature : ");
  Serial.print(t);
  Serial.print("    Humidity : ");
  Serial.println(h);
}

void setup() {
  Serial.begin(115200);

  // Initialize Buzzer Pin
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW); // Ensure buzzer is off at startup

  Blynk.begin(auth, ssid, pass);
  dht.begin();
  timer.setInterval(30000L, sendSensor);

  // Initialize LCD
  lcd.begin();
  // lcd.backlight(); // Uncomment if your LCD has backlight control and you want it on
  lcd.setCursor(3, 0);
  lcd.print("Air Quality");
  lcd.setCursor(3, 1);
  lcd.print("Monitoring");
  delay(2000); // This delay is acceptable in setup()
  lcd.clear();
}

void loop() {
  Blynk.run();
  timer.run();

  float h = dht.readHumidity();
  float t = dht.readTemperature();
  int gasValue = analogRead(gas);

  // --- LCD Display Logic (Original code's blocking delays) ---
  // Note: The delays here will make the board unresponsive for 4 seconds at a time.
  // For a more robust system, consider using millis() for non-blocking updates.
  // This section keeps your original logic for LCD updates.

  lcd.setCursor(0, 0);
  lcd.print("Temperature ");
  lcd.setCursor(0, 1);
  lcd.print(t);
  lcd.setCursor(6, 1);
  lcd.write(1);
  lcd.createChar(1, degree_symbol); // Re-create custom char (can be moved to setup if not dynamic)
  lcd.setCursor(7, 1);
  lcd.print("C");
  delay(4000);
  lcd.clear();

  lcd.setCursor(0, 0);
  lcd.print("Humidity ");
  lcd.print(h);
  lcd.print("%");
  delay(4000);
  lcd.clear();

  // --- Gas Value Display and Buzzer Control ---
  if (gasValue < 600) {
    lcd.setCursor(0, 0);
    lcd.print("Gas Value: ");
    lcd.print(gasValue);
    lcd.setCursor(0, 1);
    lcd.print("Fresh Air");
    Serial.println("Fresh Air");
    digitalWrite(BUZZER_PIN, LOW); // Turn buzzer OFF
    delay(4000);
    lcd.clear();
  } else { // gasValue >= 600
    lcd.setCursor(0, 0);
    lcd.print("Gas Value: "); // Added this line for consistency
    lcd.print(gasValue);
    lcd.setCursor(0, 1);
    lcd.print("Bad Air");
    Serial.println("Bad Air");
    digitalWrite(BUZZER_PIN, HIGH); // Turn buzzer ON
    delay(4000);
    lcd.clear();
  }

  // --- Blynk Event Log ---
  // This will log an event every time the gasValue is > 600 and this part of loop runs.
  // Consider using a flag to send it only once per "bad air" state change.
  if (gasValue > 600) {
    Blynk.logEvent("pollution_alert", "Bad Air");
  }
}# air-quality
