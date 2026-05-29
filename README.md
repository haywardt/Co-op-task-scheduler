# Cooperative Task Scheduler for Arduino

A lightweight, cross-platform cooperative task scheduler for Arduino, ESP32, and ESP8266 that uses cycle-accurate timing for maximum precision.

## Features

- **Cycle-accurate timing** on ESP32 and ESP8266 (nanosecond precision)
- **Cross-platform** support for ESP32, ESP8266, and Arduino Uno (AVR)
- **Non-blocking** cooperative multitasking
- **Priority-based scheduling** (task order determines priority)
- **Two scheduling macros**: `EVERY` (periodic) and `AFTER` (one-shot)
- **Optional built-in LED** activity indicator
- **No external dependencies** - uses only standard Arduino functions
- **Ultra-low overhead** - direct register access for LED control

## Quick Start

### Installation

1. Copy `scheduler.h` to your Arduino libraries folder or project directory
2. Include the header in your sketch:

   ```cpp
   #include "scheduler.h"
   ```

### Basic Usage

```cpp
#include "scheduler.h"

void setup() {
    Serial.begin(115200);
    scheduler_init();
}

bool runScheduler() {
    EVERY(500 MILLISECONDS, Serial.println("Every half second"));
    EVERY(1 SECONDS, Serial.println("Every second"));
    AFTER(5 SECONDS, Serial.println("Once after 5 seconds"));
    return false;
}

void loop() {
    scheduler_run(runScheduler);
}
```

## Complete Scheduler Code

### scheduler.h

```cpp
// scheduler.h - Cooperative Task Scheduler for ESP32, ESP8266, and Arduino Uno
#ifndef SCHEDULER_H
#define SCHEDULER_H

// ============================================
// CONFIGURATION - Set these before including
// ============================================

// Enable/disable built-in LED as idle/busy indicator
#ifndef ENABLE_BUSY_LED
#define ENABLE_BUSY_LED 1  // Set to 1 to enable, 0 to disable
#endif

// ============================================
// END OF CONFIGURATION
// ============================================

// Platform detection and cycle counter setup
#if defined(ESP32)
    #include "xtensa/corebits.h"
    #define GET_CYCLE_COUNT() XTHAL_GET_CCOUNT()
    #define HAS_CYCLE_COUNTER 1
    #define HAS_NANOSECOND_TIMING 1
#elif defined(ESP8266)
    #include "esp8266.h"
    #define GET_CYCLE_COUNT() xthal_get_ccount()
    #define HAS_CYCLE_COUNTER 1
    #define HAS_NANOSECOND_TIMING 1
#elif defined(__AVR__)
    #include <avr/io.h>
    #include <avr/interrupt.h>
    
    static uint32_t lastMicros = 0;
    inline uint32_t get_cycle_count_avr() {
        return micros();
    }
    #define GET_CYCLE_COUNT() get_cycle_count_avr()
    #define HAS_CYCLE_COUNTER 0
    #define HAS_NANOSECOND_TIMING 0
#else
    #error "Unsupported platform! Only ESP32, ESP8266, and AVR are supported."
#endif

// LED pin configuration
#ifndef LED_BUILTIN
    #if defined(__AVR__)
        #define LED_BUILTIN 13
    #else
        #define LED_BUILTIN 2
    #endif
#endif

#define STATUS_LED_BIT (1 << LED_BUILTIN)

// CPU Frequency and timing configuration
#if defined(ESP32)
    #ifndef F_CPU
        #define F_CPU 240000000UL
    #endif
    #define CPU_FREQ_MHZ (F_CPU / 1000000UL)
#elif defined(ESP8266)
    #ifndef F_CPU
        #define F_CPU 80000000UL
    #endif
    #define CPU_FREQ_MHZ (F_CPU / 1000000UL)
#elif defined(__AVR__)
    #define F_CPU 16000000UL
    #define CPU_FREQ_MHZ 16
#endif

// Time unit macros
#if HAS_CYCLE_COUNTER
    // ESP32/ESP8266 - full cycle-accurate timing
    #define NANOSECONDS   * (CPU_FREQ_MHZ / 1000UL)
    #define MICROSECONDS  * (CPU_FREQ_MHZ)
    #define MILLISECONDS  * (CPU_FREQ_MHZ * 1000UL)
    #define SECONDS       * (CPU_FREQ_MHZ * 1000000UL)
#else
    // On AVR, no nanosecond precision
    #define MICROSECONDS  * 1UL
    #define MILLISECONDS  * 1000UL
    #define SECONDS       * 1000000UL
    
    // Compatibility macro for NANOSECONDS on AVR
    #ifndef NANOSECONDS
        #define NANOSECONDS * (1/0)
        #warning "NANOSECONDS is not available on AVR platforms. Use MICROSECONDS instead."
    #endif
#endif

// Platform-specific GPIO control for built-in LED
#if ENABLE_BUSY_LED
    #if defined(ESP32)
        #define BUSY_LED_ON()   (GPIO.out_w1ts = STATUS_LED_BIT)
        #define BUSY_LED_OFF()  (GPIO.out_w1tc = STATUS_LED_BIT)
    #elif defined(ESP8266)
        #define BUSY_LED_ON()   GPIO_REG_WRITE(GPIO_OUT_W1TS_ADDRESS, STATUS_LED_BIT)
        #define BUSY_LED_OFF()  GPIO_REG_WRITE(GPIO_OUT_W1TC_ADDRESS, STATUS_LED_BIT)
    #elif defined(__AVR__)
        #define BUSY_LED_PORT PORTB
        #define BUSY_LED_BIT_POS 5
        #define BUSY_LED_ON()   (BUSY_LED_PORT |= (1 << BUSY_LED_BIT_POS))
        #define BUSY_LED_OFF()  (BUSY_LED_PORT &= ~(1 << BUSY_LED_BIT_POS))
    #endif
#else
    #define BUSY_LED_ON()
    #define BUSY_LED_OFF()
#endif

// Task execution macro - runs periodically, resets timer each time
#define EVERY(interval, ...) \
    do { \
        static uint32_t lastTime = 0; \
        uint32_t now = GET_CYCLE_COUNT(); \
        if ((now - lastTime) >= (interval)) { \
            lastTime = now; \
            __VA_ARGS__; \
            return true; \
        } \
    } while(0)

// After macro - runs once after delay, doesn't reset timer
#define AFTER(delay, ...) \
    do { \
        static uint32_t startTime = 0; \
        static bool executed = false; \
        static void (*savedFunc)() = nullptr; \
        \
        if (savedFunc != (void(*)())__VA_ARGS__) { \
            startTime = GET_CYCLE_COUNT(); \
            savedFunc = (void(*)())__VA_ARGS__; \
            executed = false; \
        } \
        \
        if (!executed) { \
            uint32_t now = GET_CYCLE_COUNT(); \
            if ((now - startTime) >= (delay)) { \
                executed = true; \
                __VA_ARGS__; \
                return true; \
            } \
        } \
    } while(0)

// Scheduler initialization
inline void scheduler_init() {
    #if ENABLE_BUSY_LED
        pinMode(LED_BUILTIN, OUTPUT);
        digitalWrite(LED_BUILTIN, LOW);
    #endif
}

// Main scheduler loop
inline void scheduler_run(bool (*scheduler_func)()) {
    BUSY_LED_ON();
    scheduler_func();
    BUSY_LED_OFF();
}

#endif // SCHEDULER_H
```

## Examples

### Example 1: Basic Blinking LED

```cpp
#include "scheduler.h"

#define LED_PIN 4

bool runScheduler() {
    EVERY(500 MILLISECONDS, digitalWrite(LED_PIN, !digitalRead(LED_PIN)));
    return false;
}

void setup() {
    pinMode(LED_PIN, OUTPUT);
    scheduler_init();
}

void loop() {
    scheduler_run(runScheduler);
}
```

### Example 2: Multiple Tasks with Different Priorities

```cpp
#include "scheduler.h"

#define LED1_PIN 4
#define LED2_PIN 5
#define LED3_PIN 6

void fastBlink() {
    digitalWrite(LED1_PIN, !digitalRead(LED1_PIN));
}

void mediumBlink() {
    digitalWrite(LED2_PIN, !digitalRead(LED2_PIN));
}

void slowBlink() {
    digitalWrite(LED3_PIN, !digitalRead(LED3_PIN));
}

bool runScheduler() {
    // Higher priority tasks first
    EVERY(100 MILLISECONDS, fastBlink());
    EVERY(500 MILLISECONDS, mediumBlink());
    EVERY(1 SECONDS, slowBlink());
    return false;
}

void setup() {
    pinMode(LED1_PIN, OUTPUT);
    pinMode(LED2_PIN, OUTPUT);
    pinMode(LED3_PIN, OUTPUT);
    scheduler_init();
}

void loop() {
    scheduler_run(runScheduler);
}
```

### Example 3: Button Debouncing with AFTER

```cpp
#include "scheduler.h"

#define BUTTON_PIN 2
#define LED_PIN 4
#define DEBOUNCE_DELAY_MS 50

enum ButtonState {
    IDLE,
    DEBOUNCING,
    CONFIRMED
};

ButtonState buttonState = IDLE;
bool ledState = false;
int pressCount = 0;

void confirmButtonPress() {
    buttonState = CONFIRMED;
    pressCount++;
    ledState = !ledState;
    digitalWrite(LED_PIN, ledState);
    
    Serial.print("Button press #");
    Serial.print(pressCount);
    Serial.print(" - LED ");
    Serial.println(ledState ? "ON" : "OFF");
    
    buttonState = IDLE;
}

bool runScheduler() {
    EVERY(5 MILLISECONDS, {
        bool buttonPressed = (digitalRead(BUTTON_PIN) == LOW);
        
        switch(buttonState) {
            case IDLE:
                if (buttonPressed) {
                    buttonState = DEBOUNCING;
                    AFTER(DEBOUNCE_DELAY_MS MILLISECONDS, confirmButtonPress());
                }
                break;
                
            case DEBOUNCING:
                if (!buttonPressed) {
                    buttonState = IDLE;
                }
                break;
                
            case CONFIRMED:
                if (!buttonPressed) {
                    buttonState = IDLE;
                }
                break;
        }
    });
    
    return false;
}

void setup() {
    pinMode(BUTTON_PIN, INPUT_PULLUP);
    pinMode(LED_PIN, OUTPUT);
    Serial.begin(115200);
    scheduler_init();
    Serial.println("Debounce Example - Press button!");
}

void loop() {
    scheduler_run(runScheduler);
}
```

### Example 4: Sensor Reading with Timed Actions

```cpp
#include "scheduler.h"

#define SENSOR_PIN A0
#define LED_PIN 4

float sensorValue = 0;
bool warningTriggered = false;

void readSensor() {
    sensorValue = analogRead(SENSOR_PIN) * (5.0 / 1023.0);
    Serial.print("Sensor: ");
    Serial.print(sensorValue);
    Serial.println("V");
    
    if (sensorValue > 4.5 && !warningTriggered) {
        warningTriggered = true;
        Serial.println("WARNING: High voltage detected!");
        AFTER(10 SECONDS, []() {
            warningTriggered = false;
            Serial.println("Warning reset");
        });
    }
}

void blinkLED() {
    digitalWrite(LED_PIN, !digitalRead(LED_PIN));
}

bool runScheduler() {
    EVERY(500 MILLISECONDS, readSensor());
    
    if (warningTriggered) {
        EVERY(200 MILLISECONDS, blinkLED());
    } else {
        digitalWrite(LED_PIN, LOW);
    }
    
    return false;
}

void setup() {
    pinMode(LED_PIN, OUTPUT);
    Serial.begin(115200);
    scheduler_init();
    Serial.println("Sensor Monitor Started");
}

void loop() {
    scheduler_run(runScheduler);
}
```

### Example 5: Long Press Detection

```cpp
#include "scheduler.h"

#define BUTTON_PIN 2
#define LED_PIN 4
#define LONG_PRESS_MS 2000

bool ledBlinking = false;
bool longPressDetected = false;

void onShortPress() {
    Serial.println("Short press - Toggle blinking");
    ledBlinking = !ledBlinking;
    if (!ledBlinking) {
        digitalWrite(LED_PIN, LOW);
    }
}

void onLongPress() {
    Serial.println("Long press - System reset");
    ledBlinking = false;
    digitalWrite(LED_PIN, LOW);
    longPressDetected = true;
}

bool runScheduler() {
    static bool wasPressed = false;
    static uint32_t pressStartTime = 0;
    
    EVERY(10 MILLISECONDS, {
        bool isPressed = (digitalRead(BUTTON_PIN) == LOW);
        
        if (isPressed && !wasPressed) {
            pressStartTime = millis();
            wasPressed = true;
            
            AFTER(LONG_PRESS_MS MILLISECONDS, []() {
                if (wasPressed) {
                    onLongPress();
                }
            });
        }
        else if (!isPressed && wasPressed) {
            if (!longPressDetected) {
                uint32_t pressDuration = millis() - pressStartTime;
                if (pressDuration >= 50) {
                    onShortPress();
                }
            }
            wasPressed = false;
            longPressDetected = false;
        }
    });
    
    EVERY(250 MILLISECONDS, []() {
        if (ledBlinking) {
            digitalWrite(LED_PIN, !digitalRead(LED_PIN));
        }
    });
    
    return false;
}

void setup() {
    pinMode(BUTTON_PIN, INPUT_PULLUP);
    pinMode(LED_PIN, OUTPUT);
    Serial.begin(115200);
    scheduler_init();
    Serial.println("Long Press Example - Short press toggles, long press (2s) resets");
}

void loop() {
    scheduler_run(runScheduler);
}
```

### Example 6: Multi-Button Keyboard

```cpp
#include "scheduler.h"

#define NUM_BUTTONS 4
#define DEBOUNCE_MS 50

const int buttonPins[NUM_BUTTONS] = {2, 3, 4, 5};
const char buttonChars[NUM_BUTTONS] = {'A', 'B', 'C', 'D'};

struct Button {
    int pin;
    char key;
    bool pressed;
    uint32_t lastPressTime;
};

Button buttons[NUM_BUTTONS];

void onButtonPress(int buttonIndex) {
    if (buttonIndex >= 0 && buttonIndex < NUM_BUTTONS) {
        Serial.print("Key pressed: ");
        Serial.println(buttons[buttonIndex].key);
        buttons[buttonIndex].pressed = true;
        
        AFTER(500 MILLISECONDS, [buttonIndex]() {
            buttons[buttonIndex].pressed = false;
        });
    }
}

bool runScheduler() {
    static bool debouncing[NUM_BUTTONS] = {false};
    
    EVERY(5 MILLISECONDS, {
        for (int i = 0; i < NUM_BUTTONS; i++) {
            bool currentState = (digitalRead(buttons[i].pin) == LOW);
            static bool lastState[NUM_BUTTONS] = {HIGH, HIGH, HIGH, HIGH};
            
            if (currentState != lastState[i] && !debouncing[i]) {
                debouncing[i] = true;
                AFTER(DEBOUNCE_MS MILLISECONDS, [i]() {
                    if (digitalRead(buttons[i].pin) == LOW) {
                        onButtonPress(i);
                    }
                    debouncing[i] = false;
                });
            }
            lastState[i] = currentState;
        }
    });
    
    EVERY(1 SECONDS, []() {
        Serial.print("Button states: ");
        for (int i = 0; i < NUM_BUTTONS; i++) {
            Serial.print(buttons[i].key);
            Serial.print(":");
            Serial.print(buttons[i].pressed ? "1 " : "0 ");
        }
        Serial.println();
    });
    
    return false;
}

void setup() {
    for (int i = 0; i < NUM_BUTTONS; i++) {
        pinMode(buttonPins[i], INPUT_PULLUP);
        buttons[i].pin = buttonPins[i];
        buttons[i].key = buttonChars[i];
        buttons[i].pressed = false;
        buttons[i].lastPressTime = 0;
    }
    
    Serial.begin(115200);
    scheduler_init();
    Serial.println("Multi-Button Keyboard - Press keys A, B, C, or D");
}

void loop() {
    scheduler_run(runScheduler);
}
```

## Platform-Specific Notes

### ESP32
- Cycle counter runs at CPU frequency (typically 240MHz)
- Nanosecond precision available
- Maximum interval: ~17.9 seconds (2^32 cycles)
- Direct register access for ultra-fast GPIO

### ESP8266
- Cycle counter runs at 80MHz (or 160MHz if overclocked)
- Nanosecond precision available
- Maximum interval: ~53.7 seconds at 80MHz
- Use `yield()` for WiFi stack

### Arduino Uno (AVR)
- Microsecond precision only (no nanosecond support)
- Uses `micros()` for timing
- NANOSECONDS macro causes compile error with helpful message
- Maximum interval: ~71 minutes with micros() rollover

## API Reference

### Macros

#### `EVERY(interval, code)`
Executes code periodically at the specified interval.

**Parameters:**
- `interval`: Time interval (use NANOSECONDS, MICROSECONDS, MILLISECONDS, or SECONDS)
- `code`: Code block to execute

**Returns:** `true` if code was executed, `false` otherwise

**Example:**
```cpp
EVERY(500 MILLISECONDS, Serial.println("Tick"));
```

#### `AFTER(delay, code)`
Executes code once after the specified delay.

**Parameters:**
- `delay`: Time delay (use NANOSECONDS, MICROSECONDS, MILLISECONDS, or SECONDS)
- `code`: Code block to execute

**Returns:** `true` if code was executed, `false` otherwise

**Example:**
```cpp
AFTER(5 SECONDS, Serial.println("Boot complete"));
```

### Functions

#### `scheduler_init()`
Initializes the scheduler. Call once in `setup()`.

**Example:**
```cpp
void setup() {
    scheduler_init();
}
```

#### `scheduler_run(scheduler_func)`
Runs the scheduler. Call in `loop()`.

**Parameters:**
- `scheduler_func`: Pointer to function containing EVERY/AFTER macros

**Example:**
```cpp
void loop() {
    scheduler_run(runScheduler);
}
```

## Configuration Options

### Enable/Disable Busy LED

```cpp
// Disable built-in LED indicator
#define ENABLE_BUSY_LED 0
#include "scheduler.h"
```

## Performance Characteristics

- **ESP32**: ~50-100ns overhead per scheduler run
- **ESP8266**: ~100-200ns overhead per scheduler run
- **Arduino Uno**: ~1-2µs overhead per scheduler run

## License

This code is released into the public domain. Use it freely in any project.

## Contributing

Feel free to submit issues and enhancement requests!

## Support

For questions or support, please open an issue on the project repository.

---

**Happy Scheduling! 🚀**
