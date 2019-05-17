# VFO and BFO using Arduino with Si5351 controlled by SmartPhone

## __Documentação em Português clique [aqui](https://github.com/pu2clr/VFO_BFO_OLED_ARDUINO/tree/master/Experiments/VFO_RADIO_CD2003GP/Docs).__

By PU2CLR - Ricardo Lima Caratti  - April, 2019

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

The heart of this VFO is the component Si5351. It is an I2C configurable clock generator that is very appropriate for receivers and transceivers projects in amateur radio applications. It has three outputs that you can get three distinct frequencies at the same time. A great feature of the Si5351A is the possibility of using it with a microcontroller or platform like Arduino, PIC family and others. See more on [Silicon Labs documentation](https://www.silabs.com/documents/public/data-sheets/Si5351-B.pdf).

 This project uses two outputs of the Si5351.  The VFO  uses the output 0 (CLK0) and can oscillate from 100KHz to 160MHz. The BFO uses the output 1 (CLK1) of Si5351 and it was configured to oscillate from 452KHz to 458KHz (you can modigy the BFO range to any other from 8KHz to 160MHz).  

 The Arduino sketch was projected to allow the user modify easly the band, range and step configurations.
 See Arduino Sketch below. 


## VFO and BFO interface

The user can control the VFO and BFO by using three buttons (bands, steps and VFO / BFO switch) and an encoder. The user can also connect this VFO with an iOS or Android device and control it remotelly.  The Dial was implemented with the OLED Display 128 x 64 Pixels White 0.96 Inch I2C. 


### Display Layout


| Layout |  Layout |
| ------ |  ------ |
| ![Display photo 1](https://github.com/pu2clr/VFO_SI5351_ARDUINO_SMARTPHONE/blob/master/images/OLED_01.png)| ![Displau photo 2](https://github.com/pu2clr/VFO_SI5351_ARDUINO_SMARTPHONE/blob/master/images/OLED_02.png) |
| ![Display photo 3](https://github.com/pu2clr/VFO_SI5351_ARDUINO_SMARTPHONE/blob/master/images/OLED_03.png)| ![Displau photo 4](https://github.com/pu2clr/VFO_SI5351_ARDUINO_SMARTPHONE/blob/master/images/OLED_04.png) |


### Simple Smartphone Layout

| iPhone Application | Android Application  | 
| ------ | ------- | 
| ![Smartphone Layout - iPhone](https://github.com/pu2clr/VFO_SI5351_ARDUINO_SMARTPHONE/blob/master/images/smartphone_00.png) | ![Smartphone Layout - iPhone](https://github.com/pu2clr/VFO_SI5351_ARDUINO_SMARTPHONE/blob/master/images/smartphone_01.png) | 


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
- __Freq Unit__ - Frequency unit that will be shown on the display for the current band;
- __Divider__ - Divider used to reduce the number of digits in the display;
- __decimals__ - number of decimal places (precision);
- __Initial Step Index__ - Lowest step index used for the band (see Step table)
- __Final Step Index__ - Highest step index used for the band (see Step table) 
- __Last Step Index__ - Default step index or last step index used for the band (see Step table). Useful when the user gets back to a band. 
- __callback function__ - pointer to the function that handles something when the band is selected

The implementation of the band information is shown below.  An array of band structure information is implemmented 

```cpp
// Structure for Bands database
typedef struct
{
  char *name;
  uint64_t minFreq;       // Min. frequency value for the band (unit 0.01Hz)
  uint64_t maxFreq;       // Max. frequency value for the band (unit 0.01Hz)
  uint64_t lastFreq;      // Store the current frequency before change to other band (starts with minFreq value)
  long long offset;
  char *unitFreq;         // MHz or KHz
  float divider;          // value that will be the divider of current clock (just to present on display)
  short decimals;         // number of digits after the comma
  short initialStepIndex; // Index to the initial step of incrementing
  short finalStepIndex;   // Index to the final step of incrementing
  short lastStepIndex;    // Index to the last step used (initial index value same index defult)
  void (*doSmth)(void);   // pointer to the function that will handle specific things for the band immediately after the band is selected
 } Band;
```

The code below implements the band table (array) of the VFO for a hypothetical radio. 
The hypothetical radio used as an example here has 10 bands. Each band has a frequency range, a default frequency, an IF offset, a unit used for the band (MHz or KHz), a divider to convert the unit (from 1/100Hz to KHz or MHz), decimal digits used for the band (precision), minimum step used for the band (index), maximum step used for the band, default step index and a pointer to function that will be executed when the band is selected (or NULL if no action is needed).  The values amBroadcast, fmBroadcast and defultFinishBand are pointers to functions that will execute some specific actions for the band. When the value is NULL, no action will be executed.  You might need change these values depending on your radio design. 

```cpp
// Band database. You can change the band ranges if you need.
// The unit of frequency here is 0.01Hz (1/100 Hz). See Etherkit Library at https://github.com/etherkit/Si5351Arduino
 Band band[] = {
     {"MW  ", 50000000LLU, 170000000LLU, 50000000LLU, 45500000LU, "KHz", 100000.0f, 0, 3, 6, 5, amBroadcast},
     {"SW1 ", 170000000LLU, 1000000000LLU, 170000000LLU, 45500000LU, "KHz", 100000.0f, 2, 1, 6, 3, amBroadcast},
     {"SW2 ", 1000000000LLU, 2000000000LLU, 1000000000LLU, 45500000LU, "KHz", 100000.0f, 2, 1, 6, 3, amBroadcast},
     {"SW3 ", 2000000000LLU, 3000000000LLU, 2000000000LLU, 45500000LU, "KHz", 100000.0f, 2, 1, 6, 3, amBroadcast},
     {"VHF1", 3000000000LLU, 7600000000LLU, 3000000000LLU, 45500000LU, "KHz", 100000.0f, 2, 1, 7, 3, defultFinishBand},
     {"FM  ", 8600000000LLU, 10800000000LLU, 8600000000LLU, 1075000000LLU, "MHz", 100000000.0f, 2, 5, 8, 7, fmBroadcast},
     {"AIR ", 10800000000LLU, 13700000000LLU, 10800000000LLU, 1070000000LLU, "MHz", 100000000.0f, 3, 2, 8, 5, NULL},
     {"VHF2", 13700000000LLU, 14400000000LLU, 13700000000LLU, 1070000000LLU, "MHz", 100000000.0f, 3, 2, 8, 5, defultFinishBand},
     {"2M  ", 14400000000LLU, 15000000000LLU, 14400000000LLU, 1070000000LLU, "MHz", 100000000.0f, 3, 2, 8, 5, defultFinishBand},
     {"VFH3", 15000000000LLU, 16000000000LLU, 15000000000LLU, 1070000000LLU, "MHz", 100000000.0f, 3, 2, 8, 5, defultFinishBand}};
 // Calculate the last element position (index) of the array band
```

### Band Table 

The table of bands implemented above for the hypothetical radio can be seen below. The values of Initial Step Index, Final Step Index and Default/Last Step index can be got from the Step table.  


| Band name | Initial Freq.  | Final Freq. | Last Freq. | offset | Freq Unit | Divider | Dec. | Initial Step Index | Final Step Index | Default / Last Step index | Action Function |
| --------- | ----------------------- | -------------------- | ---------------- | ---------------- | -----------------| --------------- |------------------ | ---------------- | ---------------- | -----------| -----------|  
| MW   | 50000000 | 170000000 | 50000000 | 45500000 |  KHz | 100000 | 2 | 3 | 6 | 5 | amBroadcast |
| SW1  | 170000000 | 1000000000 | 170000000 | 45500000  | KHz | 100000 | 2 | 2 | 6 | 3 | amBroadcast |
| SW2  | 1000000000 | 2000000000 | 1000000000 | 45500000  | KHz | 100000 | 2 | 2 | 6 | 3 | amBroadcast |
| SW3  | 2000000000 | 3000000000 | 2000000000 | 45500000  | KHz | 100000 | 2 | 2 | 6 | 3 | amBroadcast |
| VHF1 | 3000000000 | 7600000000 | 3000000000 | 45500000  | KHz | 100000 | 2 | 2 | 7 | 3 | defultFinishBand |
| FM   | 7600000000 | 10800000000 | 7600000000 | 1075000000  | MHz |  100000000 | 1 | 6 | 8 | 7 | fmBroadcast |
| AIR  | 10800000000 | 13700000000 | 10800000000 | 1075000000  | MHz | 100000000 | 2 | 2 | 8 | 5 | NULL |
| VHF2 | 13700000000 | 14400000000 | 13700000000 | 1075000000  | MHz | 100000000 | 2 | 2 | 8 | 5 | defultFinishBand |
| 2M  | 14400000000 | 15000000000 | 14400000000 | 1075000000  | MHz | 100000000 | 2 | 2 | 8 | 5 | defultFinishBand |
| VFH3 | 15000000000 | 16000000000 | 15000000000 | 1075000000  | MHz| 100000000 | 2 | 2 | 8| 5 | defultFinishBand |



### Step Table 

Each band uses a subset of the table of steps. This will depend on the characteristics of the band. For example, usually, 5KHz and 10 KHz are more apropriated to MW band. For SW bands, you might want 1KHz, 5KHz and 10KHz. The step table is implemented by the code below.  


```cpp
// Struct for step database
typedef struct
{
  char *name; // step label: 50Hz, 100Hz, 500Hz, 1KHz, 5KHz, 10KHz and 500KHz
  long value; // Frequency value (unit 0.01Hz See documentation) to increment or decrement
} Step;
```

The code beloow implements an array of steps (Steps)

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

The table of steps implemented above  can be seen below.


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
For example, you might need to turn a LED on, start a relay or set up a filter when you select a particular band.  If the device that will use this VFO/BFO has different behaviours for different band, you might want to use callback functions.  See Band table for more information. 


The code below shows the callback functions declaration

```cpp
// Callback functions declarations
// Depending on the selected band, you may want to perform specific actions
// The functions declared below will do something for AM and FM BAND. See implementation later.
// You can implement callback function for other bands
void amBroadcast();      // See implementation later.
void fmBroadcast();      // See implementation later.
void defultFinishBand(); // 
```

The callback functions above are referenced in band database (see table of bands -  array band[]) above.


```cpp
// Callback implementation

// Doing something spefict for MW band
void amBroadcast()
{
  digitalWrite(AM_LED, HIGH);      // Turn ON the AM LED
  // DO SOMETHING ELSE
}

// Doing something spefict for FM
// Example: set Pin 14 of the CD2003GP to HIGH; turn FM LED on etc
void fmBroadcast()
{
  digitalWrite(FM_LED, HIGH);       // Turn ON the FM LED
  // DO SOMETHING ELSE  
}

// Defaul action 
// It is another callback function that can be called when a specific band is selected 
void defultFinishBand()
{
  digitalWrite(FM_LED, LOW); // Turno the FM LED OFF
  digitalWrite(AM_LED, LOW); // Turno the AM LED OFF
  // DO SOMETHING ELSE  
}

```


The code below shows the use of callback function when the use changes the band

```cpp
// It is executed when a new band is selected.
void changeBand(short idxBand)
{
  // Save current status of the current band
  band[currentBand].lastStepIndex = currentStep;
  band[currentBand].lastFreq = vfoFreq;

  // Now change the current band  
  currentBand = idxBand;

  // Get back last information stored for this band.
  vfoFreq = band[idxBand].lastFreq;           
  currentStep = band[idxBand].lastStepIndex;

  // Call callback function if exist something to do for this band (current band)
  if (band[idxBand].doSmth != NULL)
    (band[idxBand].doSmth)();

  isFreqChanged = true;
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


