La práctica 4 consiste en construir y comparar un sistema de detección y seguimiento multiobjeto de vehículos en videos grabados por drones. La guía exige separar claramente:

1. detección de vehículos;
2. asociación de detecciones entre frames;
3. evaluación cuantitativa;
4. presentación oral de resultados.

La fuente principal es :codex-file-citation{path="/Users/angel/computer_vision/lab/practica4/Computer_vision_practicas_4.pdf" purpose="source"}.

## Qué debes hacer

Debes comparar obligatoriamente cuatro configuraciones:

| Método | Detector | Tracker | Objetivo |
|---|---|---|---|
| 1 | YOLOv11 | SORT | Línea base geométrica |
| 2 | YOLO26 | ByteTrack | Eficiencia y latencia |
| 3 | RT-DETR | BoT-SORT | Precisión, apariencia y GMC |
| 4 | Uno de los anteriores + SAHI | Tracker correspondiente | Mejorar vehículos pequeños |

Además, debes usar modelos preentrenados. No es necesario entrenar desde cero. La guía recomienda buscar pesos entrenados con datasets de drones como VisDrone o UAVDT.

La salida de cada método debe estar en formato MOT:

```text
frame, id, bb_left, bb_top, bb_width, bb_height, conf, class, vis
```

Solo deben conservarse las clases vehiculares:

- Car
- Truck
- Bus

## Cómo corresponde tu dataset

Tu organización real es válida, pero no coincide exactamente con la estructura estándar descrita en la guía:

- `M_attr/`: división de secuencias en entrenamiento y prueba.
- `UAV-benchmark-M/`: imágenes de los videos.
- `UAV-benchmark-MOTD_v1.0/GT/`: anotaciones y ground truth.
- `UAV-benchmark-MOTD_v1.0/RES_DET/`: detecciones antiguas; no deben utilizarse.
- `UAV-benchmark-MOTD_v1.0/RES_MOT/`: resultados antiguos de tracking; tampoco deberían utilizarse como solución propia.

Encontré:

- 50 secuencias de imágenes.
- 30 secuencias en `M_attr/train`.
- 20 secuencias en `M_attr/test`.
- Todas las secuencias de prueba tienen imágenes y ground truth correspondiente.

Por tanto, la evaluación final debería realizarse sobre las 20 secuencias de `M_attr/test`, por ejemplo:

```text
M0203, M0205, M0208, M0209, M0403,
M0601, M0602, M0606, M0701, M0801,
M0802, M1001, M1004, M1007, M1009,
M1101, M1301, M1302, M1303, M1401
```

Las imágenes están organizadas así:

```text
UAV-benchmark-M/M0203/img000001.jpg
UAV-benchmark-M/M0203/img000002.jpg
...
```

El ground truth correspondiente está en:

```text
UAV-benchmark-MOTD_v1.0/GT/M0203_gt.txt
UAV-benchmark-MOTD_v1.0/GT/M0203_gt_whole.txt
UAV-benchmark-MOTD_v1.0/GT/M0203_gt_ignore.txt
```

Según el README del dataset, el archivo `*_gt_whole.txt` contiene también la categoría del objeto: carro, camión o bus. El formato está descrito en :codex-file-citation{path="/Users/angel/computer_vision/lab/practica4/UAV-benchmark-MOTD_v1.0/README.md" purpose="source"}.

## Pasos recomendados

### 1. Preparar y verificar los datos

Crear un script que:

- lea las 20 secuencias de `M_attr/test`;
- localice sus imágenes;
- ordene los frames numéricamente;
- cargue el archivo `*_gt_whole.txt`;
- filtre únicamente las clases `Car`, `Truck` y `Bus`;
- convierta las anotaciones al formato MOT esperado.

Hay que tener cuidado con el archivo:

```text
M_attr/test/M0701 _attr.txt
```

Tiene un espacio antes de `_attr.txt`, por lo que conviene normalizar los nombres de archivo.

### 2. Preparar el ground truth

Para cada anotación, guardar:

```text
frame
target_id
left
top
width
height
class
```

No se debe confundir:

- `target_id`: identidad real del vehículo;
- `class`: categoría del vehículo;
- `conf`: confianza de detección;
- `vis`: visibilidad.

También conviene considerar los archivos `*_gt_ignore.txt` para no penalizar detecciones que caigan dentro de regiones ignoradas.

### 3. Implementar el detector de forma independiente

No usar directamente una función integrada como:

```python
model.track()
```

La práctica pide separar explícitamente:

```text
imagen
  -> detector
  -> filtrado de clases
  -> filtrado por confianza
  -> conversión de bounding boxes
  -> tracker
  -> resultado MOT
```

El detector debe devolver, por frame:

```text
[x1, y1, x2, y2, confidence, class_id]
```

### 4. Implementar los cuatro métodos

#### Método 1: YOLOv11 + SORT

- Detector moderno.
- SORT utiliza principalmente IoU y filtro de Kalman.
- No usa apariencia visual.
- Debe servir como línea base.

#### Método 2: YOLO26 + ByteTrack

- Comparar su velocidad con el método anterior.
- Analizar el efecto de utilizar detecciones de alta y baja confianza.
- Medir especialmente FPS y latencia.

#### Método 3: RT-DETR + BoT-SORT

- Analizar RT-DETR frente a YOLO26.
- Usar asociación basada en movimiento y apariencia.
- Considerar la compensación de movimiento global, GMC, porque la cámara del dron se mueve.

#### Método 4: SAHI

Aplicar SAHI sobre uno de los detectores anteriores:

1. dividir cada imagen en regiones;
2. detectar vehículos en cada región;
3. proyectar las cajas a la imagen original;
4. fusionar cajas superpuestas mediante NMS o alguna técnica equivalente;
5. pasar las detecciones al tracker.

Este método debe demostrar si mejora la detección de vehículos pequeños, aunque probablemente reduzca los FPS.

### 5. Ajustar los parámetros

La innovación de la práctica puede estar en probar parámetros como:

- confianza mínima del detector;
- IoU de NMS;
- distancia máxima de asociación;
- `track_buffer`;
- número máximo de frames sin detección;
- umbral de coincidencia;
- tamaño de las ventanas de SAHI;
- porcentaje de solapamiento entre ventanas;
- uso o no de GMC;
- resolución de entrada.

No conviene comparar métodos usando parámetros completamente arbitrarios. Debes documentar qué parámetros utilizaste y por qué.

### 6. Generar resultados

Para cada secuencia y método, producir archivos como:

```text
results/
├── yolov11_sort/
│   ├── M0203.txt
│   └── ...
├── yolov26_bytetrack/
├── rtdetr_botsort/
└── sahi/
```

Cada línea debe tener nueve campos:

```text
frame,id,left,top,width,height,confidence,class,visibility
```

### 7. Evaluar con `motmetrics`

Debes calcular, promediando las secuencias de prueba:

- MOTA;
- IDF1;
- IDS, número de cambios de identidad;
- FPS.

El FPS debe medir todo el proceso:

```text
lectura de imagen + detección + asociación + escritura del resultado
```

No debe medirse solamente el tiempo del detector.

### 8. Crear demostraciones visuales

Seleccionar ejemplos de:

- detecciones correctas;
- vehículos pequeños;
- oclusiones;
- vehículos que desaparecen detrás de árboles o puentes;
- cambios incorrectos de identidad;
- falsos positivos;
- casos donde SAHI mejora o empeora el resultado.

Para cada método conviene generar videos o imágenes con:

- bounding box;
- ID del tracker;
- clase;
- confianza;
- número de frame.

### 9. Preparar la presentación

La exposición debe incluir:

1. problema y características de las imágenes aéreas;
2. organización del dataset;
3. arquitectura de YOLO26;
4. arquitectura de RT-DETR;
5. diferencia entre SORT, ByteTrack y BoT-SORT;
6. geometría frente a embeddings de apariencia;
7. explicación de SAHI;
8. metodología experimental;
9. tabla de MOTA, IDF1, IDS y FPS;
10. ejemplos visuales;
11. fallos y limitaciones;
12. heurísticas o ajustes que hayas implementado;
13. conclusión sobre cuál método es más preciso y cuál es más rápido.

## Resultado esperado

La práctica no exige un informe escrito. El entregable principal es una presentación oral con resultados experimentales. La rúbrica asigna más peso a:

- conocimiento técnico;
- claridad y calidad de las diapositivas;
- capacidad de responder preguntas;
- innovación mediante ajustes o heurísticas.

La estrategia más segura sería:

1. implementar primero YOLOv11 + SORT;
2. verificar que el formato de datos y la evaluación funcionen;
3. incorporar YOLO26 + ByteTrack;
4. añadir RT-DETR + BoT-SORT;
5. finalmente integrar SAHI;
6. ejecutar todos los métodos sobre las mismas 20 secuencias;
7. comparar métricas y preparar visualizaciones.