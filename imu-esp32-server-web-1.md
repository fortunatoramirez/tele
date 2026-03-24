# Práctica: Visualización remota de movimiento y orientación con IMU, ESP32 y clientes web

## 1. Objetivo

El estudiante desarrollará un sistema distribuido capaz de adquirir datos de movimiento y orientación desde un sensor **IMU 9DoF** conectado a un **ESP32**, enviarlos a un **servidor en una PC** y visualizarlos en **clientes web** mediante una figura que se mueve y rota en tiempo real.

Al finalizar la práctica, el estudiante será capaz de:

* Comprender qué es un sensor IMU y cuáles son sus partes principales.
* Leer datos de aceleración, giro y campo magnético desde un módulo IMU.
* Enviar datos desde un ESP32 hacia un servidor mediante red WiFi.
* Utilizar un servidor Node.js con Socket.IO para redistribuir datos en tiempo real.
* Mostrar gráficamente el movimiento y la rotación de un objeto en una página web.

---

## 2. Introducción teórica

### 2.1 ¿Qué es una IMU?

Una **IMU** (*Inertial Measurement Unit*, o Unidad de Medición Inercial) es un dispositivo electrónico que permite medir el movimiento y la orientación de un objeto. Normalmente integra varios sensores internos que trabajan en conjunto para describir cómo se mueve un sistema en el espacio.

En esta práctica se utilizará una IMU de **9 grados de libertad**, lo que significa que mide información en tres ejes para tres tipos de variables diferentes:

* aceleración
* velocidad angular
* campo magnético

Esto permite obtener una descripción más completa del movimiento de un objeto.

---

### 2.2 ¿Qué es un acelerómetro?

El **acelerómetro** mide la aceleración en los ejes **X, Y y Z**.
Con este sensor es posible detectar:

* inclinación
* vibración
* cambios de movimiento
* orientación aproximada respecto a la gravedad

En esta práctica, el acelerómetro se usará para desplazar una figura en la pantalla.

---

### 2.3 ¿Qué es un giroscopio?

El **giroscopio** mide la velocidad angular en los ejes **X, Y y Z**.
Esto permite detectar giros y cambios de orientación.

En esta práctica, el giroscopio se empleará para hacer que la figura rote sobre su propio eje en la interfaz web.

---

### 2.4 ¿Qué es un magnetómetro?

El **magnetómetro** mide el campo magnético en los ejes **X, Y y Z**.
Su uso más común es funcionar como una brújula digital para estimar dirección o rumbo.

En esta práctica, sus datos serán leídos y enviados, aunque la visualización principal se enfocará en el acelerómetro y el giroscopio para mantener el desarrollo sencillo y claro.

---

### 2.5 Flujo general del sistema

El sistema desarrollado en esta práctica funcionará de la siguiente manera:

**IMU → ESP32 → servidor en PC → clientes web**

1. La IMU mide movimiento y orientación.
2. El ESP32 lee esos datos.
3. El ESP32 los envía por WiFi al servidor.
4. El servidor redistribuye la información a todos los clientes conectados.
5. Cada cliente web actualiza una figura en tiempo real.

---

## 3. Material necesario

* 1 ESP32
* 1 módulo IMU 9DoF SparkFun
* cableado de conexión o cable Qwiic
* 1 computadora con Node.js instalado
* red WiFi local
* Arduino IDE
* navegador web

---

## 5. Conexión del hardware

Si se realiza la conexión por I2C con el ESP32, se puede usar la asignación típica:

* **VCC del IMU** → **3.3V del ESP32**
* **GND del IMU** → **GND del ESP32**
* **SDA del IMU** → **GPIO 21 del ESP32**
* **SCL del IMU** → **GPIO 22 del ESP32**

> Nota: si se utiliza el sistema Qwiic, la conexión puede ser más simple y directa.

---

## 6. Desarrollo de la práctica

---

# Bloque 1. Crear la carpeta del proyecto del servidor

Crear una carpeta de trabajo con la siguiente estructura:

```text
imu_web/
├── server.js
├── package.json
└── public/
    └── index.html
```

---

# Bloque 2. Inicializar el proyecto Node.js

Abrir una terminal dentro de la carpeta del proyecto y ejecutar:

```bash
npm init -y
npm install express socket.io@1.7.2
```

### Explicación

* `npm init -y` crea el archivo `package.json`.
* `express` se utilizará para servir la página web.
* `socket.io@1.7.2` se utilizará para la comunicación en tiempo real y mantener compatibilidad con el ecosistema usado en ESP32.

---

# Bloque 3. Crear el servidor

Crear el archivo `server.js` con el siguiente contenido:

```javascript
const express = require('express');
const http = require('http');
const socketio = require('socket.io');

const app = express();
const server = http.createServer(app);
const io = socketio(server);

app.use(express.static('public'));

io.on('connection', (socket) => {
  console.log('Cliente conectado:', socket.id);

  socket.on('imu_data', (data) => {
    io.emit('imu_data', data);
  });

  socket.on('disconnect', () => {
    console.log('Cliente desconectado:', socket.id);
  });
});

server.listen(3000, () => {
  console.log('Servidor ejecutándose en http://localhost:3000');
});
```

### Explicación

Este servidor realiza tres funciones principales:

* publica la carpeta `public`
* recibe datos enviados por el ESP32
* redistribuye esos datos a todos los clientes web conectados

---

# Bloque 4. Crear la interfaz web

Dentro de la carpeta `public`, crear el archivo `index.html`:

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <title>Visualización IMU</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      text-align: center;
      background: #f2f2f2;
      margin: 0;
      padding: 20px;
    }

    canvas {
      background: white;
      border: 1px solid #ccc;
      margin-top: 20px;
    }

    .panel {
      margin-top: 20px;
      font-size: 18px;
    }
  </style>
</head>
<body>
  <h1>Visualización remota de IMU</h1>
  <canvas id="lienzo" width="700" height="500"></canvas>

  <div class="panel" id="datos"></div>

  <script src="/socket.io/socket.io.js"></script>
  <script>
    const socket = io();

    const canvas = document.getElementById("lienzo");
    const ctx = canvas.getContext("2d");
    const datos = document.getElementById("datos");

    let estado = {
      x: 350,
      y: 250,
      angulo: 0,
      ax: 0,
      ay: 0,
      az: 0,
      gx: 0,
      gy: 0,
      gz: 0
    };

    socket.on("imu_data", (data) => {
      estado = data;
      dibujar();
    });

    function dibujar() {
      ctx.clearRect(0, 0, canvas.width, canvas.height);

      ctx.save();
      ctx.translate(estado.x, estado.y);
      ctx.rotate(estado.angulo * Math.PI / 180);

      ctx.fillStyle = "#2e86de";
      ctx.fillRect(-40, -20, 80, 40);

      ctx.fillStyle = "#e74c3c";
      ctx.fillRect(20, -5, 20, 10);

      ctx.restore();

      datos.innerHTML = `
        <strong>Posición:</strong> x=${estado.x.toFixed(1)}, y=${estado.y.toFixed(1)}<br>
        <strong>Ángulo:</strong> ${estado.angulo.toFixed(1)}°<br>
        <strong>Acelerómetro:</strong> ax=${estado.ax.toFixed(2)}, ay=${estado.ay.toFixed(2)}, az=${estado.az.toFixed(2)}<br>
        <strong>Giroscopio:</strong> gx=${estado.gx.toFixed(2)}, gy=${estado.gy.toFixed(2)}, gz=${estado.gz.toFixed(2)}
      `;
    }

    dibujar();
  </script>
</body>
</html>
```

### Explicación

La interfaz web:

* recibe datos del servidor
* dibuja una figura rectangular
* mueve la figura según la posición calculada
* rota la figura según el ángulo calculado
* muestra los valores numéricos recibidos

---

# Bloque 5. Instalar bibliotecas en Arduino IDE

Instalar las siguientes bibliotecas:

* biblioteca del IMU de SparkFun para el ICM-20948
* `SocketIoClient`
* `ArduinoJson`

---

# Bloque 6. Código del ESP32

Crear el programa para el ESP32:

```cpp
#include <WiFi.h>
#include <Wire.h>
#include <SocketIoClient.h>
#include <ArduinoJson.h>
#include <ICM_20948.h>

const char* ssid = "TU_WIFI";
const char* password = "TU_PASSWORD";
const char* host = "192.168.1.100";
const uint16_t port = 3000;

SocketIoClient socket;
ICM_20948_I2C imu;

float posX = 350;
float posY = 250;
float angulo = 0;

unsigned long tAnterior = 0;

void conectarWiFi() {
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println();
  Serial.println("WiFi conectado");
  Serial.println(WiFi.localIP());
}

void conectarIMU() {
  Wire.begin(21, 22);

  bool listo = false;
  while (!listo) {
    imu.begin(Wire, 0x69);

    if (imu.status == ICM_20948_Stat_Ok) {
      listo = true;
      Serial.println("IMU detectada correctamente");
    } else {
      Serial.println("Error al detectar IMU. Reintentando...");
      delay(1000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);

  conectarWiFi();
  conectarIMU();

  socket.begin(host, port);

  tAnterior = millis();
}

void loop() {
  socket.loop();

  if (imu.dataReady()) {
    imu.getAGMT();

    float ax = imu.accX();
    float ay = imu.accY();
    float az = imu.accZ();

    float gx = imu.gyrX();
    float gy = imu.gyrY();
    float gz = imu.gyrZ();

    float mx = imu.magX();
    float my = imu.magY();
    float mz = imu.magZ();

    unsigned long tActual = millis();
    float dt = (tActual - tAnterior) / 1000.0;
    tAnterior = tActual;

    posX = 350 + ax * 30.0;
    posY = 250 - ay * 30.0;

    if (posX < 40) posX = 40;
    if (posX > 660) posX = 660;
    if (posY < 40) posY = 40;
    if (posY > 460) posY = 460;

    angulo += gz * dt;

    if (angulo >= 360) angulo -= 360;
    if (angulo < 0) angulo += 360;

    StaticJsonDocument<256> doc;
    doc["x"] = posX;
    doc["y"] = posY;
    doc["angulo"] = angulo;

    doc["ax"] = ax;
    doc["ay"] = ay;
    doc["az"] = az;

    doc["gx"] = gx;
    doc["gy"] = gy;
    doc["gz"] = gz;

    doc["mx"] = mx;
    doc["my"] = my;
    doc["mz"] = mz;

    char buffer[256];
    serializeJson(doc, buffer);

    socket.emit("imu_data", buffer);

    delay(40);
  }
}
```

---

# Bloque 7. Explicación del código del ESP32

## Lectura del IMU

La función:

```cpp
imu.getAGMT();
```

actualiza las mediciones del:

* acelerómetro
* giroscopio
* magnetómetro
* temperatura, si aplica

---

## Movimiento en pantalla

La posición horizontal y vertical de la figura se calcula a partir de `ax` y `ay`:

```cpp
posX = 350 + ax * 30.0;
posY = 250 - ay * 30.0;
```

Esto permite que la figura se desplace según la inclinación del módulo.

---

## Rotación en pantalla

La rotación se calcula acumulando la velocidad angular del eje Z:

```cpp
angulo += gz * dt;
```

Así, al girar físicamente el sensor, la figura rota en el navegador.

---

## Envío de datos

Los datos son empaquetados en formato JSON y enviados al servidor:

```cpp
socket.emit("imu_data", buffer);
```

---

# Bloque 8. Ejecutar el sistema

## Paso 1. Iniciar el servidor

En la terminal:

```bash
node server.js
```

---

## Paso 2. Abrir la interfaz web

En la misma computadora:

```text
http://localhost:3000
```

O desde otra computadora o celular conectado a la misma red:

```text
http://IP_DE_LA_PC:3000
```

---

## Paso 3. Cargar el código al ESP32

* colocar nombre de red y contraseña
* colocar la IP de la computadora donde corre el servidor
* compilar y cargar

---

## Paso 4. Probar el movimiento

Mover e inclinar el sensor para observar:

* desplazamiento del objeto
* rotación del objeto
* actualización de valores numéricos

---

## 7. Resultado esperado

Al finalizar correctamente la práctica:

* el ESP32 leerá los datos del IMU
* el servidor recibirá esos datos
* los clientes web mostrarán una figura en movimiento
* la figura rotará cuando el sensor gire
* varios clientes podrán ver simultáneamente la misma información en tiempo real

---

## 8. Ventajas de esta práctica

* introduce sensores inerciales de forma visual
* conecta hardware, red y web en una sola actividad
* permite comprender sistemas distribuidos en tiempo real
* muestra una aplicación clara del acelerómetro y el giroscopio
* sirve como base para proyectos más avanzados

---

## 9. Problemas comunes

### La IMU no responde

Revisar:

* alimentación a 3.3V
* conexiones SDA y SCL
* dirección I2C del módulo

### El ESP32 no se conecta

Revisar:

* nombre de red
* contraseña
* disponibilidad de la red WiFi

### La página no actualiza

Revisar:

* que el servidor esté ejecutándose
* que la IP del servidor sea correcta en el código del ESP32
* que la PC y el ESP32 estén en la misma red

### La figura gira muy rápido

Ajustar el factor de rotación o aplicar una escala menor a `gz`.

---

## 10. Actividad de comprobación

El estudiante deberá realizar al menos una de las siguientes modificaciones:

1. Cambiar la figura rectangular por una flecha o una nave.
2. Mostrar también los valores del magnetómetro en la página.
3. Cambiar el color de la figura según la aceleración.
4. Agregar una segunda figura que responda a otro eje del sensor.
5. Hacer que la interfaz muestre una alerta cuando la inclinación supere cierto límite.



---
---
---


# Actualización final de la práctica

### Visualización remota del módulo SparkFun Qwiic 9DoF IMU Breakout (ISM330DHCX + MMC5983MA)

En esta actualización se trabajará con el módulo de SparkFun **Qwiic 9DoF IMU Breakout - ISM330DHCX + MMC5983MA**. Para esta etapa de la práctica se utilizará principalmente la parte del **ISM330DHCX**, es decir:

* **Acelerómetro**
* **Giroscopio**

La visualización remota se realizará mediante:

* **ESP32**
* **Node.js**
* **Socket.IO**
* **Página web en HTML**

En esta versión, la figura mostrada en pantalla no representa un desplazamiento real en el espacio, sino una **inclinación relativa** de la plaquita IMU con respecto a una posición central calibrada. Esto hace que el comportamiento sea más estable y más claro para observar el efecto del acelerómetro y del giroscopio.

---

## Bibliotecas que se deben instalar en el Arduino IDE

Antes de cargar el programa al ESP32, deben instalarse las siguientes bibliotecas en el Arduino IDE:

1. **SparkFun ISM330DHCX Arduino Library**
2. **SparkFun MMC5983MA Magnetometer Arduino Library**

> Aunque en esta actualización el código utiliza principalmente el **ISM330DHCX**, también se deja instalada la biblioteca del **MMC5983MA**, ya que forma parte del mismo módulo y puede utilizarse en ampliaciones posteriores de la práctica.

---

## Descripción general del funcionamiento

El sistema final funciona de la siguiente manera:

1. El **ESP32** lee los datos del acelerómetro y del giroscopio del módulo IMU.
2. A partir de esos datos calcula:

   * una inclinación lateral (**roll**)
   * una inclinación frontal (**pitch**)
   * una velocidad angular sobre el eje Z (**gz**)
3. El ESP32 envía estos datos a un servidor **Node.js** usando **Socket.IO**.
4. La página web recibe la información y dibuja una figura en un lienzo.
5. El usuario puede calibrar la posición central desde la propia página web.

---

## Nota importante sobre la calibración

En esta implementación, la **posición central** de la plaquita IMU no está fijada por fábrica.
La posición central será la pose en la que el usuario coloque la placa al momento de presionar el botón:

**Calibrar centro**

Por lo tanto, la plaquita puede estar:

* sobre la mesa
* sostenida frente al usuario
* en una ligera inclinación
* o en cualquier orientación deseada

siempre y cuando esa pose sea tomada como referencia mediante la calibración.

---

# 1. Código del ESP32

Este código debe cargarse en el ESP32.
En donde dice `NOMBRE_DE_LA_RED`, `CONTRASENA_DE_LA_RED` y `DIRECCION_IP_DEL_SERVIDOR`, el estudiante debe colocar los datos correspondientes a su red y a la computadora donde se ejecutará el servidor Node.js.

```cpp
#include <WiFi.h>
#include <Wire.h>
#include <SocketIoClient.h>
#include <SparkFun_ISM330DHCX.h>
#include <math.h>

const char* ssid = "NOMBRE_DE_LA_RED";
const char* password = "CONTRASENA_DE_LA_RED";
const char* host = "DIRECCION_IP_DEL_SERVIDOR";
const uint16_t port = 5002;

SocketIoClient socketIO;
SparkFun_ISM330DHCX imu;

// Centro visual
const float CX = 350.0f;
const float CY = 250.0f;

// Estado visual
float posX = CX;
float posY = CY;
float angulo = 0.0f;

// Señales filtradas
float pitchFiltrado = 0.0f;
float rollFiltrado  = 0.0f;
float gzFiltrado    = 0.0f;

// Referencia calibrada
float pitch0 = 0.0f;
float roll0  = 0.0f;
float gz0    = 0.0f;

// Mapeo visual
bool invertirX = false;
bool invertirY = true;

// Ganancias
float gananciaXY = 8.0f;
float gananciaGiro = 0.20f;

// Tiempos
unsigned long tAnterior = 0;
unsigned long tLectura = 0;
unsigned long tEnvio = 0;

// Calibración no bloqueante
bool calibrando = false;
int muestrasCal = 0;
const int N_CAL = 120;

float sumaPitch = 0.0f;
float sumaRoll  = 0.0f;
float sumaGz    = 0.0f;

float aplicarZonaMuerta(float valor, float umbral) {
  if (valor > -umbral && valor < umbral) return 0.0f;
  return valor;
}

float rad2deg(float r) {
  return r * 180.0f / PI;
}

void iniciarCalibracion() {
  calibrando = true;
  muestrasCal = 0;
  sumaPitch = 0.0f;
  sumaRoll  = 0.0f;
  sumaGz    = 0.0f;
  Serial.println("Iniciando calibracion...");
}

void terminarCalibracion() {
  if (muestrasCal > 0) {
    pitch0 = sumaPitch / muestrasCal;
    roll0  = sumaRoll  / muestrasCal;
    gz0    = sumaGz    / muestrasCal;
  }

  pitchFiltrado = 0.0f;
  rollFiltrado  = 0.0f;
  gzFiltrado    = 0.0f;

  posX = CX;
  posY = CY;
  angulo = 0.0f;

  calibrando = false;

  Serial.println("Calibracion terminada");
  Serial.print("pitch0 = "); Serial.println(pitch0, 4);
  Serial.print("roll0  = "); Serial.println(roll0, 4);
  Serial.print("gz0    = "); Serial.println(gz0, 4);
}

void ponerAnguloEnCero() {
  angulo = 0.0f;
  gzFiltrado = 0.0f;
  Serial.println("Angulo en cero");
}

void resetTotalInterno() {
  posX = CX;
  posY = CY;
  angulo = 0.0f;
  pitchFiltrado = 0.0f;
  rollFiltrado  = 0.0f;
  gzFiltrado    = 0.0f;
  Serial.println("Reset total");
}

void onCalibrarOffsets(const char* payload, size_t length) {
  iniciarCalibracion();
}

void onResetOrientacion(const char* payload, size_t length) {
  ponerAnguloEnCero();
}

void onResetTotal(const char* payload, size_t length) {
  resetTotalInterno();
}

void conectarWiFi() {
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(300);
    Serial.print(".");
  }

  Serial.println();
  Serial.println("WiFi conectado");
  Serial.println(WiFi.localIP());
}

void conectarIMU() {
  Wire.begin(21, 22);

  while (!imu.begin(Wire)) {
    Serial.println("Error al detectar ISM330DHCX. Reintentando...");
    delay(1000);
  }

  imu.setAccelFullScale(ISM_2g);
  imu.setGyroFullScale(ISM_250dps);
  imu.setAccelDataRate(ISM_XL_ODR_104Hz);
  imu.setGyroDataRate(ISM_GY_ODR_104Hz);

  Serial.println("ISM330DHCX detectado correctamente");
}

void setup() {
  Serial.begin(115200);
  delay(1000);

  conectarWiFi();
  conectarIMU();

  socketIO.begin(host, port);

  socketIO.on("calibrar_offsets", onCalibrarOffsets);
  socketIO.on("reset_orientacion", onResetOrientacion);
  socketIO.on("reset_total", onResetTotal);

  tAnterior = millis();
  tLectura = millis();
  tEnvio = millis();

  iniciarCalibracion();
}

void loop() {
  socketIO.loop();

  unsigned long ahora = millis();

  if (ahora - tLectura >= 25) {
    tLectura = ahora;

    sfe_ism_data_t accel;
    sfe_ism_data_t gyro;

    bool okA = imu.getAccel(&accel);
    bool okG = imu.getGyro(&gyro);

    if (okA && okG) {
      float ax = accel.xData;
      float ay = accel.yData;
      float az = accel.zData;
      float gz = gyro.zData;

      float pitch = rad2deg(atan2(-ax, sqrt(ay * ay + az * az)));
      float roll  = rad2deg(atan2(ay, az));

      if (calibrando) {
        sumaPitch += pitch;
        sumaRoll  += roll;
        sumaGz    += gz;
        muestrasCal++;

        if (muestrasCal >= N_CAL) {
          terminarCalibracion();
          tAnterior = ahora;
        }
      } else {
        float pitchRel = pitch - pitch0;
        float rollRel  = roll  - roll0;
        float gzRel    = gz    - gz0;

        pitchRel = aplicarZonaMuerta(pitchRel, 1.0f);
        rollRel  = aplicarZonaMuerta(rollRel, 1.0f);
        gzRel    = aplicarZonaMuerta(gzRel, 1.0f);

        pitchFiltrado = 0.90f * pitchFiltrado + 0.10f * pitchRel;
        rollFiltrado  = 0.90f * rollFiltrado  + 0.10f * rollRel;
        gzFiltrado    = 0.90f * gzFiltrado    + 0.10f * gzRel;

        float dx = rollFiltrado * gananciaXY;
        float dy = pitchFiltrado * gananciaXY;

        if (invertirX) dx = -dx;
        if (invertirY) dy = -dy;

        posX = CX + dx;
        posY = CY + dy;

        if (posX < 60.0f) posX = 60.0f;
        if (posX > 640.0f) posX = 640.0f;
        if (posY < 60.0f) posY = 60.0f;
        if (posY > 440.0f) posY = 440.0f;

        float dt = (ahora - tAnterior) / 1000.0f;
        tAnterior = ahora;

        if (dt < 0.001f) dt = 0.001f;
        if (dt > 0.1f) dt = 0.1f;

        angulo += gzFiltrado * dt * gananciaGiro;

        while (angulo >= 360.0f) angulo -= 360.0f;
        while (angulo < 0.0f) angulo += 360.0f;
      }
    }
  }

  if (!calibrando && (ahora - tEnvio >= 120)) {
    tEnvio = ahora;

    char buffer[160];
    snprintf(
      buffer,
      sizeof(buffer),
      "{\"x\":%.1f,\"y\":%.1f,\"angulo\":%.1f,\"pitch\":%.2f,\"roll\":%.2f,\"gz\":%.2f}",
      posX, posY, angulo, pitchFiltrado, rollFiltrado, gzFiltrado
    );

    socketIO.emit("imu_data", buffer);
  }

  delay(1);
}
```

---

# 2. Código del servidor Node.js

Este archivo puede guardarse como `server.js`.

```js
const express = require('express');
const http = require('http');
const socketio = require('socket.io');

const app = express();
const server = http.createServer(app);
const io = socketio(server);

app.use(express.static('public'));

io.on('connection', (socket) => {
  console.log('Cliente conectado:', socket.id);

  socket.on('imu_data', (data) => {
    console.log(data);
    io.emit('imu_data', data);
  });

  socket.on('calibrar_offsets', () => {
    io.emit('calibrar_offsets');
  });

  socket.on('reset_orientacion', () => {
    io.emit('reset_orientacion');
  });

  socket.on('reset_total', () => {
    io.emit('reset_total');
  });

  socket.on('disconnect', () => {
    console.log('Cliente desconectado:', socket.id);
  });
});

server.listen(5002, () => {
  console.log('Servidor ejecutándose en http://localhost:5002');
});
```

---

# 3. Código HTML de visualización

Este archivo debe guardarse como `index.html` dentro de la carpeta `public`.

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <title>Visualización IMU</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      text-align: center;
      background: #f2f2f2;
      margin: 0;
      padding: 20px;
    }

    canvas {
      background: white;
      border: 1px solid #ccc;
      margin-top: 20px;
    }

    .panel, .controles {
      margin-top: 20px;
      font-size: 18px;
      line-height: 1.6;
    }

    button {
      margin: 5px;
      padding: 10px 16px;
      font-size: 16px;
      cursor: pointer;
    }
  </style>
</head>
<body>
  <h1>Visualización remota de IMU</h1>

  <div class="controles">
    <button onclick="calibrarReferencia()">Calibrar centro</button>
    <button onclick="ponerAnguloCero()">Poner ángulo en cero</button>
    <button onclick="resetTotal()">Reset total</button>
  </div>

  <canvas id="lienzo" width="700" height="500"></canvas>
  <div class="panel" id="datos"></div>

  <script src="/socket.io/socket.io.js"></script>
  <script>
    const socket = io();

    const canvas = document.getElementById("lienzo");
    const ctx = canvas.getContext("2d");
    const datos = document.getElementById("datos");

    let estadoObjetivo = {
      x: 350,
      y: 250,
      angulo: 0,
      pitch: 0,
      roll: 0,
      gz: 0
    };

    let estadoVisual = {
      x: 350,
      y: 250,
      angulo: 0,
      pitch: 0,
      roll: 0,
      gz: 0
    };

    socket.on("imu_data", (data) => {
      try {
        const obj = (typeof data === "string") ? JSON.parse(data) : data;

        estadoObjetivo.x = obj.x ?? estadoObjetivo.x;
        estadoObjetivo.y = obj.y ?? estadoObjetivo.y;
        estadoObjetivo.angulo = obj.angulo ?? estadoObjetivo.angulo;
        estadoObjetivo.pitch = obj.pitch ?? 0;
        estadoObjetivo.roll = obj.roll ?? 0;
        estadoObjetivo.gz = obj.gz ?? 0;
      } catch (e) {
        console.error(e);
      }
    });

    function calibrarReferencia() {
      socket.emit("calibrar_offsets");
    }

    function ponerAnguloCero() {
      socket.emit("reset_orientacion");
    }

    function resetTotal() {
      socket.emit("reset_total");
    }

    function lerp(a, b, t) {
      return a + (b - a) * t;
    }

    function normDelta(a) {
      while (a > 180) a -= 360;
      while (a < -180) a += 360;
      return a;
    }

    function animar() {
      estadoVisual.x = lerp(estadoVisual.x, estadoObjetivo.x, 0.15);
      estadoVisual.y = lerp(estadoVisual.y, estadoObjetivo.y, 0.15);

      let da = normDelta(estadoObjetivo.angulo - estadoVisual.angulo);
      estadoVisual.angulo += da * 0.12;

      estadoVisual.pitch = lerp(estadoVisual.pitch, estadoObjetivo.pitch, 0.15);
      estadoVisual.roll = lerp(estadoVisual.roll, estadoObjetivo.roll, 0.15);
      estadoVisual.gz = lerp(estadoVisual.gz, estadoObjetivo.gz, 0.15);

      dibujar();
      requestAnimationFrame(animar);
    }

    function dibujar() {
      ctx.clearRect(0, 0, canvas.width, canvas.height);

      ctx.strokeStyle = "#ddd";
      ctx.beginPath();
      ctx.moveTo(350, 0);
      ctx.lineTo(350, 500);
      ctx.moveTo(0, 250);
      ctx.lineTo(700, 250);
      ctx.stroke();

      ctx.save();
      ctx.translate(estadoVisual.x, estadoVisual.y);
      ctx.rotate(estadoVisual.angulo * Math.PI / 180);

      ctx.fillStyle = "#2e86de";
      ctx.fillRect(-40, -20, 80, 40);

      ctx.fillStyle = "#e74c3c";
      ctx.fillRect(20, -5, 20, 10);

      ctx.restore();

      datos.innerHTML = `
        <strong>Centro calibrado:</strong> la pose al presionar "Calibrar centro"<br>
        <strong>Posición visual por inclinación:</strong> x=${estadoVisual.x.toFixed(1)}, y=${estadoVisual.y.toFixed(1)}<br>
        <strong>Ángulo visual:</strong> ${estadoVisual.angulo.toFixed(1)}°<br>
        <strong>Pitch:</strong> ${estadoVisual.pitch.toFixed(2)}°<br>
        <strong>Roll:</strong> ${estadoVisual.roll.toFixed(2)}°<br>
        <strong>Velocidad angular Z:</strong> ${estadoVisual.gz.toFixed(2)} °/s<br><br>

        <strong>Interpretación:</strong><br>
        - Derecha/izquierda en pantalla: inclinación lateral (<code>roll</code>)<br>
        - Arriba/abajo en pantalla: inclinación frontal (<code>pitch</code>)<br>
        - Rotación: giro sobre eje Z (<code>gz</code>)<br>
        - No representa desplazamiento real en el espacio, sino inclinación relativa.
      `;
    }

    requestAnimationFrame(animar);
  </script>
</body>
</html>
```

---

## Organización sugerida de carpetas

La estructura del proyecto Node.js puede organizarse así:

```text
proyecto_imu/
│
├── server.js
├── package.json
└── public/
    └── index.html
```

---

## Ejecución del servidor

Dentro de la carpeta del proyecto, se pueden instalar las dependencias con:

```bash
npm install express socket.io
```

Después, el servidor puede ejecutarse con:

```bash
node server.js
```

---

## Observación final

Esta actualización permite trabajar con una visualización remota más estable del IMU, usando la inclinación como referencia principal. El resultado es más adecuado para observar el comportamiento del sensor y para comprender mejor la diferencia entre:

* **inclinación**
* **giro**
* **desplazamiento real**


