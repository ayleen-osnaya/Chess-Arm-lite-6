# Segunda prueba de visión computacional en MATLAB
**Detección de figuras geométricas y reconocimiento de color**

Para evitar repetir los procedimientos que ya se explicaron en la [primera prueba](./primer_test.md), esta sección se enfoca en las modificaciones realizadas.

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

La segunda característica permite distinguir mejor las esquinas reales de las pequeñas variaciones de una curva. Esto ya que en el documento [primer test en las conclusiones](./primer_test.md#Conclusiones) , dijimos que seria mucho mas sencillo si los vetices tienen un peso mucho mayor, ya que como las figuras seran siempre las mismas, en todo caso las podremso detectar por numero de vertice.

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
if nAgudas s == 4
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

- **Cuadrado:** se identifica cuando tiene cuatro esquinas agudas.
- **Pentágono:** se identifica cuando tiene cinco esquinas agudas.
- **Figura D:** se identifica cuando tiene dos esquinas agudas y un `Extent` mayor a 0.85.
- **Círculo:** se identifica cuando no tiene esquinas agudas y su circularidad es mayor a 0.80.

A diferencia de la primera prueba, se eliminó la relación de aspecto como criterio para diferenciar cuadrados y rectángulos. Por esta razón, en esta versión ambos pueden clasificarse como cuadrados si presentan cuatro esquinas agudas, aunque no es un problema de hceho nos evita conflicto si por posición de cámara un cuadrado se llegara a ver como rectangulo.

También se modificaron las reglas de la Figura D. Ahora se utiliza una condición más sencilla basada en dos esquinas agudas y el valor de `Extent`, en lugar de combinar el número de esquinas con `Extent` y `Solidity`.

Una concideración que debemos tener es que estas reglas dependen de la geometría de las figuras utilizadas. Por ello, será importante comprobar su funcionamiento con diferentes orientaciones y dimensiones.

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

Los intervalos establecidos son umbrales seleccionados manualmente. Por lo tanto, pueden necesitar ajustes dependiendo de la iluminación, la cámara y los tonos utilizados en las figuras, esta claibración la realizaremos con pruebas en laboratorio cuando tengamos nuestras piezas terminadas.

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


## 7. Resultados
Aqui veremos las imagenes originales y su resutado despues de ser procesadas con una conclusion sobre las mejoras que deben realizarce. 

<table>
  <tr>
    <th>Imagen original</th>
    <th>Imagen procesada</th>
  </tr>
  <tr>
    <td><img src="images/formas.png" width="400" height="400" style="object-fit: contain;"></td>
    <td><img src="images/formas_procesada2.png" width="400" height="400" style="object-fit: contain;"></td>
  </tr>
</table>

<table>
  <tr>
    <th>Imagen original</th>
    <th>Imagen procesada</th>
  </tr>
  <tr>
    <td><img src="images/formas2.png" width="400" height="400" style="object-fit: contain;"></td>
    <td><img src="images/formas2_procesada2.png" width="400" height="400" style="object-fit: contain;"></td>
  </tr>
</table>

<table>
  <tr>
    <th>Imagen original</th>
    <th>Imagen procesada</th>
  </tr>
  <tr>
    <td><img src="images/formas3.png" width="400" height="400" style="object-fit: contain;"></td>
    <td><img src="images/formas3_procesada2.png" width="400" height="400" style="object-fit: contain;"></td>
  </tr>
</table>


<table>
  <tr>
    <th>Imagen original</th>
    <th>Imagen procesada</th>
  </tr>
  <tr>
    <td><img src="images/formas4.png" width="400" height="400" style="object-fit: contain;"></td>
    <td><img src="images/formas4_procesada2.png" width="400" height="400" style="object-fit: contain;"></td>
  </tr>
</table>

<table>
  <tr>
    <th>Imagen original</th>
    <th>Imagen procesada</th>
  </tr>
  <tr>
    <td><img src="images/formas5.png" width="400" height="400" style="object-fit: contain;"></td>
    <td><img src="images/formas5_procesada2.png" width="400" height="400" style="object-fit: contain;"></td>
  </tr>
</table>

<table>
  <tr>
    <th>Imagen original</th>
    <th>Imagen procesada</th>
  </tr>
  <tr>
    <td><img src="images/formas6.png" width="400" height="400" style="object-fit: contain;"></td>
    <td><img src="images/formas6_procesada2.png" width="400" height="400" style="object-fit: contain;"></td>
  </tr>
</table>

<table>
  <tr>
    <th>Imagen original</th>
    <th>Imagen procesada</th>
  </tr>
  <tr>
    <td><img src="images/formas7.png" width="400" height="400" style="object-fit: contain;"></td>
    <td><img src="images/formas7_procesada2.png" width="400" height="400" style="object-fit: contain;"></td>
  </tr>
</table>

<table>
  <tr>
    <th>Imagen original</th>
    <th>Imagen procesada</th>
  </tr>
  <tr>
    <td><img src="images/formas8.png" width="400" height="400" style="object-fit: contain;"></td>
    <td><img src="images/formas8_procesada2.png" width="400" height="400" style="object-fit: contain;"></td>
  </tr>
</table>


## 8. Conclusiones

Lo que pudimos observar de esta nueva versión,es que imagenes limpias o hechas en computadora no tiene ningun error, detecta bien la forma y los colores sin errores, pero al pasarlo a un ambiente real, la situacion cambia, ya no detecta las figuras ni los colores. Lo que se planea hacer es usar deteccion de caracteristicas con base a bordes en imagen, sera muy util por que a la hora de agregar camara no se hara una gran modificación de código.

## Anexos

```Matlab
function detectar_formas_color(nombreImagen)

    if nargin < 1
        nombreImagen = 'formas4.png';
    end

    %% 1) LEER LA IMAGEN
    I  = imread(nombreImagen);
    Id = im2double(I);
    if size(Id,3) == 1
        Id = repmat(Id, [1 1 3]);
    end
    Id = Id(:,:,1:3);   % por si el PNG trae canal alfa

    %% 2) SEGMENTAR POR DIFERENCIA DE COLOR CON EL FONDO
    % Color del fondo = mediana de los pixeles del borde de la imagen
    bordes = [squeeze(Id(1,:,:)); squeeze(Id(end,:,:)); ...
              squeeze(Id(:,1,:)); squeeze(Id(:,end,:))];
    fondo = median(bordes, 1);

    % Distancia de cada pixel al color del fondo (en RGB)
    dif = sqrt(sum((Id - reshape(fondo, 1, 1, 3)).^2, 3));

    % Todo lo que se aleja del fondo es objeto
    BW = dif > 0.15;

    %% 3) LIMPIAR LA IMAGEN BINARIA
    BW = imfill(BW, 'holes');
    BW = bwareaopen(BW, 200);
    BW = imopen(BW, strel('disk', 2));

    %% 4) MEDIR CADA REGION
    props = regionprops(BW, 'Area', 'Perimeter', 'Centroid', ...
        'Solidity', 'MajorAxisLength', 'MinorAxisLength', ...
        'Extent', 'PixelIdxList');

    fronteras = bwboundaries(BW, 'noholes');

    %% 5) CLASIFICAR FORMA Y COLOR, Y DIBUJAR
    figure; imshow(I); hold on;
    title('Formas y colores detectados');

    for k = 1:numel(props)
        P = fronteras{k};
        P = [P(:,2), P(:,1)];     % [fila col] -> [x y]
        P(end,:) = [];            % el ultimo punto repite al primero

        % --- Forma ---
        [nVert, nAgudas] = contar_esquinas(P);
        circ = 4*pi*props(k).Area / props(k).Perimeter^2;
        relAspecto = props(k).MajorAxisLength / props(k).MinorAxisLength;

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

        % --- Color ---
        % Mediana de R, G y B de los pixeles de ESTE objeto
        idx = props(k).PixelIdxList;
        R = Id(:,:,1); G = Id(:,:,2); B = Id(:,:,3);
        rgbObj = [median(R(idx)), median(G(idx)), median(B(idx))];
        color = nombre_color(rgbObj);

        % --- Dibujar ---
        etiqueta = sprintf('%s %s', forma, color);
        plot(P(:,1), P(:,2), 'g', 'LineWidth', 2);
        c = props(k).Centroid;
        text(c(1), c(2), etiqueta, 'Color', 'r', 'FontSize', 11, ...
            'FontWeight', 'bold', 'HorizontalAlignment', 'center', ...
            'BackgroundColor', [1 1 1 0.7]);

        fprintf(['Objeto %d: %s | color=%s | vertices=%d agudas=%d ' ...
                 'circ=%.2f extent=%.2f solidez=%.2f aspecto=%.2f\n'], ...
            k, forma, color, nVert, nAgudas, circ, ...
            props(k).Extent, props(k).Solidity, relAspecto);
    end
    hold off;
end

%  FUNCIONES AUXILIARES

function nombre = nombre_color(rgb)
% Convierte un color RGB (0-1) a un nombre usando HSV.
% H (matiz) da el "color", S (saturacion) y V (brillo) separan
% negro, gris y blanco.
    hsv = rgb2hsv(reshape(rgb, 1, 1, 3));
    h = hsv(1) * 360;   % grados
    s = hsv(2);
    v = hsv(3);

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
end

function [n, nAgudas] = contar_esquinas(P)

    perim = sum(vecnorm(diff([P; P(1,:)]), 2, 2));
    tol = 0.01 * perim;

    % Un contorno cerrado se parte en dos curvas abiertas usando el
    % punto mas lejano al primero.
    [~, i2] = max(vecnorm(P - P(1,:), 2, 2));
    curva1 = P(1:i2, :);
    curva2 = [P(i2:end, :); P(1,:)];

    k1 = douglas_peucker(curva1, tol);
    k2 = douglas_peucker(curva2, tol);

    % Vertices en orden; se quitan los dos puntos de union duplicados
    V = [curva1(k1,:); curva2(k2,:)];
    V([sum(k1)+1, end], :) = [];
    n = size(V,1);

    % Angulo interior en cada vertice
    a = circshift(V,  1, 1) - V;
    b = circshift(V, -1, 1) - V;
    c = sum(a.*b, 2) ./ (vecnorm(a,2,2) .* vecnorm(b,2,2));
    ang = acosd(max(-1, min(1, c)));

    nAgudas = sum(ang < 120);
end

function keep = douglas_peucker(P, tol)
% Devuelve un vector logico indicando que puntos se conservan.
    n = size(P,1);
    keep = false(n,1);
    keep([1 n]) = true;
    if n < 3, return; end

    a = P(1,:);  b = P(n,:);  ab = b - a;  L = norm(ab);
    if L == 0
        d = vecnorm(P - a, 2, 2);
    else
        d = abs(ab(1)*(P(:,2)-a(2)) - ab(2)*(P(:,1)-a(1))) / L;
    end

    [dmax, i] = max(d);
    if dmax > tol
        keep(1:i) = douglas_peucker(P(1:i,:), tol);
        keep(i:n) = douglas_peucker(P(i:n,:), tol);
    end
end

```
