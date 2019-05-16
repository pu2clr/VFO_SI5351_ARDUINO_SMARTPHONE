# VFO and BFO using Arduino with Si5351 controlled by SmartPhone

## __Documentação em Português clique [aqui](https://github.com/pu2clr/VFO_BFO_OLED_ARDUINO/tree/master/Experiments/VFO_RADIO_CD2003GP/Docs).__

By PU2CLR - Ricardo
April, 2019

## Table of contents

1. [Introduction](https://github.com/pu2clr/VFO_SI5351_ARDUINO_SMARTPHONE#introduction)
2. [VFO and BFO Interface](https://github.com/pu2clr/VFO_SI5351_ARDUINO_SMARTPHONE#vfo-and-bfo-interface)
3. [Schematic used with Arduino Atmega328 (UNO, Pro Mini, Nano etc)](https://github.com/pu2clr/VFO_SI5351_ARDUINO_SMARTPHONE#schematic---arduino-atmega328-uno-pro-mini-nano-etc)
4. [Arduino Sketch](https://github.com/pu2clr/VFO_SI5351_ARDUINO_SMARTPHONE#arduino-sketch)
   1. [Band Table](https://github.com/pu2clr/VFO_SI5351_ARDUINO_SMARTPHONE#band-table) 
   2. [Step Table](https://github.com/pu2clr/VFO_SI5351_ARDUINO_SMARTPHONE#step-table)
   3. [Callback Functions](https://github.com/pu2clr/VFO_SI5351_ARDUINO_SMARTPHONE#callback-functions)
5. [Mobile Application](https://github.com/pu2clr/VFO_SI5351_ARDUINO_SMARTPHONE#mobile-application)
6. [Videos]()


## Introduction

This project implements a VFO and BFO  using an Arduino with Si5351 signal generator. You can controll the VFO using an encoder and also a SmartPhone via a mobile application developed here for this purpose. 

The Si5351 is an I2C configurable clock generator that is very appropriate for receivers and transceivers projects in amateur radio applications. See more on [Silicon Labs documentation](https://www.silabs.com/documents/public/data-sheets/Si5351-B.pdf).

 This project uses two outputs of the Si5351.  The VFO  uses the output 0 (CLK0) and can oscillate from 100KHz to 160MHz. The BFO uses the output 1 (CLK1) of Si5351 and it was configured to oscillate from 452KHz to 458KHz (you can modigy the BFO range to any other from 8KHz to 160MHz).  

 The Arduino sketch was projected to allow the user modify easly the band, range and step configurations.
 See Arduino Sketch below. 


## VFO and BFO interface

The user can control the VFO and BFO by using three buttons (bands, steps and VFO / BFO switch) and an encoder. The user can also connect this VFO with an iOS or Android device and control it remotelly.  The Dial was implemented with the OLED Display 128 x 64 Pixels White 0.96 Inch I2C. 


### Display Layout


| Layout |  Layout |
| ------ |  ------ |
| ![Displau photo 1](https://github.com/pu2clr/VFO_SI5351_ARDUINO_SMARTPHONE/blob/master/images/OLED_01.png)| ![Displau photo 2](https://github.com/pu2clr/VFO_SI5351_ARDUINO_SMARTPHONE/blob/master/images/OLED_02.png) |
| ![Displau photo 3](https://github.com/pu2clr/VFO_SI5351_ARDUINO_SMARTPHONE/blob/master/images/OLED_03.png)| ![Displau photo 4](https://github.com/pu2clr/VFO_SI5351_ARDUINO_SMARTPHONE/blob/master/images/OLED_04.png) |


## Schematic - Arduino Atmega328 (UNO, Pro Mini, Nano etc)

![Schematic used with Arduino Atmega328](https://github.com/pu2clr/VFO_SI5351_ARDUINO_SMARTPHONE/blob/master/schematic/vfobfo_schematic_atmega328.png)



## Arduino Sketch 

The VFO and BFO project cen be easily modified to adapt it better to your needs. 
The main feature of the Sketch is the table of band. You can modify the amount of bands and ranges. The band information is shown below. 
 
- __Band name__ - Band name that will show on display;
- __Initial Freq.__ -  Lowest band frequency (1/100 Hz);
- __Final Freq.__ - highest frequency band (1/100 Hz);
- __Last Freq.__ - Store the current frequency. Useful when the user switches the band and get back to it later (it starts with minFreq value);
- __offset__ - Consider an IF offset.  The VFO shows on the display the real frequency of the station and makes the signal generator to oscillate considering the IF used by the radio design (1/100 Hz);
- __Freq Unit__ - Frequency unit that will be show on the display for the current band;
- __Divider__ - Divider used to reduce the number of digits in the display;
- __decimals__ - number of decimal places after the comma;
- __Initial Step Index__ - Lowest step index used for the band (see Step table)
- __Final Step Index__ - Highest step index used for the band (see Step table) 
- __Last Step Index__ - Default step index or last step index used for the band (see Step table)
- __callback function__ - point to the function that handles specific things for the band

The implementation of the band information is shown below.  An array of band with this structure is implemmented 

```cpp
// Structure for Bands database
typedef struct
{
  char *name;
  uint64_t minFreq;       // Min. frequency value for the band (unit 0.01Hz)
  uint64_t maxFreq;       // Max. frequency value for the band (unit 0.01Hz)
  uint64_t lastFreq;      // Store the current frequency before change to other band (it starts with minFreq value)
  long long offset;
  char *unitFreq;         // MHz or KHz
  float divider;          // value that will be the divider of current clock (just to present on display)
  short decimals;         // number of digits after the comma
  short initialStepIndex; // Index to the initial step of incrementing
  short finalStepIndex;   // Index to the final step of incrementing
  short lastStepIndex;    // Index to the last step used (initial index value same index defult)
  void (*fstart)(void);   // pointer to the function that will handle specific things for the band immediately after the band is selected
 } Band;
```

The code below implements the band table of the VFO for a hypothetical radio. 
The values amBroadcast and  fmBroadcast are points to functions that will execute some specific actions for the band. When the value is NULL, no action will be executed.  

```cpp
// Band database. You can change the band ranges if you need.
// The unit of frequency here is 0.01Hz (1/100 Hz). See Etherkit Library at https://github.com/etherkit/Si5351Arduino
Band band[] = {
    {"MW  ", 50000000LLU, 170000000LLU, 45500000LU, "KHz", 100000.0f, 0, 3, 6, 5, amBroadcast},
    {"SW1 ", 170000000LLU, 1000000000LLU, 45500000LU, "KHz", 100000.0f, 2, 1, 6, 3, amBroadcast},
    {"SW2 ", 1000000000LLU, 2000000000LLU, 45500000LU, "KHz", 100000.0f, 2, 1, 6, 3, amBroadcast},
    {"SW3 ", 2000000000LLU, 3000000000LLU, 45500000LU, "KHz", 100000.0f, 2, 1, 6, 3, amBroadcast},
    {"VHF1", 3000000000LLU, 7600000000LLU, 45500000LU, "KHz", 100000.0f, 2, 1, 7, 3, NULL},
    {"FM  ", 7600000000LLU, 10800000000LLU, 1075000000LLU, "MHz", 100000000.0f, 2, 6, 8, 7, fmBroadcast},
    {"AIR ", 10800000000LLU, 13700000000LLU, 1075000000LLU, "MHz", 100000000.0f, 3, 2, 8, 5, NULL},
    {"VHF2", 13700000000LLU, 14400000000LLU, 1075000000LLU, "MHz", 100000000.0f, 3, 2, 8, 5, NULL},
    {"2M  ", 14400000000LLU, 15000000000LLU, 1075000000LLU, "MHz", 100000000.0f, 3, 2, 8, 5, NULL},
    {"VFH3", 15000000000LLU, 16000000000LLU, 1075000000LLU, "MHz", 100000000.0f, 3, 2, 8, 5, NULL}};
```



### Band Table 

| Band name | Initial Freq.  | Final Freq. | Last Freq. | offset | Freq Unit | Divider | Dec. | Initial Step Index | Final Step Index | Default / Last Step index | Action Function |
| --------- | ----------------------- | -------------------- | ---------------- | ---------------- | -----------------| --------------- |------------------ | ---------------- | ---------------- | -----------| -----------|  
| MW   | 50000000 | 170000000 | 50000000 | 45500000 |  KHz | 100000 | 2 | 3 | 6 | 5 | amBroadcast |
| SW1  | 170000000 | 1000000000 | 170000000 | 45500000  | KHz | 100000 | 2 | 2 | 6 | 3 | amBroadcast |
| SW2  | 1000000000 | 2000000000 | 1000000000 | 45500000  | KHz | 100000 | 2 | 2 | 6 | 3 | amBroadcast |
| SW3  | 2000000000 | 3000000000 | 2000000000 | 45500000  | KHz | 100000 | 2 | 2 | 6 | 3 | amBroadcast |
| VHF1 | 3000000000 | 7600000000 | 3000000000 | 45500000  | KHz | 100000 | 2 | 2 | 7 | 3 | NULL |
| FM   | 7600000000 | 10800000000 | 7600000000 | 1075000000  | MHz |  100000000 | 1 | 6 | 8 | 7 | fmBroadcast |
| AIR  | 10800000000 | 13700000000 | 10800000000 | 1075000000  | MHz | 100000000 | 2 | 2 | 8 | 5 | NULL |
| VHF2 | 13700000000 | 14400000000 | 13700000000 | 1075000000  | MHz | 100000000 | 2 | 2 | 8 | 5 | NULL |
| 2M  | 14400000000 | 15000000000 | 14400000000 | 1075000000  | MHz | 100000000 | 2 | 2 | 8 | 5 | NULL |
| VFH3 | 15000000000 | 16000000000 | 15000000000 | 1075000000  | MHz| 100000000 | 2 | 2 | 8| 5 | NULL |


{table-plus:border=0|class=''}
|| header1 || header 2 ||
| cell11 | cell 12 | 
| cell 21 | cell 22 |
{table-plus}


### Step Table 

The step table is implemented by the code below.  Each band uses a subset of the band table. This will depend on the characteristics of the band.


```cpp
// Struct for step database
typedef struct
{
  char *name; // step label: 50Hz, 100Hz, 500Hz, 1KHz, 5KHz, 10KHz and 500KHz
  long value; // Frequency value (unit 0.01Hz See documentation) to increment or decrement
} Step;
```


```cpp 
// Steps database. You can change the Steps and numbers of steps here if you need.
Step step[] = {
    {"10Hz  ", 1000},
    {"100Hz ", 10000},
    {"500Hz ", 50000},
    {"1KHz  ", 100000},
    {"5KHz  ", 500000},
    {"10KHz  ",1000000},
    {"50KHz ", 5000000},
    {"100KHz", 10000000},
    {"500KHz", 50000000}};
```



| Step Index | Step name | Step Value (1/100 Hz) |
| ---------- | --------- | --------------------- | 
| 0          | 10Hz      |     1000 |
| 1          | 100Hz     |    10000 |
| 2          | 500Hz     |    50000 |
| 3          | 1KHz      |   100000 |
| 4          | 5KHz      |   500000 |
| 5          | 10KHz     |  1000000 |
| 6          | 50KHz     |  5000000 |
| 7          | 100KHz    | 10000000 |
| 8          | 500KHz    | 50000000 |



### Callback functions

This sketch uses callback functions to complement specific actions for a specific band.
For example, you might need to turn on a LED, start a relay or set up a filter when yiu select a band. 
If the device that will use this VFO/BFO has different behaviours for different band, you might want use callback functions.  See Band table for more information. 


The code below shows the callback functions declaration

```cpp
// Callback functions declarations
// Depending on the selected band, you may want to perform specific actions
// The functions declared below will do something for MW and FM BAND. See implementation later.
// You can implement callback function for other bands
void amBroadcast(); // See implementation later.
void fmBroadcast(); // See implementation later.
```

The callback functions above are referenced in band database (see Band band[]) above.


```cpp
// Callback implementation

// Doing something spefict for MW band
void amBroadcast()
{
  // TO DO
  STATUSLED(HIGH); // Just testing if it is working - Turn LED ON
}

// Doing something spefict for FM
void fmBroadcast()
{
  // TO DO
  STATUSLED(LOW); // Just testing if it is working - Turn LED OFF
}
```

The code below shows the use of callback function when the use changes the band

```cpp
 if (digitalRead(BUTTON_BAND) == HIGH && (millis() - elapsedButton) > MIN_ELAPSED_TIME)
  {
    currentBand = (currentBand < lastBand) ? (currentBand + 1) : 0; // Is the last band? If so, go to the first band (AM). Else, next band.
    vfoFreq = band[currentBand].minFreq;
    currentStep = band[currentBand].starStepIndex;

    // Call callback function if exist something to do for the specific  band
    if (band[currentBand].f != NULL)
      (band[currentBand].f)();

    isFreqChanged = true;
    elapsedButton = millis();
  }
```  

## Mobile Application

The reduced length of the Oled display might limit some VFO functionalities.  The goal of this mobile application is improve the VFO control and visualization. 


### Compile and deploy this mobile application

Please copy the project to a local folder and follow the steps bellow

You have to install [Android Studio](http://developer.android.com/sdk/index.html) on your computer to compile and deploy this application on your Android.

You have to install [Xcode](https://developer.apple.com/xcode/) on your Mac computer to compile and deploy this application on your iPhone or iPad.  


### Pair your phone

You have to pair your Android phone with the HM10 Bluetooth Shield.

### Cordova 

This applications was build using the [Apache Cordova](https://cordova.apache.org/docs/en/latest/guide/overview/index.html). 
[Apache Cordova](https://cordova.apache.org/docs/en/latest/guide/overview/index.html) is an open-source tool to develop cross-platform mobile application. Click [here](https://cordova.apache.org/docs/en/latest/guide/overview/index.html) to see more about Apache Cordova.

### Install Cordova

    $ npm install cordova -g


### Install Platform and Plugin

    $ cordova platform add android
    $ cordova platform add ios
    $ cordova plugin add cordova-plugin-bluetooth-serial


### Build and Deploy

Compile and run the application

    $ cordova run android --device



## Vídeos 

- [VFO and BFO project test with an inexpencive radio based on CD2003GP](https://youtu.be/_KgBc6vYWLg)
- [Testing the VFO with a CD2003GM HOMEBREW FM RECEIVER](https://youtu.be/JfgRjDK8LTE)
- [VFO and BFO remote control with iPhone via Bluetooth](https://youtu.be/7gBRUmsrCus)


