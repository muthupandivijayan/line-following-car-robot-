line following car 
/*
  Line Following Car - Arduino Uno + L298N + 3 IR sensors

  WIRING
  L298N: ENA->5 (PWM), IN1->7, IN2->8, IN3->9, IN4->10, ENB->6 (PWM)
  Sensors: LEFT->2, CENTER->3, RIGHT->4 (VCC 5V, GND GND)
  Common ground between battery, L298N and Arduino.

  If the car turns the wrong way, swap the sensor pins or the motor wires.
  If it reads the line backwards, flip LINE_IS_HIGH.
*/

// ---------- Pins ----------
const int ENA = 5, IN1 = 7, IN2 = 8;
const int ENB = 6, IN3 = 9, IN4 = 10;
const int S_LEFT = 2, S_CENTER = 3, S_RIGHT = 4;

// ---------- Settings (tune these) ----------
const bool LINE_IS_HIGH = true;  // true: sensor outputs HIGH on black line
const int BASE_SPEED = 130;      // straight speed (0-255)
const int TURN_SPEED = 170;      // outer wheel speed in turns
const int SLOW_SPEED = 40;       // inner wheel speed in turns
const int SEARCH_SPEED = 120;    // spin speed when the line is lost
const int MAX_SPEED = 255;

int lastDir = 0;  // -1 line was last on left, +1 on right, 0 center

bool onLine(int pin) {
  bool v = digitalRead(pin);
  return LINE_IS_HIGH ? v : !v;
}

void setMotor(int en, int inA, int inB, int speed) {
  speed = constrain(speed, -MAX_SPEED, MAX_SPEED);
  if (speed >= 0) {
    digitalWrite(inA, HIGH);
    digitalWrite(inB, LOW);
  } else {
    digitalWrite(inA, LOW);
    digitalWrite(inB, HIGH);
  }
  analogWrite(en, abs(speed));
}

void drive(int left, int right) {
  setMotor(ENA, IN1, IN2, left);
  setMotor(ENB, IN3, IN4, right);
}

void setup() {
  pinMode(ENA, OUTPUT); pinMode(ENB, OUTPUT);
  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT);
  pinMode(S_LEFT, INPUT); pinMode(S_CENTER, INPUT); pinMode(S_RIGHT, INPUT);
  drive(0, 0);
  delay(1000);  // time to place the car on the track
}

void loop() {
  bool l = onLine(S_LEFT);
  bool c = onLine(S_CENTER);
  bool r = onLine(S_RIGHT);

  if (c && !l && !r) {            // centered
    drive(BASE_SPEED, BASE_SPEED);
    lastDir = 0;
  } else if (l && c && !r) {      // slight left
    drive(SLOW_SPEED + 40, TURN_SPEED);
    lastDir = -1;
  } else if (r && c && !l) {      // slight right
    drive(TURN_SPEED, SLOW_SPEED + 40);
    lastDir = 1;
  } else if (l && !c && !r) {     // sharp left
    drive(-SLOW_SPEED, TURN_SPEED);
    lastDir = -1;
  } else if (r && !c && !l) {     // sharp right
    drive(TURN_SPEED, -SLOW_SPEED);
    lastDir = 1;
  } else if (l && c && r) {       // intersection / thick line: go straight
    drive(BASE_SPEED, BASE_SPEED);
  } else {                        // line lost: spin toward where it was last seen
    if (lastDir <= 0) drive(-SEARCH_SPEED, SEARCH_SPEED);
    else drive(SEARCH_SPEED, -SEARCH_SPEED);
  }
}