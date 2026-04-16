# Prompt: 为 OLMoEarth 遥感大模型生成数据预处理管线代码

## 任务目标

为 OLMoEarth 遥感基础模型编写一套完整的数据预处理管线，将约 100TB 已下载的多源整景 GeoTIFF 数据转换为模型可直接读取训练的 H5 格式文件。

## 项目背景

OLMoEarth 是一个多模态、时空遥感基础模型，使用 MAE（掩码自编码器）进行自监督预训练。模型从 H5 文件中读取训练数据，每个 H5 文件对应地球上一个 128×128 像素（1.28km×1.28km）的区域，包含多种卫星和辅助数据模态。

现有的 OLMoEarth 代码中已实现了从"OlmoEarth 中间格式"到 H5 的转换（`olmoearth_pretrain/dataset/convert_to_h5py.py`），但**没有**从原始整景 GeoTIFF 到 OlmoEarth 中间格式的重投影和裁剪代码——这部分原本由外部工具 rslearn 完成。本任务需要编写替代 rslearn 的自定义预处理管线。

## 我拥有的数据源

### 时变卫星数据（多时相，每个区域需要 12 个月度时间步）

| 数据源 | 原始分辨率 | 波段 | QA/云掩码波段 |
|--------|-----------|------|-------------|
| Landsat 9 OLI-TIRS | 全色 15m, 多光谱 30m | B1-B7, B8(全色), B9, B10, B11 | QA_PIXEL |
| Sentinel-2 L2A | 10m/20m/60m | B01-B12, B8A (共13波段, 不含B10) | SCL (Scene Classification Layer) |
| Sentinel-1 IW GRD | 10m | VV, VH | 无（SAR 不受云影响）|
| Sentinel-3 OLCI | 300m | Oa01-Oa21 (21波段) | WQSF (质量标志位) |

### 静态辅助数据（单时相）

| 数据源 | 原始分辨率 | 波段数 |
|--------|-----------|--------|
| DEM (SRTM/Copernicus DEM) | 30m | 1 (高程值) |
| WorldCover (ESA) | 10m | 1 (土地覆盖分类) |
| OpenStreetMap 栅格化分类 | 矢量→2.5m | 30 (类别通道) |

## OLMoEarth 对输入数据的精确格式要求

### 1. 空间网格规范

- 投影：UTM 投影（每个区域自动选择对应的 UTM zone，如 EPSG:32650）
- 基础网格：256×256 像素 @ 10m/pixel = 2560m × 2560m
- H5 瓦片大小：128×128 像素（每个 256×256 被切为 4 个 128×128 子块）
- 网格坐标系统：`col = x_offset / 2560`, `row = y_offset / -2560`（整数网格索引）

### 2. 各模态在 OlmoEarth 中间格式中的存储规范

每个模态需要存为 GeoTIFF 文件，命名规则为 `{CRS}_{col}_{row}_{resolution}.tif`。

#### Sentinel-2 L2A（OLMoEarth 已支持，模态名 `sentinel2_l2a`）
- 三组不同分辨率的波段，分别存为**三个 tif 文件**：
  - 10m 波段 (B02, B03, B04, B08) → 256×256 → 文件名后缀 `_10.tif`
  - 20m 波段 (B05, B06, B07, B8A, B11, B12) → 128×128 → 文件名后缀 `_20.tif`
  - 60m→40m 波段 (B01, B09) → 64×64 → 文件名后缀 `_40.tif`
- 时变数据：12 个月度时间步，各时间步在 channel 维度堆叠
  - 例如 10m 文件的 shape 为 `(12×4=48, 256, 256)`
- 目录名：`10_sentinel2_l2a_monthly/`

#### Sentinel-1（OLMoEarth 已支持，模态名 `sentinel1`）
- 1 组波段：VV, VH @ 10m → 256×256 → `_10.tif`
- 12 个月度时间步堆叠：shape `(12×2=24, 256, 256)`
- 目录名：`10_sentinel1_monthly/`
- **特殊处理**：需要转换为 dB 值：`10 * log10(clip(data, 1e-10, None))`，此转换在 H5 转换阶段由现有代码自动完成，**你不需要在预处理中做这个转换**

#### Landsat 9（OLMoEarth 已支持，模态名 `landsat`）
- 2 组不同分辨率的波段，存为**两个 tif 文件**：
  - 15m→10m 波段 (B8 全色) → 256×256 → `_10.tif`
  - 30m→20m 波段 (B1, B2, B3, B4, B5, B6, B7, B9, B10, B11) → 128×128 → `_20.tif`
- 12 个月度时间步堆叠
- 目录名：`10_landsat_monthly/`

#### Sentinel-3 OLCI（⚠️ OLMoEarth **不支持**，需新增模态定义）
- 需要在 `olmoearth_pretrain/data/constants.py` 中新增 `SENTINEL3` ModalitySpec
- 建议配置：
  - `name="sentinel3"`, `tile_resolution_factor=16`, `is_multitemporal=True`
  - 由于原始分辨率 300m，建议存为单组波段 @ 320m（resolution_factor=512）或降采样到 10m 网格
  - 需要在 `norm_configs/computed.json` 和 `predefined.json` 中添加归一化参数
- 目录名：`10_sentinel3_monthly/`（具体取决于 ModalitySpec 配置）
- **这是一个需要额外注意的新增模态，不能直接接入现有管线**

#### SRTM/DEM（OLMoEarth 已支持，模态名 `srtm`）
- 1 个波段 @ 10m → 256×256 → `_10.tif`
- 静态数据，无时间维度
- 目录名：`10_srtm/`

#### WorldCover（OLMoEarth 已支持，模态名 `worldcover`）
- 1 个波段 @ 10m → 256×256 → `_10.tif`
- 静态数据，分类值
- 目录名：`10_worldcover/`

#### OpenStreetMap 栅格化（OLMoEarth 已支持，模态名 `openstreetmap_raster`）
- 30 个类别通道 @ 2.5m → 1024×1024 → `_2.5.tif`
- 静态数据
- 目录名：`10_openstreetmap_raster/`
- OSM 数据需要从矢量格式栅格化为 30 通道的二值栅格，每个通道对应一个地物类别：aerialway_pylon, aerodrome, airstrip, amenity_fuel, building, chimney, communications_tower, crane, flagpole, fountain, generator_wind, helipad, highway, leisure, lighthouse, obelisk, observatory, parking, petroleum_well, power_plant, power_substation, power_tower, river, runway, satellite_dish, silo, storage_tank, taxiway, water_tower, works

### 3. 元数据 CSV 格式

每个模态需要一个 CSV 文件，放在数据集根目录下，命名为 `{tile_resolution}_{modality_name}{time_suffix}.csv`。

列格式固定为：
```
crs,col,row,tile_time,image_idx,start_time,end_time
```

- `crs`：UTM 投影编号，如 `EPSG:32650`
- `col`, `row`：网格列行索引（整数）
- `tile_time`：该瓦片的中心时间（ISO 8601 格式，带时区）
- `image_idx`：时间步索引（0-11，对应 12 个月）
- `start_time`, `end_time`：该时间步对应的起止时间

**时变模态示例**（`10_sentinel2_l2a_monthly.csv`）：
```csv
crs,col,row,tile_time,image_idx,start_time,end_time
EPSG:32650,5,12,2023-06-15T00:00:00+00:00,0,2023-01-01T00:00:00+00:00,2023-01-31T00:00:00+00:00
EPSG:32650,5,12,2023-06-15T00:00:00+00:00,1,2023-02-01T00:00:00+00:00,2023-02-28T00:00:00+00:00
... (共12行)
EPSG:32650,5,12,2023-06-15T00:00:00+00:00,11,2023-12-01T00:00:00+00:00,2023-12-31T00:00:00+00:00
```

**静态模态示例**（`10_srtm.csv`）：
```csv
crs,col,row,tile_time,image_idx,start_time,end_time
EPSG:32650,5,12,2023-06-15T00:00:00+00:00,0,2000-01-01T00:00:00+00:00,2001-01-01T00:00:00+00:00
```

### 4. 最终目录结构

```
olmoearth_dataset/                          # 数据集根目录
├── 10_sentinel2_l2a_monthly/               # S2 L2A 时变
│   ├── EPSG:32650_5_12_10.tif              # 10m波段
│   ├── EPSG:32650_5_12_20.tif              # 20m波段
│   ├── EPSG:32650_5_12_40.tif              # 40m波段 (60m→40m)
│   └── ...
├── 10_sentinel1_monthly/                   # S1 时变
│   ├── EPSG:32650_5_12_10.tif
│   └── ...
├── 10_landsat_monthly/                     # Landsat 时变
│   ├── EPSG:32650_5_12_10.tif              # 全色波段
│   ├── EPSG:32650_5_12_20.tif              # 多光谱波段
│   └── ...
├── 10_srtm/                                # DEM 静态
│   ├── EPSG:32650_5_12_10.tif
│   └── ...
├── 10_worldcover/                          # WorldCover 静态
│   ├── EPSG:32650_5_12_10.tif
│   └── ...
├── 10_openstreetmap_raster/                # OSM 栅格 静态
│   ├── EPSG:32650_5_12_2.5.tif
│   └── ...
├── 10_sentinel2_l2a_monthly.csv            # 元数据 CSV
├── 10_sentinel1_monthly.csv
├── 10_landsat_monthly.csv
├── 10_srtm.csv
├── 10_worldcover.csv
└── 10_openstreetmap_raster.csv
```

## 处理管线的四个阶段

### 阶段 0：建立空间索引

扫描所有原始整景 GeoTIFF 的元数据（不读像素），提取：
- 地理边界（bounds）
- 投影信息（CRS）
- 获取时间（acquisition time）
- 云量百分比（cloud cover %，从文件名或 metadata 中提取）
- 数据源类型（Landsat/S1/S2/S3）

输出一个全局索引文件 `scene_index.parquet`，包含所有整景的空间、时间、云量信息，用于后续快速查询。使用 `rtree` 或 `geopandas.sindex` 建立空间 R-tree 索引。

### 阶段 1：确定采样网格

输入：研究区域边界（GeoJSON 或 bbox），或者经纬度列表（JSON）
输出：需要处理的所有网格瓦片列表 `grid_tiles.json`

处理逻辑：
1. 将研究区域按 UTM zone 分区
2. 在每个 UTM zone 内建立 2560m×2560m 的网格
3. 对每个网格瓦片，确定中心时间（选择卫星数据覆盖最好的时间段）
4. 输出每个瓦片的 `(crs, col, row, center_time)` 元组

### 阶段 2：逐景处理（核心，最耗时）

**关键设计原则：按整景处理（而非按网格），每个整景只打开一次，裁出所有覆盖的网格瓦片。**

对每一景 GeoTIFF：
1. 查空间索引，找到该整景覆盖的所有目标网格瓦片
2. 对该整景进行一次性读取和重投影（如果需要）
3. 逐个裁出每个网格瓦片的 256×256 区域
4. 对时变数据，多景堆叠为 12 个月度时间步

#### 云处理策略（分级处理）

对光学数据（S2、Landsat）的每个时间步：
- 使用 QA 波段（S2 的 SCL，Landsat 的 QA_PIXEL）检测云和云阴影
- 同一区域同一月份有多景时，优先选择云量最低的那一景
- 云像素占比 < 5%：保留，不做处理
- 云像素占比 5% ~ 50%：将云像素值设为 -99999（MISSING_VALUE）
- 云像素占比 > 50%：整个时间步标记为缺失
- 整景云量 > 80%：整景丢弃

**注意**：MISSING_VALUE = -99999 是 OLMoEarth 的统一缺失值标记，模型训练时会自动识别并跳过这些像素。

#### 分辨率处理

不同分辨率的波段需要分别处理并存为不同文件：
- 重投影时使用目标分辨率对应的像素尺寸
- 低分辨率波段重采样到目标分辨率时使用最近邻插值（`rasterio.enums.Resampling.nearest`），尤其是分类数据
- 连续数据（如 DEM）可以使用双线性插值

#### 每个模态的具体处理流程

**Sentinel-2 L2A**：
1. 读取 SAFE 格式或已处理的 GeoTIFF
2. 分三组波段分别处理和存储
3. 用 SCL 波段做云检测：SCL 值 8(cloud medium)、9(cloud high)、3(cloud shadow) 为云/阴影
4. 按分级策略处理云像素

**Sentinel-1**：
1. 读取 GRD 产品
2. 不需要去云（SAR 穿透云层）
3. 注意 nodata 值为 -32768，含此值的影像需要丢弃（现有代码已处理）
4. **不要在预处理中做 dB 转换**，H5 转换阶段会自动执行

**Landsat 9**：
1. 分两组波段处理
2. 用 QA_PIXEL 波段做云检测：bit 3 (cloud) 和 bit 4 (cloud shadow)
3. 15m 全色波段重采样到 10m，30m 多光谱波段重采样到 20m

**SRTM/DEM**：
1. 30m 重采样到 10m（双线性插值）
2. 静态数据只需处理一次

**WorldCover**：
1. 已经是 10m，通常只需裁剪
2. 分类数据，重采样必须用最近邻

**OpenStreetMap**：
1. 如果已有栅格化的 30 通道 tif，直接裁剪
2. 如果是矢量数据，需要栅格化为 1024×1024 @ 2.5m 的 30 通道二值栅格
3. 30 个通道分别对应 30 个地物类别（具体类别列表见上文）

### 阶段 3：生成元数据 CSV

遍历阶段 2 生成的所有 GeoTIFF 文件，为每个模态生成对应的 CSV 文件。

### 阶段 4：H5 转换（调用现有代码）

完成阶段 2 和 3 后，调用现有代码进行 H5 转换：
```bash
python -m olmoearth_pretrain.internal.run_h5_conversion \
  --tile_path=./olmoearth_dataset \
  --supported_modality_names='[sentinel2_l2a,sentinel1,landsat,srtm,worldcover,openstreetmap_raster]' \
  --compression=zstd \
  --compression_opts=3 \
  --tile_size=128
```

## 技术要求

### 并行化
- 100TB 数据量要求高效的并行处理
- 阶段 2 按整景并行：使用 `multiprocessing.Pool` 或 `concurrent.futures.ProcessPoolExecutor`
- 每个 worker 处理一景完整的 GeoTIFF（打开一次，裁出所有覆盖的网格）
- 支持断点续传：已处理的文件跳过不重复处理
- 进度跟踪：使用 tqdm 显示处理进度

### I/O 优化
- 使用 `rasterio` 的窗口读取（`rasterio.windows.from_bounds`）避免将整景全部加载到内存
- GeoTIFF 输出使用 LZW 压缩减少磁盘占用
- 写入时使用 tiling（block_size=32）

### 依赖库
- `rasterio`：GeoTIFF 读写、重投影、裁剪
- `pyproj`：坐标转换和 UTM zone 自动选择
- `numpy`：数组操作
- `pandas`：CSV 读写
- `geopandas` + `rtree`：空间索引
- `tqdm`：进度条
- `shapely`：几何操作

### 代码结构建议

```
data_pipeline/
├── __init__.py
├── config.py                  # 配置类：路径、模态定义、处理参数
├── scene_indexer.py            # 阶段0：扫描整景建立空间索引
├── grid_sampler.py             # 阶段1：生成目标网格瓦片列表
├── processors/                 # 阶段2：各模态处理器
│   ├── __init__.py
│   ├── base.py                # 基类：重投影、裁剪、窗口读取
│   ├── sentinel2.py           # S2 L2A 处理（含 SCL 云检测）
│   ├── sentinel1.py           # S1 处理
│   ├── landsat.py             # Landsat 9 处理（含 QA_PIXEL 云检测）
│   ├── sentinel3.py           # S3 OLCI 处理（新增模态）
│   ├── dem.py                 # DEM/SRTM 处理
│   ├── worldcover.py          # WorldCover 处理
│   └── osm.py                 # OSM 栅格化处理
├── cloud_masking.py            # 云检测和分级处理逻辑
├── csv_generator.py            # 阶段3：元数据 CSV 生成
├── pipeline.py                 # 主流程编排：串联四个阶段
└── utils.py                    # 工具函数：UTM zone 选择、网格计算等
```

### 关键工具函数需求

1. `get_utm_zone(lon, lat) -> str`：根据经纬度返回 UTM zone 的 EPSG 编码
2. `lonlat_to_grid(lon, lat, crs) -> (col, row)`：将经纬度转换为网格索引
3. `get_grid_bounds(crs, col, row) -> (left, bottom, right, top)`：获取网格瓦片的地理范围（UTM 坐标）
4. `reproject_and_crop(src_path, dst_crs, dst_bounds, dst_resolution, dst_size) -> np.ndarray`：重投影并裁剪到指定范围
5. `detect_cloud_pixels(qa_band, sensor_type) -> np.ndarray`：从 QA 波段提取云掩码
6. `apply_cloud_strategy(data, cloud_mask, threshold_low=0.05, threshold_high=0.50) -> np.ndarray`：应用分级云处理策略

## 输入路径约定（相对路径）

```
raw_data/                       # 原始整景 GeoTIFF 根目录
├── landsat9/                   # Landsat 9 整景
├── sentinel2/                  # Sentinel-2 L2A 整景
├── sentinel1/                  # Sentinel-1 GRD 整景
├── sentinel3/                  # Sentinel-3 OLCI 整景
├── dem/                        # DEM 数据
├── worldcover/                 # WorldCover 数据
└── osm/                        # OSM 栅格化数据

olmoearth_dataset/              # 输出：OlmoEarth 中间格式
scene_index.parquet             # 阶段0 输出：整景空间索引
grid_tiles.json                 # 阶段1 输出：目标网格列表
```

## 注意事项

1. Sentinel-3 OLCI 是 OLMoEarth 中**不存在的新模态**，需要先在 `olmoearth_pretrain/data/constants.py` 中添加 ModalitySpec 定义，并在 `olmoearth_pretrain/data/norm_configs/` 中添加归一化参数，才能被 H5 转换管线识别
2. 所有非 -99999 的像素值在 H5 加载时会被自动归一化，预处理阶段应**保留原始像素值**（如 S2 的 uint16 反射率值），不要手动归一化
3. Sentinel-1 的 dB 转换在 H5 生成阶段自动完成（`convert_to_h5py.py` 中对 S1 调用 `convert_to_db`），预处理阶段存原始线性值
4. 时变数据在 GeoTIFF 中按 channel 维度堆叠时间步：shape 为 `(T × C, H, W)`，即第 t 个时间步的第 c 个波段在 band index `t * num_bands + c`
5. 静态数据的 `image_idx` 始终为 0，CSV 只有一行
6. 确保每个模态的所有网格瓦片都出现在同一个 CSV 文件中（不是每个瓦片一个 CSV）
7. 至少一个时空变化模态（S2/S1/Landsat）必须有 12 个月度时间步，否则该样本会在 H5 转换的过滤阶段被丢弃
