#include <SPI.h>
#include <MFRC522.h>
#include <Servo_ESP32.h>

#define SS_PIN    5    // ESP32 pin GPIO5
#define RST_PIN   27   // ESP32 pin GPIO27
#define BUZZER_PIN 22   // Buzzer pin
#define BLUE_LED  12   // Blue LED GPIO pin
#define RED_LED   16   // Red LED GPIO pin
#define GREEN_LED 21   // Green LED GPIO pin
#define SERVO_PIN  15   // Servo motor GPIO pin

MFRC522 rfid(SS_PIN, RST_PIN);
Servo_ESP32 gateServo;

bool blueLEDOn = true;
bool greenLEDOn = false;
bool redLEDOn = false;

unsigned long accessStartTime = 0;
unsigned long gateCloseTime = 0;
bool gateOpen = false;

void setup() {
  Serial.begin(9600);
  SPI.begin();
  rfid.PCD_Init();

  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(BLUE_LED, OUTPUT);
  pinMode(RED_LED, OUTPUT);
  pinMode(GREEN_LED, OUTPUT);

  gateServo.attach(SERVO_PIN);

  digitalWrite(BLUE_LED, HIGH);
  blueLEDOn = true;

  Serial.println("Tap an RFID/NFC tag on the RFID-RC522 reader");
}

void loop() {
  unsigned long currentTime = millis();
  unsigned long elapsedTime = currentTime - accessStartTime;
  
  // Check if the gate was opened and it's time to close it
  if (gateOpen && (currentTime - gateCloseTime >= 8000)) {
    closeGate();
    gateOpen = false;
    digitalWrite(BLUE_LED, HIGH);
    blueLEDOn = true;
  }

  if (greenLEDOn && elapsedTime >= 8000) {
    digitalWrite(GREEN_LED, LOW);
    greenLEDOn = false;
    digitalWrite(BLUE_LED, HIGH);
    blueLEDOn = true;
  }

  if (redLEDOn && elapsedTime >= 8000) {
    digitalWrite(RED_LED, LOW);
    redLEDOn = false;
    digitalWrite(BLUE_LED, HIGH);
    blueLEDOn = true;
  }

  if (rfid.PICC_IsNewCardPresent()) {
    if (rfid.PICC_ReadCardSerial()) {
      MFRC522::PICC_Type piccType = rfid.PICC_GetType(rfid.uid.sak);
      Serial.print("RFID/NFC Tag Type: ");
      Serial.println(rfid.PICC_GetTypeName(piccType));

      Serial.print("UID:");
      for (int i = 0; i < rfid.uid.size; i++) {
        Serial.print(rfid.uid.uidByte[i] < 0x10 ? " 0" : " ");
        Serial.print(rfid.uid.uidByte[i], HEX);
      }
      Serial.println();

      if (isAuthorized(rfid.uid)) {
        beepHigh(375);
        digitalWrite(GREEN_LED, HIGH);
        greenLEDOn = true;
        if (redLEDOn) {
          digitalWrite(RED_LED, LOW);
          redLEDOn = false;
        }
        if (blueLEDOn) {
          digitalWrite(BLUE_LED, LOW);
          blueLEDOn = false;
        }
        accessStartTime = currentTime;
        openGate();
        gateCloseTime = currentTime; // Set the time for gate closing
        gateOpen = true; // Indicate that the gate is open
      } else {
        beepHigh(1000);
        digitalWrite(RED_LED, HIGH);
        redLEDOn = true;
        if (greenLEDOn) {
          digitalWrite(GREEN_LED, LOW);
          greenLEDOn = false;
        }
        if (blueLEDOn) {
          digitalWrite(BLUE_LED, LOW);
          blueLEDOn = false;
        }
        accessStartTime = currentTime;
        // You can close the gate here if needed
        closeGate();
      }

      rfid.PICC_HaltA();
      rfid.PCD_StopCrypto1();
    }
  }
}

bool isAuthorized(MFRC522::Uid uid) {
  // Modify this condition to match your authorized UID
  return (uid.uidByte[0] == 0x03 &&
          uid.uidByte[1] == 0x59 &&
          uid.uidByte[2] == 0xD8 &&
          uid.uidByte[3] == 0x1B);
}

void beepHigh(int duration) {
  tone(BUZZER_PIN, 2000, duration);
  delay(1000);
  noTone(BUZZER_PIN);
}

void openGate() {
  gateServo.write(90);
}

void closeGate() {
  gateServo.write(0);
}
