# Práctica: Visualización en tiempo real de señal muscular MyoWare con ESP32, Node.js y cliente web

## Objetivo

Desarrollar un sistema distribuido que permita adquirir la señal de un sensor muscular **MyoWare** mediante un **ESP32**, enviarla a un **servidor Node.js** y visualizarla en un **cliente web** en tiempo real por medio de:

* una gráfica de la señal,
* una barra de intensidad,
* y un indicador activado por umbral.

---

## Introducción

El sensor **MyoWare** permite medir la actividad eléctrica superficial de un músculo. En esta práctica se utilizará la salida **ENV** (envolvente), ya que entrega una señal más estable y fácil de interpretar visualmente que una señal cruda.

El sistema completo seguirá la siguiente ruta de comunicación:

**MyoWare → ESP32 → servidor Node.js → cliente web**

El ESP32 leerá el valor analógico de la salida ENV del sensor, aplicará un filtrado simple para suavizar la visualización y enviará los datos al servidor mediante **Socket.IO**. Posteriormente, el servidor reenviará esos datos al navegador para que el usuario pueda observar:

* el valor actual,
* la gráfica en tiempo real,
* una barra vertical de intensidad,
* y un indicador que cambia de estado cuando la señal supera un umbral ajustable.

---

## Material necesario

* 1 ESP32
* 1 sensor MyoWare
* 3 electrodos adhesivos para EMG
* cables de conexión
* una computadora con Node.js instalado
* red WiFi local
* Arduino IDE
* Visual Studio Code o editor equivalente

---

## Fundamento general del sistema

El sistema funciona de la siguiente manera:

1. El sensor MyoWare detecta la actividad muscular.
2. El ESP32 lee por entrada analógica la señal de la envolvente.
3. El ESP32 aplica un filtrado exponencial simple.
4. El ESP32 envía los datos al servidor Node.js usando Socket.IO.
5. El servidor reenvía los datos a todos los clientes conectados.
6. El navegador muestra la señal en forma gráfica y activa un indicador cuando el valor supera un umbral.

---

## Conexión del hardware

### Conexión del MyoWare al ESP32

Se utilizará la salida **ENV** del MyoWare.

* **VIN del MyoWare** → **3.3 V del ESP32**
* **GND del MyoWare** → **GND del ESP32**
* **ENV del MyoWare** → **GPIO 34 del ESP32**

### Importante

No debe alimentarse el MyoWare con 5 V si la salida ENV se conectará directamente al ESP32, ya que el ADC del ESP32 trabaja con niveles cercanos a 3.3 V.

---

## Colocación de electrodos

Para obtener una señal correcta, los electrodos deben colocarse adecuadamente sobre la piel.

* Los dos electrodos principales deben ir sobre el mismo músculo.
* El electrodo de referencia debe colocarse en una zona neutra o con poca actividad muscular.
* La piel debe estar limpia y seca.
* Si existe vello abundante, conviene retirarlo para mejorar el contacto.

### Ejemplo en bíceps

* **Electrodo 1**: sobre el músculo
* **Electrodo 2**: unos centímetros más abajo, sobre el mismo músculo
* **Referencia**: cerca del codo o en una zona neutra

Si la colocación es incorrecta, la señal puede verse saturada, plana o muy ruidosa.

---

## Estructura del proyecto

La carpeta del proyecto puede organizarse de la siguiente manera:

```text
myoware_web/
├── server.js
├── package.json
└── public/
    └── index.html
```

---

## Bloque 1. Creación del proyecto Node.js

Abrir una terminal y ejecutar:

```bash
mkdir myoware_web
cd myoware_web
npm init -y
npm install express socket.io
```

---

## Bloque 2. Código del ESP32

Este programa realiza las siguientes tareas:

* se conecta a la red WiFi,
* establece comunicación con el servidor Socket.IO,
* lee el valor analógico del MyoWare,
* aplica un filtrado exponencial simple,
* y envía tanto el valor crudo como el filtrado.

### Archivo para Arduino IDE

```cpp
#include <WiFi.h>
#include <SocketIoClient.h>

const char* ssid = "NOMBRE_DE_RED";
const char* password = "CONTRASENA";
const char* host = "DIRECCION_IP_DEL_SERVER";
const uint16_t port = 5002;   

SocketIoClient socketIO;

const int pinMyoWare = 34;

// Variables de lectura
unsigned long tLectura = 0;
unsigned long tEnvio = 0;

int valorRaw = 0;
float valorFiltrado = 0.0f;

// Parámetros
const int intervaloLectura = 10;   // ms
const int intervaloEnvio   = 40;   // ms
const float alpha = 0.15f;         // filtro simple

void onConnect(const char* payload, size_t length) {
  Serial.println("Conectado al servidor Socket.IO");
}

void onDisconnect(const char* payload, size_t length) {
  Serial.println("Desconectado del servidor Socket.IO");
}

void conectarWiFi() {
  WiFi.begin(ssid, password);

  Serial.print("Conectando a WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(300);
    Serial.print(".");
  }

  Serial.println();
  Serial.println("WiFi conectado");
  Serial.print("IP del ESP32: ");
  Serial.println(WiFi.localIP());
}

void setup() {
  Serial.begin(115200);
  delay(1000);

  analogReadResolution(12);   // valores de 0 a 4095

  conectarWiFi();

  socketIO.begin(host, port);

  // Eventos opcionales
  socketIO.on("connect", onConnect);
  socketIO.on("disconnect", onDisconnect);

  tLectura = millis();
  tEnvio = millis();
}

void loop() {
  socketIO.loop();

  unsigned long ahora = millis();

  // Lectura rápida del sensor
  if (ahora - tLectura >= intervaloLectura) {
    tLectura = ahora;

    valorRaw = analogRead(pinMyoWare);

    // filtro exponencial simple
    valorFiltrado = (1.0f - alpha) * valorFiltrado + alpha * valorRaw;

    Serial.print("RAW: ");
    Serial.print(valorRaw);
    Serial.print("  FILTRADO: ");
    Serial.println(valorFiltrado, 2);
  }

  // Envío al server
  if (ahora - tEnvio >= intervaloEnvio) {
    tEnvio = ahora;

    char buffer[120];
    snprintf(
      buffer,
      sizeof(buffer),
      "{\"valor\":%d,\"filtrado\":%.2f}",
      valorRaw,
      valorFiltrado
    );

    socketIO.emit("myoware_data", buffer);
  }

  delay(1);
}
```

---

## Explicación del código del ESP32

### Conexión WiFi

La función `conectarWiFi()` establece conexión con la red inalámbrica. En esta práctica, el nombre de red y la contraseña deberán ser colocados manualmente por el alumno.

### Lectura analógica

La línea:

```cpp
valorRaw = analogRead(pinMyoWare);
```

obtiene el valor del ADC del ESP32 en un rango de 0 a 4095.

### Filtrado simple

La línea:

```cpp
valorFiltrado = (1.0f - alpha) * valorFiltrado + alpha * valorRaw;
```

aplica un filtro exponencial simple. Esto ayuda a suavizar la señal para que la gráfica y la barra no se vean demasiado nerviosas.

### Envío al servidor

Los datos se empaquetan en formato JSON y se envían mediante:

```cpp
socketIO.emit("myoware_data", buffer);
```

---

## Bloque 3. Código del servidor Node.js

Este servidor recibe los datos enviados por el ESP32 y los reenvía a los clientes conectados.

### Archivo `server.js`

```javascript
const express = require('express');
const http = require('http');
const socketIO = require('socket.io');

const app = express();
const server = http.createServer(app);
const io = socketIO(server);

const PORT = 5002;

app.use(express.static('public'));

io.on('connection', function(socket) {
    console.log('Cliente conectado');

    socket.on('myoware_data', function(data) {
        // data puede llegar como string JSON o como objeto
        console.log(data);
        let paquete = data;

        if (typeof data === 'string') {
            try {
                paquete = JSON.parse(data);
            } catch (e) {
                console.log('Error al parsear myoware_data:', e);
                return;
            }
        }

        io.emit('myoware_data', paquete);
    });

    socket.on('disconnect', function() {
        console.log('Cliente desconectado');
    });
});

server.listen(PORT, function() {
    console.log('Servidor corriendo en http://localhost:' + PORT);
});
```

---

## Explicación del servidor

### Express

Se usa para servir la carpeta `public`, donde estará almacenado el archivo HTML del cliente.

### Socket.IO

Se encarga de:

* recibir los datos del ESP32,
* interpretar el contenido JSON si llega como texto,
* y reenviarlo a todos los clientes conectados.

---

## Bloque 4. Código del cliente web

El cliente web que compartiste muestra en tiempo real la señal, una barra de nivel, un valor numérico y un indicador activado por umbral. 

### Archivo `public/index.html`

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <title>MyoWare en tiempo real</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <style>
    body {
      font-family: Arial, sans-serif;
      background: #f2f2f2;
      margin: 0;
      padding: 20px;
      text-align: center;
    }

    .contenedor {
      max-width: 1000px;
      margin: auto;
      background: white;
      border-radius: 14px;
      padding: 20px;
      box-shadow: 0 4px 14px rgba(0,0,0,0.12);
    }

    canvas {
      background: white;
      border: 1px solid #ccc;
      margin-top: 20px;
    }

    .panel {
      display: flex;
      justify-content: center;
      flex-wrap: wrap;
      gap: 20px;
      margin-top: 20px;
    }

    .bloque {
      background: #f8f8f8;
      border: 1px solid #ddd;
      border-radius: 10px;
      padding: 15px;
      min-width: 220px;
    }

    .barra-externa {
      width: 70px;
      height: 220px;
      border: 2px solid #444;
      margin: auto;
      position: relative;
      border-radius: 10px;
      overflow: hidden;
      background: #eee;
    }

    .barra-interna {
      position: absolute;
      bottom: 0;
      width: 100%;
      height: 0%;
      background: linear-gradient(to top, #2e7d32, #8bc34a);
    }

    .indicador {
      width: 80px;
      height: 80px;
      border-radius: 50%;
      background: #aaa;
      margin: 10px auto;
      transition: 0.1s;
    }

    .activo {
      background: red;
      box-shadow: 0 0 20px rgba(255,0,0,0.8);
    }

    .valor {
      font-size: 24px;
      font-weight: bold;
    }

    input[type=range] {
      width: 220px;
    }
  </style>
</head>
<body>
  <div class="contenedor">
    <h1>Señal MyoWare en tiempo real</h1>

    <canvas id="grafica" width="900" height="300"></canvas>

    <div class="panel">
      <div class="bloque">
        <h3>Valor actual</h3>
        <div class="valor" id="valorActual">0</div>
      </div>

      <div class="bloque">
        <h3>Barra de intensidad</h3>
        <div class="barra-externa">
          <div class="barra-interna" id="barraNivel"></div>
        </div>
      </div>

      <div class="bloque">
        <h3>Indicador</h3>
        <div class="indicador" id="indicador"></div>
        <div id="estadoTexto">INACTIVO</div>
      </div>

      <div class="bloque">
        <h3>Umbral</h3>
        <input type="range" id="umbral" min="0" max="4095" value="1200">
        <p>Umbral: <span id="umbralTexto">1200</span></p>
      </div>
    </div>
  </div>

  <script src="/socket.io/socket.io.js"></script>
  <script>
    var socket = io();

    const canvas = document.getElementById('grafica');
    const ctx = canvas.getContext('2d');

    const valorActual = document.getElementById('valorActual');
    const barraNivel = document.getElementById('barraNivel');
    const indicador = document.getElementById('indicador');
    const estadoTexto = document.getElementById('estadoTexto');
    const umbral = document.getElementById('umbral');
    const umbralTexto = document.getElementById('umbralTexto');

    let datos = new Array(180).fill(0);

    umbral.addEventListener('input', function() {
      umbralTexto.textContent = umbral.value;
      dibujarGrafica();
    });

    socket.on('myoware_data', function(data) {
      let valor = 0;

      if (typeof data === 'string') {
        try {
          data = JSON.parse(data);
        } catch (e) {
          return;
        }
      }

      // Preferimos el valor filtrado para visualización
      valor = data.filtrado !== undefined ? data.filtrado : data.valor;

      valorActual.textContent = Number(valor).toFixed(1);

      datos.push(valor);
      if (datos.length > 180) datos.shift();

      let porcentaje = (valor / 4095) * 100;
      if (porcentaje < 0) porcentaje = 0;
      if (porcentaje > 100) porcentaje = 100;
      barraNivel.style.height = porcentaje + '%';

      if (valor >= parseFloat(umbral.value)) {
        indicador.classList.add('activo');
        estadoTexto.textContent = 'ACTIVO';
      } else {
        indicador.classList.remove('activo');
        estadoTexto.textContent = 'INACTIVO';
      }

      dibujarGrafica();
    });

    function dibujarGrafica() {
      ctx.clearRect(0, 0, canvas.width, canvas.height);

      // cuadrícula
      ctx.strokeStyle = '#e0e0e0';
      ctx.lineWidth = 1;

      for (let x = 0; x <= canvas.width; x += 50) {
        ctx.beginPath();
        ctx.moveTo(x, 0);
        ctx.lineTo(x, canvas.height);
        ctx.stroke();
      }

      for (let y = 0; y <= canvas.height; y += 50) {
        ctx.beginPath();
        ctx.moveTo(0, y);
        ctx.lineTo(canvas.width, y);
        ctx.stroke();
      }

      // línea de umbral
      let yUmbral = canvas.height - (parseFloat(umbral.value) / 4095) * canvas.height;
      ctx.strokeStyle = 'red';
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.moveTo(0, yUmbral);
      ctx.lineTo(canvas.width, yUmbral);
      ctx.stroke();

      // señal
      ctx.strokeStyle = '#1976d2';
      ctx.lineWidth = 2;
      ctx.beginPath();

      for (let i = 0; i < datos.length; i++) {
        let x = (i / (datos.length - 1)) * canvas.width;
        let y = canvas.height - (datos[i] / 4095) * canvas.height;

        if (i === 0) ctx.moveTo(x, y);
        else ctx.lineTo(x, y);
      }

      ctx.stroke();
    }

    dibujarGrafica();
  </script>
</body>
</html>
```

---

## Explicación del cliente web

El cliente web realiza tres funciones principales:

### Visualización numérica

Muestra el valor actual de la señal.

### Barra de intensidad

La barra vertical aumenta o disminuye su altura según la magnitud del valor recibido.

### Indicador con umbral

Cuando la señal supera el valor del umbral ajustado por el usuario, el indicador cambia a estado **ACTIVO**. Si la señal baja por debajo del umbral, el indicador vuelve a **INACTIVO**. Esta lógica está implementada en el código del cliente adjunto. 

### Gráfica en tiempo real

La señal se dibuja en un elemento `canvas`, mostrando una línea continua y una línea roja horizontal que representa el umbral actual. 

---

## Bloque 5. Preparación de carpetas y archivos

Dentro de la carpeta del proyecto crear:

* el archivo `server.js`
* la carpeta `public`
* dentro de `public`, el archivo `index.html`

La estructura final debe quedar así:

```text
myoware_web/
├── server.js
├── package.json
└── public/
    └── index.html
```

---

## Bloque 6. Ejecución del sistema

### Paso 1. Iniciar el servidor

Desde la carpeta del proyecto ejecutar:

```bash
node server.js
```

Si todo es correcto, en la terminal deberá mostrarse:

```text
Servidor corriendo en http://localhost:5002
```

### Paso 2. Abrir el cliente web

En el navegador abrir:

```text
http://localhost:5002
```

o bien desde otra computadora de la red:

```text
http://IP_DEL_SERVIDOR:5002
```

### Paso 3. Configurar el ESP32

Antes de cargar el código al ESP32, el alumno deberá sustituir manualmente:

* `NOMBRE_DE_RED`
* `CONTRASENA`
* `DIRECCION_IP_DEL_SERVER`

Después deberá compilar y cargar el programa.

---

## Bloque 7. Prueba de funcionamiento

Una vez que el sistema esté en ejecución:

1. Colocar correctamente los electrodos.
2. Relajar el músculo y observar el valor en reposo.
3. Contraer el músculo.
4. Verificar que:

   * la gráfica cambie en tiempo real,
   * la barra aumente su nivel,
   * el valor numérico se incremente,
   * y el indicador cambie a activo cuando se supere el umbral.

---

## Posibles problemas y recomendaciones

### Señal saturada

Si el valor permanece demasiado alto todo el tiempo:

* revisar la posición de electrodos,
* reducir la ganancia en el sensor,
* comprobar que la salida usada sea ENV,
* confirmar que VIN esté conectado a 3.3 V.

### Señal muy ruidosa

Si la señal se mueve demasiado incluso sin contracción:

* limpiar mejor la piel,
* asegurar un buen contacto de electrodos,
* revisar el cableado,
* o disminuir ligeramente la ganancia.

### Señal muy débil

Si la señal casi no cambia:

* aumentar con cuidado la ganancia,
* revisar la colocación de electrodos,
* y verificar que ambos electrodos principales estén sobre el mismo músculo.

---

## Actividad de comprobación

Modificar el sistema para agregar una acción visual adicional cuando se supere el umbral. Algunas opciones son:

* cambiar el color de toda la página,
* mover un cuadro de izquierda a derecha,
* mostrar un mensaje de “contracción detectada”,
* o hacer que aparezca una figura adicional en pantalla.
