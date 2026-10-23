# Water Module

Part of the Smart Greenhouse Controller. An ESP32 measures the water level in a tank with an ultrasonic sensor and switches a pump (an LED stands in for it) through a relay, using two-threshold hysteresis so the pump doesn't chatter. If the sensor reading can't be trusted, the pump is switched off.

water module/Screenshot 2026-10-16 194331-water module.png.png
[Add photo: finished wiring, including the Echo divider and the diode]
[Add screenshot: serial monitor output]

## Goal

- Measure the distance from the sensor down to the water surface.
- Turn the pump on when the water is low and off when the tank is full, and hold its state in between.
- Never run the pump on a bad reading.

## Parts

| Part | Role |
|---|---|
| ESP32 Dev Module (Arduino IDE) | Controller |
| HC-SR04 ultrasonic sensor | Measures distance to the water surface |
| 1k and 2k resistors | Voltage divider that brings the 5V Echo signal down to about 3.3V |
| 1-channel relay module | Switches the pump |
| 1N4007 diode | Lets the ESP32 release the relay properly |
| White LED + 150 ohm resistor | Stand-in pump (load) |

## Design

**More distance means less water.** The sensor looks down at the surface, so the pump turns on when the distance is large and off when it is small.

| Setting | Value |
|---|---|
| Pump on above | 30 cm |
| Pump off below | 15 cm |
| Dead band | 15 to 30 cm (pump keeps its current state) |
| Echo timeout | 5000 microseconds |
| Minimum trusted distance | 2 cm |

**Distance formula.** The Echo pin stays HIGH for the sound's round trip, so distance in cm is the echo time in microseconds divided by 58 (about 2 divided by the speed of sound in cm per microsecond).

**State flag.** A `pumpRunning` variable remembers whether the pump is on. Each branch checks the distance and the flag together, so it only acts when something changes, and the flag is updated on the same line as the pump function it calls.

**Safety case.** A timeout or a reading under 2 cm makes the function return -1, and the loop switches the pump off.

## Wiring

| From | To |
|---|---|
| HC-SR04 VCC | ESP32 VIN (5V) |
| HC-SR04 GND | ESP32 GND |
| HC-SR04 Trig | GPIO 25 |
| HC-SR04 Echo | 1k resistor, then GPIO 35 |
| GPIO 35 point | 2k resistor to GND |
| Relay VCC and GND | VIN and GND |
| Relay IN | Plain end of the 1N4007 |
| 1N4007 striped end | GPIO 14 |
| VIN | Relay COM |
| Relay NO | 150 ohm resistor, then LED long leg |
| LED short leg | GND |
| Relay NC | Not connected |

## Sketch

I wrote this sketch myself, section by section. The relay's on/off method: on means the pin is an output driven LOW, and off means the pin becomes an input so the ESP32 lets go of it.

```cpp
const int ECHO_PIN = 35;
const int TRIG_PIN = 25;
const int RELAY_IN = 14;

const float PUMP_ON_ABOVE_CM = 30.0; 
const float PUMP_OFF_BELOW_CM = 15.0; 

const unsigned long ECHO_TIMEOUT_US = 5000;
const float MINIMUM_DISTANCE_CM = 2.0;

bool pumpRunning = false;

void pumpOn(){
pinMode(RELAY_IN,OUTPUT);
digitalWrite(RELAY_IN,LOW);
}

void pumpOff(){
  pinMode(RELAY_IN,INPUT); 
}

float readDistanceCm(){

digitalWrite(TRIG_PIN,LOW);
delayMicroseconds(2);

digitalWrite(TRIG_PIN,HIGH);
delayMicroseconds(10);
digitalWrite(TRIG_PIN,LOW);

unsigned long t = pulseIn(ECHO_PIN,HIGH,ECHO_TIMEOUT_US);

if (t==0) return -1;

float cm = t/58.0;

if (cm < MINIMUM_DISTANCE_CM) return -1;

return cm;
}


void setup() {
Serial.begin(115200);
pinMode(ECHO_PIN,INPUT);
pinMode(TRIG_PIN,OUTPUT);
pumpOff();
}

void loop() {
 float readDist = readDistanceCm();

 if (readDist == -1){
   pumpOff();
   pumpRunning = false;
   Serial.println("No echo,Pump off");
 }

else if (readDist > PUMP_ON_ABOVE_CM && !pumpRunning)
{
  pumpRunning = true;
pumpOn();
Serial.print(readDist);
 Serial.print("cm, ");
Serial.println("Pump On");
}
else if(readDist < PUMP_OFF_BELOW_CM && pumpRunning){
  pumpRunning = false;
  pumpOff();
  Serial.print(readDist);
  Serial.print("cm, ");
  Serial.println("Pump Off");
}

else {
  Serial.print(readDist);
  Serial.print(" cm, pump ");
 if (pumpRunning){
  Serial.println("ON");
  }else {
    Serial.println("OFF");
  }
 }
 delay(500);
}
```

## Build log: problems and fixes

### 1. GPIO 35 can't send the Trig pulse
GPIO 34, 35, 36 and 39 are input-only. I swapped the planned pins, so Trig is on GPIO 25 and Echo (which only listens) is on GPIO 35.

### 2. Echo is a 5V signal
The ESP32's pins aren't 5V tolerant, so Echo goes through a divider: 5V x 2k / (1k + 2k) is about 3.3V.

### 3. Relay stuck on
The relay stayed on whatever the sketch did. The cause was wiring: the diode's striped end was connected to GND, which held the relay's IN low permanently. Moving the striped end to the GPIO pin fixed it, because then only the ESP32 can pull IN low.

### 4. GPIO 17 wasn't available
My board had no GPIO 17 pin, so I moved the relay to GPIO 14 and changed the constant in the sketch to match.

### 5. LED stayed dark through the resistor
A white LED drops about 3V, so on the 3.3V rail a 150 ohm resistor left almost nothing across it. Powering the load from VIN (5V) leaves about 2V across the resistor, around 13 mA, which lights the LED properly.

### 6. Relay not responding because of a loose jumper wire
The relay worked when I touched GND to its IN pin, but the sketch never switched it. Touching GND to the diode's striped end showed the fault was on that side. The jumper wire to GPIO 14 was loose. Reseating it fixed it.

Method that found it: test the relay by itself, then move back one connection at a time until the signal stops.

### 7. Code bugs I found and fixed
- A constant was named `RELAY_IN` but the helper functions used `RELAY_PIN`.
- The Trig and Echo pin modes were swapped in `setup()`.
- `pumpOff;` was missing its brackets, so it never ran.
- `delay(1)` made the loop far faster than the sensor can measure, so I changed it to 500 ms.
- An `else` was missing the closing brace before it.

## Results

| Test | Result |
|---|---|
| Boot | Relay off, LED dark |
| Surface more than 30 cm away | Pump on, relay clicks, LED lit |
| 15 to 30 cm | State holds, serial prints the distance and the current pump state |
| Closer than 15 cm | Pump off, relay clicks off, LED dark |
| Sensor blocked or disconnected | "No echo", pump off |

The water module works as intended.

## What I learned

- Input-only pins can listen but not drive, so check a pin's capabilities before assigning it.
- A 5V signal needs to be brought down before it reaches a 3.3V ESP32 pin.
- When a working circuit stops working, check the jumper wires and component seating before the code.
- Each branch should change its state variable on the same line as the action it takes.
- A sketch isn't finished until every branch, including the failure case, has been tested.

## Next

Air module: a gas sensor driving a fan, with the same two-threshold pattern. After all three modules work, they get merged into one sketch and reported over MQTT to AWS IoT Core.
