# Práctica: Sistema básico de monitoreo en tiempo real 

## Objetivo

El estudiante desarrollará un sistema básico de adquisición y transmisión de datos en tiempo real utilizando un ESP32 como nodo sensor, un servidor Node.js con Socket.IO como intermediario de comunicaciones, y dos clientes (navegador web y aplicación en Python con Tkinter) para visualizar las muestras.

Al finalizar la práctica, el alumno será capaz de:

* Transmitir datos desde un microcontrolador hacia un servidor en red.
* Implementar comunicación bidireccional mediante Socket.IO.
* Visualizar datos en tiempo real en una página web.
* Crear una aplicación cliente en Python que reciba datos desde el servidor.
* Comprender el flujo completo de un sistema distribuido básico de telecomunicaciones.

---

## Introducción Teórica

En sistemas industriales de telecomunicaciones es común que un dispositivo sensor:

1. Adquiera una señal física.
2. La convierta a una señal eléctrica.
3. La digitalice.
4. La transmita a un servidor.
5. Sea visualizada o procesada por distintos clientes.

En esta práctica:

* El sensor es una **fotoresistencia (LDR)** conectada en divisor de voltaje.
* El ESP32 digitaliza la señal mediante su ADC.
* El servidor Node.js recibe la muestra.
* Los clientes (web y Python) la visualizan en tiempo real.

Arquitectura del sistema:

```
LDR → ESP32 → WiFi → Servidor Node.js → Web
                                         → Python
```

Además, el sistema permitirá enviar comandos ON/OFF desde los clientes hacia el ESP32 para controlar el LED onboard, implementando comunicación bidireccional.

---

## Material Necesario

* 1 ESP32
* 1 Fotoresistencia (LDR)
* 1 Resistencia (10kΩ recomendada)
* Protoboard y cables
* Computadora con Node.js instalado
* Python 3 instalado
* Red WiFi local

---

## Parte 1: Conexión del sensor

### Divisor de voltaje

Conectar:

* LDR a 3.3V
* Resistencia de 10kΩ a GND
* El punto medio al GPIO 34 del ESP32

El punto medio es la señal analógica.

---

## Parte 2: Programación del ESP32

El ESP32 enviará una muestra cada 500 ms.

### Código del ESP32

```cpp
#include <WiFi.h>
#include <SocketIoClient.h>

const char* ssid     = "MYNETNAME_2.4";
const char* password = "*******";

const char* server   = "172.168.1.127";
const uint16_t port  = 5001;

#define ONBOARD_LED 2

SocketIoClient socketIO;
unsigned long t0 = 0;

void setup() {
  Serial.begin(115200);

  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(200);
    Serial.print(".");
  }

  socketIO.begin(server, port);
  socketIO.on("DESDE_SERVER_COMANDO", procesar_comando);

  pinMode(ONBOARD_LED, OUTPUT);
}

void loop() {
  socketIO.loop();

  unsigned long now = millis();
  if (now - t0 >= 500) {
    t0 = now;

    int adc = analogRead(34);
    String msg = String(adc);

    socketIO.emit("DESDE_ESP32_SENAL", msg.c_str());
  }
}

void procesar_comando(const char* payload, size_t length) {
  String s = String(payload);
  s.trim();
  s.replace("\"", "");

  if (s == "ON") digitalWrite(ONBOARD_LED, HIGH);
  if (s == "OFF") digitalWrite(ONBOARD_LED, LOW);
}
```

Verificar en monitor serial que el ESP32 obtenga IP correctamente.

---

## Parte 3: Servidor Node.js

### Instalación

En una carpeta nueva:

```bash
npm init -y
npm install express socket.io@1.7.2
```

### Estructura del proyecto

```
servidor/
 ├── server.js
 └── public/
      └── index.html
```

### Código del servidor (server.js)

```js
const express = require("express");
const http = require("http");
const socketIo = require("socket.io");

const app = express();
const server = http.createServer(app);
const io = socketIo(server);

const PORT = 5001;

app.use(express.static("public"));

io.on("connection", (socket) => {

  socket.on("DESDE_ESP32_SENAL", (muestra) => {
    io.emit("MUESTRA", String(muestra));
  });

  socket.on("COMANDO", (cmd) => {
    io.emit("DESDE_SERVER_COMANDO", String(cmd));
  });

});

server.listen(PORT, () => {
  console.log("Servidor en puerto", PORT);
});
```

Ejecutar con:

```bash
node server.js
```

---

## Parte 4: Visualización Web

Crear el archivo `public/index.html`.

```html
<!doctype html>
<html>
<head>
<meta charset="utf-8">
<title>Monitor ESP32</title>
<style>
body{font-family:Arial;background:#101820;color:white;margin:0;padding:20px}
.card{background:#1c2735;padding:15px;border-radius:12px;margin-bottom:15px}
.big{font-size:48px;font-weight:bold}
button{padding:10px;margin:5px;border-radius:8px;border:none}
</style>
</head>
<body>

<h2>Monitor de Muestras</h2>

<div class="card">
  <div>Última muestra:</div>
  <div class="big" id="valor">--</div>
</div>

<div class="card">
  <button onclick="enviar('ON')">ON</button>
  <button onclick="enviar('OFF')">OFF</button>
</div>

<ul id="historial"></ul>

<script src="/socket.io/socket.io.js"></script>
<script>
const socket = io();
const valor = document.getElementById("valor");
const historial = document.getElementById("historial");

socket.on("MUESTRA", (m) => {
  valor.textContent = m;

  const li = document.createElement("li");
  li.textContent = m;
  historial.insertBefore(li, historial.firstChild);

  if(historial.children.length > 20)
    historial.removeChild(historial.lastChild);
});

function enviar(cmd){
  socket.emit("COMANDO", cmd);
}
</script>

</body>
</html>
```

Abrir en navegador:

```
http://IP_DEL_SERVER:5001
```

---

## Parte 5: Cliente Python con Tkinter

Instalar:

```bash
pip install "python-socketio<5" "python-engineio<4"
```

### Código `cliente.py`

```python
import threading
import queue
import tkinter as tk
import socketio

SERVER_URL = "http://172.168.1.127:5001"

sio = socketio.Client()
q = queue.Queue()

@sio.on("MUESTRA")
def on_muestra(m):
    q.put(m)

def hilo():
    sio.connect(SERVER_URL)
    sio.wait()

def enviar(cmd):
    sio.emit("COMANDO", cmd)

root = tk.Tk()
root.title("Monitor ESP32")

valor = tk.StringVar(value="--")

tk.Label(root, text="Última muestra", font=("Arial",12)).pack()
tk.Label(root, textvariable=valor, font=("Arial",36)).pack()

tk.Button(root, text="ON", command=lambda: enviar("ON")).pack(fill="x")
tk.Button(root, text="OFF", command=lambda: enviar("OFF")).pack(fill="x")

lista = tk.Listbox(root)
lista.pack(fill="both", expand=True)

def actualizar():
    try:
        while True:
            m = q.get_nowait()
            valor.set(m)
            lista.insert(0, m)
            if lista.size() > 20:
                lista.delete(lista.size()-1)
    except:
        pass
    root.after(100, actualizar)

threading.Thread(target=hilo, daemon=True).start()
actualizar()
root.mainloop()
```

---

## Verificación de Funcionamiento

El sistema funciona correctamente si:

* Al variar la iluminación sobre la LDR, el valor cambia.
* El valor cambia simultáneamente en:

  * Página web.
  * Aplicación Python.
* Los botones ON/OFF controlan el LED del ESP32.

---

## Actividad de Comprobación

El estudiante deberá modificar el sistema para:

1. Agregar en la página web un texto que diga:

   * “Oscuro” si el valor < 1500
   * “Medio” si el valor está entre 1500 y 2500
   * “Claro” si el valor > 2500

2. Implementar la misma clasificación en la aplicación Python.

No modificar la estructura del sistema, únicamente agregar la lógica de clasificación.

---
