# 规则引擎

MediaTidy 的规则引擎包含三种规则：**分类规则**、**命名规则**、**质量评分规则**。每种规则均使用 YAML 格式配置，支持三层优先级合并：**本地 > 订阅源 > 内置**。

## 规则来源与优先级

每种规则有三个来源，按优先级从高到低：

| 来源 | 说明 | 优先级 |
|------|------|--------|
| **本地** | 在管理面板「规则引擎」页面直接编辑的 YAML | 最高 |
| **订阅源** | 从远程 URL 定期拉取的 YAML | 中 |
| **内置** | 系统自带的默认规则 | 最低 |

**合并逻辑：**
- **命名规则 / 质量规则**：按规则的 `name` 字段合并。同名规则高优先级覆盖低优先级，不同名的规则全部保留
- **分类规则**：按整体替换。如果本地有分类规则，则完全使用本地的；如果本地为空但订阅源有，则使用订阅源的；都没有则使用内置默认

---

## 分类规则

分类规则根据 TMDB 元数据（类型、语言、国家、关键词等）将媒体文件自动归类到不同的媒体库子目录。

### YAML 格式

```yaml
movie:
  目标路径:
    条件字段1: "值"
    条件字段2: "值"

tv:
  目标路径:
    条件字段1: "值"
```

顶层分为 `movie`（电影）和 `tv`（剧集）两个区块，每个区块下定义多条规则。规则**按定义顺序从上到下匹配**，命中第一条即停止。

### 匹配条件字段

| 字段 | 数据来源 | 说明 | 示例值 |
|------|----------|------|--------|
| `genre_ids` | TMDB 类型 ID | 内容类型，支持多值逗号分隔 | `"16"` (动画)、`"16,10762"` |
| `original_language` | TMDB 原始语言 | ISO 639-1 语言代码 | `"zh"`, `"ja,ko"` |
| `origin_country` | TMDB 制作国家（TV） | ISO 3166-1 国家代码 | `"CN,TW,HK"` |
| `production_countries` | TMDB 制作国家（通用） | 会同时匹配 origin_country | `"US,GB"` |
| `keywords` | TMDB 标题文本 | 标题包含关键词即匹配 | `"奥特曼,假面骑士"` |
| `include_keywords` | TMDB 标签关键词 | 匹配 TMDB 的 keyword 标签 | `"tokusatsu,superhero"` |
| `filename_keywords` | 原始文件名 | 文件名包含关键词即匹配 | `"REMUX,Atmos"` |
| `series_directors` | TMDB 导演 | 导演名称包含匹配 | `"宫崎骏"` |
| `series_actors` | TMDB 演员 | 演员名称包含匹配（始终可选） | `"成龙,李连杰"` |
| `vote_average` | TMDB 评分 | 评分范围 min-max | `"0.1-5.6"` |
| `year` / `release_year` | TMDB 年份 | 精确、范围或列表 | `"2020"`, `"2000-2020"`, `"2020-"` |
| `file_extension` | 文件后缀名 | 小写扩展名 | `"mkv,iso"` |
| `resolution` | 分辨率 | 从文件名或 ffprobe 获取 | `"2160p,1080p"` |
| `hdr` / `dynamic_range` | HDR 类型 | 从文件名或 ffprobe 获取 | `"Dolby Vision,HDR10"` |

### 条件语法详解

#### 基本语法：逗号 = OR

多个值用逗号分隔，满足任一即匹配（支持中英文逗号）：

```yaml
original_language: "zh,cn,bo,za"   # 中文 OR 藏语 OR 壮语
```

#### 排除语法：`-` 前缀

值前加 `-` 表示排除，如果匹配则整条规则失败：

```yaml
genre_ids: "16,-10762"    # 有动画(16) AND 没有儿童(10762)
original_language: "-zh"   # 不是中文
```

#### 必须语法：`+` 前缀（仅 genre_ids）

值前加 `+` 表示必须同时包含：

```yaml
genre_ids: "+16,+10762"   # 必须同时有动画(16) AND 儿童(10762)
```

#### 可选条件：`?` 前缀

字段名前加 `?` 表示该条件为可选。所有必须条件都满足 + 至少一个可选条件满足 = 匹配：

```yaml
电视剧/香港剧:
  ?include_keywords: "TVB,HK"   # 可选：TMDB 标签含 TVB 或 HK
  ?origin_country: "HK"          # 可选：制作国家为香港
  # 两个可选条件满足任一即可
```

#### 否定条件：`-` 字段前缀

字段名前加 `-` 或 `!` 表示整个条件取反：

```yaml
-origin_country: "CN"   # 制作国家不是中国
```

#### 空条件 = 保底规则

没有任何条件的规则是保底规则（catch-all），永远匹配：

```yaml
电视剧/未分类:
  # 无条件，兜底匹配所有未被上方规则命中的剧集
```

### 匹配逻辑总结

```
规则匹配 = 所有必须条件全部通过 AND (没有可选条件 OR 至少一个可选条件通过)
```

- 必须条件（无 `?` 前缀）：全部 AND 关系
- 可选条件（有 `?` 前缀）：OR 关系，至少一个通过
- `series_actors` / `series_keywords` 即使没有 `?` 前缀也自动视为可选

### 完整示例

```yaml
movie:
  # 优先级1：特摄电影（标题或TMDB标签匹配，至少一个）
  电影/特摄电影:
    ?keywords: "奥特曼,假面骑士,铠甲勇士,Power Rangers"
    ?include_keywords: "tokusatsu,ultraman"

  # 优先级2：动画电影
  电影/动画电影:
    genre_ids: "16"

  # 优先级3：纪录电影
  电影/纪录电影:
    genre_ids: "99"

  # 优先级4：华语电影
  电影/华语电影:
    original_language: "zh,cn"

  # 优先级5：日韩电影
  电影/日韩电影:
    original_language: "ja,ko"

  # 优先级6：外语电影（兜底，排除已分类语言）
  电影/外语电影:
    original_language: "-cn,-zh,-ja,-ko"

tv:
  # 优先级1：综艺
  电视剧/综艺:
    genre_ids: "10764,10767"

  # 优先级2：纪录片
  电视剧/纪录片:
    genre_ids: "99,-10762"

  # 优先级3：国漫（动画 + 中港台，排除儿童）
  动漫/国漫:
    genre_ids: "16,-10762"
    origin_country: "CN,TW,HK"

  # 优先级4：日番
  动漫/日番:
    genre_ids: "16,-10762"
    origin_country: "JP"

  # 优先级5：国产剧
  电视剧/国产剧:
    origin_country: "CN,TW"

  # 优先级6：日韩剧
  电视剧/日韩剧:
    origin_country: "KR,JP"

  # 优先级7：欧美剧
  电视剧/欧美剧:
    origin_country: "US,GB,FR,DE,ES,IT,CA"

  # 保底
  电视剧/未分类:
```

### TMDB 类型 ID 速查

| ID | 类型 | ID | 类型 |
|----|------|----|------|
| 16 | 动画 | 99 | 纪录 |
| 28 | 动作 | 12 | 冒险 |
| 35 | 喜剧 | 18 | 剧情 |
| 27 | 恐怖 | 53 | 惊悚 |
| 80 | 犯罪 | 878 | 科幻 |
| 14 | 奇幻 | 36 | 历史 |
| 10402 | 音乐 | 10749 | 爱情 |
| 10751 | 家庭 | 10752 | 战争 |
| 10762 | 儿童 | 10764 | 真人秀 |
| 10767 | 脱口秀 | 10770 | 电视电影 |

### 常用国家/语言代码

| 代码 | 国家/地区 | 代码 | 语言 |
|------|----------|------|------|
| CN | 中国大陆 | zh | 中文 |
| TW | 中国台湾 | ja | 日语 |
| HK | 中国香港 | ko | 韩语 |
| JP | 日本 | en | 英语 |
| KR | 韩国 | hi | 印地语 |
| US | 美国 | th | 泰语 |
| GB | 英国 | vi | 越南语 |

---

## 命名规则

命名规则定义整理后文件的**文件夹路径**和**文件名**格式。每条规则包含 `movie`（电影）和 `tv`（剧集）两套独立模板。

### YAML 格式

<div v-pre>

```yaml
naming:
  - name: 规则显示名称
    id: 唯一标识符
    movie:
      folder: "{{title}} ({{year}}) [tmdbid-{{tmdbid}}]"
      file: "{{title}} ({{year}}).{{fileExt}}"
    tv:
      folder: "{{title}} ({{year}}) [tmdbid-{{tmdbid}}]/Season {{season02}}"
      file: "{{title}} - {{seasonEpisode}}.{{fileExt}}"
```

</div>

- `folder`：文件夹路径模板，用 `/` 分隔多级目录
- `file`：文件名模板（含扩展名）

### 模板变量完整列表

<div v-pre>

**基本信息：**

| 变量 | 说明 | 示例 |
|------|------|------|
| `{{title}}` | TMDB 中文标题 | 流浪地球 |
| `{{originalTitle}}` | TMDB 原始标题 | The Wandering Earth |
| `{{tmdbid}}` | TMDB ID | 475132 |
| `{{year}}` | 年份 | 2019 |
| `{{mediaType}}` | 媒体类型 | movie / tv |
| `{{fileExt}}` | 文件扩展名（不含点） | mkv |

**季集信息（仅剧集有效）：**

| 变量 | 说明 | 示例 |
|------|------|------|
| `{{season}}` | 季号 | 1 |
| `{{season02}}` | 季号两位补零 | 01 |
| `{{episode}}` | 集号 | 5 |
| `{{episode02}}` | 集号两位补零 | 05 |
| `{{seasonEpisode}}` | 组合格式 | S01E05 |
| `{{episodeTitle}}` | 集标题（从文件名解析） | The Pilot |

**文件名解析字段（从文件名自动提取）：**

| 变量 | 说明 | 示例 |
|------|------|------|
| `{{videoFormat}}` | 分辨率 | 2160p, 1080p |
| `{{videoCodec}}` | 视频编码 | HEVC, AVC, AV1 |
| `{{audioCodec}}` | 音频编码 | DTS-HD MA, TrueHD |
| `{{audioChannels}}` | 声道 | 7.1, 5.1 |
| `{{audioProfile}}` | 音频特性 | Atmos |
| `{{source}}` | 片源 | BluRay, WEB-DL, BluRay Remux |
| `{{edition}}` | 版本 | Director's Cut, Extended |
| `{{dynamicRange}}` | 动态范围 | HDR, HDR10+, DV |
| `{{releaseGroup}}` | 发布组 | FRDS, CtrlHD |
| `{{streamingService}}` | 流媒体来源 | Netflix, Disney+ |
| `{{colorDepth}}` | 色深 | 10bit |
| `{{frameRate}}` | 帧率 | 60fps |
| `{{videoProfile}}` | 视频特性 | Main 10 |
| `{{regionCode}}` | 区域码 | R1, A |

**ffprobe 探测字段（开启元数据提取后可用，优先级高于文件名解析）：**

| 变量 | 说明 | 示例 |
|------|------|------|
| `{{probeResolution}}` | 探测分辨率 | 2160p, 1080p |
| `{{probeCodec}}` | 探测视频编码 | H.265, H.264 |
| `{{probeAudio}}` | 探测音频编码 | Dolby TrueHD, DTS-HD MA |
| `{{probeHDR}}` | 探测 HDR 类型 | Dolby Vision P8, HDR10 |
| `{{probeSubtitle}}` | 探测字幕 | PGS 简体中文 |
| `{{probeColorDepth}}` | 探测色深 | 10-bit |
| `{{probeFrameRate}}` | 探测帧率 | 24fps |

**多版本标签：**

| 变量 | 说明 | 示例 |
|------|------|------|
| `{{versionTag}}` | 多版本版本标签 | 2160p DV, 1080p SDR |

</div>

::: tip 变量优先级
当同时开启 ffprobe 探测时，`videoFormat`、`videoCodec`、`audioCodec`、`dynamicRange` 四个变量会被 ffprobe 结果覆盖（因为 ffprobe 从实际视频流提取，比文件名解析更准确）。如果你想明确使用 ffprobe 数据，可以直接用 `probeXxx` 系列变量。
:::

### 模板语法说明

- 文件夹路径用 `/` 分隔层级
- 缺失的变量会被替换为空字符串，系统会自动清理多余空格和连字符
- 文件名中的特殊字符 `< > : " | ? *` 会被自动过滤
- 连续的空格、连续的点号（`..`）会被自动合并

### 内置命名规则

<div v-pre>

| 规则名称 | ID | 电影文件名示例 |
|----------|----|----|
| **标准命名** | `standard` | `流浪地球 (2019).mkv` |
| **详细命名** | `detailed` | `流浪地球 (2019) - 2160p H.265 WEB-DL.mkv` |
| **完整命名** | `full` | `流浪地球 (2019) - 2160p H.265 DTS-HD MA HDR WEB-DL FRDS.mkv` |
| **原始标题** | `original-title` | `The Wandering Earth (2019).mkv` |
| **双语标题** | `bilingual` | `流浪地球 The Wandering Earth (2019).mkv` |
| **Plex 风格** | `plex` | `流浪地球 (2019).mkv`（文件夹用 `{tmdb-xxx}`） |
| **精简命名** | `minimal` | `流浪地球.mkv` |
| **发烧友** | `audiophile` | `流浪地球 (2019) - 2160p H.265 DTS-HD MA 7.1 HDR 10bit BluRay Remux FRDS.mkv` |
| **流媒体标注** | `streaming-tag` | `流浪地球 (2019) - 2160p Netflix H.265.mkv` |
| **探测数据** | `probe-based` | `流浪地球 (2019) - 2160p H.265 Dolby TrueHD Dolby Vision P8.mkv` |

所有内置规则的文件夹模板统一为 `{{title}} ({{year}}) [tmdbid-{{tmdbid}}]`（电影）和 `{{title}} ({{year}}) [tmdbid-{{tmdbid}}]/Season {{season02}}`（剧集），确保与 Emby/Jellyfin 兼容。

</div>

### 自定义命名规则示例

<div v-pre>

```yaml
naming:
  # 自定义：按语言分文件夹 + 带质量信息
  - name: 语言分类命名
    id: my-custom
    movie:
      folder: "{{title}} ({{year}}) [tmdbid-{{tmdbid}}]"
      file: "{{title}} ({{year}}) - {{videoFormat}} {{source}} {{videoCodec}} {{dynamicRange}}.{{fileExt}}"
    tv:
      folder: "{{title}} ({{year}}) [tmdbid-{{tmdbid}}]/Season {{season02}}"
      file: "{{title}} - {{seasonEpisode}} - {{videoFormat}} {{source}}.{{fileExt}}"
```

</div>

---

## 质量评分规则

质量评分规则用于**洗版升级**场景——当目标目录已有同一部影片时，对比新旧文件的质量分数，决定是否替换。

### 评分公式

```
总分 = Σ (维度权重 × 维度位置分)
```

- **位置分**：由 `priority` 列表决定，排名越靠前得分越高
- **权重 (`weight`)**：控制该维度对总分的影响程度
- 可以将权重设为 **负值**（如省空间方案中 `file_size: -10`，文件越大扣分越多）

### 评分维度

| 维度 | YAML 键 | 数据来源 | 说明 |
|------|---------|----------|------|
| **分辨率** | `resolution` | 文件名 / ffprobe | 4320p > 2160p > 1080p > 720p ... |
| **片源** | `source` | 文件名 | Remux+UHD BluRay > BluRay > WEB-DL ... |
| **视频编码** | `video_codec` | 文件名 / ffprobe | AV1 > H.265 > H.264 ... |
| **音频编码** | `audio_codec` | 文件名 / ffprobe | DTS:X > TrueHD Dolby Atmos > DTS-HD MA ... |
| **HDR** | `hdr` | 文件名 / ffprobe | Dolby Vision P7 > P8 > HDR10+ > HDR10 > SDR |
| **色深** | `color_depth` | ffprobe | 12-bit > 10-bit > 8-bit |
| **帧率** | `frame_rate` | ffprobe | 120fps > 60fps > 30fps ... |
| **字幕** | `subtitle` | ffprobe | PGS 简体中文 > ASS > SRT ... |
| **容器格式** | `file_extension` | 文件名 | mkv > mp4 > ts ... |
| **文件大小** | `file_size` | 文件属性 | 对数公式，越大越高（~50GB 饱和） |
| **码率** | `bitrate` | ffprobe | 对数公式，越高越好（~100Mbps 饱和） |

::: warning 注意
`color_depth`、`frame_rate`、`subtitle`、`bitrate` 维度需要开启 **ffprobe 探测** 才能获取数据。未开启时这些维度不参与评分。
:::

### YAML 格式

```yaml
quality:
  - name: 规则显示名称
    id: 唯一标识符
    scoring:
      resolution:
        enabled: true           # 是否启用此维度
        weight: 25              # 权重（越大影响越大）
        priority:               # 优先级列表（排名越前得分越高）
          - 2160p
          - 1080p
          - 720p
      file_size:
        enabled: true
        weight: 10              # file_size/bitrate 无 priority，使用对数公式
    exclude:
      resolutions: ["480p"]     # 排除这些分辨率的文件（不参与评分）
      codecs: ["MPEG-2", "Xvid"]
      hdr_types: []
```

### 片源 (source) 特殊格式

Remux 版本必须使用组合格式，不能单独写 `Remux`：

| 值 | 说明 |
|----|------|
| `Remux+UHD BluRay` | UHD 蓝光原盘提取 |
| `Remux+BluRay` | 蓝光原盘提取 |
| `Remux+BD` | BD 原盘提取 |
| `Unknown` | 无法识别来源的文件映射为此值 |

### 音频编码标准名称

评分规则中的音频编码名称必须与 ffprobe 输出一致：

| 名称 | 说明 |
|------|------|
| `DTS:X` | 对象式 DTS |
| `TrueHD Dolby Atmos 7.1` | TrueHD Atmos 7.1 声道 |
| `DTS-HD MA 7.1` / `DTS-HD MA 5.1` | DTS 无损（带声道） |
| `Dolby TrueHD` | TrueHD 无 Atmos |
| `DTS-HD` | DTS-HD（非 MA） |
| `Dolby Digital Plus Atmos` | EAC3 Atmos |
| `Dolby Digital Plus` | EAC3 |
| `Dolby Digital` | AC3 |

### 排除规则

排除规则可以在评分前直接过滤掉不需要的文件：

```yaml
exclude:
  resolutions:          # 排除这些分辨率
    - 480p
    - 360p
  codecs:               # 排除这些编码
    - MPEG-2
    - Xvid
    - DivX
  hdr_types: []         # 排除这些 HDR 类型（空 = 不排除）
```

### 内置评分方案

| 方案 | ID | 特点 |
|------|----|----|
| **均衡评分** | `balanced` | 不开 ffprobe，仅靠文件名 + 文件大小，速度最快 |
| **画质优先** | `quality-first` | 分辨率 + HDR + 码率权重最高，开启 ffprobe |
| **音质优先** | `audio-first` | 音频编码权重 35（最高），开启字幕评分 |
| **省空间** | `size-first` | 偏好 WEB-DL/H.265/AAC，文件大小负权重 |
| **Remux 收藏** | `remux-collector` | 片源权重最高，排除低清和过时编码 |
| **全维度探测** | `full-probe` | 开启所有 11 个维度，最全面但最慢 |
| **多版本均衡** | `multi-balanced` | 按分辨率+HDR 自动保留多版本 |
| **多版本精选** | `multi-curated` | 精选 4K DV + 4K HDR + 1080p 三个版本 |

### 多版本功能

多版本功能允许同一部影片保留多个不同规格的版本（如同时保留 4K HDR 和 1080p SDR）。

**Mode A：维度自动分槽**

```yaml
version:
  enabled: true
  dimensions: [resolution, hdr]  # 按这些维度组合自动创建槽位
  max_versions: 3                 # 最多保留 3 个版本
```

系统会自动根据 `dimensions` 的值组合创建槽位（如 2160p+DV、2160p+HDR10、1080p+SDR），每个槽位内按评分选最优。

**Mode B：显式定义槽位**

```yaml
version:
  enabled: true
  unmatched: skip              # 不匹配任何槽位的文件：skip(跳过) / best(选最佳)
  slots:
    - name: 4K DV
      match:
        resolution: [2160p]
        hdr: [Dolby Vision, Dolby Vision P7, Dolby Vision P8]
    - name: 4K HDR
      match:
        resolution: [2160p]
        hdr: [HDR10+, HDR10]
    - name: 1080p SDR
      match:
        resolution: [1080p]
        hdr: [SDR]
```

精确控制每个版本槽位的匹配条件。

---

## 自定义规则

除了上述三种整理规则外，MediaTidy 还提供自定义规则系统，用于在文件名识别阶段进行预处理和增强。所有自定义规则使用统一的 YAML 格式配置。

### YAML 结构总览

```yaml
# 渲染词（正则替换）- 识别前对文件名进行预处理
mr:
  - pattern: "正则表达式"
    replace: "替换内容"
  - pattern: "要删除的内容"
    action: remove

# 输出渲染词 - 对最终命名结果进行后处理
output_mr:
  - pattern: "正则表达式"
    replace: "替换内容"

# 季集映射 - 修正 TMDB 与实际季集号的差异
mp:
  - tmdb: 12345
    from: { season: 1, ep: [1, 24] }
    to:   { season: 2, ep: [1, 24] }

# 制作组识别
groups:
  - name: "FRDS"
    keywords: ["FRDS", "frds"]

# 流媒体服务识别
streaming:
  - name: "Netflix"
    keywords: ["Netflix", "NF"]

# 区域码识别
region:
  - name: "R1"
    keywords: ["R1", "Region A"]
```

### 渲染词（MR 规则）

渲染词是文件名识别前的正则预处理规则，按定义顺序依次执行。

#### 基本用法

```yaml
mr:
  # 删除文件名中的广告文字
  - pattern: "\\[www\\.example\\.com\\]"
    action: remove

  # 替换文字
  - pattern: "S(\\d+)EP"
    replace: "S$1E"

  # 带名称的规则（便于管理）
  - name: 修正剧集编号
    pattern: "第(\\d+)集"
    replace: "E$1"
```

#### TMDB 绑定

通过 `bind` 字段直接绑定 TMDB ID，跳过自动搜索：

```yaml
mr:
  - name: 某某动漫合集
    pattern: "某某动漫"
    bind:
      tmdb: 12345
      media: tv
      season: 1
      episode: "$1"   # 从正则捕获组提取集号
```

#### 输出渲染词（output_mr）

在命名完成后对最终文件名进行二次处理：

```yaml
output_mr:
  - pattern: "\\s{2,}"
    replace: " "
  - pattern: "\\.\\."
    replace: "."
```

#### 正则语法说明

- 默认**不区分大小写**
- 支持 Python 风格的 `(?<=...)` 后顾断言（自动转换为 Go RE2 兼容格式）
- 支持 `(?=...)` 前瞻断言
- 替换中使用 `$1`、`$2` 引用捕获组（也支持 `\1`、`\2`，自动转换）

### 季集映射（MP 规则）

当 TMDB 上的季集划分与实际文件不一致时使用。例如 TMDB 将动漫续作标为第 2 季，但文件名中集号从第 1 集开始。

```yaml
mp:
  # 将 TMDB ID 12345 的 S1E25-E48 映射到 S2E1-E24
  - tmdb: 12345
    from: { season: 1, ep: [25, 48] }
    to:   { season: 2, ep: [1, 24] }

  # 将全集编号(S1E1-E100)拆分到多季
  - tmdb: 67890
    from: { season: 1, ep: [1, 24] }
    to:   { season: 1, ep: [1, 24] }
  - tmdb: 67890
    from: { season: 1, ep: [25, 48] }
    to:   { season: 2, ep: [1, 24] }
```

映射支持以下集号表达式：
- **固定偏移**：`EP-24`（集号减 24）、`EP+1`（集号加 1）
- **原样保持**：`EP`（仅改变季号）
- **固定值**：直接指定目标集号范围

### 标签匹配（制作组 / 流媒体 / 区域码）

```yaml
# 简写方式：名称即关键词
groups:
  - "FRDS"
  - "CtrlHD"
  - "WIKI"

# 完整格式：一个名称对应多个匹配关键词
groups:
  - name: "FRDS"
    keywords: ["FRDS", "Frds"]

streaming:
  - name: "Netflix"
    keywords: ["Netflix", "NF", "NFLX"]
  - name: "Disney+"
    keywords: ["DSNP", "Disney+", "DisneyPlus"]

region:
  - name: "R1"
    keywords: ["R1", "Region A", "Region 1"]
```

关键词匹配规则：
- 不区分大小写
- 在文件名中匹配时会检查边界字符（方括号、下划线、空格、连字符等）

### 规则生效流程

```
mr 渲染词替换 → guessit 文件名解析 → TMDB 匹配 → 季集映射(MP) → 命名模板 → output_mr 输出渲染
```

### 规则管理

- **本地规则**：在「自定义规则」页面直接编辑 YAML
- **订阅源**：支持从远程 URL 同步规则，自动与本地规则合并
- **生效规则查看**：查看合并后实际生效的所有规则
- **单条禁用**：对生效规则列表中的单条规则进行启用/禁用切换
- **导出**：将当前生效规则导出为 YAML 文件

---

## 规则订阅

所有规则（分类、命名、质量、自定义规则）都支持从远程 URL 订阅。MediaTidy 使用**统一订阅管理**系统：

### 添加订阅

1. 进入「订阅管理」页面
2. 添加订阅源 URL 并选择规则类型
3. 系统定期自动同步（默认每 6 小时一次）
4. 也可手动触发同步

### 订阅类型

| 类型 | 说明 |
|------|------|
| `naming` | 命名规则订阅 |
| `quality` | 质量评分规则订阅 |
| `category` | 分类规则订阅 |
| `custom_rules` | 自定义规则订阅（渲染词 + 季集映射 + 标签） |

### 合并机制

- **命名/质量规则**：按规则的 `name` 字段合并，同名规则高优先级覆盖低优先级，不同名全部保留
- **分类规则**：整体替换，本地 > 订阅 > 内置
- **自定义规则**：本地和订阅的 YAML 按字段合并（`mr`、`mp`、`groups` 等分别拼接），本地规则排在前面（优先执行）

### GitHub Webhook 自动同步

支持配置 GitHub Webhook，当仓库推送更新时自动触发规则同步：

1. 在「订阅管理」中复制 Webhook URL
2. 在 GitHub 仓库 → Settings → Webhooks 添加
3. Content type 选 `application/json`
4. 填入 Secret（订阅管理页面可查看/重新生成）
5. 之后每次 `push` 事件都会自动同步最新规则
