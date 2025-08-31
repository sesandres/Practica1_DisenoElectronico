```
#include <WiFi.h>
#include <UbidotsEsp32Mqtt.h>
#include <TFT_eSPI.h>
#include <SPI.h>
#include <MQUnifiedsensor.h>

// === Configuración Ubidots y Wi-Fi ===
const char *UBIDOTS_TOKEN = "BBUS-GNBkQ8Kq6Vx1RqIn0SXGX1kkJYHOcx";
const char *WIFI_SSID = "Familia Zapata";
const char *WIFI_PASS = "santi2003";
const char *DEVICE_LABEL = "esp23"; // Nombre del dispositivo en Ubidots

Ubidots ubidots(UBIDOTS_TOKEN);

// === Pines y configuraciones ===
#define PIN_SENSOR 36
#define BUZZER_PIN 25
#define LED_PIN 26
#define placa "ESP32"
#define Voltage_Resolution 3.3
#define ADC_Bit_Resolution 12
#define RatioMQ135CleanAir 3.6

TFT_eSPI tft = TFT_eSPI();
MQUnifiedsensor MQ135(placa, Voltage_Resolution, ADC_Bit_Resolution, PIN_SENSOR, "MQ-135");

// === Temporización de alarma y publicación ===
unsigned long previousMillis = 0;
const unsigned long intervalAlarma = 100;  // 0.3 segundos (pipipi)
bool alarmState = false;

unsigned long ultimoEnvio = 0;
const unsigned long intervaloPublicacion = 100; // 10 segundos

// === Funciones auxiliares ===
void mostrarGas(String nombre, float valor, float umbral) {
  if (valor > umbral) {
    tft.setTextColor(TFT_BLUE, TFT_BLACK);
  } else {
    tft.setTextColor(TFT_GREEN, TFT_BLACK);
  }
  tft.printf("%s: %.1f ppm\n", nombre.c_str(), valor);
}

void conectarWiFi() {
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  Serial.print("Conectando a Wi-Fi");

  unsigned long tiempoInicio = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - tiempoInicio < 10000) {
    Serial.print(".");
    delay(500);
  }

  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\n✅ Conectado a Wi-Fi. IP: " + WiFi.localIP().toString());
  } else {
    Serial.println("\n❌ No se pudo conectar a Wi-Fi");
  }
}

// === Setup ===
void setup() {
  Serial.begin(115200);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(LED_PIN, LOW);

  // Pantalla
  tft.init();
  tft.setRotation(1);
  tft.fillScreen(TFT_BLACK);
  tft.setTextColor(TFT_WHITE, TFT_BLACK);
  tft.setTextSize(2);
  tft.setCursor(0, 0);
  tft.println("Calibrando...");

  // Sensor
  MQ135.setRegressionMethod(1);
  MQ135.init();
  float calcR0 = 0;
  for (int i = 0; i < 10; i++) {
    MQ135.update();
    calcR0 += MQ135.calibrate(RatioMQ135CleanAir);
    delay(500);
  }
  MQ135.setR0(calcR0 / 10);
  if (isinf(calcR0) || calcR0 == 0) {
    tft.setTextColor(TFT_RED, TFT_BLACK);
    tft.setCursor(0, 20);
    tft.println("Error en calibracion");
    while (1);
  }

  tft.fillScreen(TFT_BLACK);

  // Conexión Wi-Fi y Ubidots
  conectarWiFi();
  ubidots.setup();
  ubidots.reconnect();
}

// === Loop ===
void loop() {
  MQ135.update();

  // Leer gases
  MQ135.setA(605.18); MQ135.setB(-3.937);  float CO = MQ135.readSensor();
  MQ135.setA(77.255); MQ135.setB(-3.18);   float Alcohol = MQ135.readSensor();
  MQ135.setA(110.47); MQ135.setB(-2.862);  float CO2 = MQ135.readSensor() + 400;
  MQ135.setA(44.947); MQ135.setB(-3.445);  float Tolueno = MQ135.readSensor();
  MQ135.setA(102.2);  MQ135.setB(-2.473);  float NH4 = MQ135.readSensor();
  MQ135.setA(34.668); MQ135.setB(-3.369);  float Acetona = MQ135.readSensor();

  bool alarma = (CO > 50 || Alcohol > 100 || CO2 > 1000 || Tolueno > 100 || NH4 > 50 || Acetona > 200);

  // Buzzer y LED intermitente
  unsigned long currentMillis = millis();
  if (alarma) {
    if (currentMillis - previousMillis >= intervalAlarma) {
      previousMillis = currentMillis;
      alarmState = !alarmState;
      digitalWrite(BUZZER_PIN, alarmState);
      digitalWrite(LED_PIN, alarmState);
    }
  } else {
    digitalWrite(BUZZER_PIN, LOW);
    digitalWrite(LED_PIN, LOW);
    alarmState = false;
  }

  // Mostrar en pantalla
  tft.fillScreen(TFT_BLACK);
  tft.setCursor(0, 0);
  mostrarGas("CO", CO, 50);
  mostrarGas("Alcohol", Alcohol, 100);
  mostrarGas("CO2", CO2, 1000);
  mostrarGas("Tolueno", Tolueno, 100);
  mostrarGas("NH4", NH4, 50);
  mostrarGas("Acetona", Acetona, 200);

  // Enviar datos a Ubidots solo si hay alarma y cada 10 segundos
  if (alarma && WiFi.isConnected() && ubidots.connected()) {
    if (millis() - ultimoEnvio > intervaloPublicacion) {
      ubidots.add("co", CO);
      ubidots.add("alcohol", Alcohol);
      ubidots.add("co2", CO2);
      ubidots.add("tolueno", Tolueno);
      ubidots.add("nh4", NH4);
      ubidots.add("acetona", Acetona);
      ubidots.add("alarma", alarma ? 1 : 0); // 1 si hay alarma, 0 si no
      ubidots.publish(DEVICE_LABEL);
      ultimoEnvio = millis();
      Serial.println("🚨 Datos enviados a Ubidots por alarma.");
    }
  }

  ubidots.loop();
  delay(500); // Breve pausa para no saturar pantalla
}
```