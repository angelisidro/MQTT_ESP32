# MQTT con ESP32 y Adafruit IO

Este repositorio contiene información acerca del uso del protocolo de comunicación MQTT con la plataforma Adafruit IO y la tarjeta utilizada contiene el chip ESP32 de Espressif, utilizando un sensor BME280 de la marca BOSCH.
![](/Img/Back.jpg)

# Requisitos 
- Crear cuenta en Adafruit 'www.adafruit.com'
- Contar con ESP32 en cualquiera de las versiones existentes.
- Modulo BME280 

# Pines de protocolo I2C

Sensor BME280 integrado en la tarjeta Smart Home.

Nombre | GPIO PIN
--- | ---
SCL | GPIO22
SDA | GPIO23

# Dashboard 

Esta es una vista previa del dashboard implementado en la plataforma de Adafruit IO 
' https://io.adafruit.com/angelisidro/dashboards/temperatura '
![](/Img/Dashboard.PNG)

# Código Base

```cpp
/***************************************************
  Ejemplo del uso de la libreria Adafruit MQTT con ESP32
  Para más información revise el repositorio en
  ----> https://io.adafruit.com/angelisidro/dashboards/temperatura
  Puede visualizar el dashboard para medición de temperatura, humedad y presión atmosférica.
  ----> https://io.adafruit.com/angelisidro/dashboards/temperatura
  El ESP32 utilizado en este ejemplo es una tarjeta diseñada por Ángel Isidro en una implementación
  para el Foro de Innovación Tecnologica FIT en Ciudad de Guatemala, Centro America.
  
 ****************************************************/
 
/*BME280 variables y librerias*/
#include <Wire.h> // Libreria para comunicación I2C
#include <Adafruit_Sensor.h> 
#include <Adafruit_BME280.h>
#define SEALEVELPRESSURE_HPA (1013.25)

float temp=0; //Variable para la temperatura
float hume=0; // Variable para la humedad
float pressure = 0; // Variable para la presión atmosférica

Adafruit_BME280 bme; // Objeto de tipo BME280

// Librerias para WiFi y MQTT en ESP32
#include <WiFi.h>
#include "Adafruit_MQTT.h"
#include "Adafruit_MQTT_Client.h"

/************************* Credenciales para acceso a WiFi *********************************/

// Credenciales WiFi 
const char* ssid     = "Cuarto_De_Juego"; // Nombre de la Red dentro las comillas
const char* password = "A15004127";       // Contraseña de la Red dentro de las comillas


/************************* Configuración para Adafruit.io *********************************/

#define AIO_SERVER      "io.adafruit.com"       // Servidor para Adafruit.io
#define AIO_SERVERPORT  1883                    // Usar puerto 8883 para SSL
#define AIO_USERNAME    "angelisidro"           // Colocar usuario de Adafruit.io
#define AIO_KEY         "aio_tpGf08XunL8kpUJbHebfC4SHnYHW"      //Colocar la Key para Adafruit.io

/************ Global State (you don't need to change this!) ******************/

// Crea an ESP32 WiFiClient class para conectar al MQTT server.
WiFiClient client;

// Configuración para MQTT cliente con las credenciales del WiFi y el MQTT SERVER.
Adafruit_MQTT_Client mqtt(&client, AIO_SERVER, AIO_SERVERPORT, AIO_USERNAME, AIO_KEY);

/****************************** Feeds ***************************************/

// MQTT Paths para Adafruit.io con la siguiente forma <username>/feeds/<feedname>

Adafruit_MQTT_Publish temperatura = Adafruit_MQTT_Publish(&mqtt, AIO_USERNAME "/feeds/Temperatura");
Adafruit_MQTT_Publish humedad = Adafruit_MQTT_Publish(&mqtt, AIO_USERNAME "/feeds/Humedad");
Adafruit_MQTT_Publish presion = Adafruit_MQTT_Publish(&mqtt, AIO_USERNAME "/feeds/PresionAtmosferica");

Adafruit_MQTT_Subscribe onoffbutton = Adafruit_MQTT_Subscribe(&mqtt, AIO_USERNAME "/feeds/onoff");
/***************************  Codigo ************************************/

void MQTT_connect();

void setup() {
  Serial.begin(115200);
  delay(10);
  
  Serial.println(F("Adafruit MQTT demo"));

  // Conexión WiFi access point.
  
  Serial.println(); Serial.println();
  Serial.print("Conectando a ");
  Serial.println(ssid);

  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println();

  Serial.println("WiFi conectado ");
  Serial.println("IP address: "); Serial.println(WiFi.localIP());

  //configuración MQTT para la subscripción para encendido / apagado.
  mqtt.subscribe(&onoffbutton);
  // Revisión para verificar si las conexiones el BME280 estan correctas  
if (!bme.begin(0x77)) {
    Serial.println("Could not find a valid BME280 sensor, check wiring!");
    while (1);
  }
  
}

uint32_t x=0;

void loop() {
  
  MQTT_connect();

  // Espera por la llegada de paquetes a nuestra subscripción
  Adafruit_MQTT_Subscribe *subscription;
  while ((subscription = mqtt.readSubscription(1000))) {
    if (subscription == &onoffbutton) {
      Serial.print(F("Got: "));
      Serial.println((char *)onoffbutton.lastread);
    }
  }

  // Publicamos la temperatura
  Serial.print(F("\nEnviando Temperatura "));
  temp = bme.readTemperature();
  Serial.print(temp);
  Serial.print("...");
  if (! temperatura.publish(temp)) {
    Serial.println(F("Fallido"));
  } else {
    Serial.println(F("OK!"));
  }

  // Publicamos la humedad
  Serial.print(F("\nEnviando Humedad val "));
  hume = bme.readHumidity();
  Serial.print(hume);
  Serial.print("...");
  if (! humedad.publish(hume)) {
    Serial.println(F("Fallido"));
  } else {
    Serial.println(F("OK!"));
  }

    // Publicamos la presion
  Serial.print(F("\n Enviando Presión Atmosférica "));
  pressure = bme.readPressure() / 100.0F;
  Serial.print(pressure);
  Serial.print("...");
  if (! presion.publish(pressure)) {
    Serial.println(F("Fallido"));
  } else {
    Serial.println(F("OK!"));
  }

  //Nos desconectamos mientras no enviamos datos
  
  if(! mqtt.ping()) {
    mqtt.disconnect();
  }

  delay(1200000);//Con este delay enviamos cada 20 Minutos
}

// Function to connect and reconnect as necessary to the MQTT server.
// Should be called in the loop function and it will take care if connecting.
void MQTT_connect() {
  int8_t ret;

  // Para si esta lista la conexión.
  if (mqtt.connected()) {
    return;
  }

  Serial.print("Conectando a MQTT... ");

  uint8_t retries = 3;
  while ((ret = mqtt.connect()) != 0) { // Retornamos un  cuando nos conectamos
       Serial.println(mqtt.connectErrorString(ret));
       Serial.println("Intentando nuevamente conectarse en 5 segundos.....");
       mqtt.disconnect();
       delay(5000);  // Esperamos 5 secondos
       retries--;
       if (retries == 0) {
         // Basicamente esta muerto y espera un reset
         while (1);
       }
  }
  Serial.println("MQTT Conectado!");
}
```