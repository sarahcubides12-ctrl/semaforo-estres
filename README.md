# 🚦 Semáforo inteligente para la detección temprana del estrés académico

**Proyecto Integrador · Facultad de Ingeniería de Datos e Inteligencia Artificial · Universidad Santo Tomás, Seccional Tunja**

Sistema de bajo costo que estima el nivel de estrés académico de un estudiante (🟢 bajo, 🟡 moderado, 🔴 alto) a partir de señales fisiológicas. Un modelo de IA se ejecuta **directamente en un ESP32** (IA en el borde) y un dashboard muestra la alerta, recomendaciones y las fechas críticas del calendario académico.

---

## 📌 Problema

El estrés académico afecta el rendimiento y el bienestar de los estudiantes, sobre todo en épocas de parciales y entregas, pero la mayoría lo reconoce tarde porque no tiene información objetiva sobre cómo reacciona su cuerpo.

## 💡 Solución

1. **Sensores** miden ritmo cardíaco (ECG), temperatura de la piel y movimiento.
2. El **ESP32** calcula características cada 30 s y ejecuta un modelo **TensorFlow Lite Micro cuantizado a int8**.
3. El resultado se publica en la red local y el **dashboard** lo muestra como un semáforo con recomendaciones.
4. El dashboard hace **web scraping** periódico del calendario académico para anticipar semanas de alta carga.

Los datos fisiológicos crudos **nunca salen del dispositivo**: solo se envían las características resumidas y el nivel.

---

## 🧱 Arquitectura

```
[Sensores] → [ESP32: buffer circular → características → modelo int8] → JSON /data → [Dashboard Streamlit]
                                                                                        ↑
                                                              [Web scraping del calendario académico]
```

## 🔧 Hardware

| Componente | Uso |
|---|---|
| ESP32 | Microcontrolador, inferencia en el borde y servidor HTTP |
| AD8232 | Módulo ECG para el ritmo cardíaco (electrodos RA, LA, RL) |
| Sensor de temperatura | Temperatura de la piel |
| Acelerómetro | Movimiento y agitación |
| Sensor de pulso | Medición complementaria del pulso |

---

## 📁 Estructura del repositorio

```
semaforo-estres/
├── README.md
├── firmware/                  # Código del ESP32 (lectura de sensores + inferencia)
├── dataset/
│   ├── capturar_dataset.py    # Captura por puerto serial con las 3 fases del protocolo
│   ├── preparar_dataset.py    # Ventanas de 30 s, características y partición por persona
│   └── dataset_v1.csv         # Dataset propio (anonimizado)
├── modelo/
│   ├── entrenar_modelo_estres.ipynb   # Entrenamiento, evaluación y conversión a TFLite int8
│   ├── modelo_estres_int8.tflite
│   ├── modelo_estres.h        # Modelo como arreglo C para el ESP32
│   └── escalado.h             # Constantes de escalado y cuantización
├── dashboard/
│   ├── app.py                 # Dashboard Streamlit
│   ├── requirements.txt
│   ├── instalar_y_abrir.bat
│   └── .streamlit/config.toml
└── docs/                      # Póster, documentos IEEE e informe de recursos
```

---

## 📊 Dataset (propio)

Cada participante pasa por tres fases de 2 a 3 minutos:

| Fase | Etiqueta | Descripción |
|---|---|---|
| Reposo | 0 – Verde | Sentado, respiración normal |
| Test de Stroop | 1 – Amarillo | Nombrar el color de la tinta de palabras incongruentes |
| Restas seriadas | 2 – Rojo | Restar de 13 en 13 desde 1022 con presión de tiempo |

Después de cada fase el participante califica su estrés de 1 a 10 y se descartan las ventanas que no coinciden con la etiqueta.
**Características por ventana:** `bpm_prom`, `bpm_std`, `temp_prom`, `temp_pend`, `mov_prom`, `mov_std`.
**Partición por persona:** 70 % train, 15 % val, 15 % test (sin fuga de datos).

```bash
python dataset/capturar_dataset.py     # genera datos_crudos.csv
python dataset/preparar_dataset.py     # genera dataset_v1.csv
```

## 🧠 Modelo y conversión a TFLite

- Red densa: 6 entradas → 16 → 8 → 3 (softmax).
- Cuantización post-entrenamiento **int8 completa** con *representative dataset*.
- Abrir `modelo/entrenar_modelo_estres.ipynb` en Google Colab, ejecutar todo y subir `dataset_v1.csv`.
  El notebook entrega el `.tflite`, los archivos `.h` para el firmware y las métricas.

## 🗂️ Estructuras de datos

| Dónde | Estructura | Uso |
|---|---|---|
| ESP32 (borde) | Buffer circular | Últimas lecturas para calcular cada ventana |
| Dashboard | Tabla hash (`dict`) | Nivel de estrés → recomendaciones |
| Dashboard | Buffer circular (`deque`, 60) | Historial de ritmo cardíaco |
| Dashboard | Cola de prioridad (`heapq`) | Eventos académicos ordenados por cercanía |

## 🌐 Web scraping

El dashboard descarga cada 24 h el calendario académico oficial de la universidad, extrae con expresiones regulares las fechas de cortes, habilitaciones y cierres, y genera alertas para los próximos 7 días.

---

## ▶️ Cómo ejecutar el dashboard

**Windows (rápido):** doble clic en `dashboard/instalar_y_abrir.bat`.

**Manual:**
```bash
cd dashboard
pip install -r requirements.txt
streamlit run app.py
```
Se abre en `http://localhost:8501`. En la barra lateral se elige **Simulado** o se escribe la **IP del ESP32**.

### Contrato ESP32 → dashboard
`GET http://<IP_ESP32>/data`
```json
{"bpm": 82, "temp": 33.6, "mov": 0.12, "nivel": 1}
```

---

## 📏 Recursos en el ESP32

| Métrica | Valor |
|---|---|
| Tamaño del modelo | _por completar_ |
| Tensor arena (RAM) | _por completar_ |
| Tiempo de inferencia | _por completar_ (medido con `micros()`) |

## 👥 Equipo

- Sarah Alejandra Cubides Gaviria
- María José Cely
- Ricardo Moreno

**Docentes:** Zalma Valentina Moreno, Fernando Jiménez, Yashin Rueda

## 🧭 Metodología

Design Thinking: Empatizar (encuesta a estudiantes) → Definir → Idear → Prototipar (sensado, inferencia, comunicación, visualización) → Evaluar.
