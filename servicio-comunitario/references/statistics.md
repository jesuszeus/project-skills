# Estadísticas con Chart.js

Página de dashboards. Conserva el estilo (mismas clases, gradiente, tarjetas) y
**adapta solo las métricas** a las dimensiones reales del módulo. No fuerces
gráficos para los que no hay datos: elimina la tarjeta sobrante en vez de
inventar una métrica.

## Librería

Vía CDN (igual que los módulos existentes):

```html
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/chartjs-plugin-datalabels@2.2.0"></script>
```
(Existe `vendors/chart.js` local si se prefiere offline.)

## Datos en PHP (helpers seguros)

```php
function safeCount($pdo,$sql,$params=[]){ try{$s=$pdo->prepare($sql);$s->execute($params);return (int)$s->fetchColumn();}catch(\Exception $e){return 0;} }
function fetchAllAssoc($pdo,$sql,$params=[]){ try{$s=$pdo->prepare($sql);$s->execute($params);return $s->fetchAll(PDO::FETCH_ASSOC);}catch(\Exception $e){return [];} }

$total = safeCount($pdo, "SELECT COUNT(*) FROM <modulo>_registros");

// Conteo por catálogo (para barras/doughnut)
$data = fetchAllAssoc($pdo, "SELECT cat.nombre AS nombre, COUNT(*) AS total
                             FROM <modulo>_registros r JOIN <catalogo> cat ON cat.id=r.cat_id
                             GROUP BY cat.id, cat.nombre ORDER BY total DESC");
$labels = array_column($data,'nombre');
$counts = array_column($data,'total');

// Booleano Sí/No
$si = safeCount($pdo,"SELECT COUNT(*) FROM <modulo>_registros WHERE <bool>=1");
$no = safeCount($pdo,"SELECT COUNT(*) FROM <modulo>_registros WHERE <bool>=0");
```

Para "¿en Maracaibo?" usa el id de Maracaibo por nombre y `municipio_id = :m`
vs `total - si`.

## Estructura visual (conservar)

- 1 tarjeta con la métrica total (`.stat-number` en `#d2005a`).
- Fila de **doughnuts** (`.chart-container-pie`, min-height 300px) para
  booleanos y categorías cortas (≤ ~8).
- Barras **horizontales** (`.chart-container`, min-height 400px, `indexAxis:'y'`)
  para catálogos con muchas etiquetas (responsables, parroquias, municipios).
- Barras **verticales** para categorías pocas (niveles).

## JS común

```js
Chart.defaults.font.size = 12;
var tooltipConfig = {
  enabled:true, backgroundColor:'rgba(0,0,0,0.85)', padding:10, cornerRadius:6,
  titleFont:{size:13,weight:'bold'}, bodyFont:{size:12},
  callbacks:{ label:function(ctx){
    var v=ctx.raw||0, t=ctx.dataset.data.reduce((a,b)=>a+b,0), p=t>0?((v/t)*100).toFixed(1):0;
    return (ctx.label||ctx.dataset.label||'')+': '+v+' ('+p+'%)';
  }}
};
function renderChart(id,config){ var el=document.getElementById(id); if(el){ try{new Chart(el,config);}catch(e){console.error('Error '+id,e);} } }

renderChart('chartX', {
  type:'doughnut',
  data:{ labels:<?= json_encode($labels) ?>, datasets:[{ data:<?= json_encode($counts) ?>,
    backgroundColor:['#d2005a','#164377','#08e900','#6900a7','#fd7e14','#17a2b8','#ffc107','#20c997'] }] },
  options:{ responsive:true, maintainAspectRatio:false, plugins:{ legend:{position:'bottom'}, tooltip:tooltipConfig } }
});

renderChart('chartBarras', {
  type:'bar',
  data:{ labels:<?= json_encode($labels) ?>, datasets:[{ label:'Actividades', data:<?= json_encode($counts) ?>, backgroundColor:'#164377' }] },
  options:{ responsive:true, maintainAspectRatio:false, indexAxis:'y',
            scales:{ x:{ beginAtZero:true } }, plugins:{ legend:{display:false}, tooltip:tooltipConfig } }
});
```

Paleta institucional para series: `#d2005a`, `#164377`, `#198754`, `#fd7e14`,
`#0dcaf0`, `#6f42c1`, `#ffc107`, `#20c997`.
