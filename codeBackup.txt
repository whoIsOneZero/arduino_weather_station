/*
 * Program to demonstrate Data Logging/Visualisation using Arduino
 * 
 * ###Connection with SD card module###
 * Vcc->5V
 * Gnd->Gnd
 * MISO->pin 12
 * MOSI->pin 1
 * SCK-> pin 13
 * CS-> pin 4
 * 
 * ###Connection with DS3231###
 * Vcc->5V
 * Gns->Gnd
 * SCL->pin A5
 * SDA-> pin A4
 * 
 * ###Connection with DT11###
 * Vcc->5V
 * Gnd->Gnd
 * Out-> pin 7
 */

#include <DS3231.h> //Library for RTC module (Download from Link in article)
#include <SPI.h> //Library for SPI communication (Pre-Loaded into Arduino)
#include <SD.h> //Library for SD card (Pre-Loaded into Arduino)
#include <dht.h> //Library for dht11 Temperature and Humidity sensor (Download from Link in article)
#include <LiquidCrystal_I2C.h>
#include "RTClib.h"

RTC_DS3231 rtc;

char daysOfTheWeek[7][12] = {"Sunday", "Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday"};

#define DHT11_PIN 7 //Sensor output pin is connected to pin 7
#define ledPin 6
#define sensorPin A0

dht DHT; //Sensor object named as DHT
const int chipSelect = 4; //SD card CS pin connected to pin 4 of Arduino
LiquidCrystal_I2C lcd(0x3F,20,4);  // set the LCD address to 0x27 for a 16 chars and 2 line display 

// Init the DS3231 using the hardware interface

//DS3231  rtc(SDA, SCL);
Time t;
int sensorValue;

byte degree_symbol[8] = 
              {
                0b00111,
                0b00101,
                0b00111,
                0b00000,
                0b00000,
                0b00000,
                0b00000,
                0b00000
              };

void setup()
{

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0,0);
  // Setup Serial connection

  Serial.begin(9600);

  Initialize_SDcard();

  rtc.begin();

  rtc.adjust(DateTime(F(__DATE__), F(__TIME__)));
    // This line sets the RTC with an explicit date & time, for example to set
    // January 21, 2014 at 3am you would call:
  //rtc.adjust(DateTime(2023, 3, 29, 13, 23, 0));
    
  pinMode(ledPin, OUTPUT);
  digitalWrite(ledPin, LOW);
}


void loop()
{
  Read_DHT11();

  Write_SDcard();
  
  lcd.setCursor(0, 0);
  lcd.print("  Temp");
  lcd.setCursor(8,0);
  lcd.print("|");
  lcd.print(" Humidity");
  lcd.setCursor(0,1);
  lcd.print(DHT.temperature);
  lcd.createChar(1, degree_symbol);
  lcd.setCursor(5,1);
  lcd.write(1);
  lcd.setCursor(6,1);
  lcd.print("C");
  lcd.setCursor(8,1);
  lcd.print("|  ");
  lcd.print(DHT.humidity);
  lcd.print("%");

  lcd.setCursor(0,3);
  if(readSensor()<50)
  {
    lcd.setCursor(0,3);
    lcd.print("No    Rainfall");
  }
  else if((readSensor()>70)&&(readSensor()<170))
  {
    lcd.setCursor(0,3);
    lcd.print("Light Rainfall");    
  }
  else
  {
    lcd.setCursor(0,3);
    lcd.print("Heavy Rainfall");
  }

  DateTime now = rtc.now();

    Serial.print(now.year(), DEC);
    Serial.print('/');
    Serial.print(now.month(), DEC);
    Serial.print('/');
    Serial.print(now.day(), DEC);
    Serial.print(" (");
    Serial.print(daysOfTheWeek[now.dayOfTheWeek()]);
    Serial.print(") ");
    Serial.print(now.hour(), DEC);
    Serial.print(':');
    Serial.print(now.minute(), DEC);
    Serial.print(':');
    Serial.print(now.second(), DEC);
    Serial.println();
  
  delay(5000); //Wait for 5 seconds before writing the next data 

}


int readSensor() {
  sensorValue = analogRead(sensorPin);  // Read the analog value from sensor
  int outputVoltage = map(sensorValue, 0, 1023, 255, 0); // map the 10-bit data to 8-bit data

  Serial.print("Sensor Reading = ");
  Serial.println(sensorValue); 
  Serial.print("Initial output voltage = ");
  Serial.println(outputVoltage); 
  Serial.println(" "); 
       

  if(outputVoltage<50)
  {
    analogWrite(ledPin, 0);
    outputVoltage = 0;    
  }
  else if((outputVoltage>=70)&&(outputVoltage<170))
  {
    analogWrite(ledPin, outputVoltage); // generate PWM signal 
    delay(500);                      // wait for a second
    analogWrite(ledPin, 0); // generate PWM signal  
    delay(500);                     // wait for a second
    analogWrite(ledPin, outputVoltage); // generate PWM signal 
    delay(500);                      // wait for a second
    analogWrite(ledPin, 0); // generate PWM signal  
    delay(500);
  }
  else if(outputVoltage>=170)
  {
     analogWrite(ledPin, outputVoltage); // generate PWM signal 
  }
  
  //analogWrite(ledPin, outputVoltage); // generate PWM signal   
  return outputVoltage;             // Return analog moisture value
}
void Write_SDcard()
{
  File dataFile = SD.open("LoggerCD.txt", FILE_WRITE);
  
  if (dataFile) 
  {
    DateTime now = rtc.now();
    
    dataFile.print(now.year(), DEC);
    dataFile.print("-");
    dataFile.print(now.month(), DEC);
    dataFile.print("-");
    dataFile.print(now.day(), DEC); //Store date on SD card    
    dataFile.print(","); //Move to next column using a ","
    
    dataFile.print(now.hour(), DEC);
    dataFile.print(":");
    dataFile.print(now.minute(), DEC);
    dataFile.print(":");
    dataFile.print(now.second(), DEC);
    dataFile.print(","); //Move to next column using a ","

    dataFile.print(DHT.temperature); //Store date on SD card
    dataFile.print(","); //Move to next column using a ","

    dataFile.print(DHT.humidity); //Store date on SD card
    dataFile.print(","); //Move to next column using a ","

    dataFile.print(sensorValue);
    //dataFile.print(",");

    dataFile.println(); //End of Row move to next row
    dataFile.close(); //Close the file
  }

  else
  Serial.println("OOPS!! SD card writing failed");
}

void Initialize_SDcard()
{
  if (!SD.begin(chipSelect)) 
  {
    Serial.println("Card failed, or not present");
    return;
  }
  File dataFile = SD.open("LoggerCD.txt", FILE_WRITE);

  if (dataFile) 
  {
    dataFile.println("Date,Time,Temperature,Humidity,Rainfall"); //Write the first row of the excel file
    dataFile.close();
  }
}

void Read_DHT11()
{
  int chk = DHT.read11(DHT11_PIN);
}