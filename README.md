# ProyectoFinal_PDS_2026-2

# Protocolo de Transmisión de Datos Acústica

> Sistema de comunicación digital que codifica y decodifica mensajes de texto en archivos de audio WAV mediante Multiplexación por División de Frecuencia (FDM) y filtros FIR Parks-McClellan.

**Procesamiento Digital de Señales · Grupo 2 · Facultad de Ingeniería UNAM · Semestre 2026-2**

---

## Integrantes

| Nombre |
|---|
| Ángel Jesús Pérez Álvarez |
| Fredy Iker Patiño Mendoza |
| José Antonio Aguilar Lugo |
| Daniel Orlando Jiménez Chávez |

---

## Descripción

Este proyecto implementa un protocolo de transmisión de datos acústica de extremo a extremo. El sistema convierte cadenas de texto en señales de audio codificando cada bit de cada carácter ASCII-256 como una sinusoide de frecuencia única dentro del espectro audible. En el receptor, la señal se decodifica mediante un banco de filtros FIR de banda estrecha y el algoritmo de Goertzel para recuperar el mensaje original.

El sistema fue desarrollado en Python sobre Google Colab con una interfaz interactiva construida con `ipywidgets`.

---

## Especificaciones técnicas

| Parámetro | Valor |
|---|---|
| Frecuencia de muestreo (Fs) | 8 000 Hz |
| Muestras por chunk (N) | 4 000 |
| Resolución frecuencial (Δf) | 2 Hz/bin |
| Duración por chunk | 0.5 s |
| Caracteres por chunk | 33 |
| Bits por carácter | 8 |
| Frecuencias totales por chunk | 264 (33 slots × 8 bits) |
| Rango espectral de datos | 500 – 3 770 Hz |
| Codificación | ASCII-256 |
| Algoritmo de detección | Goertzel |

### Tabla de frecuencias

```
f(slot, bit) = 500 + slot × 100 + bit × 10   [Hz]

Slot 0:  500, 510, 520, 530, 540, 550, 560, 570 Hz
Slot 1:  600, 610, 620, 630, 640, 650, 660, 670 Hz
...
Slot 32: 3700, 3710, 3720, 3730, 3740, 3750, 3760, 3770 Hz
```

---

## Arquitectura del sistema

```
ENCODER
  Mensaje ASCII-256
      │
      ▼
  Codificación binaria (8 bits por carácter)
      │
      ▼
  Generación de sinusoides (hasta 264 simultáneas por chunk)
  x(t) = Σ bit(s,b) · sin(2π · f(s,b) · t)
      │
      ▼
  Normalización al pico
      │
      ▼
  Archivo WAV  (Fs = 8 000 Hz, int16)


DECODER
  Archivo WAV
      │
      ▼
  Filtro FIR global pasa-banda (Parks-McClellan)
  Banda de paso: 480 – 3 790 Hz  |  Orden: 256
  → Rechaza ruido ambiental, voz humana e interferencia fuera de banda
      │
      ▼
  Banco de 264 filtros FIR por bit (Parks-McClellan)
  Banda de paso: f_bit ± 4 Hz  |  Orden: 128 c/u
  → Aisla cada frecuencia objetivo antes de la detección
      │
      ▼
  Algoritmo de Goertzel
  → DFT selectiva en cada una de las 264 frecuencias
      │
      ▼
  Umbral relativo:  bit = 1  si  mag / max_slot ≥ umbral
      │
      ▼
  Reconstrucción ASCII → Mensaje recuperado
```

---

## Filtrado FIR Parks-McClellan

El sistema implementa dos etapas de filtrado diseñadas con el algoritmo de Parks-McClellan (intercambio de Remez):

**Capa 1 — Filtro global pasa-banda**

- Banda de paso: 480 – 3 790 Hz
- Orden: 256 coeficientes
- Atenuación: ≥ 60 dB fuera de banda
- Se aplica con `filtfilt` (latencia cero, sin distorsión de fase)
- Función: rechazar ruido e interferencia de voz antes del banco de filtros

**Capa 2 — Banco de 264 filtros por bit**

- Banda de paso: f_bit ± 4 Hz por filtro
- Banda de transición: 4 Hz
- Orden: 128 coeficientes
- Factor de peso rechazo/paso: 10:1
- Se aplica con `lfilter` (causal)
- Función: aislar cada frecuencia individual antes de Goertzel

---

## Interfaz de usuario

La interfaz está organizada en cuatro pestañas:

| Pestaña | Contenido |
|---|---|
| **Encoder** | Ingreso de mensaje, nombre del archivo, botón Generar WAV, gráficas del encoder |
| **Audio** | Reproductor HTML integrado, se actualiza al generar o cargar un WAV |
| **Decoder** | Fuente del WAV, selector de algoritmo, slider de umbral, selectores de chunk y slot, botón Decodificar, gráficas del decoder |
| **Filtros FIR** | Sliders de orden, botón Rediseñar filtros, respuestas en frecuencia |

### Gráficas generadas

**Encoder:**
- Mapa de Codificación Binaria (heatmap chunks × bits)
- Señal temporal completa
- Señal temporal por slot (primeros 4 slots)
- Espectro global promedio con bandas de slots
- Espectro pre/post filtrado por slot

**Decoder:**
- Mapa de Decodificación por Chunk y Slot
- Señal temporal completa
- Espectro global promedio
- Magnitudes Goertzel vs Umbral
- Señal raw vs post-filtrado por bit

**Filtros FIR:**
- Respuesta en frecuencia del filtro global
- Respuesta en frecuencia del banco (primeros N slots)

---

## Requisitos

```bash
pip install numpy scipy matplotlib ipywidgets
```

| Librería | Uso |
|---|---|
| `numpy` | Operaciones vectoriales sobre señales |
| `scipy.signal` | `remez`, `freqz`, `lfilter`, `filtfilt` |
| `scipy.io` | Lectura y escritura de archivos WAV |
| `matplotlib` | Visualizaciones |
| `ipywidgets` | Interfaz interactiva en Colab |

---

## Uso

1. Abrir el notebook en Google Colab.
2. Ejecutar la celda de instalación de dependencias si es necesario.
3. Ejecutar la celda principal. El sistema diseñará automáticamente el banco de filtros al cargar.
4. En la pestaña **Encoder**: escribir el mensaje, configurar el nombre del archivo y presionar **Generar WAV**.
5. En la pestaña **Audio**: reproducir el archivo generado.
6. En la pestaña **Decoder**: presionar **Decodificar** para recuperar el mensaje. Ajustar el umbral si hay errores de decodificación.
7. En la pestaña **Filtros FIR**: modificar el orden de los filtros y presionar **Rediseñar filtros** para experimentar con distintas configuraciones.

> **Nota sobre el umbral:** el valor por defecto es 0.15. Si el mensaje decodificado tiene caracteres incorrectos, revisar la gráfica de Magnitudes Goertzel vs Umbral para calibrar el valor óptimo.

---

## Conceptos DSP aplicados

| Concepto | Aplicación en el proyecto |
|---|---|
| Señales discretas | Representación del mensaje como secuencia x[n] a Fs = 8 000 Hz |
| Convolución | Operación central de cada filtro FIR (`lfilter`, `filtfilt`) |
| DFT / FFT | Análisis espectral y visualización (`numpy.fft`) |
| Goertzel | DFT selectiva por frecuencia = filtro IIR resonante |
| Filtros FIR | Aislamiento espectral con fase lineal |
| Parks-McClellan | Diseño óptimo equiripple de los filtros FIR |
| Filtros IIR | Fundamento matemático del algoritmo de Goertzel |
| Transformada Z | Análisis de estabilidad y diseño de filtros |
| FDM | Transmisión de 264 bits simultáneos por chunk |
| Teorema de Nyquist | Justificación de Fs = 8 000 Hz para el rango de datos |

---

## Trabajo a futuro

**Protocolo tolerante a fallos**
Incorporar preámbulo de sincronización, numeración secuencial de chunks y código CRC para dotar al protocolo de tolerancia básica ante pérdidas parciales del canal de audio.

**Escucha activa**
Transformar el sistema a procesamiento en tiempo real mediante `sounddevice`, con detección de inicio de transmisión basada en energía espectral y aumento de Fs a 16 000 Hz para desplazar los datos por encima de los 3 400 Hz de la voz humana.

---

## Referencias

| # | Referencia |
|---|---|
| [1] | J. A. Arias, *Señales y sistemas discretos en el tiempo*, UTM |
| [2] | E. Soria, *Transformada Discreta de Fourier*, Universitat de València |
| [3] | A. V. Oppenheim y A. S. Willsky, *Señales y sistemas*, 2.ª ed., Prentice Hall, 1998 |
| [4] | A. V. Oppenheim, R. W. Schafer y J. R. Buck, *Discrete-Time Signal Processing*, 2.ª ed., Prentice Hall, 1999 |
| [5] | T. W. Parks y J. H. McClellan, "Chebyshev approximation for nonrecursive digital filters with linear phase," *IEEE Trans. Circuit Theory*, 1972 |
| [6] | G. Goertzel, "An algorithm for the evaluation of finite trigonometric series," *The American Mathematical Monthly*, 1958 |
| [7] | ITU-T G.711 — Pulse code modulation (PCM) of voice frequencies, 1988 |
| [8] | SciPy Documentation — `scipy.signal.remez`, `lfilter`, `freqz`, `wavfile` |
| [9] | NumPy Documentation — `numpy.fft` |
| [10] | ipywidgets Documentation |
| [11] | Axiom Audio, *Frequency ranges of male, female and children's voices* |

---

## Licencia

Proyecto académico desarrollado para la asignatura de Procesamiento Digital de Señales,
Facultad de Ingeniería, UNAM, Semestre 2026-2.
