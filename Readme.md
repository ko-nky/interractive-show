# Geotiff やGeoJSONをマップ上にオーバーレイして座標をインタラクティブに表示するスクリプト
以下のコードをGoogle Colabにコピー&ペーストして実行する。
```
!pip install "holoviews==1.15.1"  "rioxarray==0.9.1" "datashader==0.14.2" "geopandas" -q
import holoviews as hv
import holoviews.operation.datashader as hd
import bokeh as bk
import rioxarray as rxr
import numpy as np
import os, param, panel as pn
import geopandas as gpd
import matplotlib, matplotlib.pyplot, numpy as np

def ReadGeofile(inf):
    lx = []
    ly = []
    df = gpd.read_file(inf).to_crs(epsg=3857)
    for line in range(len(df)): 
        p = df['geometry'][line]
        coord = list(p.coords)
        lx.append(coord[0][0])
        ly.append(coord[0][1])
    df['x'] = lx
    df['y'] = ly
    return df


def WindowSizeFilter(df, win):
    size_w = str(win)
    ndf = df[df['win'] == size_w]
    ndf = ndf.reset_index(drop=True)
    return ndf, size_w

imscale = 3    #Display scale larger number indicates smaller appearance
vmin=0
vmax=20

hv.extension('bokeh', logo=False)
dataarray = rxr.open_rasterio('unite_sep.tif', mask_and_scale=True).rio.reproject('EPSG:3857')    #WGS84
dataarray.attrs['long_name'] = 'separation'
ximg = rxr.open_rasterio('cropped_s.tif').rio.reproject('EPSG:3857')    #WGS84

# convert to hv.Dataset
r_ds = hv.Dataset(ximg[0,:,:], kdims=['x','y'], vdims='Value')
g_ds = hv.Dataset(ximg[1,:,:], kdims=['x','y'], vdims='Value')
b_ds = hv.Dataset(ximg[2,:,:], kdims=['x','y'], vdims='Value')

# scale to uint8
r = np.squeeze((r_ds.data.to_array()))
g = np.squeeze((g_ds.data.to_array()))
b = np.squeeze((b_ds.data.to_array()))

rgb = hv.RGB(
    (
        ximg['x'],
        ximg['y'],
        r.data[:],
        g.data[:],
        b.data[:]
    ),
#    vdims=list('RGB')
)
cmap_terrain_top_75_percent =  [matplotlib.colors.rgb2hex(c) for c in matplotlib.pyplot.cm.jet(np.linspace(1, 0.25, 20))]
colormap_to_use = cmap_terrain_top_75_percent
image_height, image_width = int(dataarray.sizes['y'] / imscale), int(dataarray.sizes['x'] / imscale)
map_height, map_width = image_height, image_width + 200
key_dimensions = ['x', 'y']
value_dimension = dataarray.attrs["long_name"] # 'separation'

#hv.extension('bokeh', logo=False)
clipping = {'NaN': '#00000000'}
hv.opts.defaults(
  hv.opts.Image(cmap=colormap_to_use, clim=(vmin,vmax), 
                height=image_height, width=image_width, 
                colorbar=True, alpha=0.4,
                xlabel='Longitude',
                ylabel='Latitude',
                tools=['hover'], active_tools=['wheel_zoom'], 
                clipping_colors=clipping),
  hv.opts.RGB(height=image_height, width=image_width,
                active_tools=['wheel_zoom']
              ),
  hv.opts.Points(height=image_height, width=image_width,
                size=5,
                color='red',
                line_color='white',
                active_tools=['wheel_zoom']        
                 ),
  hv.opts.Labels(text_font_size='10pt',
                xoffset=3, yoffset=-3,
                 active_tools=['wheel_zoom']
                 ),
  hv.opts.Tiles(active_tools=['wheel_zoom'])
)

# This is the JavaScript code that formats our coordinates:
# You only need to change the "digits" value if you would like to display the 
# coordinates with more or fewer digits than 4.
formatter_code = """
  var digits = 4;
  var projections = Bokeh.require("core/util/projections");
  var x = special_vars.x;
  var y = special_vars.y;
  var coords = projections.wgs84_mercator.invert(x, y);
  return "" + (Math.round(coords[%d] * 10**digits) / 10**digits).toFixed(digits)+ "";
"""

# In the code above coords[%d] gives back the x coordinate if %d is 0, so at 
# first we replace that.
formatter_code_x = formatter_code % 0
# Then we replace %d to 1 to get the y value.
formatter_code_y = formatter_code % 1

# This is the standard definition of a custom Holoviews Tooltip.
# Every line will be a line in the tooltip, with the first element being the 
# label and the second element the displayed value.
custom_tooltips = [
  # We want to use @x and @y values, but send them to a custom formatter first.
  ( 'Lon',   '@x{custom}' ),
  ( 'Lat',   '@y{custom}' ),
  # This is where you should label and format your data:
  # '@image' is the data value of your GeoTIFF
  # In this example we format it as an integer and add "m" to the end of it
  # You can find more information on number formatting here:
  # https://docs.bokeh.org/en/latest/docs/user_guide/tools.html#formatting-tooltip-fields
  ('Separation', '@image{0.00 a}m')
]


custom_formatters = {
  '@x' : bk.models.CustomJSHover(code=formatter_code_x),
  '@y' : bk.models.CustomJSHover(code=formatter_code_y)
}

custom_hover = bk.models.HoverTool(tooltips=custom_tooltips, formatters=custom_formatters)


tiles = hv.element.tiles.OSM().options(height=map_height, width=map_width)
hv_dataset = hv.Dataset(dataarray[0], vdims=value_dimension, kdims=key_dimensions)
hv_image_custom_hover = hv.Image(hv_dataset).opts(tools=[custom_hover])
hv_dyn_large = rgb

pn_windowsize = pn.widgets.Select(
  options=[2, 3, 4, 5]
)
pn_opacity = pn.widgets.FloatSlider(name='Opacity', value=0.4, start=0, end=1, step=0.1) 

@pn.depends(
  pn_ws_value = pn_windowsize.param.value,
  pn_opacity_value = pn_opacity.param.value
)
def load_map(pn_ws_value, pn_opacity_value = pn_opacity.param.value):
    used_colormap = cmap_terrain_top_75_percent
    json_in = 'LM-separation.json'
    df = ReadGeofile(json_in)
    dfd, size_w = WindowSizeFilter(df, pn_ws_value * 10)
    points = hv.Points(dfd,kdims=['x', 'y'],vdims=['id','dist'])
    labels = hv.Labels(dfd,kdims=['x', 'y'],vdims='id')
    grids = hv_image_custom_hover
    grids.apply.opts(alpha=pn_opacity_value)
    image = grids * points * labels
    return image


dynmap = hv.DynamicMap(load_map)
combined = tiles * hv_dyn_large * dynmap
#combined = tiles * hv_dyn_large * dynmap

pn.Column(
  pn.WidgetBox(
    '## Separation', 
    pn.Row(
    #  pn.Row('#### map', pn_resolution), 
      pn.Row('#### window size [m]', pn_windowsize), 
      pn.Row('#### Opacity', pn_opacity)
    ),
  ), 
  combined
)

```
