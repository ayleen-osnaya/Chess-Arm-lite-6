# Segunda prueba de visión computacional en MATLAB
**Detección de figuras geométricas y reconocimiento de color**

Para evitar repetir los procedimientos que ya se explicaron en la [primera prueba](#primera-pruebas-de-vision-computacional-en-matlab), esta sección se enfoca en las modificaciones realizadas.

El código completo se encuentra en [Anexos](#Anexos) de este reporte 

### 1. Preparación de la imagen

```matlab
Id = im2double(I);
if size(Id,3) == 1
    Id = repmat(Id, [1 1 3]);
end
Id = Id(:,:,1:3);
```

Se agregó la última línea:

`Id = Id(:,:,1:3)`

Esta instrucción conserva únicamente los tres canales RGB de la imagen. Es importante porque algunos archivos PNG pueden incluir un cuarto canal llamado **alfa**, que representa la transparencia.

Al eliminarlo, evitamos problemas al realizar operaciones que necesitan trabajar exclusivamente con los canales rojo, verde y azul.

### 2. Segmentación y limpieza

Se eliminó la condición que invertía automáticamente la imagen binaria:

```matlab
borde = [BW(1,:), BW(end,:), BW(:,1)', BW(:,end)'];
if mean(borde) > 0.5
    BW = ~BW;
end
```

En esta versión se trabaja directamente con el resultado de la comparación `dif > 0.15`, donde los píxeles diferentes al fondo representan los objetos.

Esto simplifica el procedimiento, pero también implica que debemos comprobar que la segmentación mantenga correctamente los objetos en blanco y el fondo en negro.

### 3. Nuevas propiedades de las regiones

```matlab
props = regionprops(BW, 'Area', 'Perimeter', 'Centroid', ...
    'Solidity', 'MajorAxisLength', 'MinorAxisLength', ...
    'Extent', 'PixelIdxList');
```

Se agregó una propiedad nueva:

- **`PixelIdxList`:** contiene los índices de todos los píxeles que pertenecen a una región específica. Esto permite recuperar los valores RGB de cada figura y calcular su color.


## 4. Mejoras en la clasificación de las figuras

Uno de los principales cambios de esta prueba se encuentra en la función `contar_esquinas()`.

En la primera versión se contaban los puntos importantes obtenidos mediante la simplificación Douglas-Peucker. Sin embargo, esto ocasionaba problemas porque los puntos de una curva podían interpretarse como vértices.

Para mejorar la clasificación, ahora se diferencian dos características:

- **`nVert`:** cantidad total de vértices obtenidos después de simplificar el contorno.
- **`nAgudas`:** cantidad de vértices cuyo ángulo interior es menor a 120°.

La segunda característica permite distinguir mejor las esquinas reales de las pequeñas variaciones de una curva.

### Nueva tolerancia

```matlab
perim = sum(vecnorm(diff([P; P(1,:)]), 2, 2));
tol = 0.01 * perim;
```

Se modificó la tolerancia del algoritmo Douglas-Peucker:

- Primera prueba: 2.5 % del perímetro.
- Segunda prueba: 1 % del perímetro.

Al reducir la tolerancia, se conservan más detalles del contorno, lo que permite identificar mejor sus vértices.

### Cálculo de los ángulos interiores

```matlab
a = circshift(V,  1, 1) - V;
b = circshift(V, -1, 1) - V;

c = sum(a.*b, 2) ./ ...
    (vecnorm(a,2,2) .* vecnorm(b,2,2));

ang = acosd(max(-1, min(1, c)));

nAgudas = sum(ang < 120);
```

Una vez obtenidos los vértices, se calcula el ángulo interior de cada uno.

Para ello:

1. Se obtiene el vector que conecta cada vértice con el anterior.
2. Se obtiene el vector que conecta el vértice con el siguiente.
3. Se utiliza el producto punto para calcular el ángulo entre ambos vectores.
4. `acosd()` convierte el resultado a grados.
5. Se cuentan únicamente los ángulos menores a 120°.

La intención es que las esquinas pronunciadas sean consideradas vértices reales, mientras que las pequeñas variaciones de una curva no se contabilicen como esquinas.

### Nuevas reglas de clasificación

```matlab
if nAgudas == 3
    forma = 'Triangulo';

elseif nAgudas == 4
    forma = 'Cuadrado';

elseif nAgudas == 5
    forma = 'Pentagono';

elseif nAgudas == 2 && props(k).Extent > 0.85
    forma = 'Figura D';

elseif nAgudas == 0 && circ > 0.80
    forma = 'Circulo';

else
    forma = 'Desconocida';
end
```

Se modificaron las condiciones de clasificación para utilizar principalmente el número de esquinas agudas.

- **Triángulo:** se identifica cuando tiene tres ángulos interiores menores a 120°.
- **Cuadrado:** se identifica cuando tiene cuatro esquinas agudas.
- **Pentágono:** se identifica cuando tiene cinco esquinas agudas.
- **Figura D:** se identifica cuando tiene dos esquinas agudas y un `Extent` mayor a 0.85.
- **Círculo:** se identifica cuando no tiene esquinas agudas y su circularidad es mayor a 0.80.

A diferencia de la primera prueba, se eliminó la relación de aspecto como criterio para diferenciar cuadrados y rectángulos. Por esta razón, en esta versión ambos pueden clasificarse como cuadrados si presentan cuatro esquinas agudas.

También se modificaron las reglas de la Figura D. Ahora se utiliza una condición más sencilla basada en dos esquinas agudas y el valor de `Extent`, en lugar de combinar el número de esquinas con `Extent` y `Solidity`.

**Consideración:** estas reglas dependen de la geometría de las figuras utilizadas. Por ejemplo, un triángulo con un ángulo interior de 120° o mayor podría no cumplir la condición de tres esquinas agudas. Por ello, será importante comprobar su funcionamiento con diferentes orientaciones y dimensiones.

## 5. Reconocimiento del color

Esta es la principal funcionalidad nueva de la segunda prueba.

Para identificar el color de cada figura, primero se recuperan sus píxeles mediante `PixelIdxList`.

```matlab
idx = props(k).PixelIdxList;

R = Id(:,:,1);
G = Id(:,:,2);
B = Id(:,:,3);

rgbObj = [median(R(idx)), median(G(idx)), median(B(idx))];

color = nombre_color(rgbObj);
```

Se separan los tres canales de color de la imagen y se obtiene la mediana de los valores RGB correspondientes a cada objeto.

Se utiliza la mediana en lugar de un único píxel porque permite obtener un color representativo de toda la figura y reducir la influencia de pequeñas variaciones o ruido.

El resultado es un vector RGB con tres valores entre 0 y 1, que posteriormente se envía a la función `nombre_color()`.

### Función `nombre_color(rgb)`

```matlab
hsv = rgb2hsv(reshape(rgb, 1, 1, 3));

h = hsv(1) * 360;
s = hsv(2);
v = hsv(3);
```

En esta función se convierte el color del formato RGB al formato HSV.

A diferencia de RGB, HSV permite separar tres características:

- **H (Hue o matiz):** determina el tipo de color, como rojo, amarillo o azul.
- **S (Saturation o saturación):** indica qué tan intenso o apagado es el color.
- **V (Value o brillo):** indica qué tan claro u oscuro es.

Esto facilita la identificación de colores porque podemos utilizar el matiz para reconocer los colores principales y la saturación y el brillo para distinguir el negro, gris y blanco.

### Reglas para identificar los colores

```matlab
if v < 0.20
    nombre = 'negro';

elseif s < 0.15 && v > 0.80
    nombre = 'blanco';

elseif s < 0.25
    nombre = 'gris';

elseif h < 15 || h >= 345
    nombre = 'rojo';

elseif h < 33
    nombre = 'naranja';

elseif h < 70
    nombre = 'amarillo';

elseif h < 170
    nombre = 'verde';

elseif h < 200
    nombre = 'cian';

elseif h < 260
    nombre = 'azul';

elseif h < 300
    nombre = 'morado';

else
    nombre = 'rosa';
end
```

Las condiciones se evalúan en orden:

1. Primero se revisa el brillo para identificar el negro.
2. Después se revisa la saturación y el brillo para identificar el blanco.
3. Se identifica el gris cuando la saturación es baja.
4. Finalmente, se utiliza el matiz (`h`) para clasificar los demás colores según intervalos de grados.

Los intervalos establecidos son umbrales seleccionados manualmente. Por lo tanto, pueden necesitar ajustes dependiendo de la iluminación, la cámara y los tonos utilizados en las figuras.

## 6. Etiquetado de las figuras

Otra modificación es que ahora se genera una etiqueta que combina la forma y el color detectados.

```matlab
etiqueta = sprintf('%s %s', forma, color);

plot(P(:,1), P(:,2), 'g', 'LineWidth', 2);

c = props(k).Centroid;

text(c(1), c(2), etiqueta, 'Color', 'r', ...
    'FontSize', 11, 'FontWeight', 'bold', ...
    'HorizontalAlignment', 'center', ...
    'BackgroundColor', [1 1 1 0.7]);
```

Por ejemplo, si el programa identifica un círculo azul, la etiqueta mostrará:

`Circulo azul`

El contorno se mantiene en color verde y la etiqueta en rojo, utilizando el centroide como punto de referencia.

También se modificó el fondo de la etiqueta, aumentando su transparencia a 0.7.

## 7. Resultados

En esta sección se compararán los resultados de la segunda prueba con los obtenidos anteriormente.

Se analizará si las modificaciones en la detección de esquinas mejoran el reconocimiento de las figuras y si la conversión a HSV permite identificar correctamente los colores.

<!-- Agregar aquí las comparaciones de imágenes originales y procesadas -->

## 8. Conclusiones

En esta segunda prueba se amplió el sistema de visión computacional para reconocer tanto la geometría como el color de cada objeto.

La principal mejora fue incorporar el cálculo de ángulos interiores para distinguir las esquinas reales de las variaciones presentes en los contornos curvos. Además, se implementó un clasificador de color mediante HSV, utilizando la mediana de los píxeles de cada región para obtener un color representativo.

Con estas modificaciones buscamos que el reconocimiento de las piezas del ajedrez sea más completo, ya que ahora no dependerá únicamente de la forma, sino también del color.

Los siguientes ajustes dependerán de los resultados experimentales, especialmente de la clasificación de la Figura D, la diferenciación entre cuadrados y rectángulos y la identificación de colores bajo diferentes condiciones de iluminación.

## Anexos

El código completo de esta segunda prueba se encuentra en el archivo `detectar_formas_color.m`.