# ESP32-webserver
//This is a trial to learn ESP32
//In 65/66 line of setup- when I use 
 //wifiManager.setSTAStaticIPConfig(_ip, _gw, _sn); 
//the time gets changed, if I do not use this I can not get a static IP which creates problem in connecting webserver 


#include <FS.h> 
#include <WiFi.h>
#include <DNSServer.h>
#include <AsyncTCP.h>
#include <ESPAsyncWebServer.h>
#include <NTPClient.h>
#include <ArduinoOTA.h>
#include <WiFiUdp.h>
#include <Time.h>
#include <TimeLib.h>
#include <Timezone.h>
#include <ESPAsyncWiFiManager.h>

#define NTP_OFFSET  -6  * 60 * 60 // In seconds
#define NTP_INTERVAL 60 * 1000    // In miliseconds
#define NTP_ADDRESS  "ch.pool.ntp.org"
#define templateLength 4

String totalStuff = "";

String t;
String date;
WiFiUDP ntpUDP;

NTPClient timeClient(ntpUDP, NTP_ADDRESS, NTP_OFFSET, NTP_INTERVAL);

const char * days[] = {"Sunday", "Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday"} ;
const char * months[] = {"Jan", "Feb", "Mar", "Apr", "May", "June", "July", "Aug", "Sep", "Oct", "Nov", "Dec"} ;
const char * ampm[] = {"AM", "PM"} ;

typedef struct Dictionary {
  String key; String val;
} Dictionary;

String templatey(String html, Dictionary variables[]) {
  Serial.println(templateLength);
  int i = 0;
  while (i < templateLength) {
    html.replace("%" + variables[i].key + "%", variables[i].val);
    i = i + 1;
  } return html;
}
  
AsyncWebServer server(80);
DNSServer dns;


void setup(){
 Serial.begin(115200);
 WiFi.begin();
   timeClient.begin();    
 AsyncWiFiManager wifiManager(&server,&dns);
//wifiManager.resetSettings();
 wifiManager.setBreakAfterConfig(true);
  IPAddress _ip = IPAddress(192,168, 0, 200);
  IPAddress _gw = IPAddress(192,168, 0, 1);
  IPAddress _sn = IPAddress(255, 255, 255, 0);
      wifiManager.setAPStaticIPConfig(_ip, _gw, _sn);
     //wifiManager.setSTAStaticIPConfig(_ip, _gw, _sn);
  
     if (!wifiManager.autoConnect("Green Wall")) {
    Serial.println("failed to connect, we should reset as see if it connects");
    delay(3000);
    ESP.restart();
    delay(5000);
  }
//  WiFi.disconnect()
//  WiFi.begin();
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Connecting to WiFi..");
  }

 

   Serial.println("connected...yeey :)");


  Serial.println("local ip");
  Serial.println(WiFi.localIP());
  

  server.on("/favicon.ico", [](AsyncWebServerRequest *request) {
    request -> send(200, "text/plain", "no icon found...");
  });

  server.on("/puts/text", HTTP_POST, [](AsyncWebServerRequest *request) {
    if (request->hasArg("nonsense") && request->hasArg("name")) {
      String nonsense = request->arg("nonsense");
      String namee = request->arg("name");
      if (nonsense != "0" && namee != "0") {
        Serial.print(namee); Serial.print(": "); Serial.println(nonsense);
        totalStuff = totalStuff + "<h4>" + namee + ": " + nonsense + "</h4>";
      }
      request->send(200, totalStuff, "");
    } else {
      request->send(400, "text/html", "nonsense arg not found");
    }
  });

  server.on("/sets/text", HTTP_POST, [](AsyncWebServerRequest *request) {
    totalStuff = "";
  });

  /* ROUTE: GET / */

  server.on("/", HTTP_GET, [](AsyncWebServerRequest *request) {
    
    const char html[] PROGMEM = R"=====(
      <html>
        <head>
          <meta charset="utf8">
          <title>Hydroponics Remote Control | Home</title>
        </head>

        <body>
          <h1>Hello, %name%!</h1>
          <p>%message%</p>
          <div id="app">
            <script src="https://unpkg.com/axios/dist/axios.min.js"></script>
            <input type="text" value="Lorem Ipsum">
            <input type="text" value="Bob Jones">
            <input type="submit" value="Submit" onclick="submitForm()">
            <input type="submit" value="Clear Chat" onclick="clearChat()">
          </div>

          <div id="messageContainer">
          </div>
        </body>

        <script>
          var submitForm = function() {
            var input = document.getElementsByTagName("input")[0];
            var nameInput = document.getElementsByTagName("input")[1];
            axios.post("/puts/text?nonsense=" + encodeURI(input.value) + "&name=" + encodeURI(nameInput.value), {}).then(function(response) {
              updateContainer(response);
            }).catch(function(error) {
              console.log(error);
            });
          };

          var clearChat = function() {
            axios.post("/sets/text", {}).then(function(response) {
              console.log(response);
            }).catch(function(error) {
              console.log(error);
            });
          };

          var updateSelf = function() {
            axios.post("/puts/text?nonsense=0&name=0", {}).then(function(response) {
              updateContainer(response);
            }).catch(function(error) {
              console.log(error);
            });
          };

          var updateContainer = function(res) {
            var messageContainer = document.getElementById("messageContainer");
            messageContainer.innerHTML = JSON.stringify(res.headers["content-type"]);
          };

          setInterval(updateSelf, 1000);
          
        </script>
      </html>
    )=====";

    Dictionary templateVariables[templateLength] = {
      { "name", "Papoo" },
      { "message", "This is a message coded directly in the Arduino code..." },
      { "alertMessage", "I can use these strings in the JavaScript too..." },
      { "actualCode", "document.getElementById('app').innerHTML = 'To the point that I can actually make JavaScript code here...';" }
    };
    String stringHTML = templatey(html, templateVariables);
    
    request -> send(200, "text/html", stringHTML);
  });
  
  server.begin();
  
 
  
}



void loop(){
        
    if (WiFi.status() == WL_CONNECTED) //Check WiFi connection status
  {   
    date = "";  // clear the variables
    t = "";
    
    // update the NTP client and get the UNIX UTC timestamp 
    
     timeClient.update();
    
    unsigned long epochTime =  timeClient.getEpochTime();

    // convert received time stamp to time_t object
    time_t local, utc;
    utc = epochTime;

    // Then convert the UTC UNIX timestamp to local time
    TimeChangeRule usEDT = {"EDT", Second, Sun, Mar, 2, +120};  //UTC - 5 hours - change this as needed
    TimeChangeRule usEST = {"EST", First, Sun, Nov, 2, +180};   //UTC - 6 hours - change this as needed
    Timezone usEastern(usEDT, usEST);
    local = usEastern.toLocal(utc);
    //Serial.println(usEastern.toLocal(utc));

    // now format the Time variables into strings with proper names for month, day etc
    date += days[weekday(local)-1];
    date += ", ";
    date += months[month(local)-1];
    date += " ";
    date += day(local);
    date += ", ";
    date += year(local);

    // format the time to 12-hour format with AM/PM and no seconds
    t += hourFormat12(local);
    t += ":";
    if(minute(local) < 10)  // add a zero if minute is under 10
      t += "0";
    t += minute(local);
    t += " ";
    t += ampm[isPM(local)];

//   //  Display the date and time
    Serial.println("");
    Serial.print("Local Date: ");
    Serial.print(date);
    Serial.print(" Time: ");
    Serial.print(t);


  delay(1000);
}
}
