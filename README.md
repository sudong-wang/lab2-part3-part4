### TODO:

Create a 'sequencer' that allows you to record BOOT button presses and play them on the Neopixel, and also play a sequence of read/write commands. You should be able to:
- record at a least a few seconds of button input to your RP2040 (in RAM)
- replay a recorded sequence on your NeoPixel
- loop a recording
- save a recording to your laptop (the Python Serial library is one way to do this)
- play a recording from your laptop
- record 'macros' (a sequence of console commands) based on keystrokes in your serial console
- hand-edit a list of register read/write commands on your laptop, and play them on the RP2040
- include multiple I/O sources in a recording, and remap among the following:
    - inputs: BOOT button, console commands, register read/write commands
    - outputs: neopixel color, neopixel brightness, data over serial, register read/write commands
    
### Code:

Before working on Python Serial, we firstly try to save the recording pattern as an array in C. After the program starts, it will ask user to press the button in 10s, the program will save and replay the pattern.

```
#include <stdio.h>
#include <stdlib.h>
#include "pico/stdlib.h"
#include "ws2812.h"

#include "pico/stdlib.h"
#include "hardware/pio.h"
#include "hardware/clocks.h"
#include "ws2812.pio.h"

#define IS_RGBW true
#define NUM_PIXELS 150

#ifdef PICO_DEFAULT_WS2812_PIN
#define WS2812_PIN PICO_DEFAULT_WS2812_PIN
#else

// default to pin 2 if the board doesn't have a default WS2812 pin defined

#define WS2812_PIN 12
#define WS2812_POWER_PIN 11
#define MAX 100
#endif

int main() {
    stdio_init_all();

    PIO pio = pio0;
    int sm = 0;
    uint offset = pio_add_program(pio, &ws2812_program);
    turn_on_power();
    ws2812_program_init(pio, sm, offset, WS2812_PIN, 800000, IS_RGBW);

    const uint BOOT_PIN = 21;
    // the BOOT button is GPIO21
    gpio_init(BOOT_PIN);
    gpio_set_dir(BOOT_PIN, GPIO_IN);

    int array[MAX]={0};
    int i;

    // sleep_ms(10000);
    while(stdio_usb_connected()!=true);
    printf("Start the recording after red light:\n");
    set_neopixel_color(0X00FF0000);
    sleep_ms(1000);
    for (i = 0; i < 100; i++){
        if(!gpio_get(BOOT_PIN)) {
            array[i]=1;
            set_neopixel_color(0X0000FF00);
        } 
        else {
            array[i]=0;
            set_neopixel_color(0x00000000);
        }
        sleep_ms(100);
    }
        
    printf("Finish the recording.\n");
    set_neopixel_color(0X00000000);
    sleep_ms(100);
    printf("Now play the recording:\n");

    for (i = 0; i < 100; i++)
        printf("%d ", array[i]);
    printf("\n");
    
    while(true){
        float speed = 1;
        printf("Enter the speed you want to speed up: \n");
        scanf("%f", &speed);
        printf("The speed now is x%f\n", speed);
        
        for (i = 0; i < 100; i++){
            if(array[i] == 1) {
                set_neopixel_color(0X000000FF);
                sleep_ms(100/speed);
            } else {
                set_neopixel_color(0x00000000);
                sleep_ms(100/speed);
            }
        }
    sleep_ms(1000);
    set_neopixel_color(0x00000000);
    }
}
```

Then we try to use Python Serial to interact with C. **The .c and .py files are in the attachment.**

Once the C code is run on RP2040 and Python is run on PC, on the console it will ask the user to either start recording the BOOT button pressing for 5 seconds, or replay the sequence in the speed chosen by the user. Python is in charge of literal input.

The basic logic is:
- ```qtpy = serial.Serial('COM5', 9600)``` to access PY2040 port
- ```qtpy.write(b'...')``` to choose record or replay mode: b'r' for record and b'p' for replay 

So we can choose the modes:

**R: record a blinking pattern**

we can record the pattern in a .txt file on PC and replay that pattern anytime.

- ```qtpy.readline()``` to access the print out value from 2040 port then write them in .txt file


**P: replay a recorded sequence on your NeoPixel**

In the replay mode, we can:
1）choose normal speed/speed up/speed down pattern
2) if choosing speedup or speed down, we can select a range of output rates
3) choose the number of time we want to loop the pattern
4) we can access .txt file and modify the pattern via keyboard

- ```file_to_list(path,data_out)``` to convert the .txt file to the list
- choose the speed mode and rate
- choose the time for loop
- ```qtpy.write(b'1')``` or ```qtpy.write(b'0')``` as input to C

### Result

The demo is shown in the [youtube video](https://youtu.be/-8h8KXTmqto).

The .txt file with recorded pattern is shown in the attachment.


