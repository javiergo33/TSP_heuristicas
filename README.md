import numpy as np
from math import radians, sin, cos, sqrt, atan2
import webbrowser
import folium
import pandas as pd
import time

start_time = time.time()

# Leer datos 
def leer_datos():
    df_estadios = pd.read_excel('Datos_TFG.xlsx', sheet_name='Estadios Eurocopa 2024', decimal=',') 
    df_formula1 = pd.read_excel('Datos_TFG.xlsx', sheet_name='Circuitos Formula 1 2024', decimal=',')
    df_metro    = pd.read_excel('Datos_TFG.xlsx', sheet_name='Paradas Metro Madrid', decimal=',')
    return df_estadios, df_formula1, df_metro

df_estadios, df_formula1, df_metro = leer_datos()

def calcular_distancias_entre_dos_ciudades(coord1:  pd.Series, coord2:  pd.Series) -> float:
    R = 6371.0  # Radio de la tierra en kilometros
    # Calcular diferencias
    dlon = radians(coord2['Longitud']) - radians(coord1['Longitud'])
    dlat = radians(coord2['Latitud']) - radians(coord1['Latitud'])
    a = sin(dlat/2)**2 + cos(radians(coord1['Latitud'])) * cos(radians(coord2['Latitud'])) * sin(dlon/2)**2
    c = 2 * atan2(sqrt(a), sqrt(1 - a))
    return R * c

def calcular_distancias(df):
    n = len(df)
    distancias = np.zeros((n, n))
    for i in range(n):
        for j in range(n):
            distancias[i, j] = calcular_distancias_entre_dos_ciudades(df.iloc[i], df.iloc[j])
    return distancias

# Calcular distancias para diferentes conjuntos de datos
distancias_ciudades = calcular_distancias(df_estadios)
distancias_circuitos = calcular_distancias(df_formula1)
distancias_paradas = calcular_distancias(df_metro)

distancias=distancias_paradas

def vecino_mas_cercano(distancias, inicio): 
    num=0
    ruta = [inicio] 
    distancia_total = 0 
    no_visitadas = set(range(len(distancias))) - {inicio} 

    while no_visitadas:  
        min_distancia = float("inf")  
        ciudad_mas_cercana = -1  
        # Iterar sobre todas las ciudades no visitadas para encontrar la más cercana a la última ciudad visitada en 'ruta'
        for ciudad in no_visitadas:
            num+=1
            distancia = distancias[ruta[-1]][ciudad]  
            if distancia < min_distancia:
                min_distancia = distancia  
                ciudad_mas_cercana = ciudad  
        ruta.append(ciudad_mas_cercana)  
        distancia_total += min_distancia  
        no_visitadas.remove(ciudad_mas_cercana) 

    mejor_ruta= ruta + [inicio] 
    mejor_coste= distancia_total + distancias[ruta[-1]][inicio]
    return mejor_ruta, mejor_coste, num

def vecino_mas_cercano_mejorado(distancias): 
    n = len(distancias)
    total_iteraciones = 0
    
    # Encontrar la mejor ruta considerando cada ciudad como inicio
    mejor_ruta, mejor_coste, total_iteraciones = min(
        (vecino_mas_cercano(distancias, inicio) for inicio in range(n)),
        key=lambda x: x[1] )
    # Sumar el número de iteraciones de cada ciudad de inicio
    total_iteraciones = sum(vecino_mas_cercano(distancias, inicio)[2] for inicio in range(n))

    return mejor_ruta, mejor_coste, total_iteraciones

mejor_ruta, mejor_coste, total_iteraciones = vecino_mas_cercano_mejorado(distancias)


def greedy_insert(distancias, inicio):
    no_visitadas = set(range(len(distancias))) - {inicio} 
    ruta = [inicio, inicio] 
    distancia_total= 0 
    num=0

    # Mientras haya ciudades no visitadas
    while no_visitadas:
        distancia_total, mejor_posicion, mejor_ciudad = min(
            (
                (   distancia_total
                    - distancias[ruta[posicion_i - 1]][ruta[posicion_i]] 
                    + distancias[ruta[posicion_i- 1]][ciudad] 
                    + distancias[ciudad][ruta[posicion_i]], 
                    posicion_i,                             
                    ciudad,                                 
                )
                for ciudad in no_visitadas                  
                for posicion_i in range(1, len(ruta))         
            ),
            key=lambda x: x[0],            
        )
        num += len(no_visitadas) * (len(ruta) - 1)
        # Actualiza el recorrido insertando `mejor_ciudad` en la posición `mejor_posicion`
        ruta = ruta[:mejor_posicion] + [mejor_ciudad] + ruta[mejor_posicion:]  
        # Elimina `mejor_ciudad` del conjunto de ciudades no visitadas
        no_visitadas.remove(mejor_ciudad)

    mejor_ruta= ruta + [inicio] 
    mejor_coste= distancia_total + distancias[ruta[-1]][inicio]
    return mejor_ruta, mejor_coste, num

def greedy_insert_con_mejor_inicio(distancias):
    mejor_ruta, mejor_coste, total_iteraciones = min(
        (greedy_insert(distancias, inicio) for inicio in range(len(distancias))),
        key=lambda x: x[1])
    # Sumar el número de iteraciones de cada ciudad de inicio
    total_iteraciones = sum(
        greedy_insert(distancias, inicio)[2] for inicio in range(len(distancias)))
    
    return mejor_ruta, mejor_coste, total_iteraciones

mejor_ruta, mejor_coste, total_iteraciones = greedy_insert_con_mejor_inicio(distancias)

# Mostrar resultados
print("Mejor ruta:", [f"Estadio {i}: {df_metro.iloc[i]['Lugar']}" for i in mejor_ruta])
print("Mejor coste:", round(mejor_coste, 2), "km")
print("Iteraciones: ",total_iteraciones)

# Crear un mapa centrado en la primera ciudad de la lista
m = folium.Map(location=(df_metro.iloc[mejor_ruta[0]]['Latitud'], df_metro.iloc[mejor_ruta[0]]['Longitud']), zoom_start=6)

# Agregar marcadores para cada ciudad
for i, ciudad_idx in enumerate(mejor_ruta):
    row = df_metro.iloc[ciudad_idx]  
    if i == 0 or i == len(mejor_ruta) - 1:
        # Marcador de la ciudad inicial de la mejor ruta (color rojo)
        folium.Marker([row['Latitud'], row['Longitud']], popup=row['Lugar'], icon=folium.Icon(color='red')).add_to(m)
    else:
        # Marcadores de las otras ciudades (color azul)
        folium.Marker([row['Latitud'], row['Longitud']], popup=row['Lugar'], icon=folium.Icon(color='blue')).add_to(m)

# Agregar una polilínea para el camino óptimo
path = [df_metro.iloc[i] for i in mejor_ruta] + [df_metro.iloc[mejor_ruta[0]]]  # lista ordenada de ubicaciones de ciudades
folium.PolyLine([(city['Latitud'], city['Longitud']) for city in path], color='red', weight=3).add_to(m)

# Guardar el mapa en un archivo HTML
map_path = 'optimal_path_map.html'
m.save(map_path)
# Abrir el mapa en el navegador web por defecto
webbrowser.open(map_path)

print("--- %s segundos ---" % (time.time() - start_time))
print("------ -------- ----")
