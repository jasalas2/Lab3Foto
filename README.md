# Detección de Incendios usando Sentinel-1 y Sentinel-2
## Descripción
El siguiente análisis utiliza imágenes de Sentinel-1 y Sentinel-2 para detectar incendios en la región de Palo Verde, Costa Rica, durante el año 2023.

## Análisis de incendios Sentinel-1 

### Colección de imágenes de Sentinel-1

```javascript

var s1 = ee.ImageCollection('COPERNICUS/S1_GRD') // Paquete S1
    .filter(ee.Filter.eq('instrumentMode', 'IW'))
    .filter(ee.Filter.eq('orbitProperties_pass', 'DESCENDING')) // Puede ajustarse a ASCENDING para ver otra dirección
    .filterBounds(roi)

```

### Filtrado de Imágenes de antes y después del Incendio
Se usó el rango de fechas del 4 de abril del 2023 al 6 de junio del 2023.

```javascript

var beforeinc = s1.filterDate('2023-04-01', '2023-04-28')  // Fecha antes del incendio
print(beforeinc, 'imagenes disponibles antes del incendio')

var afterinc = s1.filterDate('2023-05-10', '2023-06-01')
print(afterinc,'imagenes disponibles despues del incendio')

```

### Convertir ImageCollection a una Imagen


```javascript

var beforeinc = beforeinc.mosaic().clip(roi) // Se puede cambiar mosaic por mean o median
var afterinc =  afterinc.mosaic().clip(roi)
print(beforeinc, 'Imagen antes del incendio')
print(afterinc, 'Imagen despues del incendio')

```

### Visualización de las Imágenes de Sentinel-1 sin filtro de speckle

```javascript

Map.addLayer(beforeinc, {bands: ['VV'], min: -15, max: -5, gamma: 1.2}, 'antes del incendio sin speckle',0);
```

### Filtro para reducir el speckle
El radio del filtro de speckle es de 40 con el fin de mejorar el suavizado de las imágenes.

```javascript

var SMOOTHING_RADIUS = 40;
var beforeinc = beforeinc.focal_mean(SMOOTHING_RADIUS, 'circle', 'meters'); // Presenta filtro de speckle
var afterinc = afterinc.focal_mean(SMOOTHING_RADIUS, 'circle', 'meters');

```


### Parámetros de visualización para las imágenes de Sentinel-1
Se usa la banda VH debido a que tiene un gran impacto al analizar vegetación
```javascript

var visualization = {
  bands: ['VH'], 
  min: -20,
  max: -5,
};
```


### Visualiazr las imágenes de Sentinel-1 con filtro de speckle

```javascript

Map.addLayer( beforeinc,visualization, 'Imagen antes del incendio Sentinel-1',0);
Map.addLayer(afterinc, visualization, 'Imagen despues del incendio Sentinel-1',0);

```


### Unión de las bandas de antes y después

```javascript

var coll = beforeinc.addBands(afterinc)
print(coll, 'coleccion junta')

```


### Detección de cambio
Crea una imagen de cambio al comparar las bandas VH de dos imágenes (antes y después del incendio). 

```javascript
var change = coll.expression ('VH / VH_1', {
    'VH': coll.select ('VH'),  // Antes
    'VH_1': coll.select ('VH_1')}) // Después
    .toDouble().rename('change');

Map.addLayer(change, {min: 0,max:2},'Raster de cambio', 0);
print(change, 'cambio')

var coll2 = coll.addBands(change)
print(coll2, 'coleccion junta con cambio')
```

### Aplicación de un umbral de diferencia
Detecta zonas quemadas comparando imágenes antes y después del incendio. Usa un umbral de 0.75 para distinguir entre las zonas afectadas por el incendio y las no afectadas.
```javascript

var DIFF_UPPER_THRESHOLD = 0.75; 
var zonas_quemadas = change.lt(DIFF_UPPER_THRESHOLD);
Map.addLayer(zonas_quemadas.updateMask(zonas_quemadas),{palette:"D5421E"},'zonas quemadas',1);
```

## Análisis de incendios Sentinel-2 

### Colección de imágenes de Sentinel-2 junto al emascarado de nubes

Enmascarado de nubes
```javascript

// Sentinel-2 cloud masking
function cloudMask(image){
  var scl = image.select('SCL');
  var mask = scl.eq(3).or(scl.gte(7).and(scl.lte(10)));
  return image.updateMask(mask.eq(0));
}
```
Filtro de colección Sentinel-2
```javascript
//Filtar la coleccion de Sentinel-2
var image_collec = ee.ImageCollection("COPERNICUS/S2_SR_HARMONIZED").filterBounds(roi) 
  .filterDate('2023-01-01', '2023-12-31') 
  .filterBounds(roi) // 
  .map(cloudMask) // Se ejecuta el enmascador de nubes que previamente realizado 
  print(image_collec)  
  
```

### Filtro de imágenes antes y después del incendio en Sentinel-2

Imágenes antes del incendio S2
```javascript
var image_pre = image_collec.filter(ee.Filter.or(
 ee.Filter.date('2023-01-01', '2023-04-15')))
print(image_pre, 'imagenes disponibles antes del incendio');

```
Imágenes después del incendio S2
```javascript
var image_post = image_collec.filter(ee.Filter.or(
 ee.Filter.date('2023-04-16', '2023-06-01')))
print(image_post, 'imagenes despues del incendio');
```

### Creación de una composición y visualización de imágenes de Sentinel-2

Compuesta de imágenes antes y después del incendio
```javascript

var image_pre_median = image_pre.median().clip(roi)
var image_post_median = image_post.median().clip(roi)


print(image_pre_median, 'Imagen antes del incendio')
print(image_post_median, 'Imagen despues del incendio')
```

Visualizacion de imágenes antes y después del incendio
```javascript
Map.addLayer(image_pre_median, 
{bands: ['B8', 'B4', 'B2'], min: 193.26485306245308, max:1486.5248147482769 , gamma: 1.2}, 'Imagen pre incendio Sentinel-2'); 
Map.addLayer(image_post_median, 
{bands: ['B8', 'B4', 'B2'], min: 193.26485306245308, max:1486.5248147482769 , gamma: 1.2}, 'Imagen post incendio Sentinel-2'); 

```



