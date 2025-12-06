# Proyecto 3: CVRP - Capacitated Vehicle Routing Problem con Costos Económicos

## Nombres de los integrantes del grupo

- Rodrigo Paz Londoño
- Sebastián Palma Mogollón
- Miguel Santiago Castillo Hernández

## Instrucciones para ejecutar el código

Para ejecutar el código se usa el notebook Proyecto_3.ipynb, que contiene de forma secuencial la formulación del modelo, la carga de datos de los tres casos, la construcción de las matrices de distancia, la resolución con Pyomo y las corridas del Algoritmo Genético, junto con la generación automática de tablas de verificación, gráficos y mapas de rutas. (Advertencia: La implementacion pyomo demora mucho)

## Descripción de la estructura del proyecto

La estructura del proyecto se organiza alrededor del archivo Proyecto_3.ipynb, acompañado por carpetas de datos para cada caso de estudio (clientes, vehículos, depósito y parámetros), también por archivos de salida como las matrices de distancia y los CSV de verificación de Pyomo y de la metaheurística. Además, se incluyen funciones específicas para construir el modelo exacto en Pyomo, el GA adaptado al CVRP, los procedimientos de verificación y las rutinas de visualización que generan gráficos de desempeño y mapas HTML de las rutas.​

## Explicación de las modificaciones al ejemplo de GA para TSP 

A partir del algoritmo genético entregado se construyó un GA específico para el CVRP con un solo depósito y flota homogénea.

### 1. Representación de soluciones

En el algoritmo de referencia, una solución TSP se representa como una permutación de clientes y, en la versión MTSP, como una lista de rutas, una por viajero. El depósito no aparece explícitamente en el cromosoma y se supone que cada ruta comienza y termina allí. Para el CVRP se mantuvo esta idea, pero reinterpretada:

- Cada individuo de la población es una lista de rutas, donde cada ruta corresponde a un vehículo de la flota.
- En cada ruta aparecen únicamente los identificadores de los clientes. El depósito del proyecto se mantiene implícito y se añade solo cuando se calcula el costo.
- Los clientes se distribuyen entre las rutas de forma aleatoria pero balanceada en la inicialización, para que todos los vehículos tengan, en principio, una carga similar. Sobre esas rutas iniciales se aplica una pequeña mejora local tipo 2-opt, que busca reducir la distancia interna de cada ruta sin destruir la diversidad de la población.

### 2. Función de evaluación

En el TSP/MTSP original el fitness era simplemente la suma de distancias recorridas en las rutas. Para el CVRP del proyecto la función de evaluación se amplió para incorporar los componentes económicos y las restricciones de capacidad:

1. **Distancia de múltiples rutas**  
   Para cada vehículo se calcula la distancia del recorrido completo: desde el depósito hasta el primer cliente de la ruta, entre cada par de clientes consecutivos y del último cliente de vuelta al depósito. Así, la distancia total de la solución es la suma de las distancias de todas las rutas.

2. **Costos fijos y variables**  
   A partir del archivo de parametros se extraen las variables necesarias. De estas se deduce un costo monetario por kilómetro de combustible. Para cada ruta se construye entonces un costo total que combina:
   - Costo fijo del vehículo.
   - Costo variable por distancia.
   - Costo por tiempo. 
   - Costo de combustible.

   El fitness de una solución es la suma de los costos de todas las rutas.

3. **Penalización por violar la capacidad**  
   Además de los costos económicos, se calcula la carga atendida por cada ruta: 
   - Si la carga supera la capacidad \(Q\), se aplica una penalización proporcional al
     exceso, con un coeficiente grande.  
   - Esto hace que el algoritmo prefiera soluciones factibles y, en caso de comparar
     dos soluciones con costos similares, descarte la que viole la capacidad.

### 3. Operadores genéticos

En la adaptación se hizo lo siguiente:

- **Selección**  
  Se mantiene la selección por torneo del algoritmo original, donde a partir de un subconjunto aleatorio de individuos se elige el de menor costo. Esto favorece soluciones buenas, pero mantiene diversidad.

- **Cruce basado en rutas**  
  Se usan dos esquemas complementarios:
  1. **Intercambio de rutas completas**: se elige al azar un subconjunto de rutas y se intercambian entre dos padres. Esto permite “copiar” patrones de asignación de clientes a vehículos que ya funcionan bien.
  2. **Fusión/mezcla de rutas**: para cada posición de vehículo se combinan segmentos de la ruta del padre 1 y del padre 2, obteniendo rutas hijas que mezclan partes de ambos. Con esto se exploran nuevas combinaciones internas en las rutas sin perder por completo la estructura heredada.

- **Mutaciones específicas para CVRP**  
  Para mantener diversidad y explorar el espacio de soluciones se implementan varias
  mutaciones:
  - Intercambio de dos clientes dentro de una ruta, que modifica el orden de visita sin cambiar la asignación de clientes a vehículos.
  - Inserción de un cliente en otra posición de la misma ruta, que puede reducir la distancia al ajustar el orden de visita.
  - Inversión de un segmento de la ruta, que mejora localmente la estructura de la ruta.
  - Redistribución de clientes entre rutas. Esto mueve un cliente de la ruta de un vehículo a la de otro. Este operador es clave para CVRP porque permite explorar distintas asignaciones de clientes a vehículos y ayuda a corregir posibles sobrecargas en determinadas rutas.

En conjunto, estos operadores respetan la idea del algoritmo original pero añaden
movimientos que trabajan con la estructura multi-vehículo del CVRP.

### 4. Reparación de soluciones

Se indica que, si una ruta excede la capacidad, debe dividirse o reasignar clientes, e invita a implementar una heurística de inserción para mantener factibilidad. La adaptación incorpora una fase de reparación que actúa después de los cruces y mutaciones:

1. **Garantizar unicidad de visita**  
   Primero se reconstruye la solución para que cada cliente aparezca como máximo una vez (se eliminan ocurrencias duplicadas dejando la
     primera), y aparezca al menos una vez (los clientes que no están en ninguna ruta se insertan en las rutas con menor número de clientes). Esta inserción balanceada sirve como heurística simple para repartir carga entre vehículos.

2. **Reparación de capacidad**  
   Se identifica:
   - Qué rutas están sobrecargadas (carga > \(Q\)).
   - Cuáles tienen capacidad ociosa.
   - Cuáles están vacías.  
   Mientras existan rutas sobrecargadas y vehículos con holgura, se realiza un proceso iterativo: se toma un cliente de alta demanda de una ruta sobrecargada, se lo mueve a una ruta con más capacidad disponible o a un vehículo vacío y se actualizan las cargas y se repite el procedimiento.  

   Este mecanismo implementa, en la práctica, la idea de dividir o reasignar rutas para respetar la capacidad, antes incluso de recurrir a la penalización en la función de evaluación.

Gracias a esta combinación de reparación estructural y de capacidad, el GA tiende a trabajar con soluciones factibles o cercanas a la factibilidad, lo que acelera la convergencia hacia ruteos que respetan tanto la asignación de clientes como las limitaciones de la flota.

## Dependencias y requisitos

* Python 3.9 o superior
* pyomo>=6.6.2
* gurobipy>=10.0.0 (Gurobi con licencia académica instalada y configurada)
* pandas>=2.0.0
* numpy>=1.24.0
* matplotlib>=3.7.0
* folium>=0.14.0
* jupyter>=1.0.0

