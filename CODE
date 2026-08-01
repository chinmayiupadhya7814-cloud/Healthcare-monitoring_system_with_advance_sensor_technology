#include <OneWire.h>
#include <DallasTemperature.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <WiFi.h>
#include <ESP32Servo.h>
#include <SPIFFS.h>
#include "Adafruit_MQTT.h"
#include "Adafruit_MQTT_Client.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "freertos/semphr.h"
#include <esp_task_wdt.h>


#define TEMP_SENSOR_PIN   14
#define MOTION_PIN        12
#define GAS_SENSOR_PIN    34
#define SERVO_PIN         18

#define LED_RED_PIN        4
#define BUZZER_RED_PIN      4
#define LED_GREEN_PIN     16
#define BUZZER_GREEN_PIN   16
#define LED_BLUE_PIN      17
#define BUZZER_BLUE_PIN    17
#define LED_YELLOW_PIN     5
#define BUZZER_YELLOW_PIN   5

#define SW_TEMP_PIN       32
#define SW_HR_PIN         33
#define SW_SPO2_PIN       25
#define SW_MOTION_PIN     26

#define SCREEN_WIDTH  128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
#define OLED_ADDRESS  0x3C


#define WIFI_SSID  "Wokwi-GUEST"
#define WIFI_PASS  ""
#define IO_USERNAME  ""
#define IO_KEY       "    "
#define AIO_SERVER      "io.adafruit.com"
#define AIO_SERVERPORT  1883

#define TEMP_FEED   IO_USERNAME "/feeds/medical.body-temperature"
#define HR_FEED     IO_USERNAME "/feeds/medical.heart-rate"
#define SPO2_FEED   IO_USERNAME "/feeds/medical.spo2"
#define MOTION_FEED IO_USERNAME "/feeds/medical.motion"


#define ROOM_TEMP_FEED    IO_USERNAME "/feeds/facility.room-temperature"
#define AQI_FEED          IO_USERNAME "/feeds/facility.air-quality"
#define SYSTEM_STATE_FEED IO_USERNAME "/feeds/facility.system-state"


#define DOSAGE_SLIDER_FEED      IO_USERNAME "/feeds/control.dosage-slider"
#define BED_ANGLE_SLIDER_FEED   IO_USERNAME "/feeds/control.bed-angle-slider"
#define SAMPLE_RATE_SLIDER_FEED IO_USERNAME "/feeds/control.sample-rate-slider"

WiFiClient client;
Adafruit_MQTT_Client mqtt(&client, AIO_SERVER, AIO_SERVERPORT, IO_USERNAME, IO_KEY);

Adafruit_MQTT_Publish tempFeed           = Adafruit_MQTT_Publish(&mqtt, TEMP_FEED);
Adafruit_MQTT_Publish hrFeed             = Adafruit_MQTT_Publish(&mqtt, HR_FEED);
Adafruit_MQTT_Publish spo2Feed           = Adafruit_MQTT_Publish(&mqtt, SPO2_FEED);
Adafruit_MQTT_Publish motiondetectorFeed = Adafruit_MQTT_Publish(&mqtt, MOTION_FEED);

Adafruit_MQTT_Publish roomTempFeed    = Adafruit_MQTT_Publish(&mqtt, ROOM_TEMP_FEED);
Adafruit_MQTT_Publish aqiFeed         = Adafruit_MQTT_Publish(&mqtt, AQI_FEED);
Adafruit_MQTT_Publish systemStateFeed = Adafruit_MQTT_Publish(&mqtt, SYSTEM_STATE_FEED);

Adafruit_MQTT_Subscribe dosageSlider     = Adafruit_MQTT_Subscribe(&mqtt, DOSAGE_SLIDER_FEED);
Adafruit_MQTT_Subscribe bedAngleSlider   = Adafruit_MQTT_Subscribe(&mqtt, BED_ANGLE_SLIDER_FEED);
Adafruit_MQTT_Subscribe sampleRateSlider = Adafruit_MQTT_Subscribe(&mqtt, SAMPLE_RATE_SLIDER_FEED);


OneWire oneWire(TEMP_SENSOR_PIN);
DallasTemperature sensors(&oneWire);
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);
Servo bedServo;


typedef struct {
  float bodyTemp;
  float roomTemp;
  int   heartRate;
  int   spo2;
  bool  patientMotion;
  int   aqi;
  int   vitalsSafety;      
  char  statusMsg[32];

  float dosageTarget;
  float dosageCurrent;
  int   dosageSafety;      

  float bedAngleTarget;
  float bedAngleCurrent;

  uint32_t sampleRateMs;
  bool     autoSampleMode;

  uint8_t systemState;     
  bool    wifiOK;
  bool    mqttOK;
} SharedData;

SharedData shared;

typedef struct {
  uint32_t timestamp;
  float    bodyTemp;
  int      heartRate;
  int      spo2;
  int      aqi;
} Reading;


SemaphoreHandle_t dataMutex;
SemaphoreHandle_t alertSemaphore;
QueueHandle_t hrQueue;
QueueHandle_t spo2Queue;
QueueHandle_t offlineBufferQueue;

uint32_t backoffMs = 1000;
const uint32_t BACKOFF_MAX_MS = 32000;

bool spiffsReady = false;


void vitalsSensorTask(void *param);
void environmentTask(void *param);
void safetyEvalTask(void *param);
void dosageControlTask(void *param);
void bedControlTask(void *param);
void indicatorTask(void *param);
void displayTask(void *param);
void connectivityMqttTask(void *param);

bool MQTT_connect();
void flushOfflineBuffer();
void logToSpiffs(Reading &r);


void wdtSafeDelay(uint32_t ms) {
  const uint32_t CHUNK = 2000;
  while (ms > CHUNK) {
    vTaskDelay(pdMS_TO_TICKS(CHUNK));
    esp_task_wdt_reset();
    ms -= CHUNK;
  }
  vTaskDelay(pdMS_TO_TICKS(ms));
}


void setup() {
  Serial.begin(115200);
  randomSeed(analogRead(0));

  pinMode(LED_RED_PIN, OUTPUT);
  pinMode(LED_YELLOW_PIN, OUTPUT);
  pinMode(LED_GREEN_PIN, OUTPUT);
  pinMode(LED_BLUE_PIN, OUTPUT);
  pinMode(MOTION_PIN, INPUT);
  pinMode(SW_TEMP_PIN, INPUT_PULLUP);
  pinMode(SW_HR_PIN, INPUT_PULLUP);
  pinMode(SW_SPO2_PIN, INPUT_PULLUP);
  pinMode(SW_MOTION_PIN, INPUT_PULLUP);

  sensors.begin();

  Wire.begin();
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDRESS)) {
    Serial.println("SSD1306 allocation failed");
  }
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.println("Patient Risk Monitor");
  display.println("Booting...");
  display.display();

  spiffsReady = SPIFFS.begin(true);
  if (!spiffsReady) {
    Serial.println("SPIFFS mount failed - offline buffering will use RAM queue only");
  }

  Serial.print("Connecting to WiFi SSID: "); Serial.println(WIFI_SSID);
  Serial.print("Adafruit IO username: "); Serial.println(IO_USERNAME);
  
  WiFi.mode(WIFI_STA);
  WiFi.setSleep(false);
  WiFi.begin(WIFI_SSID, WIFI_PASS);

  dataMutex           = xSemaphoreCreateMutex();
  alertSemaphore       = xSemaphoreCreateBinary();
  hrQueue              = xQueueCreate(1, sizeof(int));
  spo2Queue            = xQueueCreate(1, sizeof(int));
  offlineBufferQueue   = xQueueCreate(50, sizeof(Reading));

  memset(&shared, 0, sizeof(shared));
  shared.sampleRateMs   = 5000;
  shared.autoSampleMode = true;
  shared.bedAngleTarget = 10;      
  strcpy(shared.statusMsg, "OK");

  mqtt.subscribe(&dosageSlider);
  mqtt.subscribe(&bedAngleSlider);
  mqtt.subscribe(&sampleRateSlider);

  
  esp_task_wdt_config_t wdtConfig = {
    .timeout_ms = 15000,
    .idle_core_mask = 0,
    .trigger_panic = true
  };
  if (esp_task_wdt_init(&wdtConfig) == ESP_ERR_INVALID_STATE) {
    esp_task_wdt_reconfigure(&wdtConfig);
  }

  xTaskCreatePinnedToCore(vitalsSensorTask,     "Vitals",     4096, NULL, 3, NULL, 1);
  xTaskCreatePinnedToCore(environmentTask,      "Environment",3072, NULL, 1, NULL, 1);
  xTaskCreatePinnedToCore(safetyEvalTask,       "SafetyEval", 3072, NULL, 3, NULL, 1);
  xTaskCreatePinnedToCore(dosageControlTask,    "Dosage",     3072, NULL, 2, NULL, 1);
  xTaskCreatePinnedToCore(bedControlTask,       "BedControl", 3072, NULL, 2, NULL, 1);
  xTaskCreatePinnedToCore(indicatorTask,        "Indicator",  2048, NULL, 2, NULL, 1);
  xTaskCreatePinnedToCore(displayTask,          "Display",    4096, NULL, 1, NULL, 0);
  xTaskCreatePinnedToCore(connectivityMqttTask, "MQTT_Link",  8192, NULL, 2, NULL, 0);
}

void loop() {
  
  vTaskDelay(pdMS_TO_TICKS(1000));
}


void vitalsSensorTask(void *param) {
  esp_task_wdt_add(NULL);
  static float lastTemp = 25.0;

  for (;;) {
    esp_task_wdt_reset();

    int heartRate = digitalRead(SW_HR_PIN) ? 0 : random(45, 181);
    int spo2      = digitalRead(SW_SPO2_PIN) ? 0 : random(85, 100);
    if (random(0, 20) == 0) heartRate = random(190, 260); 
    if (random(0, 25) == 0) spo2 = random(70, 84);

    xQueueOverwrite(hrQueue, &heartRate);
    xQueueOverwrite(spo2Queue, &spo2);

    if (!digitalRead(SW_TEMP_PIN)) {
      sensors.requestTemperatures();
      float t = sensors.getTempCByIndex(0);
      if (t > -100) lastTemp = t;   
    }
    bool motion = digitalRead(MOTION_PIN) && !digitalRead(SW_MOTION_PIN);

    uint32_t delayMs;
    bool autoMode;
    xSemaphoreTake(dataMutex, portMAX_DELAY);
    shared.bodyTemp      = lastTemp;
    shared.heartRate      = heartRate;
    shared.spo2           = spo2;
    shared.patientMotion  = motion;
    delayMs  = shared.sampleRateMs;
    autoMode = shared.autoSampleMode;
    xSemaphoreGive(dataMutex);

    
    bool abnormal = (heartRate != 0 && (heartRate < 60 || heartRate > 140)) ||
                     (spo2 != 0 && spo2 < 90) || (lastTemp > 38.0);
    if (autoMode) {
      delayMs = abnormal ? 5000UL : constrain((long)delayMs + 2000, 5000, 60000);
      xSemaphoreTake(dataMutex, portMAX_DELAY);
      shared.sampleRateMs = delayMs;
      xSemaphoreGive(dataMutex);
    }

    Reading r = { millis(), lastTemp, heartRate, spo2, 0 };
    xSemaphoreTake(dataMutex, portMAX_DELAY);
    r.aqi = shared.aqi;
    xSemaphoreGive(dataMutex);
    xQueueSend(offlineBufferQueue, &r, 0);   

    vTaskDelay(pdMS_TO_TICKS(1)); 
    wdtSafeDelay(delayMs);
  }
}


void environmentTask(void *param) {
  esp_task_wdt_add(NULL);
  for (;;) {
    esp_task_wdt_reset();

    int raw = analogRead(GAS_SENSOR_PIN);        
    int aqi = map(raw, 0, 4095, 0, 500);

    xSemaphoreTake(dataMutex, portMAX_DELAY);
    float bodyTemp = shared.bodyTemp;
    xSemaphoreGive(dataMutex);
    float roomTemp = bodyTemp - 13.0 + (random(-10, 10) / 10.0); 

    xSemaphoreTake(dataMutex, portMAX_DELAY);
    shared.aqi      = aqi;
    shared.roomTemp = roomTemp;
    xSemaphoreGive(dataMutex);

    vTaskDelay(pdMS_TO_TICKS(4000));
  }
}


void safetyEvalTask(void *param) {
  esp_task_wdt_add(NULL);
  for (;;) {
    esp_task_wdt_reset();

    int hr = 0, spo2 = 0;
    xQueuePeek(hrQueue, &hr, 0);
    xQueuePeek(spo2Queue, &spo2, 0);

    xSemaphoreTake(dataMutex, portMAX_DELAY);
    float temp = shared.bodyTemp;
    xSemaphoreGive(dataMutex);

    int level = 0;
    const char *msg = "OK";
    if (temp > 38.0)                          { level = 2; msg = "High body temperature"; }
    else if (hr != 0 && hr < 60)               { level = 2; msg = "Heart rate below normal"; }
    else if (hr != 0 && hr > 140)               { level = 2; msg = "Heart rate above normal"; }
    else if (spo2 != 0 && spo2 < 85)            { level = 2; msg = "SpO2 below normal"; }
    else if (spo2 != 0 && spo2 < 92)            { level = 1; msg = "SpO2 borderline"; }
    else if (temp > 37.4)                       { level = 1; msg = "Elevated temperature"; }

    xSemaphoreTake(dataMutex, portMAX_DELAY);
    int prevLevel = shared.vitalsSafety;
    shared.vitalsSafety = level;
    strncpy(shared.statusMsg, msg, sizeof(shared.statusMsg) - 1);
    xSemaphoreGive(dataMutex);

    if (level == 2 && prevLevel != 2) {
      xSemaphoreGive(alertSemaphore);  
    }

    vTaskDelay(pdMS_TO_TICKS(500));
  }
}


void dosageControlTask(void *param) {
  esp_task_wdt_add(NULL);
  const float MAX_STEP = 5.0;     
  static bool criticalConfirmed = false;

  for (;;) {
    esp_task_wdt_reset();

    xSemaphoreTake(dataMutex, portMAX_DELAY);
    float target  = shared.dosageTarget;
    float current = shared.dosageCurrent;
    xSemaphoreGive(dataMutex);

    float diff = target - current;
    if (fabs(diff) > MAX_STEP) current += (diff > 0 ? MAX_STEP : -MAX_STEP);
    else current = target;
    current = constrain(current, 0.0f, 100.0f);

    int level;
    if (current > 80)      level = 2;
    else if (current >= 60) level = 1;
    else                     level = 0;

    
    if (level == 2 && !criticalConfirmed) {
      current = min(current, 80.0f);
    }
    criticalConfirmed = (target <= 80);

    xSemaphoreTake(dataMutex, portMAX_DELAY);
    shared.dosageCurrent = current;
    shared.dosageSafety  = level;
    xSemaphoreGive(dataMutex);

    vTaskDelay(pdMS_TO_TICKS(200));
  }
}


void bedControlTask(void *param) {
  esp_task_wdt_add(NULL);
  bedServo.setPeriodHertz(50);
  bedServo.attach(SERVO_PIN, 500, 2400);
  const float MAX_STEP_DEG = 2.0;   

  for (;;) {
    esp_task_wdt_reset();

    xSemaphoreTake(dataMutex, portMAX_DELAY);
    float target  = shared.bedAngleTarget;
    float current = shared.bedAngleCurrent;
    int   hr       = shared.heartRate;
    int   spo2      = shared.spo2;
    xSemaphoreGive(dataMutex);

    
    if (spo2 != 0 && spo2 < 90) target = max(target, 45.0f);
    if (hr != 0 && (hr < 50 || hr > 160)) target = 90.0f;
    target = constrain(target, 0.0f, 90.0f);

    float diff = target - current;
    if (fabs(diff) > MAX_STEP_DEG) current += (diff > 0 ? MAX_STEP_DEG : -MAX_STEP_DEG);
    else current = target;
    bedServo.write((int)current);

    xSemaphoreTake(dataMutex, portMAX_DELAY);
    shared.bedAngleCurrent = current;
    xSemaphoreGive(dataMutex);

    vTaskDelay(pdMS_TO_TICKS(100));   
  }
}


void indicatorTask(void *param) {
  esp_task_wdt_add(NULL);
  bool phase = false;

  for (;;) {
    esp_task_wdt_reset();

    xSemaphoreTake(dataMutex, portMAX_DELAY);
    int  vSafety = shared.vitalsSafety;
    int  dSafety = shared.dosageSafety;
    bool motion   = shared.patientMotion;
    xSemaphoreGive(dataMutex);

    bool critical = (vSafety == 2) || (dSafety == 2);
    bool warning  = !critical && (vSafety == 1 || dSafety == 1);

    phase = !phase;
    digitalWrite(LED_RED_PIN, critical && phase);
    if (critical && phase) tone(BUZZER_RED_PIN, 2200, 150);

    digitalWrite(LED_YELLOW_PIN, warning && phase);
    if (warning && phase) tone(BUZZER_YELLOW_PIN, 1200, 100);

    digitalWrite(LED_GREEN_PIN, motion);
    if (motion) tone(BUZZER_GREEN_PIN, 900, 80);

    vTaskDelay(pdMS_TO_TICKS(critical ? 200 : 500)); 
  }
}


void displayTask(void *param) {
  esp_task_wdt_add(NULL);
  uint8_t screen = 0;

  for (;;) {
    esp_task_wdt_reset();

    xSemaphoreTake(dataMutex, portMAX_DELAY);
    SharedData s = shared;   
    xSemaphoreGive(dataMutex);

    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(0, 0);

    if (s.systemState == 2) {
      display.println("** LOGGING OFFLINE **");
      display.setCursor(0, 16);
      display.print("Buffered: ");
      display.println(uxQueueMessagesWaiting(offlineBufferQueue));
      display.setCursor(0, 32);
      display.println("Retrying connection...");
    } else if (screen == 0) {
      display.println("--- VITALS ---");
      display.print("Temp: "); display.print(s.bodyTemp, 1); display.println(" C");
      display.print("HR:   "); display.print(s.heartRate);  display.println(" bpm");
      display.print("SpO2: "); display.print(s.spo2);        display.println(" %");
      display.print("Motion: "); display.println(s.patientMotion ? "YES" : "no");
      display.print("Status: "); display.println(s.statusMsg);
    } else if (screen == 1) {
      display.println("--- DOSAGE / BED ---");
      display.print("Dose: ");   display.print(s.dosageCurrent, 1);  display.println(" mg/hr");
      display.print("Target: "); display.println(s.dosageTarget, 1);
      display.print("Bed: ");    display.print(s.bedAngleCurrent, 0); display.println(" deg");
      display.print("Rate: ");   display.print(s.sampleRateMs / 1000); display.println(" s");
    } else {
      display.println("--- FACILITY / LINK ---");
      display.print("Room T: "); display.print(s.roomTemp, 1); display.println(" C");
      display.print("AQI: ");    display.println(s.aqi);
      display.print("WiFi: ");   display.println(s.wifiOK ? "OK" : "DOWN");
      display.print("MQTT: ");   display.println(s.mqttOK ? "OK" : "DOWN");
      const char *stateStr = s.systemState == 0 ? "ONLINE" : (s.systemState == 1 ? "DEGRADED" : "OFFLINE");
      display.print("State: "); display.println(stateStr);
    }

    display.display();
    screen = (screen + 1) % 3;
    vTaskDelay(pdMS_TO_TICKS(2000));
  }
}


bool MQTT_connect() {
  if (mqtt.connected()) return true;
  Serial.print("Connecting to Adafruit IO... ");
  int8_t ret = mqtt.connect();
  if (ret != 0) {
    Serial.println(mqtt.connectErrorString(ret));  
    mqtt.disconnect();
    return false;
  }
  Serial.println("connected!");
  backoffMs = 1000;  
  return true;
}

void logToSpiffs(Reading &r) {
  if (!spiffsReady) return;
  File f = SPIFFS.open("/buffer.log", FILE_APPEND);
  if (f) {
    f.printf("%lu,%.1f,%d,%d\n", r.timestamp, r.bodyTemp, r.heartRate, r.spo2);
    f.close();
  }
}

void flushOfflineBuffer() {
  Reading r;
  int flushed = 0;
  while (xQueueReceive(offlineBufferQueue, &r, 0) == pdTRUE && flushed < 10) {
    tempFeed.publish((double)r.bodyTemp);
    hrFeed.publish((int32_t)r.heartRate);
    spo2Feed.publish((int32_t)r.spo2);
    flushed++;
    vTaskDelay(pdMS_TO_TICKS(50));
  }
  

  if (spiffsReady && SPIFFS.exists("/buffer.log")) {
    SPIFFS.remove("/buffer.log");
  }
}

void connectivityMqttTask(void *param) {
  esp_task_wdt_add(NULL);
  uint32_t lastPublish = 0;
  uint32_t lastWifiAttempt = millis();   
  bool bluePhase = false;
  bool prevWifiUp = false;

  for (;;) {
    esp_task_wdt_reset();

    bool wifiUp = (WiFi.status() == WL_CONNECTED);
    if (wifiUp != prevWifiUp) {
      if (wifiUp) {
        Serial.print("WiFi connected, IP: ");
        Serial.println(WiFi.localIP());
      } else {
        Serial.println("WiFi disconnected");
      }
      prevWifiUp = wifiUp;
    }

    
    if (!wifiUp && millis() - lastWifiAttempt > 8000) {
      Serial.println("WiFi stuck, restarting association...");
      WiFi.disconnect();
      WiFi.begin(WIFI_SSID, WIFI_PASS);
      lastWifiAttempt = millis();
    }

    bool mqttUp = wifiUp && MQTT_connect();
    
    uint8_t state = (wifiUp && mqttUp) ? 0 : (wifiUp ? 1 : 2);

    xSemaphoreTake(dataMutex, portMAX_DELAY);
    shared.wifiOK      = wifiUp;
    shared.mqttOK       = mqttUp;
    shared.systemState = state;
    xSemaphoreGive(dataMutex);

    
    bluePhase = !bluePhase;
    if (state == 0) {
      digitalWrite(LED_BLUE_PIN, LOW);
    } else if (state == 1) {                
      digitalWrite(LED_BLUE_PIN, bluePhase);
      if (bluePhase) tone(BUZZER_BLUE_PIN, 600, 150);
    } else {                                  
      digitalWrite(LED_BLUE_PIN, bluePhase);
      if (bluePhase) tone(BUZZER_BLUE_PIN, 1800, 80);
    }

    if (state != 0) {
      Reading r;
      if (xQueuePeek(offlineBufferQueue, &r, 0) == pdTRUE) logToSpiffs(r);
      Serial.print("Offline (state="); Serial.print(state);
      Serial.print("), retrying in "); Serial.print(min(backoffMs, BACKOFF_MAX_MS));
      Serial.println(" ms");
      wdtSafeDelay(min(backoffMs, BACKOFF_MAX_MS));
      backoffMs = min(backoffMs * 2, BACKOFF_MAX_MS);
      continue;
    }

    backoffMs = 1000;   
    
    if (uxQueueMessagesWaiting(offlineBufferQueue) > 0 || (spiffsReady && SPIFFS.exists("/buffer.log"))) {
      flushOfflineBuffer();
    }

    if (xSemaphoreTake(alertSemaphore, 0) == pdTRUE) {
      lastPublish = 0;
    }

    
    if (millis() - lastPublish > 5000) {
      lastPublish = millis();
      xSemaphoreTake(dataMutex, portMAX_DELAY);
      SharedData s = shared;
      xSemaphoreGive(dataMutex);

      
      bool ok = true;
      ok &= tempFeed.publish((double)s.bodyTemp);
      ok &= hrFeed.publish((int32_t)s.heartRate);
      ok &= spo2Feed.publish((int32_t)s.spo2);
      ok &= motiondetectorFeed.publish((int32_t)(s.patientMotion ? 1 : 0));

    
      ok &= roomTempFeed.publish((double)s.roomTemp);
      ok &= aqiFeed.publish((int32_t)s.aqi);
      ok &= systemStateFeed.publish((int32_t)s.systemState);

      Serial.print("Publish cycle: ");
      Serial.print(ok ? "all OK" : "ONE OR MORE FAILED - check feed keys");
      Serial.print(" | T="); Serial.print(s.bodyTemp, 1);
      Serial.print(" HR="); Serial.print(s.heartRate);
      Serial.print(" SpO2="); Serial.print(s.spo2);
      Serial.print(" Room="); Serial.print(s.roomTemp, 1);
      Serial.print(" AQI="); Serial.println(s.aqi);
    }

    
    Adafruit_MQTT_Subscribe *sub;
    while ((sub = mqtt.readSubscription(100))) {
      if (sub == &dosageSlider) {
        xSemaphoreTake(dataMutex, portMAX_DELAY);
        shared.dosageTarget = constrain((float)atof((char *)dosageSlider.lastread), 0.0f, 100.0f);
        xSemaphoreGive(dataMutex);
      } else if (sub == &bedAngleSlider) {
        xSemaphoreTake(dataMutex, portMAX_DELAY);
        shared.bedAngleTarget = constrain((float)atof((char *)bedAngleSlider.lastread), 0.0f, 90.0f);
        xSemaphoreGive(dataMutex);
      } else if (sub == &sampleRateSlider) {
        long val = atol((char *)sampleRateSlider.lastread);
        xSemaphoreTake(dataMutex, portMAX_DELAY);
        if (val <= 0) {
          shared.autoSampleMode = true;             
        } else {
          shared.autoSampleMode = false;
          shared.sampleRateMs = (uint32_t)constrain(val, 5, 60) * 1000UL;
        }
        xSemaphoreGive(dataMutex);
      }
    }

    vTaskDelay(pdMS_TO_TICKS(200));
  }
}
