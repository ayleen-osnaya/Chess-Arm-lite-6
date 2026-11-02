# Primeras Pruebas de vision computacional en matlab

## Detección de figuras sin tomar en cuenta el color

## Descripcion y requisitos

la función **DETECTAR_FORMAS**  Detecta cuadrados, círculos, pentágonos, en una imagen.

Uso:   **detectar_formas('formas.png')**
**'formas.png'** es una imagen que debe estar guardada en la misma direccion que tu código

Requiere: **Image Processing Toolbox**

### Metodología
Pipeline:
 1) Leer imagen 
 2) Escala de grises 
 3) Binarizar
 4) Limpiar (morfología)  
 5) Medir regiones (regionprops)
 6) Contar esquinas del contorno  
 7) Clasificar y dibujar


## Explicación del código 
El código completo se encuentra en Anexos de este reporte


```Matlab
codigo
```

## Anexos

```Matlab
function detectar_formas(nombreImagen)

    if nargin < 1
        nombreImagen = 'formas3.png';
    end

    %% 1) LEER LA IMAGEN
    I = imread(nombreImagen);

    %% 2) y 3) SEGMENTAR POR DIFERENCIA DE COLOR CON EL FONDO
    Id = im2double(I);
    if size(Id,3) == 1
        Id = repmat(Id, [1 1 3]);
    end
    
    % Color del fondo = mediana de los píxeles del borde de la imagen
    bordes = [squeeze(Id(1,:,:)); squeeze(Id(end,:,:)); ...
              squeeze(Id(:,1,:)); squeeze(Id(:,end,:))];
    fondo = median(bordes, 1);
    
    % Distancia de cada píxel al color del fondo (en RGB)
    dif = sqrt(sum((Id - reshape(fondo, 1, 1, 3)).^2, 3));
    
    % Todo lo que se aleja del fondo es objeto
    BW = dif > 0.15;

    % Si el fondo es blanco y las figuras oscuras, queda al revés.
    % Truco: si la mayoría de los píxeles del borde son blancos, invertimos.
    borde = [BW(1,:), BW(end,:), BW(:,1)', BW(:,end)'];
    if mean(borde) > 0.5
        BW = ~BW;
    end

    %% 4) LIMPIAR LA IMAGEN BINARIA
    BW = imfill(BW, 'holes');          % rellena huecos dentro de las figuras
    BW = bwareaopen(BW, 200);          % elimina manchas pequeñas (ruido)
    BW = imopen(BW, strel('disk', 2)); % suaviza bordes con morfología

    %% 5) MEDIR CADA REGIÓN
    props = regionprops(BW, 'Area', 'Perimeter', 'Centroid', ...
    'BoundingBox', 'Solidity', 'MajorAxisLength', 'MinorAxisLength', 'Extent');

    % Contornos de cada objeto (misma numeración que regionprops)
    fronteras = bwboundaries(BW, 'noholes');

    %% 6) y 7) CLASIFICAR Y DIBUJAR
    figure; imshow(I); hold on;
    title('Formas detectadas');

    for k = 1:numel(props)
        P = fronteras{k};
        P = [P(:,2), P(:,1)];          % [fila col] -> [x y]
        P(end,:) = [];                 % el último punto repite al primero

        nEsquinas = contar_esquinas(P);

        % Circularidad: 1 = círculo perfecto, menor = más "picudo"
        circ = 4*pi*props(k).Area / props(k).Perimeter^2;
        relAspecto = props(k).MajorAxisLength / props(k).MinorAxisLength;

        % --- Reglas de decisión ---
        if nEsquinas == 3
            nombre = 'Triangulo';
        elseif nEsquinas == 4
            if relAspecto < 1.15
                nombre = 'Cuadrado';
            else
                nombre = 'Rectangulo';
            end
        elseif nEsquinas == 5
            nombre = 'Pentagono';
        elseif nEsquinas >= 6 && circ > 0.80 && props(k).Solidity > 0.9
            nombre = 'Circulo';

        elseif nEsquinas >= 1 && nEsquinas <= 2 && ...
                props(k).Extent > 0.86 && props(k).Extent < 0.95 && ...
                props(k).Solidity > 0.95
            nombre = 'Figura D';

        else
            nombre = 'Desconocida';
        end

        % Dibujar contorno y etiqueta
        plot(P(:,1), P(:,2), 'g', 'LineWidth', 2);
        c = props(k).Centroid;
        text(c(1), c(2), nombre, 'Color', 'r', 'FontSize', 12, ...
            'FontWeight', 'bold', 'HorizontalAlignment', 'center', ...
            'BackgroundColor', [1 1 1 0.6]);

        fprintf(['Objeto %d: %s | esquinas=%d circ=%.2f ' ...
            'extent=%.2f solidez=%.2f aspecto=%.2f\n'], ...
            k, nombre, nEsquinas, circ, props(k).Extent, ...
            props(k).Solidity, relAspecto);
    end
    hold off;
end


%  FUNCIONES AUXILIARES

function n = contar_esquinas(P)

    % Tolerancia: 2.5% del perímetro. Si detecta de más/menos, ajústala.
    perim = sum(vecnorm(diff([P; P(1,:)]), 2, 2));
    tol = 0.025 * perim;

    % Un contorno cerrado se parte en dos curvas abiertas usando el
    % punto más lejano al primero.
    [~, i2] = max(vecnorm(P - P(1,:), 2, 2));
    curva1 = P(1:i2, :);
    curva2 = [P(i2:end, :); P(1,:)];

    k1 = douglas_peucker(curva1, tol);
    k2 = douglas_peucker(curva2, tol);

    n = sum(k1) + sum(k2) - 2;   % los 2 puntos de unión se cuentan doble
end

function keep = douglas_peucker(P, tol)
% Devuelve un vector lógico indicando qué puntos se conservan.
    n = size(P,1);
    keep = false(n,1);
    keep([1 n]) = true;
    if n < 3, return; end

    a = P(1,:);  b = P(n,:);  ab = b - a;  L = norm(ab);
    if L == 0
        d = vecnorm(P - a, 2, 2);
    else
        % distancia de cada punto a la recta a-b
        d = abs(ab(1)*(P(:,2)-a(2)) - ab(2)*(P(:,1)-a(1))) / L;
    end

    [dmax, i] = max(d);
    if dmax > tol
        keep(1:i) = douglas_peucker(P(1:i,:), tol);
        keep(i:n) = douglas_peucker(P(i:n,:), tol);
    end
end
```


