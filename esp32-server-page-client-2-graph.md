# Práctica: Visualización Gráfica en Tiempo Real

## Objetivo

El estudiante ampliará el sistema de monitoreo implementado anteriormente, incorporando visualización gráfica en tiempo real tanto en la página web como en el cliente Python.

Al finalizar la práctica, el alumno será capaz de:

* Representar datos en forma gráfica tipo barra.
* Representar datos como serie temporal.
* Actualizar gráficas dinámicamente con datos recibidos por Socket.IO.
* Comprender la diferencia entre valor instantáneo y comportamiento temporal.

---

# Parte 1: Modificación de la Página Web

No se modifica el servidor ni el ESP32.
Solo se modifica `index.html`.

---

## Nuevo `index.html`

Reemplazar el contenido anterior por el siguiente:

```html
<!doctype html>
<html>
<head>
<meta charset="utf-8">
<title>Monitor ESP32</title>

<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>

<style>
body{font-family:Arial;background:#101820;color:white;margin:0;padding:20px}
.card{background:#1c2735;padding:15px;border-radius:12px;margin-bottom:20px}
canvas{background:white;border-radius:8px}
.big{font-size:42px;font-weight:bold}
</style>
</head>
<body>

<h2>Monitor de Muestras con Gráficas</h2>

<div class="card">
  <div>Última muestra:</div>
  <div class="big" id="valor">--</div>
</div>

<div class="card">
  <canvas id="barChart"></canvas>
</div>

<div class="card">
  <canvas id="lineChart"></canvas>
</div>

<script src="/socket.io/socket.io.js"></script>
<script>
const socket = io();
const valor = document.getElementById("valor");

// ----- Gráfica de Barra -----
const barCtx = document.getElementById('barChart');
const barChart = new Chart(barCtx, {
    type: 'bar',
    data: {
        labels: ['LDR'],
        datasets: [{
            label: 'Valor ADC',
            data: [0],
            backgroundColor: 'rgba(0, 200, 255, 0.6)'
        }]
    },
    options: {
        responsive: true,
        scales: {
            y: { min: 0, max: 4095 }
        }
    }
});

// ----- Gráfica Temporal -----
const lineCtx = document.getElementById('lineChart');
const lineChart = new Chart(lineCtx, {
    type: 'line',
    data: {
        labels: [],
        datasets: [{
            label: 'Serie Temporal',
            data: [],
            borderColor: 'cyan',
            fill: false
        }]
    },
    options: {
        responsive: true,
        scales: {
            y: { min: 0, max: 4095 }
        }
    }
});

let contador = 0;

socket.on("MUESTRA", (m) => {

    let v = parseInt(m);

    valor.textContent = v;

    // Actualiza barra
    barChart.data.datasets[0].data[0] = v;
    barChart.update();

    // Actualiza línea
    contador++;
    lineChart.data.labels.push(contador);
    lineChart.data.datasets[0].data.push(v);

    if(lineChart.data.labels.length > 20){
        lineChart.data.labels.shift();
        lineChart.data.datasets[0].data.shift();
    }

    lineChart.update();
});
</script>

</body>
</html>
```

---

# Parte 2: Cliente Python con Gráficas

Instalar si no lo tienen:

```bash
pip install matplotlib
```

---

## Nuevo `cliente.py`

```python
import threading
import queue
import tkinter as tk
import socketio
import matplotlib.pyplot as plt
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg

SERVER_URL = "http://172.168.1.127:5001"

sio = socketio.Client()
q = queue.Queue()

@sio.on("MUESTRA")
def on_muestra(m):
    q.put(int(m))

def hilo():
    sio.connect(SERVER_URL)
    sio.wait()

# ---- Interfaz ----
root = tk.Tk()
root.title("Monitor ESP32 con Gráficas")

valor = tk.StringVar(value="--")

tk.Label(root, text="Última muestra", font=("Arial",12)).pack()
tk.Label(root, textvariable=valor, font=("Arial",28)).pack()

fig, ax = plt.subplots(figsize=(5,3))
line, = ax.plot([], [])
ax.set_ylim(0, 4095)
ax.set_title("Serie Temporal")
ax.set_xlabel("Muestra")
ax.set_ylabel("Valor ADC")

canvas = FigureCanvasTkAgg(fig, master=root)
canvas.get_tk_widget().pack()

datos = []

def actualizar():
    try:
        while True:
            v = q.get_nowait()
            valor.set(v)

            datos.append(v)
            if len(datos) > 20:
                datos.pop(0)

            line.set_data(range(len(datos)), datos)
            ax.set_xlim(0, max(20, len(datos)))
            canvas.draw()
    except:
        pass

    root.after(100, actualizar)

threading.Thread(target=hilo, daemon=True).start()
actualizar()
root.mainloop()
```

---

# Verificación

El sistema funciona correctamente si:

* La gráfica de barra cambia al variar la luz.
* La gráfica temporal muestra el comportamiento dinámico.
* Ambas visualizaciones (Web y Python) reaccionan en tiempo real.
* El rango ADC se mantiene entre 0 y 4095.

---

# Actividad de Comprobación

El estudiante deberá:

1. Modificar la gráfica temporal para que muestre solo 10 muestras.
2. Cambiar el color de la gráfica cuando el valor supere 3000.
3. Explicar la diferencia entre:

   * Visualización instantánea (barra).
   * Visualización temporal (línea).
