// ESP32FAN

// Load Wi-Fi library
#include <WiFi.h>
#include "driver/ledc.h"


// Replace with your network credentials
const char* ssid = "";
const char* password = "";

// Set web server port number to 80
WiFiServer server(80);

// Variable to store the HTTP request
String header;



// Current time
unsigned long currentTime = millis();
// Previous time
unsigned long previousTime = 0; 
// Define timeout time in milliseconds (example: 2000ms = 2s)
const long timeoutTime = 2000;


// --- fan specs ----------------------------------------------------------------------------------------------------------------------------
// fanPWM
#define FANMAXRPM      1700         // only used for showing at how many percent fan is running
#define PWMSTEP        100

// dyn var
ledc_timer_config_t   ledc_timer;
ledc_channel_config_t ledc_channel;
int pwmValue = 250;


void setup() {
  Serial.begin(115200);
  // Initialize the output variables as outputs
  pinMode(GPIO_NUM_3, OUTPUT);


  initPWMfan();
  // Connect to Wi-Fi network with SSID and password
  Serial.print("Connecting to ");
  Serial.println(ssid);
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  // Print local IP address and start web server
  Serial.println("");
  Serial.println("WiFi connected.");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
  server.begin();
}



void initPWMfan(void){
  //incc_timer.bit_num      = incC_TIMER_13_BIT;                 // resolution of PWM duty
  ledc_timer.freq_hz         = 5000;                              // frequency of PWM signal
  ledc_timer.speed_mode      = LEDC_LOW_SPEED_MODE;              // timer mode
  ledc_timer.timer_num       = LEDC_TIMER_0;   
  ledc_timer.duty_resolution = LEDC_TIMER_10_BIT;       

  // Set configuration of timer0 for high speed channels
  ledc_timer_config( &ledc_timer );

  ledc_channel.channel    = LEDC_CHANNEL_0;
  ledc_channel.duty       = 500;
  ledc_channel.gpio_num   = GPIO_NUM_3;
  ledc_channel.speed_mode = LEDC_LOW_SPEED_MODE;
  ledc_channel.timer_sel  = LEDC_TIMER_0;

  ledc_channel_config( &ledc_channel );

  updateFanSpeed();
}

void updateFanSpeed(void){
  pwmValue = min(pwmValue, 1024);
  pwmValue = max(pwmValue, 0);  
  //ledcWrite(LEDC_CHANNEL_0, pwmValue);
  ledc_set_duty(LEDC_LOW_SPEED_MODE,LEDC_CHANNEL_0,pwmValue);
  ledc_update_duty(LEDC_LOW_SPEED_MODE,LEDC_CHANNEL_0);
}

void incFanSpeed(void){
  pwmValue = min(pwmValue+PWMSTEP, 1024);
  updateFanSpeed();
}
void decFanSpeed(void){
  pwmValue = max(pwmValue-PWMSTEP, 0);  
  updateFanSpeed();
}


void loop(){
  WiFiClient client = server.available();   // Listen for incoming clients

  if (client) {                             // If a new client connects,
    currentTime = millis();
    previousTime = currentTime;
    //Serial.println("New Client.");          // print a message out in the serial port
    String currentLine = "";                // make a String to hold incoming data from the client
    while (client.connected() && currentTime - previousTime <= timeoutTime) {  // loop while the client's connected
      currentTime = millis();
      if (client.available()) {             // if there's bytes to read from the client,
        char c = client.read();             // read a byte, then
      //  Serial.write(c);                    // print it out the serial monitor
        header += c;
        if (c == '\n') {                    // if the byte is a newline character
          // if the current line is blank, you got two newline characters in a row.
          // that's the end of the client HTTP request, so send a response:
          if (currentLine.length() == 0) {
            // HTTP headers always start with a response code (e.g. HTTP/1.1 200 OK)
            // and a content-type so the client knows what's coming, then a blank line:
            client.println("HTTP/1.1 200 OK");
            client.println("Content-type:text/html");
            client.println("Connection: close");
            client.println();
        
            if (header.indexOf("GET /fan/inc") >= 0) {
              Serial.print("INC ");
              incFanSpeed();
              
              Serial.println(pwmValue);
            } else if (header.indexOf("GET /fan/dec") >= 0) {
              Serial.print("DEC ");
              decFanSpeed();

              Serial.println(pwmValue);  
            } else if (header.indexOf("GET /fan/max") >= 0) {
              Serial.print("MAX ");
              
              pwmValue = 700;
              updateFanSpeed();

              Serial.println(pwmValue);
            } else if (header.indexOf("GET /fan/min") >= 0) {
              Serial.print("MIN ");
              
              pwmValue = 200;
              updateFanSpeed();

              Serial.println(pwmValue);
            } else if (header.indexOf("GET /fan/med") >= 0) {
              Serial.print("MED ");
              
              pwmValue = 400;
              updateFanSpeed();

              Serial.println(pwmValue);
            }
            
            // Display the HTML web page
            client.print("<!DOCTYPE html><html>");
            client.print("<head><meta name=\"viewport\" content=\"width=device-width, initial-scale=1\">");
            client.print("<link rel=\"icon\" href=\"data:,\">");
            // CSS to style the on/off buttons 
            // Feel free to change the background-color and font-size attributes to fit your preferences
            client.println("<style>html { font-family: Helvetica; display: inline-block; margin: 0px auto; text-align: center;}");
            client.println(".button { background-color: #4CAF50; border: none; color: white; padding: 16px 40px;");
            client.println("text-decoration: none; font-size: 30px; margin: 2px; cursor: pointer;}");
            client.println(".button2 {background-color: #555555;}</style></head>");
            
            // Web Page Heading
            client.print("<body><h1>FAN</h1>");
            
            client.print("<p><a href=\"/fan/inc\"><button class=\"button\">+</button></a></p>");
            client.print("<p><a href=\"/fan/max\"><button class=\"button\">HI</button></a></p>");
            client.print("<p><a href=\"/fan/med\"><button class=\"button\">MED</button></a></p>");
            client.print("<p><a href=\"/fan/min\"><button class=\"button\">MIN</button></a></p>");
            client.print("<p><a href=\"/fan/dec\"><button class=\"button\">-</button></a></p>"); 
             
           
            client.print("</body></html>");
            
            // The HTTP response ends with another blank line
            client.println();
            // Break out of the while loop
            break;
          } else { // if you got a newline, then clear currentLine
            currentLine = "";
          }
        } else if (c != '\r') {  // if you got anything else but a carriage return character,
          currentLine += c;      // add it to the end of the currentLine
        }
      }
    }
    // Clear the header variable
    header = "";
    // Close the connection
    client.stop();
    //Serial.println("Client disconnected.");
    //Serial.println("");
  }
}
