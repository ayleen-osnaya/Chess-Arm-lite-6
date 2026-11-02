# Primeras Pruebas de vision computacional en matlab

## Detección de figuras sin tomar en cuenta el color

# Descripcion y requisitos

la función **DETECTAR_FORMAS**  Detecta cuadrados, círculos, pentágonos, en una imagen.

Uso:   **detectar_formas('formas.png')**
**'formas.png'** es una imagen que debe estar guardada en la misma direccion que tu código

Requiere: **Image Processing Toolbox**

## Metodología
Pipeline:
 1) Leer imagen 
 2) Escala de grises 
 3) Binarizar
 4) Limpiar (morfología)  
 5) Medir regiones (regionprops)
 6) Contar esquinas del contorno  
 7) Clasificar y dibujar


# Explicación del código 
El código completo se encuentra en Anexos de este reporte


```py
codigo
```



