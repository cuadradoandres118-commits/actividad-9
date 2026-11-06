# 🍪 Laboratorio de galletas — Actividad 9


## 🗄️ Esquema de la base de datos

| Tabla | Columnas | Descripción |
|---|---|---|
| `lotes` | `id`, `fecha`, `temp_horno`, `minutos_horno`, `gramos_bola`, `indice_dorado_prom` | Un registro por lote horneado. `id` es la clave primaria. |
| `inspecciones` | `lote_id`, `estado_real`, `estado_detector`, `indice_dorado` | Una fila por galleta inspeccionada. `lote_id` referencia `lotes.id`. |
| `paquetes` | `lote_id`, `num_galletas`, `sellado`, `quemadas`, `aprobado` | Empaques producidos. `sellado` y `aprobado` son binarios (1/0). |
| `canastas` | `carro_id`, `num_paquetes`, `fecha` | Canastas cargadas en cada carro de transporte. |
| `entregas` | `carro_id`, `tienda_id`, `fecha`, `canastas` | Despachos por carro a cada tienda. |


---

## 📊 Datos generados

Fecha de sesión: **2026-11-06**.

| Lote | Temp. (°C) | Minutos | Gramos/bola | Índice dorado prom. | Observación |
|---|---|---|---|---|---|
| 101 | 180 | 10.1 | 30 | 0.48 | Lote base, 26 inspecciones |
| 102 | 200 | 9.1 | 30 | 0.55 | Temperatura alta |
| 103 | 125 | 34.6 | 30 | 0.25 | Temperatura baja, tiempo largo |
| 104 | 160 | 5.8 | 30 | 0.03 | Muy crudo |
| 105 | 160 | 17.2 | 30 | 0.72 | Tostado fuerte |
| 106 | 160 | 14.2 | 30 | 0.48 | Lote grande, ~120 inspecciones |

**Estados de calidad** registrados en `inspecciones`: `ok`, `cruda`, `quemada`, `rota`.

**Rangos observados:** índice de dorado de 0.03 a 1.27 · tiempo de horneado de 5.8 a 34.6 min · temperatura de 125 °C a 200 °C.

---

## 🔍 Hallazgos de calidad

- **Lote 101:** 26 inspecciones, 5 galletas quemadas (índice > 1.0), 1 cruda, 1 rota. Paquetes: 11 aprobados de 12; el rechazado tenía 5 galletas en lugar de 6.
- **Lote 106:** el lote más grande y el más heterogéneo. Mezcla de `ok`, `cruda`, `rota` y `quemada`. Paquetes rechazados por sellado incompleto, por contener galletas quemadas y por conteo incorrecto (5 en vez de 6).
- **Error de detección:** al menos una galleta con `estado_real = 'rota'` fue clasificada por el detector como `'ok'`. Es el único caso de discrepancia real/detector en el dataset.
- **Logística:** 3 carros, 4 canastas de 8 paquetes cada uno por carro, entregas a 2 tiendas.

---

## 📸 Evidencia del simulador

Capturas de pantalla de las distintas etapas del proceso, tal como las ve el simulador y el detector. Cada imagen respalda una parte del análisis.

| Etapa | Captura |
|---|---|
| Mezclado y preparación de masa (ROI de chispas) | ![Mezclado](capturas/01_mezclado_roi.png) |
| Mezclado — vista cenital del bowl | ![Mezclado cenital](capturas/02_mezclado_cenital.png) |
| Interior del horno (galletas crudas y quemada) | ![Horno](capturas/03_horno.png) |
| Inspección en la cinta (galleta superpuesta) | ![Inspección](capturas/04_inspeccion_cinta.png) |
| Detector etiquetando galletas como "buena" | ![Detector](capturas/05_detector_buena.png) |
| Pesaje de galletas (29–30 g) | ![Pesaje](capturas/06_pesaje.png) |
| Logística y entregas (montacargas, muelle, tiendas) | ![Logística](capturas/07_logistica.png) |

> **Nota:** coloca las 7 capturas en la subcarpeta `capturas/` con esos nombres exactos para que las imágenes se muestren correctamente en GitHub.


### 1. Cuaderno de Colab (SQLite)

```python
import sqlite3, pandas as pd

con = sqlite3.connect("laboratorio.db")
cur = con.cursor()

cur.executescript("""
CREATE TABLE IF NOT EXISTS lotes (
  id INTEGER PRIMARY KEY, fecha TEXT, temp_horno REAL,
  minutos_horno REAL, gramos_bola REAL, indice_dorado_prom REAL);

CREATE TABLE IF NOT EXISTS inspecciones (
  id INTEGER PRIMARY KEY AUTOINCREMENT, lote_id INTEGER,
  estado_real TEXT, estado_detector TEXT, indice_dorado REAL);

CREATE TABLE IF NOT EXISTS paquetes (
  id INTEGER PRIMARY KEY AUTOINCREMENT, lote_id INTEGER,
  num_galletas INTEGER, sellado INTEGER, quemadas INTEGER, aprobado INTEGER);

CREATE TABLE IF NOT EXISTS canastas (
  id INTEGER PRIMARY KEY AUTOINCREMENT, carro_id INTEGER,
  num_paquetes INTEGER, fecha TEXT);

CREATE TABLE IF NOT EXISTS entregas (
  id INTEGER PRIMARY KEY AUTOINCREMENT, carro_id INTEGER,
  tienda_id INTEGER, fecha TEXT, canastas INTEGER);
""")

with open("laboratorio.sql", encoding="utf-8") as f:
    cur.executescript(f.read())
con.commit()

# Verificación rápida
pd.read_sql("SELECT lote_id, COUNT(*) AS n FROM inspecciones GROUP BY lote_id", con)
https://colab.research.google.com/drive/1SL1Egz7o5anubf4vZ8SHGOsDujmQZ7ii?usp=sharing

### 2. MySQL Workbench

1. Crear el esquema: `CREATE DATABASE laboratorio; USE laboratorio;`
2. Crear las tablas con `AUTO_INCREMENT` en las claves y `FOREIGN KEY (lote_id) REFERENCES lotes(id)`.
3. Ejecutar `laboratorio.sql` (o la versión deduplicada).
4. Verificar con `SELECT COUNT(*) FROM inspecciones WHERE lote_id = 101;` — debe dar **26**, no un múltiplo.

---

## 🧮 Consultas de referencia

```sql
-- Distribución de estados por lote
SELECT lote_id, estado_real, COUNT(*) AS total
FROM inspecciones
GROUP BY lote_id, estado_real
ORDER BY lote_id, total DESC;

-- Tasa de aprobación de paquetes por lote
SELECT lote_id,
       COUNT(*) AS paquetes,
       SUM(aprobado) AS aprobados,
       ROUND(100.0 * SUM(aprobado) / COUNT(*), 1) AS pct_aprobacion
FROM paquetes
GROUP BY lote_id;

-- Discrepancias entre detector y realidad
SELECT lote_id, estado_real, estado_detector, COUNT(*) AS casos
FROM inspecciones
WHERE estado_real <> estado_detector
GROUP BY lote_id, estado_real, estado_detector;

-- Relación temperatura / tiempo con el dorado promedio
SELECT id, temp_horno, minutos_horno, indice_dorado_prom
FROM lotes
ORDER BY indice_dorado_prom;

-- Paquetes entregados por tienda
SELECT tienda_id, SUM(canastas) AS canastas, SUM(canastas) * 8 AS paquetes
FROM entregas
GROUP BY tienda_id;
```

---

## 📈 Gráficas

| Gráfica | Qué muestra |
|---|---|
| Índice dorado promedio por lote | Comparación directa de los 6 lotes; evidencia el lote 104 (0.03) y el 105 (0.72) como extremos. |
| Temperatura vs. tiempo de horneado | Mapa de los parámetros de proceso; útil para ver que no hay una receta única. |
| Distribución de estados por lote | Barras apiladas `ok / cruda / quemada / rota`; el lote 106 domina en volumen. |
| Índice dorado por estado real | Dispersión que confirma los umbrales: `cruda` < 0.45, `ok` 0.41–0.67, `quemada` > 0.95. |
| Aprobación de paquetes por lote | Porcentaje de paquetes aprobados; el lote 101 pierde por conteo, el 106 por sellado y quemadas. |

---

## ⚖️ SQLite vs. MySQL

| Aspecto | SQLite (Colab) | MySQL (Workbench) |
|---|---|---|
| Instalación | Incluido en Python | Servidor + cliente |
| Tipos | Dinámicos, tolerantes | Estrictos, validan en inserción |
| Claves primarias | Opcionales | Obligatorias para integridad referencial |
| `INSERT` duplicado | Se acepta si no hay PK | Error 1062 |
| Redondeo | `ROUND()` disponible | `ROUND()` disponible |
| Ideal para | Prototipado rápido, análisis | Modelo productivo, multiusuario |

**Conclusión práctica:** SQLite perdona el archivo tal como viene; MySQL lo rechaza, y eso es justamente lo que hace visible el problema de duplicación.

---
## nuestro juego

file:///C:/Users/ESTUDIANTE/Desktop/buscaminas/.html

"C:\Users\ESTUDIANTE\Desktop\buscaminas"

## ✅ Conclusiones

1. El índice de dorado es un buen proxy de calidad: separa con claridad `cruda`, `ok` y `quemada`.
2. El tiempo de horneado pesa más que la temperatura dentro de la ventana probada: a 160 °C, pasar de 5.8 a 17.2 min eleva el índice de 0.03 a 0.72.
3. El lote 106, con parámetros "correctos" (160 °C, 14.2 min), produce la mayor variabilidad, lo que sugiere falta de uniformidad en el horno o en el tamaño de las bolas.
4. El detector falló al menos una vez (rota → ok), por lo que no puede usarse como único criterio de liberación.
5. El export del simulador necesita deduplicación antes de cargarse en un motor con restricciones.

---


