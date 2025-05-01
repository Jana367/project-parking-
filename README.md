#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <RTClib.h>
#include <SPI.h>
#include <MFRC522.h>
#include <Servo.h>

#define IR1 2
#define IR2 3
#define IR3 4

#define TRIG1 7
#define ECHO1 8

#define LED1 A2
#define LED2 A3
#define LED3 A5

#define RST_PIN 9
#define SS_PIN 10
#define SERVO_PIN 5

#define SERVO2 6
#define TRIG2 A0
#define ECHO2 A1

LiquidCrystal_I2C lcd2(0x26, 20, 4);
LiquidCrystal_I2C lcd(0x27, 20, 4);
RTC_DS3231 rtc;
Servo exitGate;
Servo gateServo;
MFRC522 rfid(SS_PIN, RST_PIN);

bool carInGarage1 = false;
bool carInGarage2 = false;
bool carInGarage3 = false;

DateTime entryTime1;
DateTime exitTime1;
DateTime entryTime2;
DateTime exitTime2;
DateTime entryTime3;
DateTime exitTime3;

const float costPerSecond = 5.0;

long duration1;
int distance1;

long duration2;
int distance2;

byte allowedUIDs[][4] = {
  {0x2A, 0x51, 0x95, 0x97},
  {0x33, 0x51, 0xB4, 0x1B}
};

void testServoFunction() {
    Serial.println("Testing Servo...");
    exitGate.attach(SERVO_PIN);
    exitGate.write(0);
    delay(1000);
    exitGate.write(120);
    delay(1000);
    exitGate.write(0);
}

void setup() {
    Serial.begin(9600);
    Serial.println("System Initializing...");

    pinMode(LED_BUILTIN, OUTPUT);
    pinMode(LED1, OUTPUT);
    pinMode(LED2, OUTPUT);
    pinMode(LED3, OUTPUT);
    digitalWrite(LED_BUILTIN, HIGH);
    Serial.println("LEDs initialized");

    Wire.begin();
    Serial.println("I2C Initialized");

    lcd2.init();
    lcd2.backlight();
    lcd2.print("LCD2 OK");
    Serial.println("LCD Initialized");
    delay(2000);
    lcd2.clear();

    lcd.init();
    lcd.backlight();
    lcd.print("LCD OK");
    Serial.println("LCD2 (0x27) Initialized");
    delay(2000);
    lcd.clear();

    if (!rtc.begin()) {
        Serial.println("RTC Error!");
        lcd2.print("RTC Error!");
        while (1);
    }
    Serial.println("RTC Initialized");
    rtc.adjust(DateTime(2023, 1, 1, 0, 0, 0));
    Serial.println("RTC Time Adjusted");

    pinMode(IR1, INPUT_PULLUP);
    pinMode(IR2, INPUT_PULLUP);
    pinMode(IR3, INPUT_PULLUP);
    Serial.println("IR Sensors Initialized");

    pinMode(TRIG1, OUTPUT);
    pinMode(ECHO1, INPUT);
    pinMode(TRIG2, OUTPUT);
    pinMode(ECHO2, INPUT);
    Serial.println("Ultrasonic Sensors Initialized");

    SPI.begin();
    rfid.PCD_Init();
    Serial.println("RFID Reader Ready");

    exitGate.attach(SERVO_PIN);
    exitGate.write(0);
    gateServo.attach(SERVO2);
    gateServo.write(0);
    Serial.println("Servos initialized and set to closed position");

    testServoFunction();

    digitalWrite(LED_BUILTIN, LOW);
    Serial.println("Setup Completed");
}

void loop() {
    DateTime now = rtc.now();

    digitalWrite(TRIG1, LOW);
    delayMicroseconds(2);
    digitalWrite(TRIG1, HIGH);
    delayMicroseconds(10);
    digitalWrite(TRIG1, LOW);
    duration1 = pulseIn(ECHO1, HIGH);
    distance1 = duration1 * 0.0344 / 2;
    if (distance1 < 2 || distance1 > 10) {
        distance1 = -1;
    }

    digitalWrite(TRIG2, LOW);
    delayMicroseconds(2);
    digitalWrite(TRIG2, HIGH);
    delayMicroseconds(10);
    digitalWrite(TRIG2, LOW);
    duration2 = pulseIn(ECHO2, HIGH);
    distance2 = duration2 * 0.0344 / 2;
    if (distance2 >= 2 && distance2 <= 10) {
        if (!areAllGaragesFull()) {
            Serial.println("Opening entry gate for 5 seconds...");
            gateServo.write(90);
            delay(2500);
            gateServo.write(0);
            Serial.println("Entry gate closed");
        }
        else {
            lcd.clear();
            lcd.setCursor(0, 0);
            lcd.print("All Garages Full");
            Serial.println("Entry blocked: All garages full");
        }
    }

    if (distance1 >= 2 && distance1 <= 10) {
        int lastActiveGarage = 0;
        unsigned long latestTime = 0;

        if (exitTime1.unixtime() > latestTime) {
            latestTime = exitTime1.unixtime();
            lastActiveGarage = 1;
        }
        if (exitTime2.unixtime() > latestTime) {
            latestTime = exitTime2.unixtime();
            lastActiveGarage = 2;
        }
        if (exitTime3.unixtime() > latestTime) {
            latestTime = exitTime3.unixtime();
            lastActiveGarage = 3;
        }

        if (lastActiveGarage > 0) {
            DateTime lastEntry, lastExit;
            if (lastActiveGarage == 1) {
                lastEntry = entryTime1;
                lastExit = exitTime1;
            }
            else if (lastActiveGarage == 2) {
                lastEntry = entryTime2;
                lastExit = exitTime2;
            }
            else if (lastActiveGarage == 3) {
                lastEntry = entryTime3;
                lastExit = exitTime3;
            }

            lcd2.clear();
            lcd2.print("Last Garage ");
            lcd2.print(lastActiveGarage);

            lcd2.setCursor(0, 1);
            lcd2.print("Time: ");
            lcd2.print((lastExit.unixtime() - lastEntry.unixtime()));
            lcd2.print(" sec");

            lcd2.setCursor(0, 2);
            lcd2.print("Cost: ");
            lcd2.print((lastExit.unixtime() - lastEntry.unixtime()) * costPerSecond);
            lcd2.print(" EGP");

            Serial.print("Last Active Garage ");
            Serial.print(lastActiveGarage);
            Serial.print(" - Time: ");
            Serial.print((lastExit.unixtime() - lastEntry.unixtime()));
            Serial.print(" sec, Cost: ");
            Serial.print((lastExit.unixtime() - lastEntry.unixtime()) * costPerSecond);
            Serial.println(" EGP");

            delay(2500);
            lcd2.clear();
        }
    }

    checkGarage(IR1, 1, entryTime1, exitTime1, carInGarage1, LED1, now);
    checkGarage(IR2, 2, entryTime2, exitTime2, carInGarage2, LED2, now);
    checkGarage(IR3, 3, entryTime3, exitTime3, carInGarage3, LED3, now);

    updateLCDWithGarageStatus();
    checkRFID();
    delay(1000);
}

void checkGarage(int irPin, int garageNum, DateTime& entryTime, DateTime& exitTime, bool& carInGarage, int ledPin, DateTime now) {
    if (digitalRead(irPin) == LOW && !carInGarage) {
        carInGarage = true;
        entryTime = now;
        Serial.print("Garage ");
        Serial.print(garageNum);
        Serial.println(": Car entered.");
        Serial.print("Entry Time: ");
        printSerialTime(entryTime);
        digitalWrite(ledPin, HIGH);
        Serial.print("Garage ");
        Serial.print(garageNum);
        Serial.println(" LED turned ON");
    }

    else if (digitalRead(irPin) == HIGH && carInGarage) {
        carInGarage = false;
        exitTime = now;
        Serial.print("Garage ");
        Serial.print(garageNum);
        Serial.println(": Car exited.");
        Serial.print("Exit Time: ");
        printSerialTime(exitTime);
        digitalWrite(ledPin, LOW);
        Serial.print("Garage ");
        Serial.print(garageNum);
        Serial.println(" LED turned OFF");

        unsigned long totalTime = (exitTime.unixtime() - entryTime.unixtime());
        float totalCost = totalTime * costPerSecond;
        Serial.print("Calculated Time: ");
        Serial.print(totalTime);
        Serial.println(" seconds");
        Serial.print("Calculated Cost: ");
        Serial.print(totalCost);
        Serial.println(" EGP");
    }
}

void printSerialTime(DateTime t) {
    if (t.hour() < 10) Serial.print("0");
    Serial.print(t.hour());
    Serial.print(":");
    if (t.minute() < 10) Serial.print("0");
    Serial.print(t.minute());
    Serial.print(":");
    if (t.second() < 10) Serial.print("0");
    Serial.println(t.second());
}

void updateLCDWithGarageStatus() {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Garage Status:");

    lcd.setCursor(0, 1);
    lcd.print("G1:");
    lcd.print(carInGarage1 ? "Occupied" : "Free");

    lcd.setCursor(0, 2);
    lcd.print("G2:");
    lcd.print(carInGarage2 ? "Occupied" : "Free");

    lcd.setCursor(0, 3);
    lcd.print("G3:");
    lcd.print(carInGarage3 ? "Occupied" : "Free");
    Serial.println("updateLCDWithGarageStatus() called");
}

bool areAllGaragesFull() {
    return carInGarage1 && carInGarage2 && carInGarage3;

}

void checkRFID() {
    if (!rfid.PICC_IsNewCardPresent() || !rfid.PICC_ReadCardSerial()) {
        return;
    }

    Serial.print("Detected Card UID:");
    for (byte i = 0; i < rfid.uid.size; i++) {
        Serial.print(rfid.uid.uidByte[i] < 0x10 ? " 0" : " ");
        Serial.print(rfid.uid.uidByte[i], HEX);
    }
    Serial.println();

    if (isCardAllowed(rfid.uid.uidByte)) {
        handleValidCard();
    }
    else {
        handleInvalidCard();
    }

    rfid.PICC_HaltA();
    rfid.PCD_StopCrypto1();
}

bool isCardAllowed(byte* uid) {
    Serial.println("Checking if card is allowed...");
    for (int i = 0; i < sizeof(allowedUIDs) / sizeof(allowedUIDs[0]); i++) {
        bool match = true;
        for (int j = 0; j < 4; j++) {
            if (uid[j] != allowedUIDs[i][j]) {
                match = false;
                break;
            }
        }
        if (match) {
            Serial.println("Card is in allowed list");
            return true;
        }
    }
    Serial.println("Card is NOT in allowed list");
    return false;
}

void handleValidCard() {
    Serial.println(">> Access Granted ✅");
    Serial.println("Opening servo gate...");
    exitGate.write(90);
                    lcd2.print("Thank you Sir Bye !! ");

    Serial.println("Servo gate is now open (90 degrees)");
    delay(2500);
        lcd2.clear();

    exitGate.write(0);
    Serial.println("Servo gate is now closed (0 degrees)");
}

void handleInvalidCard() {
    Serial.println(">> Access Denied ❌");
}
