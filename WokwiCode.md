//# Embedded-System-Design-Group6
//EET 3350 Final project
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>

// ------------------- Pin Definitions -------------------
#define DHTPIN       2
#define DHTTYPE      DHT22
#define GAS_PIN      A0

#define LED_R        9
#define LED_G        10
#define LED_B        11

#define BUZZER_PIN   5
#define BTN_PIN      4

// LED bar graph pins (low → high)
const int numLeds = 8;
int ledPins[numLeds] = { A1, A2, A3, 6, 7, 8, 12, 13 };

// ------------------- Thresholds -------------------
int   gasThreshold  = 755;
float tempThreshold = 35.0;

// ------------------- Objects -------------------
LiquidCrystal_I2C lcd(0x27, 16, 2);
DHT dht(DHTPIN, DHTTYPE);

// ------------------- State -------------------
bool          alarmActive     = false;
bool          silenced        = false;
bool          alarmWasActive  = false;  // detects new alarm after silence
bool          btnPrev         = HIGH;
unsigned long lastLCDUpdate   = 0;
unsigned long warmupStart     = 0;
bool          warmupDone      = false;

// ------------------- Helpers -------------------
void setRGB(int r, int g, int b) {
  analogWrite(LED_R, r);
  analogWrite(LED_G, g);
  analogWrite(LED_B, b);
}

void updateBar(int gasValue) {
  int level = map(gasValue, 250, 1000, 0, numLeds);
  level = constrain(level, 0, numLeds);
  for (int i = 0; i < numLeds; i++) {
    digitalWrite(ledPins[i], i < level ? HIGH : LOW);
  }
}

void clearBar() {
  for (int i = 0; i < numLeds; i++) {
    digitalWrite(ledPins[i], LOW);
  }
}

// ------------------- Setup -------------------
void setup() {
  Serial.begin(9600);

  lcd.init();
  lcd.backlight();
  dht.begin();

  pinMode(BTN_PIN,    INPUT_PULLUP);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(LED_R,      OUTPUT);
  pinMode(LED_G,      OUTPUT);
  pinMode(LED_B,      OUTPUT);

  for (int i = 0; i < numLeds; i++) {
    pinMode(ledPins[i], OUTPUT);
    digitalWrite(ledPins[i], LOW);
  }

  setRGB(0, 0, 255);  // Blue during warm-up

  warmupStart = millis();

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Warming up MQ2");
  lcd.setCursor(0, 1);
  lcd.print("Please wait...");
}

// ------------------- Loop -------------------
void loop() {
  unsigned long now = millis();

  // ---- Warm-up ----
  if (!warmupDone) {
    unsigned long elapsed = now - warmupStart;
    if (elapsed < 30000UL) {
      int secsLeft = (30000UL - elapsed) / 1000 + 1;
      if (now - lastLCDUpdate >= 1000) {
        lastLCDUpdate = now;
        lcd.setCursor(0, 1);
        lcd.print("Ready in ");
        lcd.print(secsLeft);
        lcd.print("s   ");
      }
      return;
    }
    // Warm-up done
    warmupDone = true;
    setRGB(0, 255, 0);
    lcd.clear();
  }

  // ---- Read sensors ----
  int   gasValue = analogRead(GAS_PIN);
  float t        = dht.readTemperature();
  float h        = dht.readHumidity();
  bool  dhtOK    = (!isnan(t) && !isnan(h));

  // ---- Bar graph ----
  updateBar(gasValue);

  // ---- Alarm condition ----
  bool gasAlert  = (gasValue > gasThreshold);
  bool tempAlert = (dhtOK && t > tempThreshold);
  bool condition = (gasAlert || tempAlert);

  // Reset silence when condition fully clears
  if (!condition) {
    silenced      = false;
    alarmWasActive = false;
  }

  // New alarm event resets silence
  if (condition && !alarmWasActive) {
    silenced       = false;
    alarmWasActive = true;
  }

  alarmActive = condition && !silenced;

  // ---- Button (edge-detected) ----
  bool btnNow = digitalRead(BTN_PIN);
  if (btnPrev == HIGH && btnNow == LOW) {
    if (alarmActive) {
      silenced    = true;
      alarmActive = false;
      noTone(BUZZER_PIN);
    }
  }
  btnPrev = btnNow;

  // ---- Buzzer ----
  if (alarmActive) {
    tone(BUZZER_PIN, 1000);
  } else {
    noTone(BUZZER_PIN);
  }

  // ---- RGB LED ----
  if (alarmActive) {
    setRGB(255, 0, 0);   // Red = alarm
  } else if (silenced) {
    setRGB(255, 165, 0); // Orange = silenced but condition still present
  } else {
    setRGB(0, 255, 0);   // Green = safe
  }

  // ---- LCD (update every 300ms to avoid flicker) ----
  if (now - lastLCDUpdate >= 300) {
    lastLCDUpdate = now;

    lcd.setCursor(0, 0);
    if (dhtOK) {
      lcd.print("T:");
      lcd.print(t, 1);
      lcd.print((char)223);
      lcd.print("C G:");
      lcd.print(gasValue);
      lcd.print("    ");
    } else {
      lcd.print("T:-- G:");
      lcd.print(gasValue);
      lcd.print("    ");
    }

    lcd.setCursor(0, 1);
    if (dhtOK) {
      lcd.print("H:");
      lcd.print(h, 1);
      lcd.print("% ");
    } else {
      lcd.print("H:--  ");
    }

    if (alarmActive) {
      if (gasAlert && tempAlert) lcd.print("GAS+TMP ");
      else if (gasAlert)         lcd.print("GAS ALM ");
      else                       lcd.print("TMP ALM ");
    } else if (silenced) {
      lcd.print("SILENCED");
    } else {
      lcd.print("SAFE    ");
    }
  }

  // ---- Serial debug ----
  Serial.print("Gas:");
  Serial.print(gasValue);
  Serial.print(" T:");
  Serial.print(dhtOK ? t : 0);
  Serial.print("C H:");
  Serial.print(dhtOK ? h : 0);
  Serial.print("% Alarm:");
  Serial.print(alarmActive ? "ON" : "OFF");
  Serial.print(" Silenced:");
  Serial.println(silenced ? "YES" : "NO");
}
