# VFO and BFO project test with an inexpencive radio based on CD2003GP 

## __Documentação em Português clique [aqui](https://github.com/pu2clr/VFO_BFO_OLED_ARDUINO/tree/master/Experiments/VFO_RADIO_CD2003GP/Docs).__

By PU2CLR - Ricardo
April, 2019

## Table of contents

1. [Introduction]()
2. [Display layout]()
3. [Schematic used with Arduino Atmega328 (UNO, Pro Mini, Nano etc)]()
4. [Arduino Sketch]()
5. [Step Table]()
6. [Source folder]()
   1. [Arduino sketch for Atmega328 - si5351_vfoCD2003GP_atmega328.ino]()
   2. [Arduino sketch for Atmega32U4 - si5351_vfoCD2003GP_atmega32u4.ino]() 
   3. [Arduino sketch Atmega 328 and Bluetooth BLE-HM10 - si5351_vfo_ble_atmega328.ino]()
   4. [Remote control Mobile Application for Android and iOS]()
7. [Videos]()


## Introduction

This project implements a VFO and BFO  using an Arduino with Si5351 signal generator. You can controll the VFO using an encoder and also a SmartPhone via a mobile application developed here for this purpose. 


### Display Layout


| Layout |  Layout |
| ------ |  ------ |
| ![Displau photo 1](https://github.com/pu2clr/VFO_SI5351_ARDUINO_SMARTPHONE/blob/master/images/OLED_01.png)| ![Displau photo 2](https://github.com/pu2clr/VFO_SI5351_ARDUINO_SMARTPHONE/blob/master/images/OLED_02.png) |
| ![Displau photo 3](https://github.com/pu2clr/VFO_SI5351_ARDUINO_SMARTPHONE/blob/master/images/OLED_03.png)| ![Displau photo 4](https://github.com/pu2clr/VFO_SI5351_ARDUINO_SMARTPHONE/blob/master/images/OLED_04.png) |


## Schematic - Arduino Atmega328 (UNO, Pro Mini, Nano etc)

![Schematic used with Arduino Atmega328](https://github.com/pu2clr/VFO_SI5351_ARDUINO_SMARTPHONE/blob/master/schematic/vfobfo_schematic_atmega328.png)




## Arduino Sketch 

Some original features of the VFO and BFO project was modified to adapt it better to the radio based on CD2003GP. The main modification was the band behaviour. The VFO implemented on Arduino Sketch has the follow information:  

- __Band name__ - Band name that will show on display;
- __Initial Freq.__ -  Lowest band frequency (1/100 Hz);
- __Final Freq.__ - highest frequency band (1/100 Hz);
- __offset__ - Shows on the display the frequency of the station and makes the signal generator to oscillate considering the IF (1/100 Hz);
- __Freq Unit__ - Frequency unit that will be show on the display for the current band;
- __Divider__ - Divider used to reduce the number of digits in the display;
- __decimals__ - number of decimal places after the comma;
- __Initial Step Index__ - Lowest step index used for the band (see Step table)
- __Final Step Index__ - Highest step index used for the band (see Step table) 
- __Start Step Index__ - Default step index used for the band (see Step table)
- __callback function__ - point to the function that handles specific things for the band

The implementation of the band information

```cpp
// Structure for Bands database
typedef struct
{
  char *name;
  uint64_t minFreq; // Min. frequency value for the band (unit 0.01Hz)
  uint64_t maxFreq; // Max. frequency value for the band (unit 0.01Hz)
  long long offset;
  char *unitFreq;         // MHz or KHz
  float divider;          // value that will be the divider of current clock (just to present on display)
  short decimals;         // number of digits after the comma
  short initialStepIndex; // Index to the initial step of incrementing
  short finalStepIndex;   // Index to the final step of incrementing
  short starStepIndex;    // Index to start step of incrementing
  void (*f)(void);        // pointer to the function that handles specific things for the band
} Band;
```



### Band Table 

| Band name | Initial Freq.  | Final Freq. | offset | Freq Unit | Divider | Dec. | Initial Step Index | Final Step Index | Default |
| --------- | ----------------------- | -------------------- | ---------------- | -----------------| --------------- |------------------ | ---------------- | ---------------- | -----------|  
| MW   | 50000000 | 170000000 | 45500000 |  KHz | 100000 | 2 | 3 | 6 | 5 |
| SW1  | 170000000 | 1000000000 | 45500000  | KHz | 100000 | 2 | 2 | 6 | 3 |
| SW2  | 1000000000 | 2000000000 | 45500000  | KHz | 100000 | 2 | 2 | 6 | 3 |
| SW3  | 2000000000 | 3000000000 | 45500000  | KHz | 100000 | 2 | 2 | 6 | 3 |
| VHF1 | 3000000000 | 7600000000 | 45500000  | KHz | 100000 | 2 | 2 | 7 | 3 |
| FM   | 7600000000 | 10800000000 | 1075000000  | MHz |  100000000 | 1 | 6 | 8 | 7 |
| AIR  | 10800000000 | 13700000000 | 1075000000  | MHz | 100000000 | 2 | 2 | 8 | 5 |
| VHF2 | 13700000000 | 14400000000 | 1075000000  | MHz | 100000000 | 2 | 2 | 8 | 5 |
| 2M  | 14400000000 | 15000000000 | 1075000000  | MHz | 100000000 | 2 | 2 | 8 | 5 |
| VFH3 | 15000000000 | 16000000000 | 1075000000  | MHz| 100000000 | 2 | 2 | 8| 5 |


The code below implements the band table of the VFO for the radio used here. 
The values amBroadcast, fmBroadcast and NULL are function that execute some specific actions for the band. When the value is NULL, no action will be executed.  

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



## Callback functions

This sketch uses callback functions to complement specific actions for a specific band.
For example, you will need to set the PIN 14 of the CD2003GP to HIGH if you want to work with FM.
In contrast, you need to set the PIN 14 to LOW if you want to work with AM.

The code below shows the callback functions declaration

```cpp
// Callback functions declarations
// Depending on the selected band, you may want to perform specific actions
// The functions declared below will do something for AM and FM BAND. See implementation later.
// You can implement callback function for other bands
void amBroadcast(); // See implementation later.
void fmBroadcast(); // See implementation later.
```

The callback functions above are referenced in band database (see Band band[]) above.


```cpp
// Callback implementation

// Doing something spefict for AM
// Example: set Pin 14 of the CD2003GP to LOW; switch filter, turn AM LED on etc
void amBroadcast()
{
  // TO DO
  STATUSLED(HIGH); // Just testing if it is working - Turn LED ON
}

// Doing something spefict for FM
// Example: set Pin 14 of the CD2003GP to HIGH; turn FM LED on etc
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
    currentBand = (currentBand < lastBand) ? (currentBand + 1) : 0; // Is the last band? If so, go to the first band (AM). Else. Else, next band.
    vfoFreq = band[currentBand].minFreq;
    currentStep = band[currentBand].starStepIndex;

    // Call callback function if exist something to do for the specific  band
    if (band[currentBand].f != NULL)
      (band[currentBand].f)();

    isFreqChanged = true;
    elapsedButton = millis();
  }
```  



## Vídeos 

- [VFO and BFO project test with an inexpencive radio based on CD2003GP](https://youtu.be/_KgBc6vYWLg)
- [Testing the VFO with a CD2003GM HOMEBREW FM RECEIVER](https://youtu.be/JfgRjDK8LTE)
- [VFO and BFO remote control with iPhone via Bluetooth](https://youtu.be/7gBRUmsrCus)


