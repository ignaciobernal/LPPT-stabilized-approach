## 1. Vertical rate
Using the filtered altitude profile, this function calculates the aircraft's sink rate in feet per minute (fpm). This is a critical parameter for identifying high-energy descents that breach the -1000 fpm safety limit below the 1000 ft gate.
```python
# --- BLOCK 3: ROBUST RAW VERTICAL RATE ---
print("Calculating robust raw Vertical Rate (fpm)...")

# Al usar periods=5 comparamos cada lectura con la de 5 pings atrás.
# Esto puentea el "efecto escalera" de la cuantización del radar Mode S.
dt = df_llegadas.groupby('flight_id')['time'].diff(periods=5).dt.total_seconds()
dz = df_llegadas.groupby('flight_id')['altitud_ft'].diff(periods=5)

# Tasa de descenso media de la ventana en pies por minuto
df_llegadas['vertical_rate_fpm'] = (dz / dt) * 60

# Las primeras 5 filas de cada vuelo darán NaN al no tener 5 previas. 
# Como corresponden al inicio del descenso a FL300 y a nosotros nos importa la puerta de 1000 ft, 
# rellenamos con 0 de forma segura.
df_llegadas['vertical_rate_fpm'] = df_llegadas['vertical_rate_fpm'].fillna(0)

# Limpieza estricta: si por algún microcorte dt es exactamente 0, evitamos infinitos matemáticos
df_llegadas['vertical_rate_fpm'] = df_llegadas['vertical_rate_fpm'].replace([float('inf'), float('-inf')], 0)

print("Robust Vertical Rate calculated successfully.") ´´´

## 1. Barometric Calibration (QNH)
Altitudes transmitted by Mode-S transponders are referenced to standard atmospheric pressure (1013.25 hPa). This function applies the local meteorological correction (QNH) using historical airport data to derive the true altitude above mean sea level and, subsequently, above ground level (AGL).

```python
# Paste your QNH calibration function here
