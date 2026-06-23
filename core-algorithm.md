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

print("Robust Vertical Rate calculated successfully.")
```
## 2. Barometric Calibration (QNH)
Altitudes transmitted by Mode-S transponders are referenced to standard atmospheric pressure (1013.25 hPa). This function applies the local meteorological correction (QNH) using historical airport data to derive the true altitude above mean sea level and, subsequently, above ground level (AGL).
```python
# --- BLOCK 4: QNH CORRECTION ---
print("Applying barometric correction (QNH) based on historical weather...")

# Presiones pasadas a hPa (milibares)
qnh_2023 = 30.04 * 33.8639
qnh_2024 = 29.93 * 33.8639
qnh_std = 1013.25

# Mapeamos la presión según el año del vuelo
df_llegadas['QNH_day'] = np.where(df_llegadas['time'].dt.year == 2023, qnh_2023, qnh_2024)

# Aplicamos la fórmula física
df_llegadas['altitud_ft_corr'] = df_llegadas['altitud_ft'] + 27.3 * (df_llegadas['QNH_day'] - qnh_std)

print("True altitude computed successfully.")
```
## 3. Kinematic Filtering (Savitzky-Golay)
To eliminate the quantization noise inherent in surveillance sensors and extract realistic physical derivatives, a Savitzky-Golay smoothing filter is applied to the raw altitude arrays before calculating vertical speeds.
```python
# --- BLOCK 5: SAVITZKY-GOLAY FILTER ---
from scipy.signal import savgol_filter

print("Applying Savitzky-Golay advanced smoothing filter to descent profiles...")

def apply_savgol(data):
    # Usamos una ventana de 11 lecturas para absorber el ruido y evitar picos de 30k fpm
    if len(data) < 11: 
        return data
    return savgol_filter(data, window_length=11, polyorder=2)

# Aplicamos agrupando de forma estricta por el identificador único limpio
df_llegadas['savgol_vr_fpm'] = df_llegadas.groupby('flight_id')['vertical_rate_fpm'].transform(apply_savgol)

print("Descent profiles smoothed.")

# Verificación visual comparativa
savgol_check = df_llegadas[(df_llegadas['callsign'] == 'ACA812') & 
                           (df_llegadas['altitud_ft_corr'] <= 1500) & 
                           (df_llegadas['altitud_ft_corr'] >= 800)]

print("\nComparison at Gate Approach (Raw vs. Corrected Altitude & Smoothed VR):")
display(savgol_check[['flight_id', 'time', 'altitud_ft', 'altitud_ft_corr', 'vertical_rate_fpm', 'savgol_vr_fpm']].head(8))

# Criterio: Tasa de descenso no superior a 1000 fpm
# Como nuestro avión desciende, el valor es negativo, así que buscamos valores menores a -1000
df_llegadas['vr_unstable'] = df_llegadas['savgol_vr_fpm'] < -1000
```
## 4. Descent Angle (Glide Path)
Calculates the instantaneous vertical trajectory angle of the aircraft relative to the runway threshold plane. This metric is used to continuously verify adherence to the nominal 3.0° vertical approach cone.
```python
# --- BLOCK 7: DESCENT ANGLE (FLIGHT PATH) ---
import numpy as np

print("Step 7: Calculating Descent Angle (Gamma)...")

# 1. HOMOGENEIZACIÓN DE UNIDADES (Todo a metros por segundo)
# GS: de Nudos a m/s (1 nudo = 0.514444 m/s)
gs_mps = df_llegadas['gs_smooth_kts'] * 0.514444

# VS: de fpm a m/s (1 fpm = 0.00508 m/s)
# Usamos el valor absoluto porque la tasa de descenso es negativa y queremos un ángulo positivo
vs_mps = np.abs(df_llegadas['savgol_vr_fpm']) * 0.00508


# 2. CÁLCULO TRIGONOMÉTRICO
# np.arctan devuelve el ángulo en radianes, lo pasamos a grados con np.degrees
df_llegadas['descent_angle_deg'] = np.degrees(np.arctan(vs_mps / gs_mps))


# 3. CRITERIO SKYBRARY: "On correct flight path"
# Límite ideal 3º. Tolerancia aceptable entre 2.5º y 3.5º.
df_llegadas['angle_unstable'] = (df_llegadas['descent_angle_deg'] < 2.5) | (df_llegadas['descent_angle_deg'] > 3.5)

print("Success! Descent angle computed and evaluated.")

# --- VERIFICACIÓN VISUAL ---
print("\nAuditoría del Criterio 3 (Ángulo de Descenso) para ACA812:")
check_angulo = df_llegadas[(df_llegadas['callsign'] == 'ACA812') & 
                           (df_llegadas['altitud_ft_corr'] <= 1500) & 
                           (df_llegadas['altitud_ft_corr'] >= 800)].copy()

columnas_angulo = [
    'flight_id', 'altitud_ft_corr', 'ground_speed_kts', 
    'savgol_vr_fpm', 'descent_angle_deg', 'angle_unstable'
]

display(check_angulo[columnas_angulo].head(5))
```
## 5. Lateral Deviation
Evaluates the localizer tracking error by computing the perpendicular cross-track distance from the aircraft's current position to the extended runway centerline.
```python
# --- BLOCK 8: LATERAL ALIGNMENT (DIRECTIONAL STABILITY) ---
import numpy as np

print("Step 8: Calculating Lateral Alignment and Track Deviation...")

# Orientación magnética de las pistas de LPPT
RWY_02_HDG = 22.0
RWY_20_HDG = 202.0
LATERAL_TOLERANCE_DEG = 5.0

# 1. CÁLCULO DE DESVIACIÓN PARA CADA PISTA
# Calculamos la diferencia absoluta con la Pista 02. 
# np.minimum soluciona el "efecto brújula" (ej: pasar de 359º a 1º es una diferencia de 2º, no de 358º)
diff_02 = np.abs(df_llegadas['aircraft_heading'] - RWY_02_HDG)
diff_02 = np.minimum(diff_02, 360 - diff_02)

diff_20 = np.abs(df_llegadas['aircraft_heading'] - RWY_20_HDG)
diff_20 = np.minimum(diff_20, 360 - diff_20)

# 2. SELECCIÓN DE LA PISTA OBJETIVO
# El avión estará intentando aterrizar por la pista cuya desviación sea menor
df_llegadas['track_deviation_deg'] = np.minimum(diff_02, diff_20)

# 3. CRITERIO SKYBRARY: "On correct flight path (Localizer)"
# Inestable si el desvío es mayor a 5 grados
df_llegadas['lateral_unstable'] = df_llegadas['track_deviation_deg'] > LATERAL_TOLERANCE_DEG

print("Success! Lateral alignment evaluated.")

# --- VERIFICACIÓN VISUAL FINAL ---
print("\nAuditoría del Criterio 4 (Alineación Lateral) para ACA812:")
check_lateral = df_llegadas[(df_llegadas['callsign'] == 'ACA812') & 
                            (df_llegadas['altitud_ft_corr'] <= 1500) & 
                            (df_llegadas['altitud_ft_corr'] >= 800)].copy()

columnas_lat = [
    'flight_id', 'altitud_ft_corr', 'aircraft_heading', 
    'track_deviation_deg', 'lateral_unstable'
]

display(check_lateral[columnas_lat].head(5))
```
## 6. Dynamic Reference Speed Estimation (Vref)
In the absence of landing weight telemetry, this empirical algorithm calculates a customized baseline $V_{REF}$ for each individual flight. It achieves this by extracting the median Indicated Airspeed (IAS) during the most stable spatial window of the short final segment (e.g., between 3 NM and 1 NM).
```python
# --- BLOCK 6: PROFILING ENERGÉTICO FINAL (VIENTO + VREF EMPÍRICA DATA-DRIVEN) ---
import pandas as pd
import numpy as np
from scipy.signal import savgol_filter

print("Step 6.1: Procesando METAR y filtrando ruido de radar...")

# Limpieza de memoria
cols_to_drop = [
    'wind_dir', 'wind_speed', 'ground_speed_kts', 'gs_smooth_kts', 
    'headwind_kts', 'tas_est_kts', 'ias_est_kts', 'dynamic_vref', 'speed_unstable']
df_llegadas = df_llegadas.drop(columns=[col for col in cols_to_drop if col in df_llegadas.columns], errors='ignore')

# 1. INTEGRACIÓN DE METAR
weather_data = [
    ('2023-05-01 00:00:00', 340, 7), ('2023-05-01 01:00:00', 340, 9),
    ('2023-05-01 02:00:00', 350, 10), ('2023-05-01 03:00:00', 330, 11),
    ('2023-05-01 06:00:00', 340, 11), ('2023-05-01 09:00:00', 350, 8),
    ('2023-05-01 12:00:00', 70, 7), ('2023-05-01 16:00:00', 310, 12),
    ('2023-05-01 19:00:00', 330, 14), ('2023-05-01 22:00:00', 340, 6),
    ('2024-06-17 00:00:00', 290, 6), ('2024-06-17 03:00:00', 280, 4),
    ('2024-06-17 06:00:00', 240, 3), ('2024-06-17 09:00:00', 210, 7),
    ('2024-06-17 12:00:00', 220, 12), ('2024-06-17 14:00:00', 230, 14),
    ('2024-06-17 17:00:00', 220, 14), ('2024-06-17 20:00:00', 210, 10),
]
df_metar = pd.DataFrame(weather_data, columns=['time', 'wind_dir', 'wind_speed'])
df_metar['time'] = pd.to_datetime(df_metar['time'])
df_metar = df_metar.sort_values('time')

df_llegadas = df_llegadas.dropna(subset=['time']).sort_values('time')
df_llegadas = pd.merge_asof(df_llegadas, df_metar, on='time', direction='backward')

# 2. SUAVIZADO DE GROUND SPEED
gs_mps_raw = np.sqrt(df_llegadas['vx']**2 + df_llegadas['vy']**2)
df_llegadas['ground_speed_kts'] = gs_mps_raw * 1.94384

df_llegadas['gs_smooth_kts'] = df_llegadas.groupby('flight_id')['ground_speed_kts'].transform(
    lambda x: savgol_filter(x, window_length=15, polyorder=2) if len(x) > 15 else x
)

print("Step 6.2: Cálculo aerodinámico (Gradiente de viento y TAS/IAS)...")

# 3. CÁLCULO DE VIENTO Y IAS
df_llegadas['aircraft_heading'] = (np.degrees(np.arctan2(df_llegadas['vx'], df_llegadas['vy'])) % 360)
angulo_relativo = np.radians(df_llegadas['wind_dir'] - df_llegadas['aircraft_heading'])

df_llegadas['headwind_kts'] = df_llegadas['wind_speed'] * 1.5 * np.cos(angulo_relativo)
df_llegadas['tas_est_kts'] = df_llegadas['gs_smooth_kts'] + df_llegadas['headwind_kts']
df_llegadas['ias_est_kts'] = df_llegadas['tas_est_kts'] / (1 + (0.02 * (df_llegadas['altitud_ft_corr'] / 1000.0)))

print("Step 6.3: Extracción de VREF Empírica (Data-Driven: 1000ft - 300ft)...")

# 4. EXTRACCIÓN DE VREF DINÁMICA (Data-Driven Estimation)
# Calculamos la mediana en el tramo de aproximación final en lugar del umbral de pista
tramo_estable = df_llegadas[(df_llegadas['altitud_ft_corr'] <= 1000) & (df_llegadas['altitud_ft_corr'] >= 300)]
df_vref = tramo_estable.groupby('flight_id')['ias_est_kts'].median().reset_index()
df_vref.rename(columns={'ias_est_kts': 'dynamic_vref'}, inplace=True)

df_llegadas = pd.merge(df_llegadas, df_vref, on='flight_id', how='left')

# 5. CRITERIOS OPERATIVOS (Adaptados a los Tips del Documento)
# Tolerancia: +20 kts / -10 kts respecto a la VREF extraída
limite_superior = df_llegadas['ias_est_kts'] > (df_llegadas['dynamic_vref'] + 20)
limite_inferior = df_llegadas['ias_est_kts'] < (df_llegadas['dynamic_vref'] - 10)

df_llegadas['speed_unstable'] = limite_superior | limite_inferior

print("\nSuccess! VREF basada en mediana y nuevos márgenes de tolerancia aplicados.")
```
