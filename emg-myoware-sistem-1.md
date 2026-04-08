
# PRÁCTICA

# Sistema dual de señales musculares en tiempo real

## Arduino + ESP32 + Node.js + Socket.IO 1.7.2

---

## 1. Objetivo

Desarrollar un sistema distribuido capaz de adquirir, transmitir y visualizar **dos señales musculares (EMG)** en tiempo real utilizando:

* Arduino (RedBoard del kit MyoWare) + Python
* ESP32 con comunicación directa al servidor
* Node.js con Socket.IO 1.7.2
* Interfaz web interactiva

El sistema permitirá:

* visualizar ambas señales en tiempo real
* mostrar barras dinámicas de nivel
* ajustar un umbral desde la interfaz
* activar un evento visual al superar el umbral
* operar correctamente aunque solo uno de los canales esté activo

---

## 2. Introducción teórica

El sensor **MyoWare 2.0** permite medir la actividad muscular mediante electromiografía superficial (EMG). La señal de salida es analógica y proporcional al nivel de contracción muscular.

En esta práctica se implementa una arquitectura distribuida:

* adquisición de señal biomédica
* transmisión de datos por diferentes medios
* procesamiento en servidor
* visualización en tiempo real

Se integran conceptos de:

* adquisición analógica
* comunicación serial
* comunicación HTTP
* WebSockets (Socket.IO)
* visualización de datos

---

## 3. Material necesario

* Kit SparkFun MyoWare 2.0 (KIT-21269)
* RedBoard Plus (incluido)
* ESP32
* sensores MyoWare
* electrodos
* computadora con Node.js y Python
* red WiFi
* cables USB

---

## 4. Arquitectura del sistema

```text
Canal A:
MyoWare → Arduino → Serial → Python → HTTP → Node.js → Web

Canal B:
MyoWare → ESP32 → Socket.IO → Node.js → Web
```

---

## 5. Estructura del proyecto

```text
dual_emg/
│
├── server.js
├── package.json
├── serial_bridge.py
│
└── public/
    ├── index.html
    ├── style.css
    └── app.js
```

---

# 6. Instalación

## Node.js

```bash
npm install
```

## Python

```bash
pip install pyserial requests
```

---

# 7. Desarrollo

---

# BLOQUE 1. package.json

```json
{
  "name": "dual-emg-monitor",
  "version": "1.0.0",
  "description": "Sistema dual EMG",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "body-parser": "^1.20.2",
    "express": "^4.18.2",
    "socket.io": "1.7.2"
  }
}
```

---

# BLOQUE 2. server.js

```javascript
const express = require('express');
const http = require('http');
const bodyParser = require('body-parser');
const path = require('path');

const app = express();
const server = http.createServer(app);
const io = require('socket.io')(server);

app.use(bodyParser.json());
app.use(express.static(path.join(__dirname, 'public')));

const PORT = 3000;

const state = {
  arduino: { connected:false, level:0, last:0 },
  esp32: { connected:false, level:0, last:0 }
};

function normalize(v,max){
  return Math.round((v/max)*100);
}

function update(ch,val){
  const max = ch==="esp32"?4095:1023;
  const level = normalize(val,max);

  state[ch].level = level;
  state[ch].connected = true;
  state[ch].last = Date.now();

  io.emit('signal',{channel:ch,level});
}

app.post('/arduino',(req,res)=>{
  update('arduino',req.body.value);
  res.send({ok:true});
});

io.on('connection',(socket)=>{
  socket.emit('init',state);

  socket.on('esp32',(data)=>{
    let d = typeof data==="string"?JSON.parse(data):data;
    update('esp32',d.value);
  });
});

setInterval(()=>{
  const now=Date.now();
  ['arduino','esp32'].forEach(c=>{
    state[c].connected = (now-state[c].last<3000);
  });
  io.emit('status',state);
},500);

server.listen(PORT,()=>{
  console.log("http://localhost:3000");
});
```

---

# BLOQUE 3. index.html

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Dual EMG</title>
<link rel="stylesheet" href="style.css">
</head>
<body>

<h1>Monitor EMG</h1>

Umbral: <span id="tv">60</span>%
<input type="range" id="th" min="0" max="100" value="60">

<div class="panel">
<h2>Arduino</h2>
<canvas id="ca"></canvas>
<div class="bar"><div id="ba"></div></div>
<div id="oa" class="orb"></div>
</div>

<div class="panel">
<h2>ESP32</h2>
<canvas id="ce"></canvas>
<div class="bar"><div id="be"></div></div>
<div id="oe" class="orb"></div>
</div>

<script src="/socket.io/socket.io.js"></script>
<script src="app.js"></script>

</body>
</html>
```

---

# BLOQUE 4. style.css

```css
body{
font-family:Arial;
text-align:center;
background:#eef;
}

.panel{
background:white;
padding:20px;
margin:20px;
}

canvas{
width:300px;
height:150px;
background:#ddd;
}

.bar{
width:30px;
height:150px;
background:#ccc;
margin:auto;
}

.bar div{
background:blue;
width:100%;
height:0%;
}

.orb{
width:40px;
height:40px;
background:orange;
border-radius:50%;
margin:auto;
}
```

---

# BLOQUE 5. app.js

```javascript
const s=io();

let thr=60;
const tv=document.getElementById("tv");
const slider=document.getElementById("th");

slider.oninput=()=>{
thr=slider.value;
tv.innerText=thr;
};

const data={arduino:[],esp32:[]};

function draw(id,arr){
const c=document.getElementById(id);
const ctx=c.getContext("2d");

ctx.clearRect(0,0,300,150);

ctx.beginPath();
arr.forEach((v,i)=>{
const x=i*3;
const y=150-v*1.5;
if(i==0)ctx.moveTo(x,y);
else ctx.lineTo(x,y);
});
ctx.stroke();
}

function update(ch,val){
const bar=document.getElementById(ch==='arduino'?'ba':'be');
const orb=document.getElementById(ch==='arduino'?'oa':'oe');

bar.style.height=val+"%";

if(val>thr)orb.style.background="green";
else orb.style.background="orange";

data[ch].push(val);
if(data[ch].length>100)data[ch].shift();
}

s.on('signal',d=>update(d.channel,d.level));

setInterval(()=>{
draw("ca",data.arduino);
draw("ce",data.esp32);
},50);
```

---

# BLOQUE 6. Arduino

```cpp
void setup(){
Serial.begin(115200);
}

void loop(){
int v=analogRead(A0);
Serial.println(v);
delay(20);
}
```

---

# BLOQUE 7. Python

```python
import serial,requests

ser=serial.Serial('COM3',115200)

while True:
 try:
  v=int(ser.readline())
  requests.post("http://localhost:3000/arduino",json={"value":v})
 except:
  pass
```

---

# BLOQUE 8. ESP32

```cpp
#include <WiFi.h>
#include <SocketIoClient.h>

const char* ssid="TU_RED";
const char* pass="TU_PASS";
const char* host="192.168.1.100";

SocketIoClient socket;

void setup(){
Serial.begin(115200);

WiFi.begin(ssid,pass);
while(WiFi.status()!=WL_CONNECTED)delay(500);

socket.begin(host,3000);
}

void loop(){
socket.loop();

int v=analogRead(34);

String msg="{\"value\":";
msg+=v;
msg+="}";

socket.emit("esp32",msg.c_str());

delay(20);
}
```

---

# 8. Conexiones

### Arduino

```text
MyoWare → A0
GND → GND
```

### ESP32

```text
MyoWare → GPIO34
GND → GND
```

---

# 9. Ejecución

1. `npm install`
2. `node server.js`
3. abrir navegador
4. ejecutar Python
5. cargar Arduino y ESP32

---

# 10. Resultados esperados

* gráficas en tiempo real
* barras dinámicas
* umbral ajustable
* evento visual
* funcionamiento independiente

---

# 11. Actividad

Modificar para:

* dos umbrales
* sonido
* guardar datos
* control XY

---


