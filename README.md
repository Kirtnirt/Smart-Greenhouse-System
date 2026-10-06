# Smart Greenhouse Controller: Light Module

An ESP32 module that reads ambient light with an LDR and switches a lamp through a relay, using two-threshold hysteresis so the relay doesn't chatter. This is the first of three modules (Light, Water, Air).

## Goal

Read light level, turn the lamp on when it gets dark, turn it off when it gets bright, and hold its state in between. Later, report readings over MQTT to AWS IoT Core.

## Parts

| Part | Role |
|---|---|
| ESP32 Dev Module (Arduino IDE) | Controller |
| LDR + fixed resistor | Voltage divider for light sensing |
| 1-channel relay module | Switches the lamp |
| LED + resistor | Stand-in lamp (load) |
| 1N4007 diode | Lets the ESP32 release the relay properly |

[Add photo: breadboard build]
[Add screenshot: Proteus schematic]

## Design

**Schematic first.** I drew the control side in Proteus (LDR + resistor divider into the ESP32, relay module on GPIO 26) and the load side (12V source, relay contacts, resistor, LED, ammeters). The simulation ran, and the voltmeter across the divider resistor read about 2.65V at 100 lux on the 3.3V rail.

**Hysteresis.** Hysteresis means separate turn-on and turn-off thresholds with a dead band between them. In the dead band the output keeps its current state. This prevents relay chatter, wear, and message flooding.

| Setting | Value |
|---|---|
| Lamp turns on below | 50 |
| Lamp turns off above | 400 |
| Dead band | 50 to 400 (room light, about 150, sits inside it) |

The code keeps the state in a `lampOn` flag. Each `if` branch is only allowed to act in one state, so only one threshold is active at a time.

## Build log: problems and fixes

### 1. No usable LDR input
The LDR I had was a 3-pin comparator module with a digital output (DO). I needed the raw analog value, so I built my own LDR + resistor divider on a breadboard.

### 2. ADC pins reading 0
GPIO 34 and 35 both read 0, even with a jumper to 3V3. GPIO 32 read 4095 on the same test. That showed the sketch and the 3V3 rail were fine, so I moved the divider output to GPIO 32.

### 3. Divider readings that made no sense
Room light and covered both read 0, and the flashlight gave random values. Checks with the pin touched to the LDR's top leg (4095) and the resistor's bottom leg (0) proved the wiring was right. The fault was the fixed resistor: I had misread its colour bands, so my assumed value was wrong. A 100k swap then gave odd readings too.

The lesson: the fixed resistor should be roughly the same size as the LDR's resistance in the light range you care about, so the node swings through the middle of the ADC range. With no multimeter available, I tried resistors one at a time and read covered, room, and flashlight values on serial until they separated clearly.

| Condition | Reading |
|---|---|
| Covered | 0 |
| Room light | about 150 |
| Phone flashlight | 2000+ |

### 4. Relay wouldn't switch the load
The relay module only clicked after I moved its VCC to the ESP32's VIN pin (5V). The LED on the COM/NO contacts then lit when switched.

### 5. Relay stuck on
With the sketch running, the relay stayed on whatever the light did. Pulling the IN wire out made it click off, which proved the ESP32 pin was holding it on.

My explanation: the module is powered from 5V, and its IN input is an optocoupler LED pulled toward 5V. When the ESP32 writes HIGH it only reaches 3.3V, leaving about 1.7V across that LED. That is above the roughly 1.2V such an LED typically needs, so the relay never released. (The 1.2V figure is typical, not measured on my module.)

Fix, in two steps:
1. Change what "off" means. Relay on is `pinMode(OUTPUT)` plus `digitalWrite(LOW)`. Relay off is `pinMode(INPUT)`, which lets go of the pin.
2. The relay's green light then dimmed instead of turning fully off, so something was still leaking current. Adding a **1N4007 diode in series on the IN wire** (stripe end toward GPIO 26) blocked it. The likely cause is leakage through the ESP32 pin's protection diodes, but I did not confirm that directly.

### 6. Lamp stayed lit after the relay clicked off
The load was wired to the wrong terminals. I rewired it as:

| From | To |
|---|---|
| ESP32 3V3 | Relay COM |
| Relay NO | One leg of the resistor |
| Other resistor leg | LED anode (long leg) |
| LED cathode (short leg) | ESP32 GND |
| Relay NC | Left empty |

NC is connected to COM when the relay is off, so anything on NC lights at the wrong time.

## Final code

```cpp
const int LDR_PIN = 32;
const int RELAY_PIN = 26;
const int ON_BELOW = 50;
const int OFF_ABOVE = 400;
bool lampOn = false;

void relayOn() {
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW);
}

void relayOff() {
  pinMode(RELAY_PIN, INPUT);
}

void setup() {
  Serial.begin(115200);
  relayOff();
}

void loop() {
  int light = analogRead(LDR_PIN);

  if (!lampOn && light < ON_BELOW) {
    lampOn = true;
    relayOn();
  } else if (lampOn && light > OFF_ABOVE) {
    lampOn = false;
    relayOff();
  }

  Serial.print(light);
  Serial.print(" ");
  Serial.println(lampOn);
  delay(200);
}
```

The relay is only touched when the state changes, not every loop.

## Result

| Test | Outcome |
|---|---|
| Boot | Relay off, lamp dark |
| Cover the LDR (reads 0) | Relay clicks on, lamp lights |
| Back to room light (about 150) | Lamp stays on (dead band) |
| Flashlight (2000+) | Relay clicks off, lamp goes dark |
| Back to room light | Lamp stays off |

The light module works as intended.

## What I learned

- A voltage divider only works well when the fixed resistor matches the sensor's range.
- A 3.3V GPIO can't fully switch off a relay module that idles at 5V. "Off" has to mean releasing the pin, and a series diode helped.
- Check polarity and terminals (COM, NO, NC) before every power-up.
- Isolate faults with simple tests: touching a pin to 3V3 or GND, or pulling one wire out, told me more than guessing.

## Next

1. Report the light value and lamp state over MQTT to AWS IoT Core.
2. Water module: ultrasonic tank level driving a pump.
3. Air module: gas sensor driving ventilation.
