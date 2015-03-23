/*
  Arduino clock on a standard 4-digit clock display
  Uses a Liteon LTC-617D1G clock display 
 
 Connections:
 LTC - Arduino
 1 - nc
 2 - nc
 3 - nc
 4 - d7
 5 - d3
 6 - d2
 7 - d11
 8 - d10
 9 - d4
 10 - gnd
 22 - d9
 23 - d5
 24 - d6
 25 - A0  // because d13 already has a built-in LED getting in the way
 26 - d8
 27 - d12
 28 - A1
 29 - gnd
 
 button:
 5v - button - A5 - 10k resistor - gnd
 
 crazy wires:
 5v - red jumper wire - A4 - 10k resistor - gnd
 5v - blue jumper wire - A3 - 10k resistor - gnd
 5v - yellow jumper wire - A2 - 10k resistor - gnd

 Action:
 pin D13 is already hooked up to an LED
 
*/

#define DIGIT1 2
#define DIGIT2 3
#define DIGIT3 5
#define DIGIT4 6

#define SEGMENTA 7
#define SEGMENTB 8
#define SEGMENTC 9
#define SEGMENTD 10
#define SEGMENTE 11
#define SEGMENTF 12
#define SEGMENTG A0

#define COLON 4
#define AMPM A1

#define BUTTON A5

#define STOPWIRE  A2
#define SPEEDWIRE A3
#define ZEROWIRE  A4

#define ACTION 13

#define ON  HIGH
#define OFF LOW

#define DELAYTIME 50
int x = 0;
unsigned short hours, minutes, seconds;
unsigned long lastTime; // keeps track of when the previous second happened

int buttonState;             // the current reading from the button pin
int lastButtonState = LOW;   // the previous reading from the button pin
unsigned long button_down_start = 0; // how long the button was held down
unsigned long lastDebounceTime = 0;  // the last time the output pin was toggled
unsigned long debounceDelay = 50;    // the debounce time
byte flash;    // indicates when display should be flashing
byte flash_on; // indicates that display is current in "on" part of a flash
byte timer_stopped; // indicates that the timer is not counting down

#define ONE_SECOND 1000
#define FLASH_TIME 100 // 10 times as fast
unsigned long time_chunk;
unsigned long interval=2000;  // the time we need to wait
unsigned long previousMillis=0; // millis() returns an unsigned long.
 

void setup() {
  
  Serial.begin(9600);
  
  // initialize all the required pins as output.
  pinMode(DIGIT1, OUTPUT);
  pinMode(DIGIT2, OUTPUT);
  pinMode(DIGIT3, OUTPUT);
  pinMode(DIGIT4, OUTPUT);

  pinMode(SEGMENTA, OUTPUT);
  pinMode(SEGMENTB, OUTPUT);
  pinMode(SEGMENTC, OUTPUT);
  pinMode(SEGMENTD, OUTPUT);
  pinMode(SEGMENTE, OUTPUT);
  pinMode(SEGMENTF, OUTPUT);
  pinMode(SEGMENTG, OUTPUT);

  pinMode(COLON, OUTPUT);
  pinMode(AMPM, OUTPUT);
  
  // button is input
  pinMode(BUTTON, INPUT);
  
  // wires are inputs
  pinMode(STOPWIRE, INPUT);
  pinMode(SPEEDWIRE, INPUT);
  pinMode(ZEROWIRE, INPUT);
  
  // the action is output
  pinMode(ACTION, OUTPUT);
  
  // set the initial time
  hours = 0;
  minutes = 0;
  seconds = 10;

  flash = 0;
  flash_on = 0;
  timer_stopped = 0;

  time_chunk = ONE_SECOND;

  lastTime = millis();
}

void loop() {
  
  // Keep showing the display while waiting for timer to expire  
  while (millis() - lastTime < time_chunk) {
        
    if (!flash || flash_on) {
      
      if (hours > 0) {
        clock_show_time(hours, minutes);
    
        // and blink the colon every even second
        if (seconds % 2 == 0) {
          clock_show_colon();
        }
      }
      else {
        clock_show_time(minutes, seconds);
        clock_show_colon(); // show a steady colon
      }
      
    }
    
    // check the crazy wires
    
    if (digitalRead(STOPWIRE) == LOW) {  // stops time
      timer_stopped = true;
    }
    else {
      timer_stopped = false;
    }
    
    if (digitalRead(SPEEDWIRE) == LOW) { // speeds up the time and flashes display
      flash = 1;
      time_chunk = FLASH_TIME;
    }
    
    if (digitalRead(ZEROWIRE) == LOW) {  // sets time to zero
      hours = 0;
      minutes = 0;
      seconds = 0;
      time_chunk = FLASH_TIME;
    }
    
    // button presses increase minutes
    int reading = digitalRead(BUTTON);
 
     // If the switch changed, due to noise or pressing:
    if (reading != lastButtonState) {
      // reset the debouncing timer
      lastDebounceTime = millis();
    }
    
    if ((millis() - lastDebounceTime) > debounceDelay) {
      // whatever the reading is at, it's been there for longer
      // than the debounce delay, so take it as the actual current state:
      
      if (buttonState != reading) {
        button_down_start = millis(); // record the start of the current button state
      }
      
      buttonState = reading;
      
      // buttonState is now either on or off
      if (buttonState == HIGH) {
        x=0;
        flash = 0; // takes it out of panic mode
        digitalWrite(ACTION, OFF); // turns the action OFF.
        time_chunk = ONE_SECOND; // reset to regular time counting.
          
        // slow it down by only doing this every 10th millisecond
        if ((millis() % 10) == 0) {
          // if the button was held down more than 5 seconds, make it go faster
          if ((millis() - button_down_start) > 5000) {
            seconds += 10;
            if (seconds > 59) seconds = 59;
          }
          
          // button has been pressed
          incrementTime();
        }
      }
    } 
 
    lastButtonState = reading;
  }

  lastTime += time_chunk;
  
  if (!timer_stopped) {
    decrementTime();
  }
  
  if (flash) {
    flash_on = !flash_on;
  }

}

//
// a call to decrementTime decreases time by one second.
//
void decrementTime() {

      if (seconds == 0) {
      
        if (minutes == 0) {
        
          if (hours == 0) {          
            // time is at 00:00, flash the zeroes
            flash = 1;
            time_chunk = FLASH_TIME;
              if (x <=1){    
            // and do the action
            do_action();
           }
          }
          else {
            minutes = 59;
            hours--;
          }
        }
        else {
          seconds = 59;
          minutes--;
        }
        
      }
      else {
        seconds--;  
      }
}

//
// a call to incrementTime increases time by one second.
//
void incrementTime() {
  
  if (seconds == 59) {
    seconds = 0;
    
    if (minutes == 59) {
      minutes = 0;
      
      if (hours == 12) {          
        hours = 1;
      }
      else {
        hours++;
      }
    }
    else {
      minutes++;
    }
  }
  else {
    seconds++;  
  }
}

//
// clock_show_time - displays the given time on the clock display
//   Note that instead of hr/min the user can also send min/sec
//   Maximum hr is 99, Maximum min is 59, and minimum is 0 for both (it's unsigned, heh).
//
void clock_show_time(unsigned short hours, unsigned short minutes) {
  unsigned short i;
  unsigned short delaytime;
  unsigned short num_leds[10] = { 6, 2, 5, 5, 4, 5, 6, 3, 7, 6 };
  unsigned short digit[4];
  unsigned short hide_leading_hours_digit;
    
  // convert minutes and seconds into the individual digits
  // check the boundaries
  if (hours > 99) hours = 99;
  if (minutes > 59) minutes = 59;
  
  // convert hr
  if (hours < 10 && hours > 0) {
    hide_leading_hours_digit = 1;
  }
  else {
    hide_leading_hours_digit = 0;
  }
  
  digit[0] = hours / 10;
  digit[1] = hours % 10; // remainder
  digit[2] = minutes / 10;
  digit[3] = minutes % 10; // remainder  

  for (i = hide_leading_hours_digit; i < 4; i++) {
    clock_all_off();
    clock_show_digit(i, digit[i]);

    // fewer leds = brighter display, so delay depends on number of leds lit.
    delaytime = num_leds[digit[i]] * DELAYTIME;   
    delayMicroseconds(delaytime);
  }
    
  clock_all_off();
}

//
// clock_all_off - turns off all the LEDs on the clock to give a blank display
//
void clock_all_off(void) {
  
  // digits must be ON for any LEDs to be on
  digitalWrite(DIGIT1, OFF);
  digitalWrite(DIGIT2, OFF);
  digitalWrite(DIGIT3, OFF);
  digitalWrite(DIGIT4, OFF);
  
  // segments must be ON for any LEDs to be on
  digitalWrite(SEGMENTA, OFF);
  digitalWrite(SEGMENTB, OFF);
  digitalWrite(SEGMENTC, OFF);
  digitalWrite(SEGMENTD, OFF);
  digitalWrite(SEGMENTE, OFF);
  digitalWrite(SEGMENTF, OFF);
  digitalWrite(SEGMENTG, OFF);
  
  // turn off colon and alarm too
  digitalWrite(COLON, OFF);
  digitalWrite(AMPM, OFF);
}

//
// clock_show_digit - turns on the LEDs for the digit in the given position
//      position can be from 0 through 3: 0 and 1 being the hour, 2 and 3 being the seconds
//      value can be from 0 through 9, ie, a valid single digit.
//
//      (if value is out of range, it displays a 9. if digit is out of range display remains blank)
//
void clock_show_digit(unsigned short position, unsigned short value) {
  byte a;
  byte b;
  byte c;
  byte d;
  byte e;
  byte f;
  byte g;

  switch (position) {
    case 0:
      digitalWrite(DIGIT1, ON);
      break;
    case 1:
      digitalWrite(DIGIT2, ON);
      break;
    case 2:
      digitalWrite(DIGIT3, ON);
      break;
    case 3:
      digitalWrite(DIGIT4, ON);
      break;
  }

  a = !(value == 1 || value == 4);
  b = !(value == 5 || value == 6);
  c = !(value == 2);
  d = !(value == 1 || value == 4 || value == 7);
  e =  (value == 0 || value == 2 || value == 6 || value == 8);
  f = !(value == 1 || value == 2 || value == 3 || value == 7);
  g = !(value == 0 || value == 1 || value == 7);
  
  if (a) digitalWrite(SEGMENTA, ON);
  if (b) digitalWrite(SEGMENTB, ON);
  if (c) digitalWrite(SEGMENTC, ON);
  if (d) digitalWrite(SEGMENTD, ON);
  if (e) digitalWrite(SEGMENTE, ON);
  if (f) digitalWrite(SEGMENTF, ON);
  if (g) digitalWrite(SEGMENTG, ON);
}

//
// clock_show_colon - shows the colon that separates minutes from seconds
//
void clock_show_colon(void) {
  unsigned short delaytime;

  digitalWrite(COLON, ON);  
                              // 2 leds = 2 delays needed
  delaytime = DELAYTIME * 2;  // must use variable to have similar delay to rest of clock
  delayMicroseconds(delaytime);   //   because use of variable slows it down slightly.
  digitalWrite(COLON, OFF);
}

//
// clock_show_alarm - shows the ampm dot (bottom right of clock display)
//
void clock_show_ampm(void) {
  unsigned short delaytime;

  digitalWrite(AMPM, ON);
                      
  delaytime = DELAYTIME;  // must use variable to have similar delay to rest of clock
  delayMicroseconds(delaytime);   //   because use of variable slows it down slightly.
  digitalWrite(AMPM, OFF);
}

//
// do_action - this function gets called when the timer completes.
//
 static void do_action(void) {
  // the exciting action here is just to turn on a LED
  
  digitalWrite(ACTION,HIGH);
  if ((unsigned long)(millis() - previousMillis) >= interval) {
  previousMillis = millis();
  digitalWrite(ACTION, LOW);  
  x++;
 }
    
  Serial.println("ACTION!");
}

