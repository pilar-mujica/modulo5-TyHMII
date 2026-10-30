# Sistema de Control de Temperatura por Termostato

Modelado en **Vensim PLE 10.5** del caso *Mistakes and Misunderstandings: Examining Dimensional Inconsistency* (M. S. Gary, MIT System Dynamics in Education Project, 1992). El trabajo recorre tres versiones del modelo: una **errónea**, una **corregida** y una **mejorada con histéresis**.

## Contenido del repositorio

| Archivo | Descripción |
|---|---|
| [`Modelo_erroneo.mdl`](<ModelodeSistema_erróneo.mdl>) | Modelo inicial con inconsistencia dimensional |
| [`Modelo_corregido.mdl`](<Modelocorregido.mdl>) | Modelo dimensionalmente consistente |
| [`Modelo_mejorado.mdl`](<Modelomejorado.mdl>) | Modelo con termostato con histéresis |
| [`Informe.pdf`](<Informe-Módulo5.pdf>) | Informe completo |

---

## 1. Breve introducción al sistema analizado

El modelo representa un sistema físico cotidiano: **el control de temperatura de una vivienda mediante un calefactor y un termostato**.

El propósito del sistema es mantener la temperatura interior en un valor deseado. Para ello, el termostato calcula la brecha entre la temperatura deseada y la actual. Si la temperatura interna cae por debajo del valor deseado, el termostato activa la producción de calor del calefactor. Al mismo tiempo, la vivienda pierde calor de manera continua hacia el exterior.

---

## 2. Modelo inicial y detección del error

![Modelo erróneo](<Zoommodelo.jpeg>)

La primera formulación toma la **temperatura de la casa como stock**, al que entra la producción de calor y del que sale la pérdida de calor.

### ¿Cómo revisamos los errores en Vensim?

Vensim ofrece dos verificaciones:

- **Check Model (Ctrl + T):** controla sintaxis y conexiones. En este caso **no detecta problemas**: el modelo es formalmente válido.
- **Units Check (Ctrl + U):** analiza la coherencia de las unidades a ambos lados de cada ecuación. Es el que revela el error.

### El error identificado (Dimensional Inconsistency)

Al ejecutar el chequeo de unidades, Vensim reporta:

> Analysis of units error: Right hand and left hand units do not match

El error surge en la ecuación del nivel `House Temperature`:

![Units Check del modelo erróneo](<error-unitcheck.jpeg>)

- **Origen de la falla:** la variable acumuladora (`House Temperature`) está definida en unidades de temperatura (`DegF`), mientras que los flujos (`Heat Production` y `Heat Loss`) están en energía por tiempo (`BTU/Minute`).
- **Falla conceptual:** integrar una tasa de energía en el tiempo da energía (BTU/min × min = BTU), nunca temperatura (DegF).

---

## 3. Plan de acción y modelo corregido

Para obtener un modelo dimensionalmente correcto se reestructura el diagrama según la termodinámica:

1. **Redefinir el stock:** lo que se acumula es la energía térmica de la vivienda (`Heat in House`, en BTU).
2. **Relacionar calor con temperatura:** la temperatura pasa a ser una variable auxiliar, que se calcula dividiendo el calor acumulado por la capacidad térmica de la casa.
3. **Introducir un factor de conversión:** `Conversion Factor` (BTU/DegF) = calor específico del aire × densidad del aire × volumen de la casa.

### Corrección del modelo

1. Reemplazar el stock `House Temperature` por el nuevo stock **`Heat in House`** (BTU).
2. Conectar `Heat Production` y `Heat Loss` (BTU/Minute) directamente a `Heat in House`.
3. Crear la variable auxiliar **`House Temperature`** (DegF) con la fórmula:

   ```
   House Temperature = Heat in House / Conversion Factor
   ```

4. Conectar `House Temperature` con `Temperature Gap` para cerrar el bucle de retroalimentación del termostato.
5. Agregar la constante `Furnace Capacity` (BTU/Minute) como potencia del calefactor.

![Modelo corregido](<modelocorregido.png>)

Al ejecutar nuevamente el **Units Check**, Vensim confirma que el modelo es **dimensionalmente consistente** (*Units are OK*).

![Units OK](<chequeounidades.jpeg>)

### Comportamiento simulado

La casa parte de unos 72,2 °F (13 000 BTU) y alcanza los 75 °F a los ~17 minutos. A partir de ahí aparece un problema: **el calefactor se enciende y apaga en cada paso de simulación** (*chattering*), porque el termostato tiene un único umbral.

![Temperatura - modelo corregido](<gráficotempcasa.jpeg>)
![Producción de calor - chattering](<chattering(heatprod).jpeg>)

---

## 4. Variación final: termostato con histéresis

### Analogía con la electrónica

El termostato del modelo corregido se comporta como un **comparador sin realimentación** (amplificador operacional a lazo abierto): cuando la entrada está cerca del umbral, la salida oscila sin parar entre sus dos estados.

La solución es la misma que en electrónica: un **disparador Schmitt**, es decir, un comparador con **realimentación positiva** que define **dos umbrales**. Con una banda de 2 °F:

- El calefactor **se enciende** por debajo de **74 °F**.
- **Se apaga** por encima de **76 °F**.
- **Entre 74 y 76 °F mantiene su estado anterior** (memoria).

### Implementación en Vensim

La memoria se modela con un **stock**, `Furnace On` (0 = apagado, 1 = encendido). La realimentación positiva aparece en `Desired Furnace State`, que dentro de la banda devuelve el propio valor de `Furnace On`. `TIME STEP` se incorpora como variable sombra.

![Modelo mejorado](<simulacionmejorada.png>)

### Resultados

La temperatura describe un **diente de sierra regular entre 74 y 76 °F**, y el calefactor pasa de conmutar en cada paso a completar **un ciclo aproximadamente cada 25 minutos**.


![Temperatura - modelo mejorado](<graficotempcasa.png>)
![Producción de calor - modelo mejorado](<graficoprodcalor.png>)

> **Alcance de la mejora:** como la pérdida de calor es constante, la energía total consumida es la misma en ambos modelos. La mejora está en la **drástica reducción de arranques**, que en un equipo real implica menos desgaste y menos pérdidas de encendido. El ancho de banda define el compromiso: más angosta, más confort y más arranques; más ancha, menos arranques y más oscilación.

---
## 5. Simplificación: modelado del calor del horno

En los tres modelos el horno funciona en modo **todo o nada**: cuando está encendido entrega siempre su potencia máxima (`Furnace Capacity` = 60 BTU/min) y, si no, entrega 0. La producción de calor no depende de *cuánto* falta para llegar a la temperatura de referencia, sino solo de *si* falta. A eso se suma una pérdida de calor constante. Es una simplificación válida para analizar la consistencia dimensional, pero no la forma más eficiente de controlar el sistema.

**Analogía: llenar un vaso con agua**

- **Control on/off (nuestro modelo):** es como llenar un vaso con una manguera a máxima presión, abriéndola y cerrándola de golpe. Se pasa del nivel, salpica y obliga a abrir y cerrar continuamente. Con histéresis se abre y cierra menos veces, pero siempre a todo o nada.
- **Control proporcional (alternativa):** es como usar una canilla regulable que se va cerrando a medida que el vaso se llena. Mucho caudal cuando está vacío, poco cuando está casi lleno. El nivel se acerca suavemente al objetivo, sin pasarse ni cortar el agua de golpe.

En el modelo, esto se lograría haciendo que la producción de calor dependa de la brecha de temperatura, con un límite en la potencia máxima del horno:

    Heat Production = MIN(Furnace Capacity, MAX(0, Gain * Temperature Gap))

donde `Gain` (BTU/(Minute*DegF)) define cuánto calor se entrega por cada grado de diferencia.

En electrónica, es la diferencia entre un comparador, que satura a uno u otro extremo, y un amplificador con **realimentación negativa** que trabaja en su zona lineal. Los calefactores modernos *modulantes* funcionan así: regulan la llama en lugar de prenderla y apagarla, con lo que reducen los arranques y las pérdidas de encendido.

> **Nota:** un control proporcional puro deja un pequeño error permanente. Para compensar la pérdida constante de 30 BTU/min, la casa se estabiliza un poco por debajo de la referencia (Temperature Gap = 30 / Gain). Un `Gain` más alto reduce ese error.
---
## 6. Conclusiones

- Un modelo puede ser sintácticamente válido y simular sin errores, y aun así ser **conceptualmente incorrecto**.
- **Todo flujo debe tener unidades de stock por unidad de tiempo.** El *Units Check* permite detectar la falla, pero corregirla exige revisar el modelo mental: el calor se acumula; la temperatura se deriva de él.
- La simulación del modelo corregido reveló un segundo problema, de comportamiento: el *chattering* del termostato con un único umbral.
- Incorporar **histéresis**, igual que en un disparador Schmitt, requirió agregar memoria (un stock) y una realimentación positiva. El resultado es un calefactor que trabaja en ciclos largos y regulares, con la temperatura acotada en ±1 °F alrededor de 75 °F.

---

## Referencias

- Gary, M. S. (1992). *Mistakes and Misunderstandings: Examining Dimensional Inconsistency* (D-4272-7). System Dynamics in Education Project, MIT.
- Forrester, J. W. (1961). *Industrial Dynamics*. MIT Press.

