# Tile Map Việt Nam — Hướng dẫn tích hợp (bản local)

Tài liệu này dành cho **project khác muốn dùng tile map này**. Phạm vi: chạy và gọi ở
mức local / mạng nội bộ. Bản đưa lên Internet sẽ làm sau — phần còn thiếu liệt kê ở
mục [Chưa có](#chưa-có--cần-làm-trước-khi-đưa-lên-internet).

---

## 1. Đây là gì

Một tile server chạy bằng Docker, phục vụ bản đồ nền Việt Nam theo chuẩn
[OpenMapTiles](https://openmaptiles.org/schema) v3.16.0. Client gọi vào để lấy
vector tile, raster tile, hoặc ảnh tĩnh.

| Thông tin | Giá trị |
|---|---|
| Nguồn dữ liệu | OpenStreetMap — extract Việt Nam của Geofabrik |
| Ngày chốt dữ liệu | **14/07/2026** (không tự cập nhật) |
| Schema | OpenMapTiles 3.16.0, 16 layer |
| Zoom | 0–14 |
| Số tile | 611.014 |
| Dung lượng | 537 MB (`data/tiles.mbtiles`) |
| Chiếu | Web Mercator (EPSG:3857), tile scheme XYZ |
| Định dạng tile | Mapbox Vector Tile (protobuf, gzip) |

Nội dung trong cơ sở dữ liệu: 1.252.679 toà nhà, 3.121.488 đoạn đường,
118.053 POI, 21.567 điểm địa danh. Nhãn đa ngôn ngữ lấy thêm từ Wikidata
(1.708 ID).

---

## 2. Khởi động server

Cần Docker Desktop đang chạy.

```bash
cd /đường/dẫn/tới/openmaptiles

# Khởi động
docker compose -f docker-compose.yml -f docker-compose-MINGW64.yml up -d tileserver-gl

# Kiểm tra
curl http://localhost:8080/styles.json

# Dừng
docker compose -f docker-compose.yml -f docker-compose-MINGW64.yml stop tileserver-gl
```

Server nghe ở `0.0.0.0:8080` nên máy khác trong mạng LAN gọi được bằng IP của máy chủ.

> **Trên Windows** phải dùng cả hai file compose như trên. `docker-compose-MINGW64.yml`
> chuyển thư mục `cache/` sang named volume; thiếu nó một số thao tác sẽ lỗi vì
> filesystem mount kiểu `9p`. Lệnh `make` **không dùng được** trên Windows — Makefile
> tự abort qua guard `win_fs_error`.

Đổi cổng: đặt biến `TPORT` trước khi `up`, ví dụ `TPORT=9000`.

Để server tự chạy lại sau khi khởi động máy, thêm vào service `tileserver-gl` trong
`docker-compose.yml`:

```yaml
    restart: unless-stopped
```

---

## 3. Endpoint

Thay `HOST` bằng `localhost` hoặc IP LAN của máy chủ (ví dụ `192.168.110.227`).

Style id là `OSM OpenMapTiles` — **có dấu cách**, khi đưa vào URL phải mã hoá thành
`OSM%20OpenMapTiles`.

| Endpoint | Trả về | Dùng cho |
|---|---|---|
| `GET /data/openmaptiles.json` | TileJSON 2.1 | Client tự khám phá nguồn tile |
| `GET /data/openmaptiles/{z}/{x}/{y}.pbf` | protobuf | **Vector tile** |
| `GET /styles/OSM%20OpenMapTiles/style.json` | JSON | Style hoàn chỉnh (MapLibre) |
| `GET /styles/OSM%20OpenMapTiles/{z}/{x}/{y}.png` | PNG 256px | **Raster tile** |
| `GET /styles/OSM%20OpenMapTiles/{z}/{x}/{y}@2x.png` | PNG 512px | Raster cho màn Retina |
| `GET /styles/OSM%20OpenMapTiles/static/{lon},{lat},{zoom}/{w}x{h}.png` | PNG | **Ảnh tĩnh** |
| `GET /styles/OSM%20OpenMapTiles/sprite.json` + `.png` | JSON/PNG | Bộ icon |
| `GET /fonts/{fontstack}/{range}.pbf` | protobuf | Font nhãn |
| `GET /styles.json`, `GET /index.json` | JSON | Liệt kê tài nguyên |

Đặc điểm đã kiểm chứng:

- **CORS**: `Access-Control-Allow-Origin: *` — web client bất kỳ gọi được, không cần proxy.
- **URL tự khớp host**: gọi qua IP LAN thì tile URL trong TileJSON trả về đúng IP đó,
  không cứng `localhost`. Cũng tôn trọng `X-Forwarded-Host` / `X-Forwarded-Proto`
  nên chạy được sau reverse proxy.

---

## 4. Tích hợp

### MapLibre GL JS — cách khuyến nghị

Vector tile, nhãn xoay theo hướng, đổi style động được. Cần WebGL.

```html
<link href="https://unpkg.com/maplibre-gl/dist/maplibre-gl.css" rel="stylesheet" />
<script src="https://unpkg.com/maplibre-gl/dist/maplibre-gl.js"></script>
<div id="map" style="width:100%;height:600px"></div>

<script>
const map = new maplibregl.Map({
  container: 'map',
  style: 'http://localhost:8080/styles/OSM%20OpenMapTiles/style.json',
  center: [105.8542, 21.0285],   // [kinh độ, vĩ độ] — Hà Nội
  zoom: 12,
  maxZoom: 18                     // >14 sẽ phóng to tile z14
});
map.addControl(new maplibregl.NavigationControl());
</script>
```

Lưu ý: `style.json` nặng **491 KB** (xem [mục 7](#7-tuỳ-chỉnh-riêng-của-bản-này)).
Nếu client tải style nhiều lần, nên cache lại.

### Leaflet — không cần WebGL

Nhẹ, chạy được trên máy yếu và trình duyệt cũ. Server render PNG nên tốn CPU máy chủ.

```html
<link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css" />
<script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
<div id="map" style="width:100%;height:600px"></div>

<script>
const map = L.map('map').setView([21.0285, 105.8542], 12);  // [vĩ độ, kinh độ]
L.tileLayer('http://localhost:8080/styles/OSM%20OpenMapTiles/{z}/{x}/{y}{r}.png', {
  maxNativeZoom: 14,
  maxZoom: 18,
  detectRetina: true,
  attribution: '&copy; <a href="https://openmaptiles.org/">OpenMapTiles</a> ' +
               '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap contributors</a>'
}).addTo(map);
</script>
```

### OpenLayers

```js
import Map from 'ol/Map';
import View from 'ol/View';
import TileLayer from 'ol/layer/Tile';
import XYZ from 'ol/source/XYZ';
import { fromLonLat } from 'ol/proj';

new Map({
  target: 'map',
  layers: [new TileLayer({
    source: new XYZ({
      url: 'http://localhost:8080/styles/OSM%20OpenMapTiles/{z}/{x}/{y}.png',
      maxZoom: 14,
      attributions: '© OpenMapTiles © OpenStreetMap contributors'
    })
  })],
  view: new View({ center: fromLonLat([105.8542, 21.0285]), zoom: 12 })
});
```

### Ảnh tĩnh — nhúng báo cáo, gửi email, PDF

```
GET /styles/OSM%20OpenMapTiles/static/{lon},{lat},{zoom}/{width}x{height}.png
```

```bash
curl -o hanoi.png \
  "http://localhost:8080/styles/OSM%20OpenMapTiles/static/105.8542,21.0285,13/800x500.png"
```

Thêm `@2x` trước `.png` để lấy ảnh gấp đôi độ phân giải.

### Gọi từ backend

Tile là protobuf đã gzip. Ví dụ Node.js lấy tile thô:

```js
const res = await fetch(
  'http://localhost:8080/data/openmaptiles/14/13009/7212.pbf'
);
// res.status === 204  -> ô đó không có tile (ngoài vùng phủ)
// res.status === 200 nhưng body rỗng -> có tile nhưng không có đối tượng nào
const buf = Buffer.from(await res.arrayBuffer());
```

Công thức đổi toạ độ sang chỉ số tile:

```js
function lonLatToTile(lon, lat, z) {
  const n = 2 ** z;
  const x = Math.floor((lon + 180) / 360 * n);
  const latRad = lat * Math.PI / 180;
  const y = Math.floor((1 - Math.asinh(Math.tan(latRad)) / Math.PI) / 2 * n);
  return { x, y };
}
// lonLatToTile(105.848, 21.0245, 14) -> { x: 13009, y: 7212 }
```

> Tile scheme là **XYZ** (gốc toạ độ ở góc trên-trái). File mbtiles bên trong lưu
> theo TMS, nhưng tileserver đã lật trục Y giúp rồi — client cứ dùng XYZ.

---

## 5. Phạm vi phủ dữ liệu

Đây là phần quan trọng nhất phải nắm, vì **không phải chỗ nào cũng có tile**.

| Zoom | Phạm vi có tile | Số tile |
|---|---|---|
| 0–7 | **Toàn thế giới** | 21.845 |
| 8–12 | Việt Nam + biển đảo (100,6–114,6 / 6,2–23,5) | 44.362 |
| 13–14 | Đất liền (102,1–109,5 / 8,2–23,4) **và** dải biển đông (109,5–114,65 / 6,2–23,5) | 544.807 |

Hệ quả cụ thể:

- **z0–7**: nhìn được cả thế giới, nhưng ngoài Việt Nam chỉ có dữ liệu Natural Earth
  (đường bờ biển, biên giới quốc gia, tên nước). Không có đường sá.
- **z8–12 ngoài Việt Nam**: tile tồn tại nhưng gần như trống (HTTP 200, 0 byte).
- **z13–14 ngoài hai khung trên**: **không có tile**, server trả HTTP **204 No Content**.
  Client phải xử lý được trường hợp này — MapLibre coi 204 là ô trống và vẽ nền.
- **Zoom > 14**: không có tile z15+. Đặt `maxZoom: 18` kèm `maxNativeZoom: 14`
  (Leaflet) hoặc để MapLibre tự phóng to tile z14.

Tên đường phố chỉ xuất hiện từ **z14**. Đây là thiết kế của schema OpenMapTiles:
ở z≤13 layer `transportation_name` lọc qua hàm `LineLabel()` nên chỉ giữ đoạn đường
đủ dài. Đo thực tế trên một vùng Hà Nội: z12 có 47 tên đường, z13 có 157, z14 có 15.570.

---

## 6. 16 layer trong vector tile

Dùng khi bạn tự viết style hoặc truy vấn đối tượng bằng `queryRenderedFeatures`.

| Layer | Nội dung |
|---|---|
| `water` | Mặt nước: biển, hồ, sông dạng vùng |
| `waterway` | Sông suối dạng đường |
| `landcover` | Lớp phủ: rừng, cát, băng, cỏ |
| `landuse` | Mục đích sử dụng đất: dân cư, công nghiệp, bãi đỗ xe |
| `mountain_peak` | Đỉnh núi (có cao độ) |
| `park` | Công viên, khu bảo tồn |
| `boundary` | Ranh giới hành chính dạng đường |
| `aeroway` | Đường băng, sân đỗ máy bay |
| `transportation` | **Đường bộ, đường sắt, đường thuỷ** (hình học) |
| `building` | Toà nhà (có `render_height`) |
| `water_name` | Nhãn thuỷ vực |
| `transportation_name` | **Nhãn tên đường** (tách riêng khỏi hình học) |
| `place` | Địa danh: tỉnh, thành phố, phường xã, đảo |
| `housenumber` | Số nhà |
| `poi` | Điểm quan tâm: cửa hàng, trường, bệnh viện… |
| `aerodrome_label` | Nhãn sân bay (có mã IATA/ICAO) |

Chi tiết trường dữ liệu từng layer: <https://openmaptiles.org/schema>

**Tên tiếng Việt**: trường `name` giữ nguyên tên gốc OSM (có dấu). Ngoài ra có
`name:vi`, `name:en` và nhiều ngôn ngữ khác. Style mặc định ưu tiên `name:en`
rồi mới `name` — nếu muốn hiện tiếng Việt trước, sửa biểu thức `text-field`
trong style thành:

```json
["coalesce", ["get", "name:vi"], ["get", "name"]]
```

---

## 7. Tuỳ chỉnh riêng của bản này

Bản tile map này **khác style OpenMapTiles gốc** ở hai điểm, đều nằm trong source
GeoJSON tên `vn-overlay` khai báo ở `style/style-header.json`:

1. **Nhãn hai quần đảo** — hai điểm nhãn "Quần đảo Hoàng Sa" và "Quần đảo Trường Sa",
   hiện từ z4. Phải thêm thủ công vì schema OpenMapTiles không map
   `place=archipelago`, và file OSM gốc cũng không có hai đối tượng này.

2. **Biên giới quốc gia** — trích từ quan hệ OSM `r49915` ngay trong file pbf
   (600 way, đơn giản hoá Douglas–Peucker dung sai 0,0002° ≈ 22 m). Cần làm vì
   imposm3 không dựng được `admin_level=2` từ quan hệ, khiến bảng
   `osm_border_linestring` **không có dòng nào ở cấp quốc gia** — biên giới sẽ biến
   mất hoàn toàn từ z5 trở lên nếu không có lớp overlay này.

### ⚠️ Rủi ro cần biết

Hai layer vẽ (`VN national border`, `VN archipelago labels`) hiện **chỉ tồn tại
trong file đã build `build/style/style.json`**, không có trong file nguồn
`layers/boundary/style.json` và `layers/place/style.json` — hai file này bị hoàn tác
sau mỗi lần sửa trong môi trường hiện tại.

**Chạy lại `make build-style` hoặc `style-tools recompose` sẽ làm mất hai layer đó.**
Nếu cần rebuild style, phải thêm lại thủ công. Nên sao lưu `build/style/style.json`
trước khi rebuild.

### Về hai quần đảo

- **Trường Sa**: có dữ liệu OSM — Song Tử Tây, Đá Tây A, Đảo Đá Nam, Đá Đông C,
  Đá Lát, An Bang. Hiện tên đảo, đường nội bộ, công trình.
- **Hoàng Sa**: **không có dữ liệu OSM** trong file pbf này (Geofabrik xếp vào extract
  Trung Quốc). Đảo vẫn hiện đúng **hình dạng** nhờ dữ liệu đường bờ biển toàn cầu,
  nhưng **không có tên đảo, đường sá, công trình**.
- **Không có đường chủ quyền biển** (đường cơ sở, lãnh hải, EEZ). Dữ liệu này không
  tồn tại trong OSM lẫn Natural Earth. Đường màu tím chạy ra biển là hình học của
  quan hệ biên giới OSM, không phải đường chủ quyền. Muốn có thì phải chồng thêm
  dataset bên ngoài vào source `vn-overlay`.

---

## 8. Ghi công — bắt buộc

Giấy phép OpenMapTiles yêu cầu **hiển thị rõ** dòng ghi công trên bản đồ:

```
© OpenMapTiles © OpenStreetMap contributors
```

kèm liên kết tới <https://openmaptiles.org/> và
<https://www.openstreetmap.org/copyright>.

Chuỗi HTML sẵn sàng dùng đã nằm trong `/data/openmaptiles.json` trường `attribution` —
MapLibre tự đọc và hiển thị. Với Leaflet/OpenLayers phải truyền tay như ví dụ ở mục 4.

Bản đồ in hoặc ảnh tĩnh cũng phải ghi công tương tự ở phần chú thích.

---

## 9. Xử lý sự cố

| Hiện tượng | Nguyên nhân | Cách xử lý |
|---|---|---|
| Bản đồ trắng, chỉ có nền | Client không tải được tile | Mở DevTools → Network, xem `/data/openmaptiles.json` có trả 200 không. Nếu URL trong đó bị lặp `/data//data/` thì mục `data` trong `style/config.json` đã bị xoá — phải khôi phục. |
| Trắng khi zoom sâu ở một số vùng | Ngoài khung phủ z13–14 | Xem [mục 5](#5-phạm-vi-phủ-dữ-liệu). Đây là giới hạn dữ liệu, không phải lỗi. |
| Không thấy tên đường | Đang ở zoom < 14 | Zoom tới z14. |
| Tile cũ vẫn hiện sau khi sinh lại | Trình duyệt cache | Ctrl+F5. |
| Chữ hiện thành ô vuông | Thiếu font | Kiểm tra `data/fonts/` có đủ 10 fontstack; `GET /fonts/Open%20Sans%20Regular/0-255.pbf` phải trả 200. |
| `SQLITE_CANTOPEN` khi sinh lại tile | tileserver-gl đang giữ file mbtiles | **Dừng tileserver-gl trước** khi xoá hoặc sinh lại `data/tiles.mbtiles`, xong bật lại. |
| Sinh tile báo `Invalid minzoom param` | File yaml có line ending CRLF, script `grep` ra `"0\r"` | Truyền thẳng `-e TILESET_MIN_ZOOM=0 -e TILESET_MAX_ZOOM=14` cho `docker compose run`. |

Xem log:

```bash
docker compose -f docker-compose.yml -f docker-compose-MINGW64.yml logs -f tileserver-gl
```

---

## 10. Sinh lại tile

Chỉ cần khi cập nhật dữ liệu OSM mới hoặc đổi phạm vi phủ. Bình thường **không cần
đụng tới** — file `data/tiles.mbtiles` dùng được ngay.

Toàn bộ quy trình (Windows, không dùng `make`):

```bash
DC="docker compose -f docker-compose.yml -f docker-compose-MINGW64.yml"

# 0. Dừng tileserver để nhả file mbtiles
$DC stop tileserver-gl

# 1. Khởi động database
$DC up -d postgres

# 2. Nạp dữ liệu toàn cầu (Natural Earth, đường bờ biển) — chỉ cần lần đầu
$DC run --rm import-data

# 3. Nạp file OSM  (~5 phút cho 310 MB)
$DC run --rm openmaptiles-tools sh -c "pgwait && import-osm data/vietnam-260714.osm.pbf"

# 4. Nhãn đa ngôn ngữ từ Wikidata
$DC run --rm openmaptiles-tools import-wikidata --cache /cache/wikidata-cache.json openmaptiles.yaml

# 5. Nạp hàm SQL của các layer
$DC run --rm openmaptiles-tools sh -c "pgwait && import-sql"

# 6. Sinh tile — ba lượt, xem bảng phạm vi ở mục 5
$DC run -T --rm -e TILESET_MIN_ZOOM=0 -e TILESET_MAX_ZOOM=14 openmaptiles-tools sh -c \
  "BBOX='-180.0,-85.0511,180.0,85.0511' MIN_ZOOM=0 MAX_ZOOM=7 generate-tiles && \
   BBOX='100.6067931,6.2183523,114.6407887,23.5024689' MIN_ZOOM=8 MAX_ZOOM=12 generate-tiles && \
   BBOX='102.1,8.2,109.5,23.4' MIN_ZOOM=13 MAX_ZOOM=14 generate-tiles && \
   BBOX='109.5,6.2,114.65,23.5' MIN_ZOOM=13 MAX_ZOOM=14 generate-tiles"

# 7. Cập nhật metadata
$DC run --rm openmaptiles-tools mbtiles-tools meta-generate /export/tiles.mbtiles openmaptiles.yaml --auto-minmax
$DC run --rm openmaptiles-tools mbtiles-tools meta-set /export/tiles.mbtiles center "106.5,16.2,6"

# 8. Bật lại
$DC up -d tileserver-gl
```

Thời gian tham khảo đo được trên máy đã chạy:

| Bước | Thời gian |
|---|---|
| `import-osm` (310 MB) | 4 phút 50 giây |
| Sinh z0–7 toàn cầu | ~3 phút (21.845 tile) |
| Sinh z8–12 | ~9 phút (44.362 tile) |
| Sinh z13–14 đất liền | 48 phút (305.000 tile) |
| Sinh z13–14 biển đông | 30 phút (240.800 tile) |
| **Tổng** | **khoảng 1 giờ 35 phút** |

`import-osm` làm **full import có xoay bảng** — dữ liệu vùng cũ bị thay thế hoàn toàn.
Muốn ghép nhiều khu vực thì phải `osmium merge` các file `.osm.pbf` trước.

---

## Chưa có — cần làm trước khi đưa lên Internet

Bản hiện tại **chỉ an toàn trong mạng nội bộ**. Thiếu:

- **Xác thực và giới hạn tốc độ.** Ai vào được cổng 8080 đều tải không giới hạn.
  Một tile z14 trung tâm Hà Nội nặng 989 KB — vài client quét tile là nghẽn.
- **HTTPS.** Trang web chạy HTTPS sẽ bị trình duyệt chặn khi gọi sang HTTP
  (mixed content).
- **Reverse proxy.** Đã hỗ trợ `X-Forwarded-Host`, nhưng cổng `:8080` vẫn bị chèn vào
  URL sinh ra — cần đặt thêm `--public_url` cho tileserver-gl.
- **Cache tầng biên (CDN).** Tile là dữ liệu tĩnh, rất hợp để cache.
- **Giảm kích thước style.json.** Đang 491 KB, trong đó ~381 KB là toạ độ biên giới
  nhúng inline. Nên đưa đường biên vào chính vector tile thay vì nhúng vào style.
- **Cơ chế cập nhật dữ liệu.** Hiện chốt cứng ngày 14/07/2026.
- **`restart: unless-stopped`** cho container.

---

## Tham khảo

- Schema OpenMapTiles: <https://openmaptiles.org/schema>
- tileserver-gl: <https://github.com/maptiler/tileserver-gl>
- MapLibre GL JS: <https://maplibre.org/maplibre-gl-js/docs/>
- Đặc tả style: <https://maplibre.org/maplibre-style-spec/>
- Giấy phép: [LICENSE.md](./LICENSE.md) — code theo BSD, thiết kế bản đồ theo CC-BY
