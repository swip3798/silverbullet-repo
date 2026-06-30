---
name: "Library/swip3798/EmbedOsm"
tags: meta/library
pageDecoration.prefix: "🗺️"
---
# # Embed OpenStreetMap in your pages

## The widget accepts the following arguments:

### embed_latlng
*   `qry` - required - Coordinates in `"lat, lon"` format (e.g., `"48.8566, 2.3522"`)
*   `height` - optional - in pixels, default is `480`
*   `zoom` - optional
    *   min: `1` (zoomed out, whole world)
    *   default: `14`
    *   max: `19` (OSM's maximum for embed)

### embed_location
*   `qry` - required - A place name, address, or search query (e.g., `"London"`, `"Statue of Liberty"`, `"Eiffel Tower, Paris"`)
*   `height` - optional - in pixels, default is `480`
*   `zoom` - optional, default is `14`

⚠️ Undeclared/Empty arguments must be skipped with double-quotes `""``

## Example:
${embed_latlng("48.8566, 2.3522","400","")}

${embed_latlng("40.6892, -74.0445","480","15")}

${embed_location("Karlsruhe","","14")}

## Implementation
```space-lua
function embed_latlng(qry, height, zoom)
  -- Set default values
  local height = height or "480"
  local zoom = zoom or "14"

  -- Parse coordinates from query (format: "lat, lon")
  local lat_str, lon_str = qry:match("^(%-?%d+%.?%d*),%s*(%-?%d+%.?%d*)$")
  if not lat_str then
    return 'Provide coordinates as "lat, lon" (e.g., "48.8566, 2.3522")'
  end

  local lat = tonumber(lat_str)
  local lon = tonumber(lon_str)
  local z = tonumber(zoom)
  local w = 640
  local h = tonumber(height) or 480

  -- Calculate bounding box from center point + zoom (Web Mercator)
  local world_size = 256 * (2 ^ z)

  -- Center → pixel coordinates
  local center_x = (lon + 180) / 360 * world_size
  local lat_rad = math.rad(lat)
  local center_y = (1 - math.log(math.tan(lat_rad) + 1 / math.cos(lat_rad)) / math.pi) / 2 * world_size

  -- Pixel extents based on iframe dimensions
  local left = center_x - w / 2
  local right = center_x + w / 2
  local top = center_y - h / 2
  local bottom = center_y + h / 2

  -- Pixel coordinates back to geographic
  local west = left / world_size * 360 - 180
  local east = right / world_size * 360 - 180
  local north = math.atan(math.sinh(math.pi * (1 - 2 * top / world_size))) * 180 / math.pi
  local south = math.atan(math.sinh(math.pi * (1 - 2 * bottom / world_size))) * 180 / math.pi

  -- Build OSM embed URL with bbox + marker
  local src = string.format(
    "https://www.openstreetmap.org/export/embed.html?bbox=%f,%f,%f,%f&layer=mapnik&marker=%f,%f",
    west, south, east, north, lat, lon
  )

  return widget.new {
    html = '<iframe src="' .. src .. '" style="width:100% !important; height:' .. height .. 'px !important; border:0;" allowfullscreen="" loading="lazy" referrerpolicy="no-referrer-when-downgrade"></iframe>',
    display = "inline",
    cssClasses = {"my-map"}
  }
end


function embed_location(qry, height, zoom)
  -- Set defaults (height and zoom will be passed through to embed_latlng)
  local height = height or "480"
  local zoom = zoom or "14"

  -- URL-encode the query for the API call
  local encoded_qry = qry:gsub(" ", "+")

  -- Call Photon geocoding API
  local result = net.proxyFetch("https://photon.komoot.io/api/?limit=1&q=" .. encoded_qry)

  if not result.ok then
    return 'Geocoding API request failed for "' .. qry .. '"'
  end

  local data = result.body
  if not data or not data.features or #data.features == 0 then
    return 'No results found for "' .. qry .. '"'
  end

  -- Extract coordinates from the first result
  local coords = data.features[1].geometry.coordinates
  local lon = coords[1]
  local lat = coords[2]

  -- Get the name for display (optional, for the return fallback)
  local name = data.features[1].properties.name or qry

  -- Delegate to embed_latlng with the resolved coordinates
  return embed_latlng(lat .. ", " .. lon, height, zoom)
end

```
