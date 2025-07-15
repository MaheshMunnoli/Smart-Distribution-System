// -------------------------------------------------------------
// Project: Smart Distribution System - Fault Detection Module
// Author : Mahesh Munnolli
// Date   : July 2025
// Description:
// This project detects overhead line faults such as short circuit 
// and line-to-line faults. It measures real-time voltage and current, 
// displays the values on an I2C LCD, and activates relays on fault detection.
// -------------------------------------------------------------

#include <LiquidCrystal_I2C.h>
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Analog input pins for voltage measurement
#define ANALOG_IN_PIN1 A1
#define ANALOG_IN_PIN2 A2
#define ANALOG_IN_PIN3 A3

// Digital input pins for fault detection
#define Short 7
#define Line 6

// Resistor values for voltage divider (Ohms)
float R11 = 30000.0, R12 = 7500.0;
float R21 = 30000.0, R22 = 7500.0;
float R31 = 30000.0, R32 = 7500.0;

// Reference voltage
float ref_voltage1 = 5.0, ref_voltage2 = 5.0, ref_voltage3 = 5.0;

// ADC values
int adc_value1 = 0, adc_value2 = 0, adc_value3 = 0;

// Voltage values
float adc_voltage1 = 0.0, in_voltage1 = 0.0;
float adc_voltage2 = 0.0, in_voltage2 = 0.0;
float adc_voltage3 = 0.0, in_voltage3 = 0.0;

// Current sensor
const int analogchannel = A0; // ACS712 connected here
int sensitivity = 185;        // mV per Amp (185 for 5A module)
int adcvalue = 0;
int offsetvoltage = 2500;
double Voltage = 0;
double val1, val2, result, ecurrent = 0;

// Relays
#define Relay1 11
#define Relay2 10
#define Relay3 9
#define Relay4 8

// Digital fault inputs
float sensorValue1, sensorValue2;

void setup() {
  Serial.begin(9600);

  pinMode(Relay1, OUTPUT);
  pinMode(Relay2, OUTPUT);
  pinMode(Relay3, OUTPUT);
  pinMode(Relay4, OUTPUT);

  digitalWrite(Relay1, LOW);
  digitalWrite(Relay2, LOW);
  digitalWrite(Relay3, LOW);
  digitalWrite(Relay4, LOW);

  Serial.println("SMART DISTRIBUTION SYSTEM - FAULT DETECTION");

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("SMART DISTRIBUTION");
  lcd.setCursor(0, 1);
  lcd.print("FAULT DETECTION");
  delay(3000);
}

void loop() {
  // Voltage readings
  adc_value1 = analogRead(ANALOG_IN_PIN1);
  adc_voltage1 = (adc_value1 * ref_voltage1) / 1024.0;
  in_voltage1 = adc_voltage1 * (R11 + R12) / R12;

  adc_value2 = analogRead(ANALOG_IN_PIN2);
  adc_voltage2 = (adc_value2 * ref_voltage2) / 1024.0;
  in_voltage2 = adc_voltage2 * (R21 + R22) / R22;

  adc_value3 = analogRead(ANALOG_IN_PIN3);
  adc_voltage3 = (adc_value3 * ref_voltage3) / 1024.0;
  in_voltage3 = adc_voltage3 * (R31 + R32) / R32;

  // Current reading loop for max detection
  for (int i = 0; i < 10; i++) {
    currentsen();
    val1 = result;
    currentsen();
    val2 = ecurrent;

    result = (val2 > val1) ? val2 : val1;
  }

  // Display on Serial Monitor
  Serial.print("Current: ");
  Serial.print(result);
  Serial.print("\t");

  Serial.print("Input Voltage1 = ");
  Serial.print(in_voltage1, 2);
  Serial.print("\t");

  Serial.print("Input Voltage2 = ");
  Serial.print(in_voltage2, 2);
  Serial.print("\t");

  Serial.print("Input Voltage3 = ");
  Serial.println(in_voltage3, 2);
  delay(2000);

  // Display on LCD
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("R1=");
  lcd.setCursor(3, 0);
  lcd.print(in_voltage1, 2);

  lcd.setCursor(0, 1);
  lcd.print("Y1=");
  lcd.setCursor(3, 1);
  lcd.print(in_voltage2, 2);

  lcd.setCursor(9, 0);
  lcd.print("B1=");
  lcd.setCursor(12, 0);
  lcd.print(in_voltage3, 2);

  lcd.setCursor(9, 1);
  lcd.print("C1=");
  lcd.setCursor(12, 1);
  lcd.print(result);
  delay(1000);

  // Fault detection
  sensorValue1 = digitalRead(Short);
  sensorValue2 = digitalRead(Line);

  if (sensorValue1 == LOW) {
    Serial.println(" | SHORT CIRCUIT FAULT DETECTED!");
    triggerRelays();
    lcd.clear();
    lcd.backlight();
    lcd.setCursor(0, 0);
    lcd.print(" SHORT CIRCUIT");
    lcd.setCursor(0, 1);
    lcd.print(" FAULT DETECTED");
    delay(5000);
  }

  if (sensorValue2 == HIGH) {
    Serial.println(" | LINE TO LINE FAULT DETECTED!");
    triggerRelays();
    lcd.clear();
    lcd.backlight();
    lcd.setCursor(0, 0);
    lcd.print(" LINE TO LINE");
    lcd.setCursor(0, 1);
    lcd.print(" FAULT DETECTED");
    delay(5000);
  } else {
    resetRelays();
  }
}

void triggerRelays() {
  digitalWrite(Relay1, HIGH);
  digitalWrite(Relay2, HIGH);
  digitalWrite(Relay3, HIGH);
  digitalWrite(Relay4, HIGH);
}

void resetRelays() {
  digitalWrite(Relay1, LOW);
  digitalWrite(Relay2, LOW);
  digitalWrite(Relay3, LOW);
  digitalWrite(Relay4, LOW);
  delay(500);
}

void currentsen() {
  adcvalue = analogRead(analogchannel);
  Voltage = (adcvalue / 1024.0) * 5000;
  ecurrent = (Voltage - offsetvoltage) / sensitivity;
  if (ecurrent < 0) {
    ecurrent = -ecurrent;
  }
  delay(100);
}
