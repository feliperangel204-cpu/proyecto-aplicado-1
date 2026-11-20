# Clasificación de etiquetas para prevenir el crosspacking

Proyecto Aplicado 1 · Maestría en Ciencia de Datos · Universidad Icesi
Empresa aliada: MV Perception (Cali)

Equipo: Luis Felipe Rangel · Daniel Alarcón · Edward Atehortua · Angie Tamara · Frank Mendoza

Estado al 20 de noviembre de 2026. La versión 2 se midió en la prueba (sección 3.24 del cuaderno). En un día nuevo, la cadena de red y lectores de texto acierta 93,3 % con MobileNetV3-Small y 87,0 % con ResNet18, frente a cerca de 63 % de la versión 1. No llega a la meta de 95 %, y la cadena completa tarda unos 320 ms en un Mac M4, por encima de los 240 ms de la meta. Con estas cifras el modelo no supera todavía la evaluación, y llevarlo a la planta queda como una decisión posterior. El informe final está en [`informe/`](informe/).

---

## El problema

En una línea de envasado de shampoo puede salir un frasco con la etiqueta de otra referencia distinta a la de la orden de producción. Eso es el *crosspacking*. La línea procesa 250 unidades por minuto, una cada 240 ms, y alterna entre decenas de referencias de cuatro marcas: Sedal, Savital, Savile y Dove.

El proyecto construye un sistema que reconoce la referencia a partir de una foto de la etiqueta, para compararla con la que se espera en la orden.

## Metas

| Meta | Cómo se mide en este proyecto |
|---|---|
| Exactitud de al menos 95 % | En el nivel día nuevo de la prueba: imágenes de un día que el modelo nunca vio |
| Latencia menor a 240 ms | Percentil 95 de la cadena completa: lectura de la imagen, preproceso, inferencia y decisión |

La empresa formuló la primera meta como "precisión superior al 95 %". El equipo la mide como exactitud en un día nuevo, que es la lectura más exigente. Falta que MV Perception confirme esa definición.

Estas son las dos metas del proyecto. Los demás números de operación que aparecen más abajo (paradas falsas y rollos no detectados) describen cómo se comportaría el sistema como verificador y no se juzgan contra un umbral. Llevar el modelo a la planta depende de que pase esta evaluación.

## Datos

Las imágenes son de MV Perception y no están en este repositorio: pesan cerca de 10 GB y se guardan en Google Drive.

| Paso | Imágenes | Clases |
|---|---|---|
| Entrega de la empresa (BMP de 1440 × 944) | 2.605 | 43 carpetas |
| Sin duplicados exactos (163 por huella MD5) | 2.442 | 43 |
| Partición (sin 4 imágenes de otro tamaño ni 9 sin producto) | 2.429 | 42 |

Las clases son 42 porque las carpetas 2 y 10 (Colágeno Savital 510) parecen la misma etiqueta con dos artes. Esa unión es provisional, en consulta con la empresa.

Por qué la partición es por día. Las fotos se tomaron en ráfagas, casi una por segundo: hay 18 jornadas, 95 sesiones y 336 ráfagas. Si se reparten al azar, casi la misma foto queda en entrenamiento y en prueba. La sección 1.8 lo mide: un clasificador de vecino más cercano acierta 96,3 % cuando la prueba comparte ráfaga con el entrenamiento y 13,4 % cuando la prueba es un día nuevo.

| Subconjunto | Imágenes |
|---|---|
| Entrenamiento | 1.415 |
| Validación | 410 |
| Prueba | 604 |

En la prueba, 24 clases se evalúan en un día nuevo. Otras 5 se evalúan en otra sesión y 4 en otra ráfaga del mismo día; esas dos lecturas no miden generalización. Las 9 restantes solo tienen imágenes para entrenar.

## Método

### Versión 1: tres redes con transferencia

ResNet18, MobileNetV3-Small y EfficientNet-B0 preentrenadas, con imágenes de 448 × 294 a color, pérdida ponderada por clase, AdamW y parada temprana. Se compararon con una línea base clásica (histograma de color, miniatura y regresión logística). Se evaluaron sobre la prueba.

El análisis de errores mostró que el modelo reconoce bien la marca y el diseño, pero confunde las referencias que solo cambian en el volumen impreso (por ejemplo, Sedal Prebióticos 340 y 650) y dos variantes de Dove del mismo tamaño.

### Versión 2: la red más lectores de texto

La versión 2 cambia la forma de elegir y la forma de clasificar.

Validación dejando un día afuera (sección 2.4). Se arman 9 pliegues sobre los días de desarrollo (entrenamiento y validación, sin tocar la prueba). En cada pliegue se retira un día completo, se entrena con el resto y se mide sobre ese día. Cada experimento se corre con ResNet18 y MobileNetV3-Small y con 3 semillas (42, 43 y 44), y se reporta como media ± desviación. La regla para aceptar cada experimento se escribe antes de medir.

La cadena E3c combina tres piezas:

| Pieza | Qué hace | Sección |
|---|---|---|
| E0, la red | Predice la referencia a partir de la imagen | 3.7 |
| Lector de tamaño | Lee el volumen: por geometría en Sedal y con OCR (EasyOCR) en la franja izquierda en Savital y Savile | 3.10, 3.13 |
| Lector de variante | Lee con OCR la marca y las palabras de la variante ("Nutrición", "Zero Caspa"…) y decide cuando las encuentra todas | 3.12 a 3.18 |

Encima de la cadena hay una regla de revisión (R5, sección 3.15): las unidades en las que el sistema no está seguro van a revisión manual en lugar de aceptarse.

La sección 3.19 y la 3.21 prueban otra forma de usar el sistema: si la línea sabe qué referencia está empacando, el sistema solo tiene que confirmar o rechazar esa referencia.

## Resultados

### Versión 1, sobre la prueba (nivel día nuevo, 418 imágenes)

| Modelo | Exactitud [IC 95 %] | Exactitud balanceada |
|---|---|---|
| ResNet18 | 63,6 % [46,2 a 80,3] | 58,0 % |
| MobileNetV3-Small | 62,7 % [47,2 a 76,0] | 53,1 % |
| EfficientNet-B0 | 54,5 % [38,1 a 71,3] | 47,1 % |
| Línea base clásica | 14,8 % | 10,0 % |

Una sola semilla por modelo. Intervalos por remuestreo de ráfagas.

### Versión 2, en validación dejando un día afuera (ResNet18 / MobileNetV3-Small)

| Medida | Vista día (4 clases medibles) | Vista todas (1.331 imágenes, peor caso) |
|---|---|---|
| Exactitud de E3c sin revisión | 96,4 ± 0,8 / 95,3 ± 2,1 % | 83,4 ± 0,2 / 83,3 ± 0,6 % |
| Con la regla R5 (0,95): unidades aceptadas | 88,1 / 86,6 % | 81,7 / 81,4 % |
| Con R5 (0,95): exactitud en lo aceptado | 99,8 / 99,8 % | 99,1 / 99,1 % |

La vista día cubre solo las clases 2, 11, 13 y 27, las únicas con dos o más días de desarrollo. La vista todas incluye clases que la red nunca vio al retirar su único día, y por eso es un peor caso.

En la verificación de la referencia esperada (3.21), los crosspackings que pasan sin detectarse quedan en 0,6 % o menos en todos los escenarios. Acepta entre 0,2 y 0,8 puntos menos unidades correctas que la clasificación con R5.

### Latencia (percentil 95 de la cadena completa)

| Cadena | Equipo | ResNet18 | MobileNetV3-Small |
|---|---|---|---|
| v1, red sin lectores de texto | Colab, CPU Xeon de 2 núcleos | 235,0 ms | 47,3 ms |
| v1, red sin lectores de texto | Colab, GPU T4 | 40,0 ms | 34,2 ms |
| v2, red y lectores de texto | Mac M4 con MPS, 3 repeticiones | 320,8 ± 0,7 ms | 317,2 ± 0,7 ms |

Casi todo el tiempo de la v2 está en el OCR. Entre sesiones distintas, la misma cadena varió de 316 a 345 ms.

### Prueba de la versión 2 (nivel día nuevo, 418 imágenes, 24 clases)

La prueba se miró una sola vez, con la cadena ya fijada en validación (sección 3.24). Media de 3 semillas, con intervalo del 95 % por remuestreo de ráfagas.

| Medida | MobileNetV3-Small + lectores | ResNet18 + lectores |
|---|---|---|
| Exactitud de la cadena [IC 95 %] | 93,3 ± 0,2 % [86,9 a 97,9] | 87,0 ± 1,2 % [75,2 a 96,4] |
| La misma red de la versión 1, en las mismas imágenes | 62,7 % | 63,6 % |
| La red sola de la versión 2 | 63,2 % | 59,0 % |
| Con R5: unidades aceptadas / exactitud en lo aceptado | 91,1 / 94,6 % | 90,0 / 94,4 % |
| Clases con 100 % de acierto | 17 de 24 | 16 de 24 |

La mejora sobre la versión 1 viene de los lectores: la red sola queda al nivel de la versión 1, y el texto decide la variante en el 92,1 % de las imágenes. Los dos intervalos incluyen el 95 %, pero ninguna exactitud lo alcanza.

Los errores se concentran en pocas clases. Dove Reconstrucción 370 acierta 20 % en los dos modelos: en 16 de sus 20 imágenes, el lector encuentra en el texto las palabras de Dove Repair 350 y corrige la predicción hacia esa referencia, y ese error explica 3,8 puntos de exactitud. Dove Nutrición 370 acierta 94,8 % con MobileNetV3-Small y 25,0 % con ResNet18. Quedan además errores de tamaño en Keratina Sedal 650, Sérum Savital 510 y Prebióticos Sedal 650.

### Como verificador de la referencia esperada (prueba, modo A)

Si la línea envía al sistema la referencia de la orden, el sistema solo confirma o rechaza esa referencia (sección 3.23). Con MobileNetV3-Small y parando en la primera alarma:

| Medida | Valor |
|---|---|
| Paradas falsas por frasco correcto | 5,0 % (0,68 % en validación; unos 3,8 puntos vienen de Dove Reconstrucción) |
| Frascos de otro tamaño detenidos | 93,8 % |
| Frascos de la misma marca detenidos | 95,2 % |
| Frascos de otra marca detenidos | 92,9 % |
| Rollos de 10 frascos que no se detienen | 0,00 % de otro tamaño (76 ventanas), 0,05 % de la misma marca (1.920 ventanas), 0,00 % de otra marca |

Con cero escapes observados no puede afirmarse que la tasa real sea cero. Una tasa de escape baja solo se verifica con monitoreo en producción. Que el PLC pueda enviar el código esperado a la visión está pendiente de confirmar con la empresa.

## Cumplimiento de las metas

| Meta | Resultado | Estado |
|---|---|---|
| Exactitud de 95 % o más en día nuevo | 93,3 % [86,9 a 97,9] con MobileNetV3-Small y lectores | No se cumple; el intervalo incluye la meta |
| Latencia menor a 240 ms | 320 ms en el percentil 95, en un Mac M4 sin GPU NVIDIA | No se cumple en ese equipo; falta medirla en el equipo de la línea, un Jetson Orin |

La versión 1, sin lectores de texto, cumplía la latencia en CPU y GPU de Colab, pero acertaba 63,6 % en día nuevo. Los lectores dan la exactitud y consumen unos 303 ms de los 320. Por eso el paso a la planta depende de una versión 3 y de medir en el Jetson. El informe propone cuatro caminos para bajar el tiempo: correr el OCR en la GPU, leer en el modo A solo la zona y las palabras de la referencia esperada, ejecutar la red y los lectores en paralelo, y terminar antes cuando la primera lectura confirma la referencia.

## Limitaciones

- La validación midió pocas clases. Dejar un día afuera exige que la clase tenga al menos dos jornadas de fotos, y solo 4 de las 42 las tienen. Es una limitación de los datos, no del procedimiento: por eso la validación (96,4 y 95,3 %) no anticipó el error de Dove Reconstrucción que apareció en la prueba, y el orden de los dos modelos se invirtió.
- La elección entre MobileNetV3-Small y ResNet18 se hizo con la prueba, porque la validación no los distinguía. Debe confirmarse con imágenes nuevas.
- 18 clases no tienen evidencia de día nuevo en ningún conjunto, entre ellas los pares de 490 y 510 de Fusión y Sérum Savital.
- La prueba no está del todo limpia. El análisis de errores de la versión 1 se hizo sobre ella y los lectores se diseñaron para esos errores, así que la exactitud de la versión 2 puede estar algo inflada.
- Varias reglas y el umbral de R5 se eligieron sobre los mismos datos de validación.
- La latencia se midió en equipos de desarrollo (Colab y un Mac) y varía unos 30 ms entre sesiones.
- Los lectores dependen de reglas por marca. Agregar una referencia nueva exige revisar sus palabras y su lectura de tamaño, además de reentrenar la red.

## Pasos propuestos para una versión 3

1. Corregir el lector de Dove Reconstrucción y los tamaños que fallan.
2. Medir la cadena en el Jetson de la línea.
3. Evaluar con imágenes de prueba nuevas.
4. Capturar al menos tres días separados por referencia, en turnos de mañana y de noche, con los dos miembros de cada par de volumen en la misma sesión y con registro de fecha, turno, lote de etiqueta y código SAP.
5. Comparar el modelo con AI Classify, la herramienta instalada, sobre la misma partición de prueba.

## Preguntas abiertas para MV Perception

1. ¿Qué módulo Jetson Orin tendrá el equipo de producción? Define si el OCR corre en GPU y con qué tiempo.
2. ¿El PLC puede enviar al sistema de visión el código de la referencia esperada? Si puede, el sistema opera como verificador (modo A).
3. ¿Se pueden capturar más días de las referencias que hoy tienen uno solo?
4. ¿El 95 % de precisión se mide como exactitud global, por clase o por decisión?
5. ¿Se confirman las carpetas 2 y 10 (Colágeno Savital 510) como una sola referencia?

## Estructura del repositorio

```
informe/
  informe_proyecto_aplicado.pdf     informe final (versión 13 del 20 de noviembre)
  informe_proyecto_aplicado.docx    el mismo informe, editable
notebooks/
  Proyecto-aplicado-Desa.ipynb      cuaderno de trabajo (versión 2)
  Proyecto-aplicado-v1-1-Prod.ipynb versión 1 congelada; no se edita
requirements.txt                    dependencias del entorno local
src/                                vacío por ahora
```

Las imágenes (`data/`) y el entorno virtual (`.venv/`) están excluidos en `.gitignore`.

El cuaderno sigue el ciclo de un proyecto de ciencia de datos:

| Sección | Contenido |
|---|---|
| 0 | Configuración: rutas, semillas y parámetros |
| 1 | Comprensión de los datos: inventario, integridad, duplicados, procedencia, distribución, luz y color |
| 2 | Preparación: depuración, partición por día, preprocesamiento y validación dejando un día afuera |
| 3 | Modelado: v1 (3.1 a 3.6), experimentos de la v2 (3.7 a 3.23) y prueba de la v2 (3.24) |
| 4 | Evaluación de la v1 sobre la prueba |
| 5 | Latencia de la v1 |
| 4.5 | Evaluación de la v2 sobre la prueba (lee lo que guardó la 3.24) |
| 6 | Comparativo, conclusiones y estado de la v2 (6.1b y 6.3 incluyen la prueba) |

Cada celda de código va seguida de una celda de "Lectura de resultados" que interpreta su salida.

## Cómo reproducirlo

### Carpetas en Google Drive

| Carpeta | Contenido |
|---|---|
| `pa1-imagenes` | Las 43 carpetas de imágenes BMP |
| `pa1-artefactos` | Particiones, predicciones y textos leídos por OCR, compartidos por el equipo (`*_v1_*.csv`, `*_v2_*.csv`) |
| `pa1-salidas` | Caché local de cada persona (por ejemplo, `miniaturas.npz`) |

### En Colab

1. Abre `notebooks/Proyecto-aplicado-Desa.ipynb` en Colab.
2. Corre la sección 0: monta Drive y comprueba que estén las 43 carpetas y las 2.605 imágenes.
3. EasyOCR se instala en la celda 3.10b.

### En un Mac con Apple Silicon

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

La sección 0 busca Google Drive para escritorio en `~/Library/CloudStorage/GoogleDrive-*/Mi unidad` (o `My Drive`). La red usa MPS. EasyOCR corre en CPU, o en MPS donde el cuaderno lo indica.

### Antes de correr

- **No vuelvas a correr el OCR sin necesidad.** Las celdas reutilizan los textos guardados en `pa1-artefactos` si coinciden las imágenes, la versión y el dispositivo. Las banderas `REHACER_*` deben quedar en `False`. En un dispositivo distinto del que leyó los textos, la celda vuelve a leer todas las imágenes, y eso tarda horas.
- `GUARDAR_V1` y `CORRER_3` están en `False`: la versión 1 está cerrada y volver a correrla no crea archivos nuevos.
- Los resultados nuevos se guardan con `guardar_v2`, que escribe `tema_v2_ddmes_hhmm.csv` y nunca sobrescribe un archivo `_v1`.

## Reglas de trabajo

1. La partición es por día, nunca aleatoria, y la prueba no cambia.
2. Ningún modelo ni experimento se elige mirando la prueba. Se elige con la validación que deja un día afuera, y la prueba se mira una sola vez con el experimento final.
3. Cada experimento se corre con al menos 3 semillas y se reporta como media ± desviación. La latencia también se mide con repeticiones.
4. La regla para aceptar un experimento se escribe antes de medirlo. Si cambia después de ver un resultado, se declara.
5. Cada cifra del texto sale de la salida de una celda.
