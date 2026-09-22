#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

#define DOUT 3
#define CLK  2
#define TDS_PIN A0
#define BUZZER 9

long offset = 0;
float scale = 408.0;

float FULL_WEIGHT = 550.0;

// -------- HX711 --------
long readHX711()
{
  long count = 0;
  int timeout = 0;

  while (digitalRead(DOUT))
  {
    delay(1);
    if (++timeout > 1000) return 0;
  }

  for (int i = 0; i < 24; i++)
  {
    digitalWrite(CLK, HIGH);
    count = count << 1;
    digitalWrite(CLK, LOW);
    if (digitalRead(DOUT)) count++;
  }

  digitalWrite(CLK, HIGH);
  count ^= 0x800000;
  digitalWrite(CLK, LOW);

  return count;
}

long getAvg(int n)
{
  long sum = 0;
  for (int i = 0; i < n; i++)
    sum += readHX711();
  return sum / n;
}

// -------- SETUP --------
void setup()
{
  Serial.begin(9600);

  pinMode(CLK, OUTPUT);
  pinMode(DOUT, INPUT);
  pinMode(BUZZER, OUTPUT);

  lcd.init();
  lcd.backlight();

  delay(1000);
  offset = getAvg(30);
}

// -------- LOOP --------
void loop()
{
  // -------- WEIGHT --------
  long raw = getAvg(5);
  float weight = (raw - offset) / scale;

  if (weight < 0) weight = 0;
  if (weight < 5) weight = 0;

  // -------- TDS --------
  int tdsRaw = analogRead(TDS_PIN);

  static float tdsFiltered = 0;
  tdsFiltered = (tdsFiltered * 0.5) + (tdsRaw * 0.5);

  float tdsPPM = (tdsFiltered - 150) * 900.0 / (500 - 150);

  if (tdsPPM < 50) tdsPPM = 0;
  if (tdsPPM > 1200) tdsPPM = 1200;

  // -------- LCD LINE 1 --------
  lcd.setCursor(0, 0);
  lcd.print("W:");
  lcd.print(weight, 1);
  lcd.print("g   ");

  // -------- LCD LINE 2 --------
  lcd.setCursor(0, 1);
  lcd.print("T:");
  lcd.print((int)tdsPPM);
  lcd.print("ppm ");

  // -------- SAFE / NOT SAFE TAG --------
  if (tdsPPM >= 700 && tdsPPM <= 1000)
  {
    lcd.print("SAFE ");
  }
  else
  {
    lcd.print("BAD  ");
  }

  // -------- ALERT SYSTEM --------
  float remainingPercent = (weight / FULL_WEIGHT) * 100;

  // -------- PRIORITY 1: EMPTY --------
  if (weight <= 40)
  {
    lcd.setCursor(12, 0);
    lcd.print("EMPTY");

    // 🔥 CONTINUOUS LOUD BEEP (no pause)
    tone(BUZZER, 4000);
  }

  // -------- PRIORITY 2: LOW IV (<=25%) --------
  else if (remainingPercent <= 25)
  {
    lcd.setCursor(12, 0);
    lcd.print("LOW ");

    // 🔊 beep-pause (unchanged)
    tone(BUZZER, 3000);
    delay(300);
    noTone(BUZZER);
    delay(200);
  }

  // -------- PRIORITY 3: TDS HIGH --------
  else if (tdsPPM > 1000)
  {
    lcd.setCursor(12, 0);
    lcd.print("HIGH");

    tone(BUZZER, 3200);
  }

  // -------- PRIORITY 4: TDS LOW --------
  else if (tdsPPM < 700)
  {
    lcd.setCursor(12, 0);
    lcd.print("LOW ");

    tone(BUZZER, 2000);
  }

  // -------- NORMAL --------
  else
  {
    lcd.setCursor(12, 0);
    lcd.print("SAFE");

    noTone(BUZZER);
  }

  delay(200);
}
