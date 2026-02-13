// -----------------------------------------------------
// Arduino Morse Input with Calibration
// -----------------------------------------------------

// Pins
const int buttonPin = 3;   // dot / dash
const int resetPin  = 7;   // ^r  (also starts calibration)
const int spacePin  = 9;   // /
const int backPin   = 5;   // -
const int redPin    = 13;
const int greenPin  = 12;
const int bluePin   = 11;

// Button states
int buttonState = HIGH;
int prevButtonState = HIGH;

int resetState = HIGH;
int prevResetState = HIGH;

int spaceState = HIGH;
int prevSpaceState = HIGH;

int backState = HIGH;
int prevBackState = HIGH;

// Timing
unsigned long startTime;
unsigned long duration;

// Default durations BEFORE calibration
unsigned long short_duration = 10;   
unsigned long long_duration  = 150;   

// Calibration state
bool calibrating = false;
int calibration_step = 0;      // 0 = waiting for dot; 1 = waiting for dash
unsigned long cal_start = 0;

// -----------------------------------------------------
// Calibration Functions
// -----------------------------------------------------

void startCalibration() {
  calibrating = true;
  calibration_step = 0;

  Serial.println("^r");
  Serial.println("CALIBRATION STARTED");

  digitalWrite(redPin, HIGH);
  digitalWrite(greenPin, HIGH);
  delay(200);
  digitalWrite(redPin, LOW);
  digitalWrite(greenPin, LOW);
}

void processCalibration(int buttonState, int prevButtonState) {
  if (!calibrating) return;

  // Press → start timing
  if (buttonState == LOW && prevButtonState == HIGH) {
    cal_start = millis();
    tone(bluePin, 600);
    digitalWrite(bluePin, HIGH);
  }

  // Release → determine short or long
  if (buttonState == HIGH && prevButtonState == LOW) {
    noTone(bluePin);
    digitalWrite(bluePin, LOW);

    unsigned long dur = millis() - cal_start;

    if (calibration_step == 0) {
      short_duration = dur;
      Serial.print("DOT calibrated = ");
      Serial.println(short_duration);

      calibration_step = 1;

      digitalWrite(greenPin, HIGH);
      delay(150);
      digitalWrite(greenPin, LOW);
    }
    else if (calibration_step == 1) {
      long_duration = dur;
      Serial.print("DASH calibrated = ");
      Serial.println(long_duration);

      Serial.println("CALIBRATION COMPLETE");

      digitalWrite(redPin, HIGH);
      delay(200);
      digitalWrite(redPin, LOW);

      calibrating = false;
      calibration_step = 0;
    }
  }
}

// -----------------------------------------------------
// Setup
// -----------------------------------------------------

void setup() {
  pinMode(buttonPin, INPUT_PULLUP);
  pinMode(resetPin, INPUT_PULLUP);
  pinMode(spacePin, INPUT_PULLUP);
  pinMode(backPin, INPUT_PULLUP);

  pinMode(redPin, OUTPUT);
  pinMode(greenPin, OUTPUT);
  pinMode(bluePin, OUTPUT);

  Serial.begin(9600);
}

// -----------------------------------------------------
// Main Loop
// -----------------------------------------------------

void loop() {

  // Read buttons
  buttonState = digitalRead(buttonPin);
  resetState  = digitalRead(resetPin);
  spaceState  = digitalRead(spacePin);
  backState   = digitalRead(backPin);

  // -------- RESET BUTTON = Start Calibration --------
  if (resetState == LOW && prevResetState == HIGH) {
    startCalibration();
    delay(250);
  }

  // -------- If calibrating, only do calibration --------
  if (calibrating) {
    processCalibration(buttonState, prevButtonState);

    // Update states and exit loop early
    prevButtonState = buttonState;
    prevResetState  = resetState;
    prevSpaceState  = spaceState;
    prevBackState   = backState;
    delay(1);
    return;
  }

  // -----------------------------------------------------
  // DOT / DASH INPUT (normal mode)
  // -----------------------------------------------------
  if (buttonState == LOW && prevButtonState == HIGH) {
    startTime = millis();
    tone(bluePin, 500);
    delay(10);
    digitalWrite(bluePin, HIGH);
  }

  if (buttonState == HIGH && prevButtonState == LOW) {
    noTone(bluePin);
    digitalWrite(bluePin, LOW);

    duration = millis() - startTime;

    // Debounce threshold
    if (duration > 30) {
      if ((duration > 11) && (duration < long_duration)) {
        Serial.println("dot");
      }
      else if (duration > long_duration) {
        Serial.println("dash");
      }
      else {
        Serial.println("NA");
      }
    }

    delay(250);
  }

  // -----------------------------------------------------
  // SPACE BUTTON
  // -----------------------------------------------------
  if (spaceState == LOW && prevSpaceState == HIGH) {
    Serial.println("/");
    digitalWrite(greenPin, HIGH);
    delay(120);
    digitalWrite(greenPin, LOW);
    delay(60);
  }

  // -----------------------------------------------------
  // BACKSPACE BUTTON
  // -----------------------------------------------------
  if (backState == LOW && prevBackState == HIGH) {
    Serial.println("-");
    digitalWrite(redPin, HIGH);
    delay(120);
    digitalWrite(redPin, LOW);
    delay(60);
  }

  // -----------------------------------------------------
  // Update previous states
  // -----------------------------------------------------
  prevButtonState = buttonState;
  prevResetState  = resetState;
  prevSpaceState  = spaceState;
  prevBackState   = backState;

  delay(1);
}
