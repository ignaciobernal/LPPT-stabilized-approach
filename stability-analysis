""" Core kinematic and scoring functions for the LPPT Stabilized Approach analysis """
import pandas as pd
# 1. Cargar el CSV completo
print("Loading data...")
df = pd.read_csv("datos_lisboa_completos.csv")
# 2. Ver el tamaño real
print(f"Total rows: {len(df)}")
# 3. Ver el nombre exacto de las columnas (crucial para los siguientes pasos)
print("\n--- COLUMN NAMES ---")
print(df.columns.tolist())
# 4. Echar un vistazo a las primeras 5 filas
print("\n--- FIRST 5 ROWS ---")
display(df.head()) # Si falla 'display', cámbialo por print(df.head())
# 5. Diccionario para traducir de los nombres feos a nombres limpios
traduccion_columnas = {
    'Column1.properties.Time update (HH:mm:ss.SSS):': 'time',
    'Column1.properties.Callsign (String)': 'callsign',
    'Latitude (º)': 'lat',
    'Longitude (º)': 'lon',
    'Column1.properties.Measured flight level (int)': 'flight_level',
    'Column1.properties.ADEP': 'adep',
    'Column1.properties.ADES': 'ades',
    'Column1.properties.ATYPE': 'aircraft_type',
    'vx (m/s)': 'vx',
    'vy (m/s)': 'vy'
}
print("Cleaning columns...")
# Nos quedamos solo con esas columnas y las renombramos
df_limpio = df[list(traduccion_columnas.keys())].rename(columns=traduccion_columnas)
# 6. Filtrar el ruido: Solo vuelos relacionados con Lisboa (LPPT)
print("Filtering flights to Lisbon...")
df_lppt = df_limpio[(df_limpio['adep'] == 'LPPT') | (df_limpio['ades'] == 'LPPT')].copy()
# 7. Convertir Flight Level a Altitud en pies (Fórmula: FL * 100)
# Así ya tenemos los famosos 1000 pies y 500 pies listos para usar
df_lppt['altitud_ft'] = df_lppt['flight_level'] * 100
print(f"Filtering completed! We have kept {len(df_lppt)} useful rows.")
# Vemos el resultado final, mucho más amigable
display(df_lppt.head())
