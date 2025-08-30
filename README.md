# Trabajo_1

#include <ArduinoJson.h>
#include <WiFi.h>
#include <HTTPClient.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <max6675.h>

// Librerias del AP/STA
#include "ESPaccesspoint.h"
#include "Settings.h"

#define SCREEN_WIDTH 128 // Ancho del display OLED
#define SCREEN_HEIGHT 64 // Alto del display OLED
#define OLED_RESET -1 // Reset por software (usualmente -1 para ESP32)

/******************\*\*\*\*******************\*\*\*******************\*\*\*\*******************
Importante: para la placa ESP32 con Display OLED, los pines de SDA = Pin 5
y SCL = pin 4. Se definen en las siguientes macros
******************\*\*\*\*******************\*\*\*\*******************\*\*\*\*******************/

#define SDA_PIN 5 // Define el pin SDA del I2C  
#define SCL_PIN 4 // Define el pin SCL del I2C

// Inicialización del objeto display
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Definimos los pines para el sensor MAX6675
#define SCK_PIN 12 // Pin del reloj (SCK)
#define CS_PIN 13 // Pin Chip Select (CS)
#define SO_PIN 15 // Pin de datos del sensor (SO)

// Macro para corrección de error de linealidad
#define LINEAR_FIT(raw_data) ((-0.00403 \* (raw_data)) + 1.601)

// Creamos el objeto del sensor MAX6675
MAX6675 thermocouple(SCK_PIN, CS_PIN, SO_PIN);

// Interpolación y corrección de error
double error;
int ajuste;

unsigned long lastReading = 0;
const unsigned long interval = 1000; // Intervalo de 1 segundo para las lecturas

// Configuracion del servidor quemado
WebServer server(80);
Settings settings;

const char \*serverName = "http://169.254.202.32:1026/v2/entities/AmbientMonitor_001/attrs";

unsigned long lastTime = 0;
unsigned long timerDelay = 5000;

void setup() {
Serial.begin(115200);
Wire.begin(SDA_PIN, SCL_PIN); // Inicializa el bus I2C en los pines definidos

    // Setup para el AP/STA
    EEPROM.begin(4096);
    settings.load();
    settings.info();
    startAPorSTA(settings);


    // Inicializa la pantalla OLED
    if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {  // Cambia 0x3C si la dirección es diferente
        Serial.println("No se pudo encontrar una pantalla OLED SSD1306");
        while (true);  // Detiene el programa si no encuentra la pantalla
    }

    display.clearDisplay();       // Limpia la pantalla
    display.setTextSize(2);       // Tamaño del texto
    display.setTextColor(SSD1306_WHITE);  // Color del texto
    display.setCursor(0, 0);      // Posición inicial del texto

    String mensaje = "Iniciando...";
    Serial.println(mensaje);      // Envía el mensaje por el puerto serial
    display.println(mensaje);     // Muestra el mensaje en la pantalla
    display.display();            // Actualiza el display para mostrar el contenido

    delay(500);  // Espera un momento para que todo esté listo

}

void loop() {

server.handleClient();

    if (millis() - lastReading >= interval) {
        lastReading = millis();
        actualizarTemperatura();
    }

    unsigned long currentMillis = millis();
    if ((currentMillis - lastTime) > timerDelay) {
        if (WiFi.status() != WL_CONNECTED) {
            Serial.println("WiFi Disconnected. Attempting to reconnect...");
            WiFi.disconnect();
            int wifiRetries = 0;
            const int maxReconnectRetries = 5;
            while (WiFi.status() != WL_CONNECTED && wifiRetries < maxReconnectRetries) {
                WiFi.reconnect();
                delay(500);
                Serial.print(".");
                wifiRetries++;
            }
            if (WiFi.status() != WL_CONNECTED) {
                Serial.println("\nFailed to reconnect to WiFi after multiple attempts.");
            }
        }

        if (WiFi.status() == WL_CONNECTED) {
            enviarDatos();
        }

        lastTime = currentMillis; // Actualiza lastTime para el siguiente ciclo
    }

}

void actualizarTemperatura() {
// Leemos la temperatura del MAX6675
double temperatura = thermocouple.readCelsius();
// Ajuste y corrección de error
error = LINEAR*FIT(temperatura);
temperatura -= error;
temperatura = round(temperatura * 4) \_ 0.25; // Redondear al múltiplo más cercano de 0.25

    // Mostramos la temperatura en el monitor serial
    Serial.print("Temp cal: ");
    Serial.print(temperatura);
    Serial.println(" °C");

    // Actualizamos la pantalla OLED
    actualizarPantalla(temperatura);

}

void actualizarPantalla(double temperatura) {
display.fillRect(0, 0, SCREEN_WIDTH, 16, SSD1306_BLACK); // Borra sólo la parte necesaria
display.setCursor(0, 0);
display.print("Temp = ");
display.print(temperatura);
display.println(" C");
display.setCursor(0, 30);
if (WiFi.status() == WL_CONNECTED)
display.println("WiFi OK");
else if (WiFi.getMode() == WIFI_AP)
display.println("Modo AP");

    display.display();

}

void enviarDatos() {
WiFiClient client;
HTTPClient http;
if (http.begin(client, serverName)) {
// Leemos la temperatura
double temperatura = thermocouple.readCelsius();
// Ajuste y corrección de error
error = LINEAR*FIT(temperatura);
temperatura -= error;
temperatura = round(temperatura * 4) \_ 0.25; // Redondear al múltiplo más cercano de 0.25

        // Serializa datos JSON
        String payload = SerializeComplex(temperatura);
        http.addHeader("Content-Type", "application/json");

        // Enviar PATCH request
        int httpResponseCode = http.sendRequest("PATCH", payload);
        Serial.print("HTTP Response code: ");
        Serial.println(httpResponseCode);

        http.end();
    } else {
        Serial.println("Unable to connect to server.");
    }

}

String SerializeComplex(double temperatura) {
String json;
const size_t capacity = JSON_OBJECT_SIZE(3) + JSON_OBJECT_SIZE(2) + 100;
StaticJsonDocument<capacity> doc;

    // Datos de temperatura
    JsonObject temperature_k = doc.createNestedObject("temperature_k");
    temperature_k["value"] = temperatura;
    temperature_k["type"] = "Number";

    //Si se necesita enviar más variables, se crean más instancias tipo JsonObject

    // Datos de geolocalización
    // JsonObject location = doc.createNestedObject("geolocalizacion");
    // location["latitude"] = 6.241508;    // Valor para la latitud
    // location["longitude"] = -75.589644; // Valor para la longitud
    // location["type"] = "GeoCoordinates";

    serializeJson(doc, json);
    Serial.println(json);

    return json;

}

# codigo de un sensor de temperatura
