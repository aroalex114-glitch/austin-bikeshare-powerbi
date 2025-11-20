# Uso del sistema de bicicletas en Austin (2014–2023)

Dashboard desarrollado en **Power BI** utilizando los datos públicos de *Austin Bikeshare* en **BigQuery**.  
El proyecto cubre el periodo **2014–2023** y analiza cómo se utiliza el sistema de bicicletas compartidas de la ciudad.

---

## 1. Objetivo del proyecto

- Analizar la evolución del uso del sistema de bicicletas en Austin entre 2014 y 2023.
- Entender **patrones de uso** por:
  - año, mes y día de la semana  
  - hora del día  
  - tipo de suscripción  
  - tipo de bicicleta (clásica vs eléctrica)  
  - duración del viaje  
- Construir un modelo de datos en esquema estrella y un dashboard que pueda servir como ejemplo de portafolio en Power BI.

---

## 2. Fuente de datos

- **Origen:** `bigquery-public-data.austin_bikeshare.bikeshare_trips` (Google BigQuery).  
- **Nivel de detalle:** cada registro corresponde a un **viaje individual**.  
- **Periodo analizado:** viajes con fecha de inicio entre **2014 y 2023**.

---

## 3. Preparación de datos en SQL

La primera fase de limpieza y enriquecimiento se realizó en **BigQuery SQL** para:

- Estandarizar campos de texto (`TRIM`).  
- Derivar campos de fecha/hora (año, mes, día, nombre del día, etc.).  
- Crear un campo de **ruta** (origen-destino).  
- Clasificar el **tipo de suscripción** en categorías más legibles.  
- Filtrar registros no válidos (estaciones nulas, duración incorrecta, etc.).

### 3.1. Script principal de limpieza

~~~sql
SELECT
     trip_id                                                                  AS id_viaje,
     TRIM(subscriber_type)                                                    AS tipo_subscripcion,
     bike_id                                                                  AS id_bici,
     TRIM(bike_type)                                                          AS tipo_bici,
     start_station_id                                                         AS id_inicio_estacion,
     TRIM(start_station_name)                                                 AS nombre_inicio_estacion,
     end_station_id                                                           AS id_fin_estacion,
     TRIM(end_station_name)                                                   AS nombre_fin_estacion,
     CONCAT(TRIM(start_station_name),'-', TRIM(end_station_name))             AS ruta,
     duration_minutes                                                         AS duracion_min,

     -- Limpieza de fecha y hora
     DATE(start_time)                                                         AS fecha_viaje,
     TIME(start_time)                                                         AS hora_inicio,
     EXTRACT(HOUR FROM start_time)                                            AS hora_num,
     EXTRACT(YEAR FROM start_time)                                            AS anio_viaje,
     EXTRACT(DAYOFWEEK FROM start_time)                                       AS num_dia,
     CASE EXTRACT(DAYOFWEEK FROM start_time)
        WHEN 1 THEN 'Domingo' 
        WHEN 2 THEN 'Lunes' 
        WHEN 3 THEN 'Martes'
        WHEN 4 THEN 'Miercoles' 
        WHEN 5 THEN 'Jueves'
        WHEN 6 THEN 'Viernes'
        ELSE 'Sabado'
     END                                                                      AS dia_viaje,
     EXTRACT(MONTH FROM start_time)                                           AS num_mes,
     CASE EXTRACT(MONTH FROM start_time)
        WHEN 1 THEN 'Enero'
        WHEN 2 THEN 'Febrero'
        WHEN 3 THEN 'Marzo'
        WHEN 4 THEN 'Abril'
        WHEN 5 THEN 'Mayo'
        WHEN 6 THEN 'Junio'
        WHEN 7 THEN 'Julio'
        WHEN 8 THEN 'Agosto'
        WHEN 9 THEN 'Septiembre'
        WHEN 10 THEN 'Octubre'
        WHEN 11 THEN 'Noviembre'
        ELSE 'Diciembre'
     END                                                                      AS mes,
     EXTRACT(QUARTER FROM start_time)                                         AS Q_viaje,

     -- Clasificación del tipo de suscripción
     CASE 
        -- Membresía anual
        WHEN TRIM(subscriber_type) IN (
          'Annual','Annual Member','Annual Membership',
          'Annual Pass','Annual Pass (30 minute)','Annual Pass (Original)',
          'Annual Plus','Annual Plus Membership',
          'Republic Rider','Republic Rider (Annual)',
          'Heartland Pass (Annual Pay)',
          'Membership: pay once one-year commitment',
          'Membership: pay once, one-year commitment',
          'Local365','Local365 ($80 plus tax)',
          'Local365+Guest Pass','Local365+Guest Pass- 1/2 off Anniversary Special',
          'Denver B-cycle Founder','Founding Member','HT Ram Membership'
        ) THEN 'Membresía anual'

        -- Membresía mensual
        WHEN TRIM(subscriber_type) IN (
          'Heartland Pass (Monthly Pay)','Madtown Monthly',
          'Local30','Local30 ($11 plus tax)','Local31'
        ) THEN 'Membresía mensual'

        -- Membresía semestral
        WHEN TRIM(subscriber_type) = 'Semester Membership'
        THEN 'Membresía semestral'

        -- Membresía estudiantil
        WHEN TRIM(subscriber_type) IN (
          'Student Membership','UT Student Membership','U.T. Student Membership'
        ) THEN 'Membresía estudiantil'

        -- Membresía juvenil
        WHEN TRIM(subscriber_type) LIKE 'Local365 Youth%'
        THEN 'Membresía juvenil'

        -- Pago por viaje
        WHEN TRIM(subscriber_type) IN (
          '$1 Pay by Trip Fall Special','$1 Pay by Trip Winter Special',
          '24 Hour Walk Up Pass','Pay-as-you-ride',
          'Single Trip','Single Trip Ride','Single Trip (Pay-as-you-ride)',
          'RideScout Single Ride','RideScout Single Tide','Walk Up'
        ) THEN 'Pago por viaje'

        -- Pase corto plazo / evento
        WHEN TRIM(subscriber_type) IN (
          '3-Day Explorer','Explorer','Explorer ($8 plus tax)',
          '3-Day Weekender','Weekender','Weekender ($15 plus tax)',
          '7-Day','ACL 2019 Pass','ACL Weekend Pass Special',
          'FunFunFun Fest 3 Day Pass'
        ) THEN 'Pase corto plazo / evento'

        WHEN TRIM(subscriber_type) = 'RESTRICTED'
        THEN 'Restringido / interno'

        ELSE 'Otro'
     END AS tipo_suscripcion

FROM bigquery-public-data.austin_bikeshare.bikeshare_trips
WHERE 
      start_station_name IS NOT NULL AND TRIM(start_station_name) <> '' 
      AND start_station_id IS NOT NULL 
      AND end_station_name IS NOT NULL AND TRIM(end_station_name) <> '' 
      AND end_station_id IS NOT NULL
      AND subscriber_type IS NOT NULL
      AND duration_minutes > 0
      AND end_station_id <> 'Event'
      AND duration_minutes > 0 AND  duration_minutes <= 300
      AND EXTRACT(YEAR FROM start_time) BETWEEN 2014 AND 2023;
~~~

---

## 4. Modelado en Power BI (Power Query + modelo de datos)

En **Power Query** se realizó:

- Importación de la tabla resultante del SQL.  
- Separación del modelo en:

**Tabla de hechos**

- `Hechos`  
  - Viaje a nivel granular (`id_viaje`).  
  - Contiene métricas como `duracion_min`, `id_bici`, `tipo_bici`, `ruta`, fechas y horas.

**Tablas de dimensiones**

- `Dim_start_estaciones`: id y nombre de la estación de inicio.  
- `Dim_end_estaciones`: id y nombre de la estación de fin.  
- `Dim_subscripcion`: categorías de tipo de suscripción.  
- `Dim_tipo_bici`: tipos de bicicleta (clásica, eléctrica).  
- `Calendario`: tabla de calendario marcada como tabla de fechas, con columnas de año, mes, día, fecha corta, etc.

El modelo final es un **esquema estrella**, con la tabla `Hechos` en el centro y relaciones 1:* hacia cada dimensión.

---

## 5. Medidas DAX y columnas calculadas

### 5.1. Medidas principales

~~~DAX
Total_viajes =
COUNT ( Hechos[id_bici] )

total_viajes_AA =
CALCULATE ( [Total_viajes], DATEADD ( Calendario[Fecha], -1, YEAR ) )

cambio % vs AA =
VAR actual = [Total_viajes]
VAR aa     = [total_viajes_AA]
RETURN
    IF (
        ISBLANK ( aa ),
        BLANK (),
        DIVIDE ( actual - aa, aa )
    )

numero de estaciones =
DISTINCTCOUNT ( Hechos[id_inicio_estacion] )

Sum_duracion_minutos =
SUM ( Hechos[duracion_min] )

tiempo promedio de viaje =
AVERAGE ( Hechos[duracion_min] )

total suscripcion =
COUNT ( Hechos[tipo_subscripcion] )

Total_bicis_activas =
DISTINCTCOUNT ( Hechos[id_bici] )
~~~

### 5.2. Columna calculada – Rango de duración

Para analizar en qué rangos de tiempo se concentran más los viajes, se creó la columna:

~~~DAX
Rango_duracion =
VAR dur = Hechos[duracion_min]
RETURN
    SWITCH (
        TRUE (),
        dur <= 10,  "0 - 10 min",
        dur <= 20,  "10 - 20 min",
        dur <= 30,  "20 - 30 min",
        dur <= 40,  "30 - 40 min",
        dur <= 60,  "40 - 60 min",
        dur <= 80,  "60 - 80 min",
        dur <= 100, "80 - 100 min",
        dur <= 120, "100 - 120 min",
        dur <= 140, "120 - 140 min",
        dur <= 160, "140 - 160 min",
        dur <= 180, "160 - 180 min",
        "180 min - 300 min"
    )
~~~

*(Puedes ajustar los textos de los rangos según prefieras.)*

---

## 6. Diseño del dashboard

El reporte está organizado en **3 páginas**:

1. **General**  
   - KPIs:
     - Total de viajes (≈ 2 millones).  
     - Bicis activas.  
     - Tiempo promedio de viaje.  
     - Número de estaciones.  
   - Serie temporal de **Total de viajes por año** con línea de tendencia.  
   - Tabla de **tasa de cambio anual (%)** vs año anterior.  
   - Tabla de detalle de viajes por año.

2. **Patrones del sistema**  
   - Total de viajes por **tipo de suscripción**.  
   - Total de viajes por **hora del día**.  
   - Total de viajes por **mes**.  
   - Total de viajes por **día de la semana**.

3. **Duración y tipo de bicicleta**  
   - Total de viajes por **rango de duración**.  
   - Total de viajes por **año y tipo de bici** (clásica vs eléctrica).

Todas las páginas incluyen slicers para filtrar por **año, mes y día**, permitiendo explorar distintos periodos.

---

## 7. Principales insights

Algunos hallazgos que se pueden obtener del dashboard:

- Evolución del uso del sistema a lo largo de los años e identificación de años con crecimientos/caídas fuertes.  
- Horas pico de uso diario y días de la semana con mayor demanda.  
- Distribución de la duración de los viajes (la mayoría concentrados en viajes cortos).  
- Importancia relativa de cada tipo de suscripción (pago por viaje vs membresías).  
- Penetración de las bicicletas eléctricas por año frente a las clásicas.

---

## 8. Archivos del proyecto

- `austin_bikeshare_model.pbix` – archivo de Power BI con el modelo y el dashboard.  
- `sql/limpieza_bikeshare.sql` – script SQL usado para la limpieza y transformación inicial.  
- `img/` – capturas de pantalla de las páginas del dashboard (General, Patrones del sistema, Duración y tipo de bici).

## Descarga del Archivo Power BI.
Puedes descargar el pbix desde este enlace 
[Descargar archivo Power BI](https://drive.google.com/file/d/1gfQPPvyzLxI96xcK-uUy_k-99Dv0vitN/view?usp=drive_link)
