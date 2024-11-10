#include <WiFi.h>
#include <ESPAsyncWebServer.h>

// WiFi credentials
const char* ssid = "Rover";
const char* password = "123456789";

// Pin definitions for L298N motor driver
const int motor1In1Pin = 26;
const int motor1In2Pin = 25;
const int motor2In1Pin = 33;
const int motor2In2Pin = 32;

// Ultrasonic sensor pin definitions
const int trigPin = 13;
const int echoPin = 12;

// HTML page content
const char* htmlPage = R"HTML(
<!DOCTYPE html>
<html>
<head>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <style>
    .arrows {
      font-size: 70px;
      color: black;
      position: center;
    }
    .stopButton {
      font-size: 25px;
      position: center;
      color: black;
    }
    .button {
      width: 100px;
      height: 100px;
      font-size: 20px;
      margin: 10px;
      border-radius: 25%;
    }
    .distance {
      color: lightgreen;
      font-size: 30px;
      text-align: center;
      margin-bottom: 10px;
    }
    .obstacle {
      color: red;
      font-size: 34px;
      text-align: center;
      margin-top: 20px;
    }
  </style>
</head>
<body class="noselect" align="center" style="background-color:black">
  <h1 style="color: white; text-align: center;">Control Your Car!!</h1>
  <p class="distance">Distance: <span id="distance">-</span></p>
  <p class="obstacle" id="obstacleText" style="display: none;">Obstacle Detected</p>
  
  <button class="button" onclick="move('forward')"><span class="arrows">&#8679;</span></button>
  <br>
  <button class="button" onclick="move('left')"><span class="arrows">&#8678;</span></button>
  <button class="button" onclick="move('stop')"><span class="stopButton">STOP</span></button>
  <button class="button" onclick="move('right')"><span class="arrows">&#8680;</span></button>
  <br>
  <button class="button" onclick="move('backward')"><span class="arrows">&#8681;</span></button>
  <p style="color: white; text-align: center; font-size: 30px;">IOT Project by :</p>
  <p style="color: white; text-align: center; font-size: 30px;">Yogeshwar.jb (Reg. No. 21BEC0521)</p>
  <p style="color: white; text-align: center; font-size: 30px;">S K Thinagaran (Reg. No. 21BEC2136)</p>
<p style="color: white; text-align: center; font-size: 30px;">Nithish Kumar C(Reg. No. 21BEC2155)</p>


  <script>
    function move(direction) {
      var xhttp = new XMLHttpRequest();
      xhttp.open("GET", "/move?dir=" + direction, true);
      xhttp.send();
    }

    function updateDistance(distance) {
      var distanceElement = document.getElementById("distance");
      var obstacleTextElement = document.getElementById("obstacleText");

      distanceElement.textContent = distance + " cm";

      if (distance < 10) {
        obstacleTextElement.style.display = "block";
      } else {
        obstacleTextElement.style.display = "none";
      }
    }
    
    setInterval(function() {
      var xhttp = new XMLHttpRequest();
      xhttp.onreadystatechange = function() {
        if (this.readyState == 4 && this.status == 200) {
          var distance = parseInt(this.responseText);
          updateDistance(distance);
        }
      };
      xhttp.open("GET", "/getDistance", true);
      xhttp.send();
    }, 1000);
  </script>
</body>
</html>
)HTML";

AsyncWebServer server(80);

void setup() {
  Serial.begin(115200); // Start UART communication

  WiFi.softAP(ssid, password);
  IPAddress IP = WiFi.softAPIP();
  Serial.print("AP IP address: ");
  Serial.println(IP);  // Log AP IP address

  // Configure motor and sensor pins as output/input
  pinMode(motor1In1Pin, OUTPUT);
  pinMode(motor1In2Pin, OUTPUT);
  pinMode(motor2In1Pin, OUTPUT);
  pinMode(motor2In2Pin, OUTPUT);
  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);

  // Logging the status of the setup
  Serial.println("Pins configured and server starting...");

  server.on("/", HTTP_GET, [](AsyncWebServerRequest *request){
    request->send(200, "text/html", htmlPage);
  });

  server.on("/move", HTTP_GET, [](AsyncWebServerRequest *request){
    String direction = request->getParam("dir")->value();

    if (direction == "forward") {
      digitalWrite(motor1In1Pin, HIGH);
      digitalWrite(motor1In2Pin, LOW);
      digitalWrite(motor2In1Pin, HIGH);
      digitalWrite(motor2In2Pin, LOW);
      Serial.println("Moving forward");  // Log movement
    } else if (direction == "backward") {
      digitalWrite(motor1In1Pin, LOW);
      digitalWrite(motor1In2Pin, HIGH);
      digitalWrite(motor2In1Pin, LOW);
      digitalWrite(motor2In2Pin, HIGH);
      Serial.println("Moving backward");
    } else if (direction == "left") {
      digitalWrite(motor1In1Pin, LOW);
      digitalWrite(motor1In2Pin, HIGH);
      digitalWrite(motor2In1Pin, HIGH);
      digitalWrite(motor2In2Pin, LOW);
      Serial.println("Turning left");
    } else if (direction == "right") {
      digitalWrite(motor1In1Pin, HIGH);
      digitalWrite(motor1In2Pin, LOW);
      digitalWrite(motor2In1Pin, LOW);
      digitalWrite(motor2In2Pin, HIGH);
      Serial.println("Turning right");
    } else if (direction == "stop") {
      digitalWrite(motor1In1Pin, LOW);
      digitalWrite(motor1In2Pin, LOW);
      digitalWrite(motor2In1Pin, LOW);
      digitalWrite(motor2In2Pin, LOW);
      Serial.println("Stop");
    }

    request->send(200, "text/plain", "OK");
  });

  server.on("/getDistance", HTTP_GET, [](AsyncWebServerRequest *request){
    long duration, distance;
    digitalWrite(trigPin, LOW);
    delayMicroseconds(2);
    digitalWrite(trigPin, HIGH);
    delayMicroseconds(10);
    digitalWrite(trigPin, LOW);
    duration = pulseIn(echoPin, HIGH);
    distance = duration * 0.034 / 2;

    // Log distance measurement
    Serial.print("Distance: ");
    Serial.print(distance);
    Serial.println(" cm");

    if (distance < 10) {
      digitalWrite(motor1In1Pin, LOW);
      digitalWrite(motor1In2Pin, LOW);
      digitalWrite(motor2In1Pin, LOW);
      digitalWrite(motor2In2Pin, LOW);
      Serial.println("Obstacle detected, motors stopped");
    }

    request->send(200, "text/plain", String(distance));
  });

  server.begin();
  Serial.println("Server started");
}

void loop() {
  // Code in the loop, if any
}
