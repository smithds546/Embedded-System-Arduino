# C++ Smart-Home-Device Code



```cpp
#include <Wire.h>
#include <Adafruit_RGBLCDShield.h>
#include <utility/Adafruit_MCP23017.h>

Adafruit_RGBLCDShield lcd = Adafruit_RGBLCDShield();

// Define variables for storing device information
const int MAX_DEVICES = 10; // maximum number of devices that can be stored in the array
char devices[MAX_DEVICES][30]; // array of devices, each device can be up to 30 characters long
int num_devices = 0; // variable to keep track of the number of devices currently stored
char status1[MAX_DEVICES][7]; // array to store the status of each device
char power1[MAX_DEVICES][7]; // array to store the power of each device
String the_devices[5] = {"PZW", "FDG", "LHT", "BDA", "HJL"}; // list of valid device types
char d_type[3]; // array to store the device type extracted from the user input
bool present = false; // flag to check if the device type is present in the list of valid types
boolean sync = true; // flag to keep track of whether the LCD display is in sync with the device information in memory
int currentIndex = 0; // variable to keep track of which device is currently selected on the LCD display
char device[30]; // array to store the name of the currently selected device
char symbol1 = ' '; // character to represent the up arrow on the LCD display
char symbol2 = ' '; // character to represent the down arrow on the LCD display

// Function to extract the device type from the user input
void Device_type(char input1[]) {
    d_type[0] = input1[2];
    d_type[1] = input1[3];
    d_type[2] = input1[4];
    present = false; // reset present flag for each call
    for (int i = 0; i < 5; i++) {
        if (strcmp(the_devices[i].c_str(), d_type) == 0) { // check if the current device type matches one in the list
            present = true;
            break;
        }
    }
}

// Function to check the amount of free memory
int freeRam() {
    extern int __heap_start, *__brkval;
    int free_memory;
    if ((int)__brkval == 0)
        free_memory = ((int)&free_memory) - ((int)&__heap_start);
    else
        free_memory = ((int)&free_memory) - ((int)__brkval);
    return free_memory;
}

// Define constants for LCD custom characters
byte downArrow[8] = {
    0b00100,
    0b01110,
    0b11111,
    0b00100,
    0b00100,
    0b00100,
    0b00100,
    0b00000
};

byte upArrow[8] = {
    0b00000,
    0b00100,
    0b00100,
    0b00100,
    0b00100,
    0b11111,
    0b01110,
    0b00100
};

// Define constants for FSM states
enum States {
    IDLE,
    TAKING_INPUT,
    STUDENT_ID,
};

States currentState = IDLE;

/* This function adds a new device to the system. The input parameter is a
   string that contains the device ID. The function checks if the ID is valid
   and not already added to the system. If the ID is valid and there is space
   to add a new device, the function adds the device to the system and
   updates the LCD display. */
void adding(char input1[]) {
    Device_type(input1);
    if (input1[1] != '-' || input1[7] != '-' || 
        (input1[6] != 'S' && input1[6] != 'L' && input1[6] != 'O' &&
         input1[6] != 'T' && input1[6] != 'C') || input1[8] == '\0' || 
        present == false) {
        if (num_devices == 0) {
            lcd.setBacklight(7);
            lcd.clear();
            Serial.println("Error, not a valid device");
        } else if (num_devices > 0) {
            selectDevice(0);
            Serial.println("Error, not a valid device");
        }
        return;
    } else if (num_devices < MAX_DEVICES) { // make sure we don't exceed the maximum number of devices
        // Add device to the devices array
        strcpy(devices[num_devices], input1); // copy the input1 string to the next available slot in the devices array
        memset(devices[num_devices] + strlen(input1), '\0', 22 - strlen(input1)); // fill the remaining characters with null terminators
        // Set device status in the status1 array
        status1[num_devices][0] = input1[2]; // assign the first character of the input1 string to the first element of the status1 array
        status1[num_devices][1] = input1[3]; // assign the second character of the input1 string to the second element of the status1 array
        status1[num_devices][2] = input1[4]; // assign the third character of the input1 string to the third element of the status1 array
        status1[num_devices][3] = '-';
        status1[num_devices][4] = 'O';
        status1[num_devices][5] = 'F';
        status1[num_devices][6] = 'F';
        lcd.setBacklight(3);
        power1[num_devices][0] = input1[2];
        power1[num_devices][1] = input1[3];
        power1[num_devices][2] = input1[4];
        power1[num_devices][3] = '-';
        power1[num_devices][4] = '0';
        power1[num_devices][5] = '0';
        power1[num_devices][6] = '0';

        num_devices++;

        Serial.print("Device added: ");
        Serial.println(input1);
        Serial.println(num_devices);
        printDevices();
        lcdprintdevice(input1);
    } else if (num_devices == MAX_DEVICES) {
        Serial.println("Error: Too many devices");
    }
    currentState = TAKING_INPUT;
}

/* This function updates the power value of a device. The input parameter is
   a string that contains the device ID and the new power value. The function
   searches for the device with the given ID and updates its power value if
   the ID is valid and the new power value is within the allowed range. The
   function also checks if the device is a temperature sensor and the new value
   is within the allowed temperature range. The function updates the LCD display
   with the new power value if the update is successful. */
void power(char input1[]) {
    bool found = false;
    if (strlen(input1) <= 9) {
        for (int i = 0; i < num_devices; i++) {
            if (power1[i][0] == input1[2] && power1[i][1] == input1[3] && power1[i][2] == input1[4]) {
                found = true;
                int powerValue = atoi(&input1[6]); // extract the power value from the input
                if (input1[2] == 'B' && input1[3] == 'D' && (powerValue < 9 || powerValue > 32)) {
                    lcd.clear();
                    Serial.print("Error, outside temperature range");
                    break;
                } else if (input1[2] == 'H' && input1[3] == 'J') {
                    lcd.clear();
                    Serial.println("Error: No feature for device");
                    break;
                } else {
                    sprintf(power1[i], "%c%c%c-%03d", input1[2], input1[3], input1[4], powerValue); // store the power value in the power1 array
                    lcdprintdevice(devices[currentIndex]);
                    break;
                }
            }
        }
        if (!found) {
            Serial.println("Device not found");
        }
    } else {
        Serial.println("Error: ");
        Serial.println(input1);
        Serial.println("Too long");
    }
}

/* This function takes a device ID as input and changes its status to either
   "ON" or "OFF" based on the command received. The status is updated in the
   status1 array and the change is reflected on the LCD screen. */
void status(char input1[]) {
    bool found = false;
    if (input1[6] == 'O') {
        for (int i = 0; i < num_devices; i++) {
            if (status1[i][0] == input1[2] && status1[i][1] == input1[3] && status1[i][2] == input1[4]) {
                found = true;
                char searchValue[] = {input1[2], input1[3], input1[4]};
                if (input1[7] == 'N') {
                    status1[i][4] = ' ';
                    status1[i][5] = 'O';
                    status1[i][6] = 'N';
                } else {
                    status1[i][4] = 'O';
                    status1[i][5] = 'F';
                    status1[i][6] = 'F';
                }
                lcdprintdevice(devices[currentIndex]);
                break;
            }
        }
        if (!found) {
            Serial.println("Device not found");
        }
    }
}

/* This function prints the current status of all devices stored in the system.
   The function iterates over the devices array and prints the device ID, 
   status, and power to the serial monitor. */
void printDevices() {
    Serial.println("Devices:");
    for (int i = 0; i < num_devices; i++) {
        Serial.print("Device ID: ");
        Serial.print(devices[i]);
        Serial.print(", Status: ");
        Serial.print(status1[i]);
        Serial.print(", Power: ");
        Serial.println(power1[i]);
    }
}

/* This function displays the currently selected device on the LCD screen.
   The display includes the device ID, its status, and the power level. */
void lcdprintdevice(char input1[]) {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print(input1);
    lcd.setCursor(0, 1);
    lcd.print(status1[currentIndex]);
    lcd.print("  "); // Clear the rest of the line
    lcd.setCursor(10, 1);
    lcd.print(power1[currentIndex]);
}

void setup() {
    Serial.begin(9600); // Start serial communication
    lcd.begin(16, 2); // Initialize the LCD
    lcd.setBacklight(7); // Set backlight color
    lcd.createChar(0, downArrow); // Create custom character for down arrow
    lcd.createChar(1, upArrow); // Create custom character for up arrow
    lcd.clear(); // Clear the display
    Serial.println("System Initialized");
}

void loop() {
    // Check for serial input
    if (Serial.available() > 0) {
        char input1[30]; // Buffer to store incoming data
        int len = Serial.readBytesUntil('\n', input1, sizeof(input1) - 1);
        input1[len] = '\0'; // Null-terminate the string

        // Handle different commands based on the input format
        if (len > 0) {
            if (input1[0] == 'A') { // Adding a device
                adding(input1);
            } else if (input1[0] == 'P') { // Power command
                power(input1);
            } else if (input1[0] == 'S') { // Status command
                status(input1);
            } else if (input1[0] == 'D') { // Display device
                selectDevice(input1[1] - '0'); // Assuming input1[1] is the index of the device
            } else {
                Serial.println("Unknown command");
            }
        }
    }

    // Handle LCD navigation
    if (sync) {
        lcdprintdevice(devices[currentIndex]); // Refresh LCD display if needed
    }

    // Example of navigating devices with buttons (if using physical buttons)
    // Adjust based on your specific button setup
    if (digitalRead(UP_BUTTON_PIN) == HIGH) {
        currentIndex = (currentIndex > 0) ? currentIndex - 1 : num_devices - 1;
        sync = true;
        delay(200); // Simple debounce
    }
    if (digitalRead(DOWN_BUTTON_PIN) == HIGH) {
        currentIndex = (currentIndex < num_devices - 1) ? currentIndex + 1 : 0;
        sync = true;
        delay(200); // Simple debounce
    }

    // Check for free memory
    Serial.print("Free RAM: ");
    Serial.println(freeRam());
}

/* Function to select a device based on index */
void selectDevice(int index) {
    if (index >= 0 && index < num_devices) {
        currentIndex = index; // Update current index
        lcdprintdevice(devices[currentIndex]); // Display selected device info
    }
}

