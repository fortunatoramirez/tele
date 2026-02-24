## Comunicación en Tiempo Real entre ESP32 y Servidor Node.js

### Objetivo:

Desarrollar una aplicación utilizando Node.js y socket.io para establecer una comunicación en tiempo real entre un servidor y un dispositivo ESP32. El ESP32 enviará valores leídos de una entrada analógica al servidor para su visualización y análisis.

---

### Capítulo 1. Fundamentos Teóricos

En este capítulo, exploramos los conceptos fundamentales necesarios para entender la comunicación en tiempo real entre un dispositivo ESP32 y un servidor utilizando Node.js y socket.io. Cubriremos las bases de la conversión analógica-digital (ADC), la tecnología WebSocket, y conceptos adicionales como la estructura de un servidor y el protocolo de comunicación.

#### 1.1 Conversión Analógica-Digital (ADC)

La conversión de señales analógicas a digitales es un proceso crítico en la interfaz entre el hardware físico y el software digital. Esta conversión permite que los microcontroladores y otros dispositivos procesen y almacenen información del mundo físico, como temperatura, luz, y otros tipos de datos sensoriales.

- **Resolución del ADC:** Define el número de divisiones discretas que un ADC puede diferenciar. Una resolución más alta permite una mayor precisión en las mediciones.
- **Frecuencia de Muestreo:** Es crucial que la frecuencia de muestreo sea adecuada para capturar la variabilidad de la señal analógica, siguiendo el Teorema de Nyquist para evitar el aliasing.

#### 1.2 Tecnología WebSocket

WebSocket es una tecnología que permite canales de comunicación bidireccional y full-duplex sobre un único socket persistente. Es fundamental para aplicaciones que requieren una comunicación en tiempo real entre el cliente y el servidor.

- **Protocolo WebSocket:** Permite una interacción continua y en tiempo real, eliminando la necesidad de sondear repetidamente el servidor para obtener nuevos datos o actualizaciones.
- **socket.io:** Una biblioteca que simplifica el uso de WebSockets, proporcionando una API robusta para emitir y recibir mensajes de manera eficiente y con manejo de errores.

#### 1.3 Estructura de un Servidor

Un servidor no es solo un hardware que ejecuta software, sino que en el contexto de programación, se refiere al software que gestiona los recursos y proporciona servicios a otros programas, generalmente llamados clientes.

- **Node.js:** Un entorno de ejecución para JavaScript del lado del servidor, ideal para desarrollar servidores rápidos y escalables debido a su modelo de operaciones basado en eventos no bloqueantes.
- **Express.js:** Un marco de trabajo para Node.js que simplifica la creación de servidores web y API, proporcionando herramientas y funcionalidades que facilitan la gestión de rutas, solicitudes y respuestas.

#### 1.4 Protocolos de Comunicación

Los protocolos de comunicación definen las reglas que los dispositivos deben seguir para intercambiar mensajes. En el contexto de esta práctica, es crucial entender cómo estos protocolos influyen en la eficiencia y fiabilidad de la transmisión de datos.

- **HTTP vs. WebSocket:** Mientras que HTTP es un protocolo de solicitud-respuesta sin estado, WebSocket proporciona una conexión persistente que facilita una comunicación en tiempo real mucho más rápida y eficiente.
- **MIME Types:** En la comunicación web, los tipos MIME definen el formato de los datos enviados, asegurando que tanto el servidor como el cliente interpreten correctamente los datos intercambiados.
---

### Capítulo 2. Desarrollo

#### 2.1 Configuración del Hardware

**Materiales Necesarios:**

- ESP32.
- Sensor analógico (ejemplo: potenciómetro).
- Cable USB para programación y comunicación.

**Procedimiento:**

1. **Conectar el Sensor al ESP32:**
   - Conecte el sensor analógico al pin adecuado del ESP32.
   - Asegúrese de que el sensor esté correctamente alimentado.

2. **Programación del ESP32:**
   - Utilice el IDE de Arduino para cargar el sketch que leerá los datos del sensor y los enviará al servidor utilizando la biblioteca SocketIoClient.

**Código del ESP32:**

```cpp
#include <WiFi.h>
#include <SocketIoClient.h>

// Configuración de la red WiFi
const char* ssid = "Tu_SSID";
const char* password = "Tu_Password";

// Configuración del servidor WebSocket
const char* host = "direccion_del_servidor";
const int port = 3000;

// Crear instancia del cliente socket.io
SocketIoClient webSocket;

void setup() {
  Serial.begin(115200);
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("WiFi conectado");
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());

  // Conectar al servidor WebSocket
  webSocket.begin(host, port);
  webSocket.on("connect", handleWebSocketConnect);
}

void loop() {
  if (WiFi.status() == WL_CONNECTED) {
    webSocket.loop();
    int sensorValue = analogRead(A0);  // Leer valor analógico del pin A0
    // Enviar valor al servidor
    webSocket.emit("data", String(sensorValue).c_str());
  }
  delay(1000);  // Esperar un segundo antes de enviar el siguiente valor
}

void handleWebSocketConnect(const char * payload, size_t length) {
  Serial.println("Conectado al WebSocket Server");
}
```

#### 2.2 Configuración del Servidor

**Desarrollo del Servidor en Node.js:**

- Instalación de Node.js y configuración del entorno.
- Creación de un servidor utilizando Express y socket.io para manejar conexiones WebSocket.
- Implementación de la lógica para recibir datos del ESP32 y enviar comandos de respuesta si fuera necesario.

**Código del Servidor:**

```javascript
const express = require('express');
const http = require('http');
const socketIo = require('socket.io');

const app = express();
const server = http.createServer(app);
const io = socketIo(server);

io.on('connection', (socket) => {
    console.log('Nuevo dispositivo conectado.');

    socket.on('data', (data) => {
        console.log(`Datos recibidos del ESP32: ${data}`);
        // Procesamiento o respuesta
    });

    socket.on('disconnect', () => {
        console.log('Dispositivo desconectado.');
    });
});

server.listen(3000, () => {
    console.log('Servidor escuchando en puerto 3000');
});
```

### Capítulo 3. Resultados

#### Documentación de Resultados

- **Visualización de Datos:** Muestre cómo los datos enviados por el ESP32 se visualizan en tiempo real en el servidor.
- **Análisis de Conectividad:** Discuta la estabilidad y la eficacia de la conexión WebSocket entre el ESP32 y el servidor.

### Capítulo 4. Conclusiones

#### Reflexiones Sobre la Práctica

- Evaluar la eficacia de la comunicación en tiempo real mediante WebSocket y socket.io.
- Discutir los desafíos encontrados durante la integración del ESP32 con el servidor Node.js y cómo se superaron.

### Capítulo 5. Anexos

#### Materiales Complementarios

- **Códigos Completos:** Se incluyen los códigos fuente completos y bien documentados tanto para el servidor Node.js como para el ESP32.
- **Documentación de la Bitácora de Prácticas:** Se sugiere registrar todas las observaciones y ajustes realizados durante la práctica.

---

### Material Adicional 1: Configuración del IDE de Arduino para Programar el ESP32

Para programar el ESP32 usando el IDE de Arduino, es necesario configurar el entorno de desarrollo para soportar este hardware específico. Aquí te proporciono una guía paso a paso sobre cómo configurar el IDE de Arduino:

#### Paso 1: Instalación del IDE de Arduino
Si aún no tienes instalado el IDE de Arduino, puedes descargarlo desde el sitio web oficial de Arduino en [https://www.arduino.cc/en/software](https://www.arduino.cc/en/software). Sigue las instrucciones de instalación proporcionadas en el sitio para tu sistema operativo.

#### Paso 2: Agregar el ESP32 a la Lista de Tarjetas en el IDE de Arduino
1. **Abrir el IDE de Arduino** y dirigirse a **Archivo > Preferencias**.
2. En el campo **"Gestor de URLs Adicionales de Tarjetas"**, debes agregar la siguiente URL para incluir el soporte para el ESP32:
   ```
   https://dl.espressif.com/dl/package_esp32_index.json
   ```
   Pulsa **OK** para guardar la configuración.

#### Paso 3: Instalación del Paquete ESP32
1. Ve a **Herramientas > Placa > Gestor de Tarjetas**.
2. Escribe **"ESP32"** en la barra de búsqueda.
3. Encuentra el paquete llamado **"esp32 by Espressif Systems"** y haz clic en **Instalar**.

#### Paso 4: Seleccionar la Placa Correcta
Una vez instalado el paquete:
1. Ve a **Herramientas > Placa** y selecciona la placa ESP32 que estás utilizando (por ejemplo, ESP32 Dev Module).

#### Paso 5: Configurar los Parámetros del Puerto
1. Conecta tu ESP32 a la computadora mediante un cable USB.
2. Ve a **Herramientas > Puerto** y selecciona el puerto que muestra tu ESP32.

#### Paso 6: Instalar Bibliotecas Específicas
Para utilizar el ESP32 con comunicación en tiempo real, necesitarás instalar las bibliotecas **SocketIoClient** de Vincent Wyszynski y **WebSockets** de Markus Sattler.

1. **Instalación de la Biblioteca SocketIoClient:**
   - Ve a **Programa > Incluir Biblioteca > Gestionar Bibliotecas...**
   - Busca **"SocketIoClient"** y haz clic en **Instalar**.

2. **Instalación de la Biblioteca WebSockets:**
   - En el mismo **Gestor de Bibliotecas**, busca **"WebSockets"** por Markus Sattler.
   - Selecciona e instala la biblioteca para habilitar la comunicación WebSocket.

#### Modificaciones en el Código Fuente de la Biblioteca

Para evitar errores de compilación con la biblioteca SocketIoClient, es necesario realizar una pequeña modificación en el código fuente:

1. **Localiza el Archivo `SocketIoClient.cpp`:**
   - Navega a la carpeta de bibliotecas de Arduino, que generalmente se encuentra en `Documentos/Arduino/libraries/SocketIoClient/src`.

2. **Modificar el Archivo `SocketIoClient.cpp`:**
   - Abre el archivo `SocketIoClient.cpp` con un editor de texto.
   - Busca la línea 41 que contiene la función `hexdump(payload, length)`.
   - Comenta esta línea añadiendo `//` al inicio de la línea para desactivarla:
     ```cpp
     // hexdump(payload, length);  // Comentar esta línea para evitar errores de compilación.
     ```

#### Nota Adicional:
- **Por qué Comentar la Línea:** La función `hexdump` se utiliza para depurar mostrando los datos en formato hexadecimal. Sin embargo, si esta función no está definida en otras partes del código o en las bibliotecas del sistema de Arduino, su llamada producirá un error de compilación. Comentando esta línea, evitas estos errores y aseguras una compilación exitosa.

#### Paso 7: Cargar el Código
Una vez configurado el entorno, puedes cargar el código proporcionado para el ESP32. Asegúrate de que el código está libre de errores y es compatible con tu modelo de ESP32.


---

### Material Adicional 2: Instalación de Drivers para ESP32

En algunos casos, especialmente con nuevos dispositivos o en sistemas operativos específicos, puede ser necesario instalar drivers adicionales para que el ESP32 sea reconocido correctamente por la computadora. Esto es común con chips que usan interfaces USB a serial como el CP210x de Silicon Labs, que es frecuentemente utilizado en módulos ESP32.

#### Instalación de Drivers para CP210x

1. **Identificación del Chip:** Verifica si tu módulo ESP32 usa un chip CP210x para la comunicación USB a serial. Esto suele estar indicado en la documentación del módulo o visible en el chip en la placa.

2. **Descarga del Driver:**
   - Visita el sitio web de Silicon Labs para descargar los drivers más recientes para el chip CP210x: [Silicon Labs CP210x USB to UART Bridge VCP Drivers](https://www.silabs.com/developers/usb-to-uart-bridge-vcp-drivers).
   - Selecciona y descarga la versión del driver adecuada para tu sistema operativo (Windows, Mac OS, Linux).

3. **Instalación del Driver:**
   - Ejecuta el instalador descargado y sigue las instrucciones proporcionadas para completar la instalación.
   - En sistemas Windows, puede ser necesario reiniciar la computadora después de instalar el driver para asegurar que se cargue correctamente.

4. **Verificación:**
   - Una vez instalado el driver, conecta el ESP32 a la computadora mediante el cable USB.
   - Ve a **Administrador de dispositivos** en Windows o usa el comando `lsusb` en Linux para verificar que el dispositivo está siendo reconocido correctamente.
   - Deberías ver un nuevo puerto serial listado, indicando que el dispositivo está listo para ser utilizado.

#### Solución de Problemas

- **No Reconocimiento del Dispositivo:** Si el dispositivo no es reconocido después de instalar el driver, prueba con otro cable USB y asegúrate de que el ESP32 esté alimentado correctamente.
- **Problemas con la Conexión Serial:** Si encuentras errores al intentar cargar programas en el ESP32, verifica la configuración del puerto serial en el IDE de Arduino y asegúrate de que coincide con el puerto que muestra el administrador de dispositivos.

### Importancia de los Drivers

Los drivers como los de CP210x facilitan la comunicación entre el microcontrolador y la computadora, permitiendo la programación del dispositivo y la comunicación serial durante la ejecución de aplicaciones. Es esencial asegurarse de que estos drivers están correctamente instalados y actualizados para evitar problemas durante el desarrollo y la depuración proyectos con ESP32.


---


###  Material Adicional 3: Instrucciones para Instalar Paquetes de Node.js

Para desarrollar una aplicación con Node.js utilizando Express y socket.io en la versión específica 1.7.2, sigue estos pasos para configurar tu entorno en Windows con Visual Studio Code.

#### Paso 1: Instalar Visual Studio Code y Node.js
Asegúrate de que tienes instalados tanto Visual Studio Code como Node.js:
- **Visual Studio Code**: Puedes descargarlo desde [su sitio oficial](https://code.visualstudio.com/).
- **Node.js**: Descárgalo desde [nodejs.org](https://nodejs.org/en/). Esto incluirá npm (Node Package Manager), que es necesario para instalar paquetes de Node.js.

#### Paso 2: Configuración del Proyecto
1. **Abrir Visual Studio Code** y seleccionar o crear un nuevo directorio para tu proyecto.
2. **Abrir una terminal en Visual Studio Code**: Puedes hacer esto desde `Terminal > Nueva Terminal`.

#### Paso 3: Inicialización del Proyecto
1. En la terminal, ejecuta:
   ```bash
   npm init -y
   ```
   Esto creará un archivo `package.json` que gestionará las configuraciones y las dependencias de tu proyecto.

#### Paso 4: Instalación de Express y Socket.io
1. **Instalar Express**:
   - Ejecuta el siguiente comando en la terminal:
     ```bash
     npm install express
     ```
   - Esto instalará la última versión de Express y la añadirá a las dependencias en tu `package.json`.

2. **Instalar Socket.io versión 1.7.2**:
   - Dado que necesitas una versión específica de socket.io, asegúrate de especificar la versión en el comando de instalación:
     ```bash
     npm install socket.io@1.7.2
     ```
   - Este comando instala la versión 1.7.2 de socket.io y la registra en tu archivo `package.json`.

#### Paso 5: Verificación de las Instalaciones
1. **Verifica que los paquetes están instalados correctamente**:
   - Revisa el archivo `package.json` en tu proyecto y asegúrate de que las dependencias listan express y socket.io con las versiones correctas.

#### Paso 6: Creación del Servidor
1. **Crea un archivo nuevo** como `server.js` en tu directorio de proyecto.
2. **Escribe o copia el código del servidor** que utiliza express y socket.io.

#### Paso 7: Ejecutar el Servidor
1. **Inicia tu servidor** ejecutando:
   ```bash
   node server.js
   ```
   - Asegúrate de que no hay errores y que el servidor está corriendo correctamente escuchando en el puerto configurado.

---



