#include <WiFi.h>
#include <PubSubClient.h>
#include "DHTesp.h"
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <Wire.h>

#define SCREEN_WIDTH 128 
#define SCREEN_HEIGHT 64 
const int DHT_PIN = 13;
const int ledBlue = 18;
const int ledRed = 19;
const int Buzzer = 17;
const int PB = 4;
float Temp;

bool ButtonStatus = LOW;
bool lastButtonStatus = LOW; 
const long postInterval = 2000;
unsigned long previousMillis = 0;
unsigned long silenceTimestamp = 0; 
const long snoozeDuration = 5000;   

const char *ssid = "Wokwi-GUEST";
const char *password = "";
const char *mqtt_server = "broker.hivemq.com";
const int mqtt_port = 1883;
const char *topic_subscribe = "TempRM";
const char *topic_stopbuzz = "STOPBUZZ";
bool buzzerSilenced = false;

WiFiClient wifiClient;
PubSubClient mqttClient(wifiClient);

DHTesp dhtSensor;
Adafruit_SSD1306 oled(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

void setup_wifi()
{
  delay(10);
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);

  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED)
  {
    delay(500);
    Serial.print(".");
  }

  randomSeed(micros());

  Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
}

void mqttCallback(char *topic, byte *payload, unsigned int length)
{

  Serial.print("Message arrived [");
  Serial.print(topic);
  Serial.print("]: ");
  String message;
  for (int i = 0; i < length; i++)
  {
    message += (char)payload[i];
  }
  Serial.println(message);

  if (strcmp(topic, topic_stopbuzz) == 0)
  {
    if (message == "false")
    {
      Serial.println("Received 'false' on STOPBUZZ. Silencing buzzer.");
      buzzerSilenced = true;     
      tone(Buzzer, 0);           
      silenceTimestamp = millis(); 
    }
  }
}

void setupMQTT()
{
  mqttClient.setServer(mqtt_server, mqtt_port);
  mqttClient.setCallback(mqttCallback);
}

void reconnect()
{
  Serial.println("Connecting to MQTT Broker (HiveMQ Public)...");
  while (!mqttClient.connected())
  {
    Serial.println("Reconnecting to MQTT Broker...");
    String clientId = "ESP32Client-";
    clientId += String(random(0xffff), HEX);

    if (mqttClient.connect(clientId.c_str()))
    {
      Serial.println("Connected to MQTT Broker.");

      mqttClient.subscribe(topic_stopbuzz);
      Serial.print("Subscribed to topic: ");
      Serial.println(topic_stopbuzz);
    }
    else
    {
      Serial.print("Failed, rc=");
      Serial.print(mqttClient.state());
      Serial.println(" try again in 5 seconds");
      delay(5000);
    }
  }
}

void publishMessage()
{
  float temperature = Temp;
  char msg[50];
  dtostrf(temperature, 4, 2, msg); 

  Serial.print("Publishing message: ");
  Serial.println(msg);

  mqttClient.publish("TempRM", msg, true);
}

void setup()
{
  pinMode(ledBlue, OUTPUT);
  pinMode(ledRed, OUTPUT);
  pinMode(Buzzer, OUTPUT);
  pinMode(PB, INPUT);

  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED)
  {
    delay(500);
    Serial.print(".");
  }
  Serial.println("");
  Serial.println("Connected to Wi-Fi");

  setupMQTT();
  if (!oled.begin(SSD1306_SWITCHCAPVCC, 0x3C))
  {
    Serial.println(F("failed to start SSD1306 OLED"));
    while (1)
    {
      // loop indefinitely
    }
  }

  delay(2000);       
  oled.clearDisplay();

  oled.setTextSize(1);    
  oled.setTextColor(WHITE); 
  oled.setCursor(0, 2);  
  oled.println("Start");    
  oled.display();
  Serial.begin(115200);
  dhtSensor.setup(DHT_PIN, DHTesp::DHT22);
}

void loop()
{
  if (!mqttClient.connected())
  {
    reconnect();
  }
  mqttClient.loop();

  ButtonStatus = digitalRead(PB);
  if (ButtonStatus == HIGH && lastButtonStatus == LOW)
  {
    buzzerSilenced = true;
    mqttClient.publish("STOPBUZZ", "false", true);
    Serial.println("Button Pressed - Silencing Buzzer");
    tone(Buzzer, 0);
    silenceTimestamp = millis(); 
  }
  lastButtonStatus = ButtonStatus; 

  unsigned long currentMillis = millis();
  if (currentMillis - previousMillis >= postInterval)
  {
    previousMillis = currentMillis;

    TempAndHumidity data = dhtSensor.getTempAndHumidity();
    Temp = float(data.temperature);
    Serial.println("Temp: " + String(data.temperature, 2) + "C");

    oled.clearDisplay();
    oled.setTextColor(WHITE);

    oled.setTextSize(2);
    oled.setCursor(0, 5);
    oled.print(String(data.temperature, 1));
    oled.setTextSize(1);
    oled.print(" ");
    oled.print((char)247);
    oled.setTextSize(2);
    oled.print("C");

    oled.setTextSize(1);
    oled.setCursor(0, 30);
    String pb_state = (ButtonStatus == HIGH) ? "PRESSED" : "Ready";
    oled.print("Button: ");
    oled.println(pb_state);

    oled.setCursor(0, 42);
    oled.print("Buzzer: ");
    oled.println(buzzerSilenced ? "SNOOZE" : "Ready");

    oled.display();
    publishMessage();

    if (buzzerSilenced)
    {
      unsigned long now = millis();
      if (now - silenceTimestamp >= snoozeDuration)
      {
        if (Temp > 30)
        {
          Serial.println("Snooze time over. Still hot! Re-alarming.");
          buzzerSilenced = false; 
        }
      }
    }

    if (Temp > 30)
    {
      digitalWrite(ledRed, HIGH);
      digitalWrite(ledBlue, LOW);

      if (!buzzerSilenced)
      {
        tone(Buzzer, 1000); 
        mqttClient.publish("STOPBUZZ", "true", true);
      }
      else
      {
        tone(Buzzer, 0); 
      }
    }
    else
    {
      digitalWrite(ledRed, LOW);
      
      if (Temp < 25)
      {
        digitalWrite(ledBlue, HIGH); 
      }
      else
      {
        digitalWrite(ledBlue, LOW); 
      }

      tone(Buzzer, 0);         
      buzzerSilenced = false;   
      
     
      mqttClient.publish("STOPBUZZ", "false", true); 
    }
  } 
} 
