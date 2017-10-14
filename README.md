# fricosk
Pokusy meran
//Meter reading program using Itron smart meter
//Added LED that turns on/off with each watt

#include "FileLogger.h" // http://code.google.com/p/arduino-filelogger/
#include <Time.h>  
#include <Wire.h>  
#include <DS1307RTC.h> // a basic DS1307 library - http://www.arduino.cc/playground/Code/Time

const int ledPin = 14; // LED watt indicator initialization. Turns on or off each time a watt is measured
int ledState = LOW;

volatile unsigned long wattSensor=0;  //Counts power pulses in interrupt, 1 pulse = 1 watt
unsigned long wattSensorTemp=0;  //Temporary power pulse counts for printing - watts
unsigned long totalWatts=0; //Temporary total power used for printing - watts

byte start[7]= {
 'B','e','g','i','n',0x0D,0x0A};
byte Buffer[31]; //Buffer for printing to SD card
time_t oldtime=0; //last time loop has ran
int result; //status of SD card write - 0 = "OK", 1 = "Fail initializing", 2 = "Fail appending"
unsigned long loggingRate=5; //logging rate in seconds
int length; //length of buffer string

void setup(void)
{
 pinMode(ledPin, OUTPUT);  // LED watt indicator initialization
 attachInterrupt(1, wattSensorInterrupt, FALLING);  //interrupt1 - digital pin 3 - If we detect a change from HIGH to LOW we call wattSensorInterrupt

 do
 {
   result = FileLogger::append("data.txt", start, 7);//Initialize the SD Card
 }
 while(result != 0);

 // the next two lines can be removed if the RTC has been set
 //  setTime(17,05,0,1,3,10); // set time to 17:05:00  1 Mar 2010
 //  RTC.set(now());  // set the RTC to the current time (set in the previous line)

 // format for setting time - setTime(hr,min,sec,day,month,yr);

 setSyncProvider(RTC.get);   // the function to get the time from the RTC

 Serial.begin(9600);
}

void loop(void)
{
 time_t t = now();
 //  if(t >= oldtime + loggingRate) //proceed with loop if loggingRate seconds have elapsed
 if( minute(t) != oldtime) //use this if 1 minute intervals is desired
 {
   wattSensorTemp=wattSensor; //number of watts for interval
   wattSensor=0; //reset watts counter
   totalWatts=totalWatts+wattSensorTemp; //total watts counter
   oldtime = minute(t); //use this for one minute interval
   //oldtime = t; //or this for loggingRate interval

   do
   {
     Buffer[0]='A'; //start packet/Site ID
     Buffer[1]=','; //start of data string
     //format of printDigits - printDigits(tempBuffer,bufferPntr,numberDigits)
     printDigits(t,2,10); //t = now()
     Buffer[12]=',';
     printDigits(wattSensorTemp,13,5); //wattSensorTemp
     Buffer[18]=',';
     printDigits(totalWatts,19,8); //totalWatts
     Buffer[27]=',';
     Buffer[28]='>'; //end data packet
     Buffer[29]=0x0D;
     Buffer[30]=0x0A;

     length=sizeof(Buffer);
     result = FileLogger::append("data.txt", Buffer, length);
   }
   while( result != 0);

   for (int i=0; i < length ;i++) {
     Serial.print(Buffer[i]); //Serial print buffer string to serial monitor or xBee
   }
 }
}

void wattSensorInterrupt() //Interrupt1 counter routine for counting watts
{
 wattSensor=wattSensor+1;  //Update number of pulses, 1 pulse = 1 watt
 if (ledState == LOW) //Cycle LED on or off each time one watt is counted
   ledState = HIGH;
 else
   ledState = LOW;
 digitalWrite(ledPin, ledState);
}

void printDigits(unsigned long tempBuffer, int bufferPntr, int numberDigits) //assign data to output array
{
 unsigned long modDivide = 1;
 for (int i = 0; i < numberDigits; i++) {
   Buffer[bufferPntr+numberDigits-1-i]=((tempBuffer%(modDivide*10))/modDivide)+'0';
   modDivide=modDivide*10;
 }
}
