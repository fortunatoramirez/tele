## Visualización de Datos en Tiempo Real mediante Comunicación Serial

#### Objetivo:

Desarrollar una aplicación que permita visualizar gráficamente los datos capturados en tiempo real desde un dispositivo basado en microcontrolador. Además, se acondicionará la señal para cambiar su rango de valores.

---

### Capítulo 1. Fundamentos Teóricos

#### 1. Conversión Analógica-Digital (ADC)

La conversión analógica-digital es un proceso que permite transformar señales analógicas en valores digitales procesables. Esto es esencial en sistemas de adquisición de datos, control de sensores y automatización industrial.

1.1 **Principios Básicos de ADC:**
El ADC convierte la señal analógica en valores discretos. La precisión de la conversión depende de:

- **Resolución del ADC:** Determina el número de niveles digitales en los que se divide la señal de entrada. Una mayor resolución proporciona mayor precisión.
- **Frecuencia de Muestreo:** Debe ser al menos el doble de la frecuencia máxima de la señal según el Teorema de Nyquist.

1.2 **Acondicionamiento de la Señal**
Para adaptar la señal a un rango deseado, se pueden utilizar diferentes técnicas como:

- **Normalización:** Escalar la señal a un nuevo rango.
- **Desplazamiento de Nivel:** Ajustar el punto de referencia de la señal.
- **Filtrado:** Suavizar la señal para reducir el ruido.

### Capítulo 2. Desarrollo

Esta sección detalla la configuración del hardware y la implementación del software para la captura y visualización de datos.

#### 2.1 Configuración del Hardware

**Materiales Necesarios:**

- Dispositivo basado en microcontrolador.
- Sensor analógico (potenciómetro, fotoresistencia, etc.).
- Cable USB para la comunicación con la computadora.

**Procedimiento:**

1. Conectar el sensor al pin analógico A0.
2. Asegurar la correcta alimentación del sensor con 5V y GND.
3. Verificar las conexiones antes de ejecutar el código.

#### 2.2 Código del Dispositivo

El siguiente código configura el dispositivo para leer datos desde el pin analógico y enviarlos a la computadora mediante comunicación serial.

```c++
char  car;
int   val_int;
float val_float;
int   val_rec;

void setup(){
  Serial.begin(9600);
}

void loop(){
  if(Serial.available()>0)
  {
    car = Serial.read();
    if(car == 'r')
    {
        val_int = analogRead(A0);
       //val_float = ((float)val_int/1023)*5; // Convierte la lectura a un rango de voltaje
       //val_rec = val_int>>9; // Reduce el rango de la señal
       Serial.println(val_int);
    }
  }
}
```

#### 2.3 Código de la Aplicación en Python

Este script en Python se encarga de capturar y graficar en tiempo real las muestras enviadas por el dispositivo.
- Interfaz 1:


```python
import tkinter as tk
import serial
import time
from threading import Thread

# Configuración del puerto serial
puerto = 'COM6'  # Cambiar según el puerto en el que esté conectado el Arduino
baudrate = 9600

print("Conectando al Arduino...")
try:
    arduino = serial.Serial(puerto, baudrate, timeout=3.0)
    print("Conectado al Arduino.")
except Exception as e:
    print(f"Error de conexión: {e}")
    arduino = None

# Función para solicitar datos del Arduino
def leer_sensor():
    if arduino:
        arduino.write("r".encode())  # Envía el comando 'r' al Arduino
        val = arduino.readline().strip()  # Lee el valor enviado por el Arduino
        try:
            val = int(val)  # Convierte el valor a entero
            etiqueta_valor.config(text=f"Valor Sensor: {val}")  # Actualiza la interfaz
        except ValueError:
            etiqueta_valor.config(text="Error en la lectura")

    ventana.after(500, leer_sensor)  # Llama a esta función cada 500ms

# Crear la ventana principal
ventana = tk.Tk()
ventana.title("Lectura de Sensor con Arduino")
ventana.geometry("300x200")

# Crear widgets
etiqueta = tk.Label(ventana, text="Leyendo datos del sensor...", font=("Arial", 14))
etiqueta.pack(pady=10)

etiqueta_valor = tk.Label(ventana, text="Valor Sensor: --", font=("Arial", 18), fg="blue")
etiqueta_valor.pack(pady=10)

boton_actualizar = tk.Button(ventana, text="Actualizar", command=leer_sensor)
boton_actualizar.pack(pady=10)

# Iniciar la actualización automática
leer_sensor()

# Iniciar la interfaz gráfica
ventana.mainloop()
```

- Interfaz 2:
```python
import datetime as dt
import matplotlib.pyplot as plt
import matplotlib.animation as animation
import serial

# Configuración de la figura
fig = plt.figure()
ax = fig.add_subplot(1, 1, 1)
xs = []
ys = []
muestra = None

# Inicialización de la comunicación serial
print('Conectando serial...')
arduino = serial.Serial('COM6', 9600, timeout=3.0)
print('Conectado.')

# Función para actualizar la gráfica
def animate(i, xs, ys, muestra):
    arduino.write('r'.encode())
    if arduino.inWaiting() > 0:
        muestra = arduino.readline().strip()
        if not muestra:
            return
        muestra = int(muestra)

        # Agregar datos a la lista
        xs.append(dt.datetime.now().strftime('%H:%M:%S.%f'))
        ys.append(muestra)

        # Limitar la cantidad de datos visualizados
        xs = xs[-20:]
        ys = ys[-20:]

        # Graficar
        ax.clear()
        ax.plot(xs, ys)

        # Formatear gráfica
        plt.xticks(rotation=45)
        plt.subplots_adjust(bottom=0.30)
        plt.title('Leyendo puerto serial')
        plt.ylabel('Amplitud')

# Configurar animación en tiempo real
ani = animation.FuncAnimation(fig, animate, fargs=(xs, ys, muestra), interval=500)
plt.show()
```



### Capítulo 3. Resultados

#### Documentación de Resultados

- **Gráficos en Tiempo Real:** La señal capturada se visualizará dinámicamente a medida que se recibe.
- **Análisis de Señal:** Se podrá evaluar la estabilidad, ruido y rango de la señal mediante la gráfica.
- **Comparación de Acondicionamiento:** Se podrá comparar la señal original con su versión acondicionada.

### Capítulo 4. Conclusiones

#### Reflexiones Sobre la Práctica

- **Visualización de Datos:** Se demuestra la utilidad de graficar datos en tiempo real para su análisis inmediato.
- **Efectividad de la Comunicación Serial:** Se evalúa el desempeño del sistema en la transmisión continua de datos.
- **Comparación de Señales:** Se discuten las diferencias antes y después del acondicionamiento.

### Capítulo 5. Anexos

#### Materiales Complementarios

- **Códigos Completos:** Se incluyen los códigos fuente completos y bien documentados.
- **Documentación de la Bitácora de Prácticas:** Se sugiere registrar observaciones y ajustes realizados para futuras mejoras.

