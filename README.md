---
jupyter:
  kernelspec:
    display_name: Python 3 (ipykernel)
    language: python
    name: python3
  language_info:
    codemirror_mode:
      name: ipython
      version: 3
    file_extension: .py
    mimetype: text/x-python
    name: python
    nbconvert_exporter: python
    pygments_lexer: ipython3
    version: 3.12.3
  nbformat: 4
  nbformat_minor: 5
---

::: {#b7ab5a63-783b-4c2b-9a5f-f214100eb6ba .cell .markdown}
## Análise geoespacial de dados vetoriais
:::

::: {#87b24856-4522-4f18-a861-7314e49acd7f .cell .markdown}
### Introdução
:::

::: {#95acaea0-7c1c-426a-b424-82eb1f0697c7 .cell .markdown}
Os dados vetoriais permitem representar objetos geográficos de forma
precisa e estruturada. Como exemplo, utilizamos dois temas já tratados
anteriormente: o **Museu de Percurso Raphael Arcuri** e o **Centro
Histórico** da cidade de Juiz de Fora, em Minas Gerais, cada um
representado por entidades geométricas distintas -- pontos e polígonos,
respectivamente.

O [Museu de Percurso Raphael
Arcuri](https://www.instagram.com/museuraphaelarcuri?igsh=MWRjNWV1cnZnczE5aQ==),
idealizado por [Letícia
Rabelo](https://www.instagram.com/leticiarabelo.arq?igsh=dndsYTdsemM4ZWdw),
ganha ainda mais relevância quando analisado em conjunto com o Decreto
Municipal Nº 17.025/2025, que estabelece os limites oficiais do **Centro
Histórico**. Este decreto, publicado em janeiro de 2025, visa não apenas
preservar o patrimônio histórico, mas também impulsionar o comércio
local e o turismo na região.

Ao sobrepor os pontos do **Museu de Percurso** ao polígono do **Centro
Histórico** -- utilizando ferramentas de análise espacial em Python --,
observamos que obras icônicas como o *Paço Municipal* e o *Cine Theatro
Central* se encontram dentro da área delimitada, enquanto outras
igualmente significativas, como as *residências da família Arcuri* e a
*Vila Iracema,* situam-se além desses limites. Essa análise não apenas
destaca a importância do núcleo histórico central, mas também levanta
questões sobre a extensão do patrimônio arquitetônico pela cidade.
:::

::: {#1d61c1f6-0d7d-4ec2-afad-76067bab639e .cell .markdown}
**Importamos as bibliotecas**
:::

::: {#f0e227ca-5cb2-4ba9-9dbc-67cf4a56d5fc .cell .code}
``` python
import pandas as pd
import folium
from folium.plugins import Fullscreen
from pyproj import Transformer, Proj
from shapely.geometry import Polygon, Point
from geopy.distance import geodesic
```
:::

::: {#ad7ce2bf-7b0a-4a71-b03a-df13a818d4bd .cell .markdown}
**Carregamos os pontos do Museu de Percurso**
:::

::: {#2f1e6bb4-5d2d-40b9-83e7-6381c0222b3e .cell .code execution_count="3"}
``` python
df_obras = pd.DataFrame(
    [['Paço Municipal', -21.761600, -43.349999],
     ['ED. CIAMPI', -21.7608312, -43.3496202],
     ['Galeria Pio X', -21.7606305, -43.3484793],
     ['Cine Theatro Central', -21.7615775, -43.3478918],
     ['Palacete Pinho', -21.7608331, -43.3468349],
     ['Cia. Dias Cardoso', -21.7601146, -43.3448549],
     ['Hotel Príncipe', -21.7599733, -43.3440394],
     ['Associação Comercial', -21.7597099, -43.3441725],
     ['Cia. Pantaleone Arcuri', -21.762536, -43.342788],
     ['Vila Iracema', -21.763336, -43.344481],
     ['Palacete dos Fellet', -21.763280, -43.345717],
     ['Residência Raphael Arcuri', -21.763699, -43.342097],
     ['Castelinho dos Bracher', -21.763666, -43.341959],
     ['Casa D\'Itália', -21.764455, -43.348467]],
    columns=['obra', 'latitude', 'longitude']
)
```
:::

::: {#9c37294c-a991-4e21-b7a8-2eaae823c156 .cell .markdown}
**Definimos o polígono do Centro Histórico**
:::

::: {#0ff39a14-bcc2-4ced-b71c-bd684ced2b8d .cell .code execution_count="4"}
``` python
vertices_utm = [
    (671631.01298, 7592562.9258),  
    (670483.76674, 7593577.0383),  
    (670398.10899, 7593532.9851),  
    (670322.59534, 7593692.0525),  
    (670290.67635, 7593669.7837),  
    (670377.32049, 7593520.1438),  
    (670302.54290, 7593484.0129),  
    (670391.01186, 7593079.2177),  
    (670353.52763, 7593071.3745),  
    (670291.86401, 7593099.9004),  
    (670188.72948, 7593021.7110),  
    (670392.36957, 7593068.9978),  
    (670471.79272, 7592740.6347),  
    (670401.69082, 7592724.5514),  
    (670423.71811, 7592633.6294),  
    (670231.47881, 7592584.9345),  
    (670234.18470, 7592574.6061),  
    (670494.86684, 7592637.8945),  
    (670565.63917, 7592323.5994)
]
```
:::

::: {#ec2f0fca-4677-46da-9cb3-54bef6e5be44 .cell .markdown}
**Identificamos quais pontos estão dentro e fora do polígono**
:::

::: {#b099ab52-5885-4e3e-b3fe-39541bc98803 .cell .code execution_count="5"}
``` python
# Converter coordenadas UTM para WGS84 (lat/lon)
proj_utm = Proj(proj='utm', zone=23, south=True, ellps='GRS80', datum='WGS84')
transformer = Transformer.from_proj(proj_utm, 'epsg:4326')
vertices_wgs84 = [transformer.transform(easting, northing) for easting, northing in vertices_utm]

# Criar polígono Shapely 
polygon = Polygon([(lon, lat) for lat, lon in vertices_wgs84])

# Função para verificar se ponto está dentro do polígono
def is_inside(lat, lon):
    point = Point(lon, lat)  # Shapely usa (longitude, latitude)
    return polygon.contains(point)

# Aplicar a função corrigida
df_obras['dentro'] = df_obras.apply(lambda row: is_inside(row['latitude'], row['longitude']), axis=1)
```
:::

::: {#631c7d63-de95-4fa2-9bc9-166b8cfe739c .cell .markdown}
**Criamos um mapa interativo**
:::

::: {#697cf670-a77d-4f5e-9612-aaf6aee0ded0 .cell .code execution_count="6"}
``` python
# Centralizar o mapa
avg_location = df_obras[['latitude', 'longitude']].mean().tolist()
mapa = folium.Map(location=avg_location, zoom_start=14)

# Adicionar polígono do Centro Histórico (folium usa [latitude, longitude])
folium.Polygon(
    locations=[[lat, lon] for lat, lon in vertices_wgs84],
    color='blue',
    weight=2,
    fill=True,
    fill_color='blue',
    fill_opacity=0.2,
    tooltip='Centro Histórico de Juiz de Fora'
).add_to(mapa)

# Adicionar pontos do Museu de Percurso
for idx, row in df_obras.iterrows():
    color = 'green' if row['dentro'] else 'red'
    icon = folium.Icon(color=color, icon='home' if row['dentro'] else 'info-sign')
    
    folium.Marker(
        location=[row['latitude'], row['longitude']],
        popup=f"<b>{row['obra']}</b><br>Status: {'Dentro' if row['dentro'] else 'Fora'} do Centro Histórico",
        tooltip=row['obra'],
        icon=icon
    ).add_to(mapa)

# Adicionar controles de tela cheia
Fullscreen(
    position='bottomleft',
    title='Expandir',
    title_cancel='Sair',
    force_separate_button=True
).add_to(mapa)

# Adicionar legenda
legend_html = '''
     <div style="position: fixed; 
     bottom: 50px; right: 50px; width: 150px; height: 90px; 
     border:2px solid grey; z-index:9999; font-size:14px;
     background-color:white;
     ">
     &nbsp; <b>Legenda</b> <br>
     &nbsp; <i class="fa fa-home" style="color:green"></i>&nbsp; Dentro <br>
     &nbsp; <i class="fa fa-info-circle" style="color:red"></i>&nbsp; Fora <br>
     &nbsp; <i class="fa fa-map" style="color:blue"></i>&nbsp; Centro Histórico
     </div>
     '''
mapa.get_root().html.add_child(folium.Element(legend_html))

mapa
```

::: {.output .execute_result execution_count="6"}
```{=html}
<div style="width:100%;"><div style="position:relative;width:100%;height:0;padding-bottom:60%;"><span style="color:#565656">Make this Notebook Trusted to load map: File -> Trust Notebook</span><iframe srcdoc="&lt;!DOCTYPE html&gt;
&lt;html&gt;
&lt;head&gt;
    
    &lt;meta http-equiv=&quot;content-type&quot; content=&quot;text/html; charset=UTF-8&quot; /&gt;
    &lt;script src=&quot;https://cdn.jsdelivr.net/npm/leaflet@1.9.3/dist/leaflet.js&quot;&gt;&lt;/script&gt;
    &lt;script src=&quot;https://code.jquery.com/jquery-3.7.1.min.js&quot;&gt;&lt;/script&gt;
    &lt;script src=&quot;https://cdn.jsdelivr.net/npm/bootstrap@5.2.2/dist/js/bootstrap.bundle.min.js&quot;&gt;&lt;/script&gt;
    &lt;script src=&quot;https://cdnjs.cloudflare.com/ajax/libs/Leaflet.awesome-markers/2.0.2/leaflet.awesome-markers.js&quot;&gt;&lt;/script&gt;
    &lt;link rel=&quot;stylesheet&quot; href=&quot;https://cdn.jsdelivr.net/npm/leaflet@1.9.3/dist/leaflet.css&quot;/&gt;
    &lt;link rel=&quot;stylesheet&quot; href=&quot;https://cdn.jsdelivr.net/npm/bootstrap@5.2.2/dist/css/bootstrap.min.css&quot;/&gt;
    &lt;link rel=&quot;stylesheet&quot; href=&quot;https://netdna.bootstrapcdn.com/bootstrap/3.0.0/css/bootstrap-glyphicons.css&quot;/&gt;
    &lt;link rel=&quot;stylesheet&quot; href=&quot;https://cdn.jsdelivr.net/npm/@fortawesome/fontawesome-free@6.2.0/css/all.min.css&quot;/&gt;
    &lt;link rel=&quot;stylesheet&quot; href=&quot;https://cdnjs.cloudflare.com/ajax/libs/Leaflet.awesome-markers/2.0.2/leaflet.awesome-markers.css&quot;/&gt;
    &lt;link rel=&quot;stylesheet&quot; href=&quot;https://cdn.jsdelivr.net/gh/python-visualization/folium/folium/templates/leaflet.awesome.rotate.min.css&quot;/&gt;
    
            &lt;meta name=&quot;viewport&quot; content=&quot;width=device-width,
                initial-scale=1.0, maximum-scale=1.0, user-scalable=no&quot; /&gt;
            &lt;style&gt;
                #map_36c4f48ebb45eb450be95a6a0229dfea {
                    position: relative;
                    width: 100.0%;
                    height: 100.0%;
                    left: 0.0%;
                    top: 0.0%;
                }
                .leaflet-container { font-size: 1rem; }
            &lt;/style&gt;

            &lt;style&gt;html, body {
                width: 100%;
                height: 100%;
                margin: 0;
                padding: 0;
            }
            &lt;/style&gt;

            &lt;style&gt;#map {
                position:absolute;
                top:0;
                bottom:0;
                right:0;
                left:0;
                }
            &lt;/style&gt;

            &lt;script&gt;
                L_NO_TOUCH = false;
                L_DISABLE_3D = false;
            &lt;/script&gt;

        
    &lt;script src=&quot;https://cdn.jsdelivr.net/npm/leaflet.fullscreen@3.0.0/Control.FullScreen.min.js&quot;&gt;&lt;/script&gt;
    &lt;link rel=&quot;stylesheet&quot; href=&quot;https://cdn.jsdelivr.net/npm/leaflet.fullscreen@3.0.0/Control.FullScreen.css&quot;/&gt;
&lt;/head&gt;
&lt;body&gt;
    
    
     &lt;div style=&quot;position: fixed; 
     bottom: 50px; right: 50px; width: 150px; height: 90px; 
     border:2px solid grey; z-index:9999; font-size:14px;
     background-color:white;
     &quot;&gt;
     &amp;nbsp; &lt;b&gt;Legenda&lt;/b&gt; &lt;br&gt;
     &amp;nbsp; &lt;i class=&quot;fa fa-home&quot; style=&quot;color:green&quot;&gt;&lt;/i&gt;&amp;nbsp; Dentro &lt;br&gt;
     &amp;nbsp; &lt;i class=&quot;fa fa-info-circle&quot; style=&quot;color:red&quot;&gt;&lt;/i&gt;&amp;nbsp; Fora &lt;br&gt;
     &amp;nbsp; &lt;i class=&quot;fa fa-map&quot; style=&quot;color:blue&quot;&gt;&lt;/i&gt;&amp;nbsp; Centro Histórico
     &lt;/div&gt;
     
    
            &lt;div class=&quot;folium-map&quot; id=&quot;map_36c4f48ebb45eb450be95a6a0229dfea&quot; &gt;&lt;/div&gt;
        
&lt;/body&gt;
&lt;script&gt;
    
    
            var map_36c4f48ebb45eb450be95a6a0229dfea = L.map(
                &quot;map_36c4f48ebb45eb450be95a6a0229dfea&quot;,
                {
                    center: [-21.761874435714287, -43.34581435714286],
                    crs: L.CRS.EPSG3857,
                    ...{
  &quot;zoom&quot;: 14,
  &quot;zoomControl&quot;: true,
  &quot;preferCanvas&quot;: false,
}

                }
            );

            

        
    
            var tile_layer_09d6338cf14e54079d88163dcae9f61a = L.tileLayer(
                &quot;https://tile.openstreetmap.org/{z}/{x}/{y}.png&quot;,
                {
  &quot;minZoom&quot;: 0,
  &quot;maxZoom&quot;: 19,
  &quot;maxNativeZoom&quot;: 19,
  &quot;noWrap&quot;: false,
  &quot;attribution&quot;: &quot;\u0026copy; \u003ca href=\&quot;https://www.openstreetmap.org/copyright\&quot;\u003eOpenStreetMap\u003c/a\u003e contributors&quot;,
  &quot;subdomains&quot;: &quot;abc&quot;,
  &quot;detectRetina&quot;: false,
  &quot;tms&quot;: false,
  &quot;opacity&quot;: 1,
}

            );
        
    
            tile_layer_09d6338cf14e54079d88163dcae9f61a.addTo(map_36c4f48ebb45eb450be95a6a0229dfea);
        
    
            var polygon_7bd3792e2e853841b610de998933850b = L.polygon(
                [[-21.762281043001835, -43.34016652775217], [-21.753233459476547, -43.351362572075494], [-21.753639557293926, -43.352186110828214], [-21.75221027520308, -43.35293251378737], [-21.752414458261512, -43.353238788656014], [-21.753757529707904, -43.35238575673133], [-21.754091028762748, -43.35310493520417], [-21.75773825072305, -43.352207947099004], [-21.757812692768333, -43.35256952148563], [-21.75756100908489, -43.353168600600966], [-21.758277067755575, -43.354157612890276], [-21.75783041668798, -43.35219376769888], [-21.760788237237197, -43.35139206518308], [-21.760940240650374, -43.35206813839776], [-21.761759241939266, -43.35184580552671], [-21.762217519769386, -43.35369933505048], [-21.76231053597393, -43.35367211067517], [-21.76171386749516, -43.351158388857264], [-21.764545464282058, -43.350441727160124]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;blue&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: true, &quot;fillColor&quot;: &quot;blue&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 1.0, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 2}
            ).addTo(map_36c4f48ebb45eb450be95a6a0229dfea);
        
    
            polygon_7bd3792e2e853841b610de998933850b.bindTooltip(
                `&lt;div&gt;
                     Centro Histórico de Juiz de Fora
                 &lt;/div&gt;`,
                {
  &quot;sticky&quot;: true,
}
            );
        
    
            var marker_b55dc7b3bb44adc23bff70e37bea12be = L.marker(
                [-21.7616, -43.349999],
                {
}
            ).addTo(map_36c4f48ebb45eb450be95a6a0229dfea);
        
    
            var icon_4a089a913f886db23936050bb5feb366 = L.AwesomeMarkers.icon(
                {
  &quot;markerColor&quot;: &quot;green&quot;,
  &quot;iconColor&quot;: &quot;white&quot;,
  &quot;icon&quot;: &quot;home&quot;,
  &quot;prefix&quot;: &quot;glyphicon&quot;,
  &quot;extraClasses&quot;: &quot;fa-rotate-0&quot;,
}
            );
        
    
        var popup_1601ff99e9f98f604156c72ac326df3f = L.popup({
  &quot;maxWidth&quot;: &quot;100%&quot;,
});

        
            
                var html_deb002bfa8e32593bcdd98bc12d9235f = $(`&lt;div id=&quot;html_deb002bfa8e32593bcdd98bc12d9235f&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;&gt;&lt;b&gt;Paço Municipal&lt;/b&gt;&lt;br&gt;Status: Dentro do Centro Histórico&lt;/div&gt;`)[0];
                popup_1601ff99e9f98f604156c72ac326df3f.setContent(html_deb002bfa8e32593bcdd98bc12d9235f);
            
        

        marker_b55dc7b3bb44adc23bff70e37bea12be.bindPopup(popup_1601ff99e9f98f604156c72ac326df3f)
        ;

        
    
    
            marker_b55dc7b3bb44adc23bff70e37bea12be.bindTooltip(
                `&lt;div&gt;
                     Paço Municipal
                 &lt;/div&gt;`,
                {
  &quot;sticky&quot;: true,
}
            );
        
    
                marker_b55dc7b3bb44adc23bff70e37bea12be.setIcon(icon_4a089a913f886db23936050bb5feb366);
            
    
            var marker_3ac236bdba434c11f4330eea1681f04a = L.marker(
                [-21.7608312, -43.3496202],
                {
}
            ).addTo(map_36c4f48ebb45eb450be95a6a0229dfea);
        
    
            var icon_d1ccc9d4d94e8f56a35264cbfedcc42a = L.AwesomeMarkers.icon(
                {
  &quot;markerColor&quot;: &quot;green&quot;,
  &quot;iconColor&quot;: &quot;white&quot;,
  &quot;icon&quot;: &quot;home&quot;,
  &quot;prefix&quot;: &quot;glyphicon&quot;,
  &quot;extraClasses&quot;: &quot;fa-rotate-0&quot;,
}
            );
        
    
        var popup_b170d33ef287de9c6656ce45babb0566 = L.popup({
  &quot;maxWidth&quot;: &quot;100%&quot;,
});

        
            
                var html_e43bb6673038d443ce122124c0756ed3 = $(`&lt;div id=&quot;html_e43bb6673038d443ce122124c0756ed3&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;&gt;&lt;b&gt;ED. CIAMPI&lt;/b&gt;&lt;br&gt;Status: Dentro do Centro Histórico&lt;/div&gt;`)[0];
                popup_b170d33ef287de9c6656ce45babb0566.setContent(html_e43bb6673038d443ce122124c0756ed3);
            
        

        marker_3ac236bdba434c11f4330eea1681f04a.bindPopup(popup_b170d33ef287de9c6656ce45babb0566)
        ;

        
    
    
            marker_3ac236bdba434c11f4330eea1681f04a.bindTooltip(
                `&lt;div&gt;
                     ED. CIAMPI
                 &lt;/div&gt;`,
                {
  &quot;sticky&quot;: true,
}
            );
        
    
                marker_3ac236bdba434c11f4330eea1681f04a.setIcon(icon_d1ccc9d4d94e8f56a35264cbfedcc42a);
            
    
            var marker_287a4497a5e6dbeb725041bca904465b = L.marker(
                [-21.7606305, -43.3484793],
                {
}
            ).addTo(map_36c4f48ebb45eb450be95a6a0229dfea);
        
    
            var icon_e2ce41521003e8a892ab2091e26f501c = L.AwesomeMarkers.icon(
                {
  &quot;markerColor&quot;: &quot;green&quot;,
  &quot;iconColor&quot;: &quot;white&quot;,
  &quot;icon&quot;: &quot;home&quot;,
  &quot;prefix&quot;: &quot;glyphicon&quot;,
  &quot;extraClasses&quot;: &quot;fa-rotate-0&quot;,
}
            );
        
    
        var popup_16a2afd75a492b3de0b172eb8c979367 = L.popup({
  &quot;maxWidth&quot;: &quot;100%&quot;,
});

        
            
                var html_82df6522f2df2c019aaa9e46c30545ab = $(`&lt;div id=&quot;html_82df6522f2df2c019aaa9e46c30545ab&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;&gt;&lt;b&gt;Galeria Pio X&lt;/b&gt;&lt;br&gt;Status: Dentro do Centro Histórico&lt;/div&gt;`)[0];
                popup_16a2afd75a492b3de0b172eb8c979367.setContent(html_82df6522f2df2c019aaa9e46c30545ab);
            
        

        marker_287a4497a5e6dbeb725041bca904465b.bindPopup(popup_16a2afd75a492b3de0b172eb8c979367)
        ;

        
    
    
            marker_287a4497a5e6dbeb725041bca904465b.bindTooltip(
                `&lt;div&gt;
                     Galeria Pio X
                 &lt;/div&gt;`,
                {
  &quot;sticky&quot;: true,
}
            );
        
    
                marker_287a4497a5e6dbeb725041bca904465b.setIcon(icon_e2ce41521003e8a892ab2091e26f501c);
            
    
            var marker_09198f0b339660c82e26cb67e9466026 = L.marker(
                [-21.7615775, -43.3478918],
                {
}
            ).addTo(map_36c4f48ebb45eb450be95a6a0229dfea);
        
    
            var icon_377626e547c857284af97d9bc1e216da = L.AwesomeMarkers.icon(
                {
  &quot;markerColor&quot;: &quot;green&quot;,
  &quot;iconColor&quot;: &quot;white&quot;,
  &quot;icon&quot;: &quot;home&quot;,
  &quot;prefix&quot;: &quot;glyphicon&quot;,
  &quot;extraClasses&quot;: &quot;fa-rotate-0&quot;,
}
            );
        
    
        var popup_2a6ac6917780f801a26610fb6bfe17a0 = L.popup({
  &quot;maxWidth&quot;: &quot;100%&quot;,
});

        
            
                var html_a32cda4e0f1f8aea1796a35b1df20224 = $(`&lt;div id=&quot;html_a32cda4e0f1f8aea1796a35b1df20224&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;&gt;&lt;b&gt;Cine Theatro Central&lt;/b&gt;&lt;br&gt;Status: Dentro do Centro Histórico&lt;/div&gt;`)[0];
                popup_2a6ac6917780f801a26610fb6bfe17a0.setContent(html_a32cda4e0f1f8aea1796a35b1df20224);
            
        

        marker_09198f0b339660c82e26cb67e9466026.bindPopup(popup_2a6ac6917780f801a26610fb6bfe17a0)
        ;

        
    
    
            marker_09198f0b339660c82e26cb67e9466026.bindTooltip(
                `&lt;div&gt;
                     Cine Theatro Central
                 &lt;/div&gt;`,
                {
  &quot;sticky&quot;: true,
}
            );
        
    
                marker_09198f0b339660c82e26cb67e9466026.setIcon(icon_377626e547c857284af97d9bc1e216da);
            
    
            var marker_f2e8ab267d1de28e17e961d1ef042beb = L.marker(
                [-21.7608331, -43.3468349],
                {
}
            ).addTo(map_36c4f48ebb45eb450be95a6a0229dfea);
        
    
            var icon_cfe83153e9a1cfebe41b7f54f4bed7e3 = L.AwesomeMarkers.icon(
                {
  &quot;markerColor&quot;: &quot;green&quot;,
  &quot;iconColor&quot;: &quot;white&quot;,
  &quot;icon&quot;: &quot;home&quot;,
  &quot;prefix&quot;: &quot;glyphicon&quot;,
  &quot;extraClasses&quot;: &quot;fa-rotate-0&quot;,
}
            );
        
    
        var popup_e1bb21ed6e283897c45b48a346d7873c = L.popup({
  &quot;maxWidth&quot;: &quot;100%&quot;,
});

        
            
                var html_1ef4efd3ad12cce742a8f044530cd8cc = $(`&lt;div id=&quot;html_1ef4efd3ad12cce742a8f044530cd8cc&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;&gt;&lt;b&gt;Palacete Pinho&lt;/b&gt;&lt;br&gt;Status: Dentro do Centro Histórico&lt;/div&gt;`)[0];
                popup_e1bb21ed6e283897c45b48a346d7873c.setContent(html_1ef4efd3ad12cce742a8f044530cd8cc);
            
        

        marker_f2e8ab267d1de28e17e961d1ef042beb.bindPopup(popup_e1bb21ed6e283897c45b48a346d7873c)
        ;

        
    
    
            marker_f2e8ab267d1de28e17e961d1ef042beb.bindTooltip(
                `&lt;div&gt;
                     Palacete Pinho
                 &lt;/div&gt;`,
                {
  &quot;sticky&quot;: true,
}
            );
        
    
                marker_f2e8ab267d1de28e17e961d1ef042beb.setIcon(icon_cfe83153e9a1cfebe41b7f54f4bed7e3);
            
    
            var marker_6f66e9555eac0ca352b23db2e1778d8f = L.marker(
                [-21.7601146, -43.3448549],
                {
}
            ).addTo(map_36c4f48ebb45eb450be95a6a0229dfea);
        
    
            var icon_47c44b9a4da20866ce018706eb7ad94a = L.AwesomeMarkers.icon(
                {
  &quot;markerColor&quot;: &quot;green&quot;,
  &quot;iconColor&quot;: &quot;white&quot;,
  &quot;icon&quot;: &quot;home&quot;,
  &quot;prefix&quot;: &quot;glyphicon&quot;,
  &quot;extraClasses&quot;: &quot;fa-rotate-0&quot;,
}
            );
        
    
        var popup_8b1d348cb3d852f607719696370835e8 = L.popup({
  &quot;maxWidth&quot;: &quot;100%&quot;,
});

        
            
                var html_fef5b0416b1289b5ba0c8fed6f0f07c9 = $(`&lt;div id=&quot;html_fef5b0416b1289b5ba0c8fed6f0f07c9&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;&gt;&lt;b&gt;Cia. Dias Cardoso&lt;/b&gt;&lt;br&gt;Status: Dentro do Centro Histórico&lt;/div&gt;`)[0];
                popup_8b1d348cb3d852f607719696370835e8.setContent(html_fef5b0416b1289b5ba0c8fed6f0f07c9);
            
        

        marker_6f66e9555eac0ca352b23db2e1778d8f.bindPopup(popup_8b1d348cb3d852f607719696370835e8)
        ;

        
    
    
            marker_6f66e9555eac0ca352b23db2e1778d8f.bindTooltip(
                `&lt;div&gt;
                     Cia. Dias Cardoso
                 &lt;/div&gt;`,
                {
  &quot;sticky&quot;: true,
}
            );
        
    
                marker_6f66e9555eac0ca352b23db2e1778d8f.setIcon(icon_47c44b9a4da20866ce018706eb7ad94a);
            
    
            var marker_f8f44c43f67b5b0dce0da1af86f9f2e5 = L.marker(
                [-21.7599733, -43.3440394],
                {
}
            ).addTo(map_36c4f48ebb45eb450be95a6a0229dfea);
        
    
            var icon_fe794ba67ed9e83c0f39f548680950dc = L.AwesomeMarkers.icon(
                {
  &quot;markerColor&quot;: &quot;green&quot;,
  &quot;iconColor&quot;: &quot;white&quot;,
  &quot;icon&quot;: &quot;home&quot;,
  &quot;prefix&quot;: &quot;glyphicon&quot;,
  &quot;extraClasses&quot;: &quot;fa-rotate-0&quot;,
}
            );
        
    
        var popup_84323a5551f228e98c86ff4df2ee1fa3 = L.popup({
  &quot;maxWidth&quot;: &quot;100%&quot;,
});

        
            
                var html_a1eabe512200442c8803f827815a54f3 = $(`&lt;div id=&quot;html_a1eabe512200442c8803f827815a54f3&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;&gt;&lt;b&gt;Hotel Príncipe&lt;/b&gt;&lt;br&gt;Status: Dentro do Centro Histórico&lt;/div&gt;`)[0];
                popup_84323a5551f228e98c86ff4df2ee1fa3.setContent(html_a1eabe512200442c8803f827815a54f3);
            
        

        marker_f8f44c43f67b5b0dce0da1af86f9f2e5.bindPopup(popup_84323a5551f228e98c86ff4df2ee1fa3)
        ;

        
    
    
            marker_f8f44c43f67b5b0dce0da1af86f9f2e5.bindTooltip(
                `&lt;div&gt;
                     Hotel Príncipe
                 &lt;/div&gt;`,
                {
  &quot;sticky&quot;: true,
}
            );
        
    
                marker_f8f44c43f67b5b0dce0da1af86f9f2e5.setIcon(icon_fe794ba67ed9e83c0f39f548680950dc);
            
    
            var marker_fe2a5acc27eb07715065fcaae96057b5 = L.marker(
                [-21.7597099, -43.3441725],
                {
}
            ).addTo(map_36c4f48ebb45eb450be95a6a0229dfea);
        
    
            var icon_7356d2f663a6213510b2332c0912ef5c = L.AwesomeMarkers.icon(
                {
  &quot;markerColor&quot;: &quot;green&quot;,
  &quot;iconColor&quot;: &quot;white&quot;,
  &quot;icon&quot;: &quot;home&quot;,
  &quot;prefix&quot;: &quot;glyphicon&quot;,
  &quot;extraClasses&quot;: &quot;fa-rotate-0&quot;,
}
            );
        
    
        var popup_762e5a49b108ba7e4d9fa8cdcc0659ee = L.popup({
  &quot;maxWidth&quot;: &quot;100%&quot;,
});

        
            
                var html_983a2dc7e37e8bd6a3bbb44f7ac02c37 = $(`&lt;div id=&quot;html_983a2dc7e37e8bd6a3bbb44f7ac02c37&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;&gt;&lt;b&gt;Associação Comercial&lt;/b&gt;&lt;br&gt;Status: Dentro do Centro Histórico&lt;/div&gt;`)[0];
                popup_762e5a49b108ba7e4d9fa8cdcc0659ee.setContent(html_983a2dc7e37e8bd6a3bbb44f7ac02c37);
            
        

        marker_fe2a5acc27eb07715065fcaae96057b5.bindPopup(popup_762e5a49b108ba7e4d9fa8cdcc0659ee)
        ;

        
    
    
            marker_fe2a5acc27eb07715065fcaae96057b5.bindTooltip(
                `&lt;div&gt;
                     Associação Comercial
                 &lt;/div&gt;`,
                {
  &quot;sticky&quot;: true,
}
            );
        
    
                marker_fe2a5acc27eb07715065fcaae96057b5.setIcon(icon_7356d2f663a6213510b2332c0912ef5c);
            
    
            var marker_fb8551ece4936457a8bf95f75a65b2e2 = L.marker(
                [-21.762536, -43.342788],
                {
}
            ).addTo(map_36c4f48ebb45eb450be95a6a0229dfea);
        
    
            var icon_287010b472cae91172cf602ca37f1279 = L.AwesomeMarkers.icon(
                {
  &quot;markerColor&quot;: &quot;green&quot;,
  &quot;iconColor&quot;: &quot;white&quot;,
  &quot;icon&quot;: &quot;home&quot;,
  &quot;prefix&quot;: &quot;glyphicon&quot;,
  &quot;extraClasses&quot;: &quot;fa-rotate-0&quot;,
}
            );
        
    
        var popup_e6fa84c87e0c5ecbc144a32245bede8b = L.popup({
  &quot;maxWidth&quot;: &quot;100%&quot;,
});

        
            
                var html_c46d74c820d64430d40a912642b5b767 = $(`&lt;div id=&quot;html_c46d74c820d64430d40a912642b5b767&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;&gt;&lt;b&gt;Cia. Pantaleone Arcuri&lt;/b&gt;&lt;br&gt;Status: Dentro do Centro Histórico&lt;/div&gt;`)[0];
                popup_e6fa84c87e0c5ecbc144a32245bede8b.setContent(html_c46d74c820d64430d40a912642b5b767);
            
        

        marker_fb8551ece4936457a8bf95f75a65b2e2.bindPopup(popup_e6fa84c87e0c5ecbc144a32245bede8b)
        ;

        
    
    
            marker_fb8551ece4936457a8bf95f75a65b2e2.bindTooltip(
                `&lt;div&gt;
                     Cia. Pantaleone Arcuri
                 &lt;/div&gt;`,
                {
  &quot;sticky&quot;: true,
}
            );
        
    
                marker_fb8551ece4936457a8bf95f75a65b2e2.setIcon(icon_287010b472cae91172cf602ca37f1279);
            
    
            var marker_a710539b495eab72a71629096654ce0f = L.marker(
                [-21.763336, -43.344481],
                {
}
            ).addTo(map_36c4f48ebb45eb450be95a6a0229dfea);
        
    
            var icon_635801cb2841771a440e318a33db5293 = L.AwesomeMarkers.icon(
                {
  &quot;markerColor&quot;: &quot;red&quot;,
  &quot;iconColor&quot;: &quot;white&quot;,
  &quot;icon&quot;: &quot;info-sign&quot;,
  &quot;prefix&quot;: &quot;glyphicon&quot;,
  &quot;extraClasses&quot;: &quot;fa-rotate-0&quot;,
}
            );
        
    
        var popup_9fd3876d60711532a0fba53c50570f4b = L.popup({
  &quot;maxWidth&quot;: &quot;100%&quot;,
});

        
            
                var html_19a95875405153520648b5e71ab46454 = $(`&lt;div id=&quot;html_19a95875405153520648b5e71ab46454&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;&gt;&lt;b&gt;Vila Iracema&lt;/b&gt;&lt;br&gt;Status: Fora do Centro Histórico&lt;/div&gt;`)[0];
                popup_9fd3876d60711532a0fba53c50570f4b.setContent(html_19a95875405153520648b5e71ab46454);
            
        

        marker_a710539b495eab72a71629096654ce0f.bindPopup(popup_9fd3876d60711532a0fba53c50570f4b)
        ;

        
    
    
            marker_a710539b495eab72a71629096654ce0f.bindTooltip(
                `&lt;div&gt;
                     Vila Iracema
                 &lt;/div&gt;`,
                {
  &quot;sticky&quot;: true,
}
            );
        
    
                marker_a710539b495eab72a71629096654ce0f.setIcon(icon_635801cb2841771a440e318a33db5293);
            
    
            var marker_4eda47fcfdfb13bfbff54e5b46f950ad = L.marker(
                [-21.76328, -43.345717],
                {
}
            ).addTo(map_36c4f48ebb45eb450be95a6a0229dfea);
        
    
            var icon_00db8211e3807ccfa84b1a21dcb1d226 = L.AwesomeMarkers.icon(
                {
  &quot;markerColor&quot;: &quot;green&quot;,
  &quot;iconColor&quot;: &quot;white&quot;,
  &quot;icon&quot;: &quot;home&quot;,
  &quot;prefix&quot;: &quot;glyphicon&quot;,
  &quot;extraClasses&quot;: &quot;fa-rotate-0&quot;,
}
            );
        
    
        var popup_690f3762030d5b044a3b1e52d2c82d94 = L.popup({
  &quot;maxWidth&quot;: &quot;100%&quot;,
});

        
            
                var html_6ffcce1d11c682780c0fe6e7115f56b1 = $(`&lt;div id=&quot;html_6ffcce1d11c682780c0fe6e7115f56b1&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;&gt;&lt;b&gt;Palacete dos Fellet&lt;/b&gt;&lt;br&gt;Status: Dentro do Centro Histórico&lt;/div&gt;`)[0];
                popup_690f3762030d5b044a3b1e52d2c82d94.setContent(html_6ffcce1d11c682780c0fe6e7115f56b1);
            
        

        marker_4eda47fcfdfb13bfbff54e5b46f950ad.bindPopup(popup_690f3762030d5b044a3b1e52d2c82d94)
        ;

        
    
    
            marker_4eda47fcfdfb13bfbff54e5b46f950ad.bindTooltip(
                `&lt;div&gt;
                     Palacete dos Fellet
                 &lt;/div&gt;`,
                {
  &quot;sticky&quot;: true,
}
            );
        
    
                marker_4eda47fcfdfb13bfbff54e5b46f950ad.setIcon(icon_00db8211e3807ccfa84b1a21dcb1d226);
            
    
            var marker_8421381241bd15999a7e8abf84f5351e = L.marker(
                [-21.763699, -43.342097],
                {
}
            ).addTo(map_36c4f48ebb45eb450be95a6a0229dfea);
        
    
            var icon_f1404149f7a6f0a57e431e8ea9303fc5 = L.AwesomeMarkers.icon(
                {
  &quot;markerColor&quot;: &quot;red&quot;,
  &quot;iconColor&quot;: &quot;white&quot;,
  &quot;icon&quot;: &quot;info-sign&quot;,
  &quot;prefix&quot;: &quot;glyphicon&quot;,
  &quot;extraClasses&quot;: &quot;fa-rotate-0&quot;,
}
            );
        
    
        var popup_f099f96bc52d1493e577559213828d78 = L.popup({
  &quot;maxWidth&quot;: &quot;100%&quot;,
});

        
            
                var html_72a8d9ed4bb323402561429fce1cda1e = $(`&lt;div id=&quot;html_72a8d9ed4bb323402561429fce1cda1e&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;&gt;&lt;b&gt;Residência Raphael Arcuri&lt;/b&gt;&lt;br&gt;Status: Fora do Centro Histórico&lt;/div&gt;`)[0];
                popup_f099f96bc52d1493e577559213828d78.setContent(html_72a8d9ed4bb323402561429fce1cda1e);
            
        

        marker_8421381241bd15999a7e8abf84f5351e.bindPopup(popup_f099f96bc52d1493e577559213828d78)
        ;

        
    
    
            marker_8421381241bd15999a7e8abf84f5351e.bindTooltip(
                `&lt;div&gt;
                     Residência Raphael Arcuri
                 &lt;/div&gt;`,
                {
  &quot;sticky&quot;: true,
}
            );
        
    
                marker_8421381241bd15999a7e8abf84f5351e.setIcon(icon_f1404149f7a6f0a57e431e8ea9303fc5);
            
    
            var marker_5291f57d652b939f2e38f46e3605f22e = L.marker(
                [-21.763666, -43.341959],
                {
}
            ).addTo(map_36c4f48ebb45eb450be95a6a0229dfea);
        
    
            var icon_0880df88e6d50c6fd27aacc1088d2dbb = L.AwesomeMarkers.icon(
                {
  &quot;markerColor&quot;: &quot;red&quot;,
  &quot;iconColor&quot;: &quot;white&quot;,
  &quot;icon&quot;: &quot;info-sign&quot;,
  &quot;prefix&quot;: &quot;glyphicon&quot;,
  &quot;extraClasses&quot;: &quot;fa-rotate-0&quot;,
}
            );
        
    
        var popup_90435f2d2dfe97ef944488f3a5c83656 = L.popup({
  &quot;maxWidth&quot;: &quot;100%&quot;,
});

        
            
                var html_fcf292a0996ee1af1b04c490154194db = $(`&lt;div id=&quot;html_fcf292a0996ee1af1b04c490154194db&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;&gt;&lt;b&gt;Castelinho dos Bracher&lt;/b&gt;&lt;br&gt;Status: Fora do Centro Histórico&lt;/div&gt;`)[0];
                popup_90435f2d2dfe97ef944488f3a5c83656.setContent(html_fcf292a0996ee1af1b04c490154194db);
            
        

        marker_5291f57d652b939f2e38f46e3605f22e.bindPopup(popup_90435f2d2dfe97ef944488f3a5c83656)
        ;

        
    
    
            marker_5291f57d652b939f2e38f46e3605f22e.bindTooltip(
                `&lt;div&gt;
                     Castelinho dos Bracher
                 &lt;/div&gt;`,
                {
  &quot;sticky&quot;: true,
}
            );
        
    
                marker_5291f57d652b939f2e38f46e3605f22e.setIcon(icon_0880df88e6d50c6fd27aacc1088d2dbb);
            
    
            var marker_2a1808e76379b13e2d3db658e20d6e74 = L.marker(
                [-21.764455, -43.348467],
                {
}
            ).addTo(map_36c4f48ebb45eb450be95a6a0229dfea);
        
    
            var icon_06279769845ce8ca3dbedf834d745edf = L.AwesomeMarkers.icon(
                {
  &quot;markerColor&quot;: &quot;red&quot;,
  &quot;iconColor&quot;: &quot;white&quot;,
  &quot;icon&quot;: &quot;info-sign&quot;,
  &quot;prefix&quot;: &quot;glyphicon&quot;,
  &quot;extraClasses&quot;: &quot;fa-rotate-0&quot;,
}
            );
        
    
        var popup_62520e03d3f71bdf62393c208e3ef2a9 = L.popup({
  &quot;maxWidth&quot;: &quot;100%&quot;,
});

        
            
                var html_bf9793402e18d44dcd2875e6f3aaec6e = $(`&lt;div id=&quot;html_bf9793402e18d44dcd2875e6f3aaec6e&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;&gt;&lt;b&gt;Casa D&#x27;Itália&lt;/b&gt;&lt;br&gt;Status: Fora do Centro Histórico&lt;/div&gt;`)[0];
                popup_62520e03d3f71bdf62393c208e3ef2a9.setContent(html_bf9793402e18d44dcd2875e6f3aaec6e);
            
        

        marker_2a1808e76379b13e2d3db658e20d6e74.bindPopup(popup_62520e03d3f71bdf62393c208e3ef2a9)
        ;

        
    
    
            marker_2a1808e76379b13e2d3db658e20d6e74.bindTooltip(
                `&lt;div&gt;
                     Casa D&#x27;Itália
                 &lt;/div&gt;`,
                {
  &quot;sticky&quot;: true,
}
            );
        
    
                marker_2a1808e76379b13e2d3db658e20d6e74.setIcon(icon_06279769845ce8ca3dbedf834d745edf);
            
    
            L.control.fullscreen(
                {
  &quot;position&quot;: &quot;bottomleft&quot;,
  &quot;title&quot;: &quot;Expandir&quot;,
  &quot;titleCancel&quot;: &quot;Sair&quot;,
  &quot;forceSeparateButton&quot;: true,
}
            ).addTo(map_36c4f48ebb45eb450be95a6a0229dfea);
        
&lt;/script&gt;
&lt;/html&gt;" style="position:absolute;width:100%;height:100%;left:0;top:0;border:none !important;" allowfullscreen webkitallowfullscreen mozallowfullscreen></iframe></div></div>
```
:::
:::

::: {#b1e9557e-689f-4752-9370-a63f526278b6 .cell .markdown}
:::

::: {#180dbc06-dd05-4a54-8242-26a20f078c06 .cell .markdown}
**Considerações finais**

Ferramentas de georreferenciamento e análise espacial permitem
visualizar a relação entre o **Museu de Percurso Raphael Arcuri** e o
**Centro Histórico** de forma clara. A interseção entre dados históricos
e geográficos não só enriquece nossa compreensão do patrimônio cultural,
mas também apoia tomadas de decisão mais embasadas.
:::

::: {#cb735345-4ad2-41a4-9749-bccac67faf56 .cell .markdown}
:::

::: {#b290af9d-8bfd-417a-b45b-2b6bf0bbbd40 .cell .markdown}
**Referências**

Centro Histórico de Juiz de Fora é delimitado. **Tribuna de Minas**,
Juiz de Fora, publicado em 24/01/2025 às 13h30. Caderno Cidade.
Disponível em:
<https://tribunademinas.com.br/noticias/cidade/24-01-2025/centro-historico-jf.html>.

DECRETO Nº 17.025, de 23 de janeiro de 2025. Prefeitura Municipal de
Juiz de Fora. Publicado no Diário Oficial Eletrônico do Município de
Juiz de Fora - Atos do Governo do do Poder Executivo, em 24/01/2025.
Disponível em:
<https://www.pjf.mg.gov.br/e_atos/e_atos_vis.php?id=126353>.

Ferreira, G. (2025). Centro Histórico de Juiz de Fora. Publicado em
26/01/2025. LinkedIn. Disponível em:
<https://www.linkedin.com/pulse/centro-hist%C3%B3rico-de-juiz-fora-guilherme-ferreira-5wshf/?trackingId=mNyOlJLqld7YcMdTpzpi6w%3D%3D>.

Ferreira, G. (2025). Museu de Percurso Raphael Arcuri. Publicado em
08/06/2025. LinkedIn. Disponível em:
<https://www.linkedin.com/pulse/museu-de-percurso-raphael-arcuri-guilherme-ferreira-v8i2f/?trackingId=1jNy2WyuQiqb8VEqlsBznQ%3D%3D>.

Uribe, C. J. (2024). *An intelligent decision support system for tourism
in Python*. Publicado em Jan 16, 2024. Disponível em:
<https://medium.com/@carlosjuribe/list/an-intelligent-decision-support-system-for-tourism-in-python-b6ba165b4236>.
:::

::: {#ea70b146-532e-4ac1-bf85-7932aa973ae2 .cell .code}
``` python
```
:::
