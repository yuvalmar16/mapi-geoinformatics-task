# Raster Analysis (Q1–3) and E-mail Data Extraction (Q4)

## Contents
| File / folder | Description |
|---|---|
| `raster_analysis.ipynb` | Q1–3 notebook: reads every raster in `רסטרים לניתוח/`, extracts its metadata, shows previews, classifies the file type, builds the summary table, histograms and defect detection |
| `raster_analysis_table.csv` | Q1 table (written by the notebook) |
| `raster_defects.csv` | Q3 valid/invalid classification with defect type (written by the notebook) |
| `email_extraction.ipynb` | Q4 notebook, all the code in one place: tools, extraction rules, the code (reading the .msg/.eml files, extracting the fields, building the HTML page), extracted data, HTML display |
| `email_tasks/emails.html` | Q4 output: the extracted e-mail data as HTML (self-contained, opens in any browser) |

**The input data is not included in this repository**: the rasters (`רסטרים לניתוח/`, 28 files, about 7.6 GB) and the e-mails (`מיילים/`, 5 .msg files). To re-run, place those two folders, with these exact names, in the repository root.

## Tools and libraries
All runs were done with **Python 3.14.0** on Windows 11.

| Library / tool | Version used | Used in | Purpose |
|---|---|---|---|
| **rasterio** (GDAL bindings; bundled GDAL **3.12.1** with the OpenJPEG driver for JPEG 2000) | 1.5.0 | Q1–3 | Opening GeoTIFF / JP2 files; reading metadata (bands, dtype, geotransform, CRS, NoData, units, ColorInterp) and pixel previews from the overviews |
| **numpy** | 2.3.5 | Q1–3 | Data type → pixel type / bit depth, array statistics, stretching, masks |
| **pandas** | 2.3.3 | Q1–4 | Result tables and CSV export |
| **matplotlib** | 3.10.8 | Q1–3 | Previews, histograms, defect overlays; `matplotlib.colors.rgb_to_hsv` for the saturation-based cloud test |
| **scipy** (`scipy.ndimage`) | 1.17.0 | Q3 | Connected-component labelling and morphological opening of defect blobs |
| **extract-msg** | 0.58.2 | Q4 | Reading Outlook `.msg` files: headers, plain/HTML body, attachments, Content-IDs (installs **olefile** 0.47) |
| **beautifulsoup4** | 4.15.0 | Q4 | Parsing HTML bodies: finding `<img>` tags inside links, to tell signature logos from real attachments |
| **Pillow** | 12.1.0 | Q4 | Image size (signature-banner test) and small previews embedded in the HTML as base64 |
| **IPython** | 9.9.0 | Q4 | Showing the generated HTML page inside the notebook |
| **Jupyter**: ipykernel / nbconvert / nbclient / nbformat | 7.1.0 / 7.17.1 / 0.11.0 / 5.10.4 | all | Running the notebooks (`jupyter nbconvert --execute`) |
| Python standard library: `math`, `pathlib`, `sys`, `re`, `email`, `html`, `hashlib`, `base64`, `io`, `dataclasses`, `datetime` | – | all | Degree→metre conversion on the WGS84 ellipsoid, file handling, regex field extraction, `.eml` reading, HTML escaping, SHA-1 of signature images, embedding images |

Install:
```
pip install rasterio numpy pandas matplotlib scipy extract-msg beautifulsoup4 pillow jupyter
```

## How to run
First copy the input folders `רסטרים לניתוח/` and `מיילים/` into the repository root (they are not in the repository).

* **Q1–3:** open `raster_analysis.ipynb` (next to the folder `רסטרים לניתוח`) and run all cells. It writes `raster_analysis_table.csv` and `raster_defects.csv`.
* **Q4:** open `email_extraction.ipynb` and run all cells. It writes `email_tasks/emails.html`.

Both notebooks were re-run end to end with the input data, with no errors (the saved outputs are from that run):
`jupyter nbconvert --to notebook --execute --inplace raster_analysis.ipynb email_extraction.ipynb`

## Question 1 – Raster metadata and file type
### Method
* **Bands, pixel type, pixel depth:** `src.count`, `src.dtypes` (uint8 → 8 bit unsigned integer, uint16 → 16 bit, float32 → 32 bit floating point).
* **Resolution:** pixel size from the geotransform. For geographic CRSs (degrees) it is converted to metres using the WGS84 meridional / prime-vertical radii at the raster's centre latitude.
* **Coordinate system:** `src.crs` (EPSG code and name; an undefined `LOCAL_CS` is flagged; a file whose geotransform is the identity matrix is reported as not georeferenced, even if it carries an empty `LOCAL_CS`).
* **File type**, decided by these checks:
  1. *Georeferencing & alignment (CRS + GeoTransform):* a standard CRS, a realistic square pixel → orthophoto / elevation. A placeholder georeference (pixel < 1 cm) → raw aerial photo. Frame naming `<x>_L_<strip>_F_<frame>` + non-square pixels → aerial frame with an approximate georeference. No geotransform at all → not georeferenced (resolution and CRS reported as unknown); with a frame name or camera EXIF tags it is an aerial photo, otherwise an image.
  2. *Data type:* a single band of float32, or of an integer wider than 8 bit / with units = metre, = height (elevation model; BSM_2222 stores whole metres as int32). A single uint8 band = grayscale image. uint16 = raw sensor (HDR_2580's values never exceed 255 → 8-bit data in a 16-bit container).
  3. *Band meaning (ColorInterp + values):* Gray + metre + NoData = DEM; RGB = brightness DN; the 4th band tested as Alpha (only 0/255) vs NIR (continuous, dark over water, high NDVI).

### Result (28 files: 12 orthophotos, 8 aerial photos, 8 elevation models)
| File | Type | Bands | Resolution (m) | Pixel type | Pixel depth | Coordinate system |
|---|---|---|---|---|---|---|
| A_L_013_F_0006.jp2 | Aerial photo (frame, approx. georeferenced) | 3 | 0.164 × 0.226 | Unsigned integer | 8 bit | EPSG:2039 Israel TM Grid |
| A_L_013_F_0010.jp2 | Aerial photo (frame, approx. georeferenced) | 3 | 0.200 × 0.313 | Unsigned integer | 8 bit | EPSG:2039 Israel TM Grid |
| AFQ_2617.tif | Orthophoto | 4 | 0.2 | Unsigned integer | 8 bit | EPSG:2039 Israel TM Grid |
| ARD_2147.jp2 | Orthophoto (grayscale) | 1 | 0.2 | Unsigned integer | 8 bit | EPSG:2039 Israel TM Grid |
| ARL_1827.jp2 | Orthophoto | 4 | 0.2 | Unsigned integer | 8 bit | EPSG:2039 Israel TM Grid |
| BSM_1111.tif | Elevation (DSM/DTM) | 1 | 0.5 | Floating point | 32 bit | EPSG:2039 Israel TM Grid |
| BSM_2222.tif | Elevation (DSM/DTM) | 1 | 0.5 | Signed integer | 32 bit | EPSG:2039 Israel TM Grid |
| ELT_1110.jp2 | Orthophoto | 3 | 0.044 × 0.050 | Unsigned integer | 8 bit | EPSG:4326 WGS 84 |
| G_L_030_F_0137.jp2 | Aerial photo (raw frame) | 3 | not meaningful (nominal ≈ 0.002) | Unsigned integer | 8 bit | EPSG:4326 WGS 84 (nominal only) |
| G_L_030_F_0139.jp2 | Aerial photo (raw frame) | 3 | not meaningful (nominal ≈ 0.002) | Unsigned integer | 8 bit | EPSG:4326 WGS 84 (nominal only) |
| G_L_034_F_0067.jp2 | Aerial photo (raw frame) | 3 | not meaningful (nominal ≈ 0.002) | Unsigned integer | 8 bit | EPSG:4326 WGS 84 (nominal only) |
| G_L_034_F_0070.jp2 | Aerial photo (raw frame) | 3 | not meaningful (nominal ≈ 0.002) | Unsigned integer | 8 bit | EPSG:4326 WGS 84 (nominal only) |
| HDR_1234.tif | Elevation (DSM/DTM), not georeferenced | 1 | unknown | Floating point | 32 bit | none (empty LOCAL_CS, no geotransform) |
| HDR_2580.tif | Orthophoto | 3 | 0.4 | Unsigned integer | 16 bit | EPSG:2039 Israel TM Grid |
| HEF_1000.tif | Elevation (DSM/DTM) | 1 | 0.4 | Floating point | 32 bit | EPSG:2039 Israel TM Grid |
| HEF_2000.tif | Elevation (DSM/DTM) | 1 | 0.4 | Floating point | 32 bit | EPSG:2039 Israel TM Grid |
| KSB_1626.jp2 | Orthophoto | 4 (RGB + NIR) | 0.2 | Unsigned integer | 8 bit | EPSG:2039 Israel TM Grid |
| Nadir_L_018_F_0044.jpg | Aerial photo (raw frame, not georeferenced) | 3 | unknown | Unsigned integer | 8 bit | none |
| Nadir_L_138_F_0102.jpg | Aerial photo (raw frame, not georeferenced) | 3 | unknown | Unsigned integer | 8 bit | none |
| NTN_0912.tif | Elevation (DSM/DTM) | 1 | 0.15 | Floating point | 32 bit | EPSG:2039 Israel TM Grid |
| NZR_0408.jp2 | Orthophoto | 3 | 0.05 | Unsigned integer | 8 bit | EPSG:2039 Israel TM Grid |
| QGT_0408.tif | Elevation (DSM/DTM), not georeferenced | 1 | unknown | Floating point | 32 bit | none (empty LOCAL_CS, no geotransform) |
| QLY_2437.jp2 | Orthophoto | 4 | 0.170 × 0.199 (nominal 0.2) | Unsigned integer | 8 bit | EPSG:4326 WGS 84 |
| RSH_0932.jp2 | Orthophoto | 4 | 0.2 | Unsigned integer | 8 bit | EPSG:2039 Israel TM Grid |
| SHF_0117.tif | Elevation (DSM/DTM) | 1 | 0.15 | Floating point | 32 bit | EPSG:2039 Israel TM Grid |
| TLY_1008.jp2 | Orthophoto | 3 | 0.05 | Unsigned integer | 8 bit | EPSG:2039 Israel TM Grid |
| TLY_1226.jp2 | Orthophoto | 4 (RGB + NIR) | 0.169 × 0.199 (nominal 0.2) | Unsigned integer | 8 bit | LOCAL_CS (undefined in the file); the coordinates are geographic lon/lat degrees ≈ WGS84 |
| YRS_2137.jp2 | Orthophoto | 4 | 0.170 × 0.199 (nominal 0.2) | Unsigned integer | 8 bit | EPSG:4326 WGS 84 |

## Question 2 – Histograms
Per-band histograms (0–255 for imagery, 256 bins min–max for DEMs, NoData excluded). A file is flagged as **non-uniform** if any band has: an empty band (std = 0), a narrow histogram (entropy < 6 bit or the 1–99 % range < 40 % of 0–255), a dominant spike (> 5 % of pixels in one value) or clipping (≥ 1 % at 0/255).

| File | Uniform? | Reason |
|---|---|---|
| G_L_030_F_0137/0139, G_L_034_F_0067/0070 | No | Narrow, low-contrast peaks: homogeneous desert, haze, raw (unstretched) aerial photo |
| AFQ_2617, A_L_013_F_0006/0010, ELT_1110, NZR_0408, QLY_2437, RSH_0932, TLY_1008 | No | Spike at 255 in all bands (7–66 % of pixels): white fill outside the captured footprint. ELT, NZR and RSH are also narrow (little real content; RSH is only sea) |
| HDR_2580 | No | Empty red band, 19 % spike at 0: missing band, shadows clipped to black, NoData polygon |
| TLY_1226 | No | Sharp spikes from the large homogeneous sea area (bimodal sea/city) |
| KSB_1626 | No | Spike at 255 in band 3: saturation of the brightest roofs |
| ARD_2147, ARL_1827, YRS_2137, Nadir frames | Yes | Well spread (ARL is spread per band, but the bands are offset: colour cast, see Q3) |
| All 8 elevation models | Yes | Elevation values spread over the terrain range (NoData excluded) |

## Question 3 – Defects
Rules: missing data (NoData ≥ 1 %, an empty band, a black block ≥ 0.2 %, or **white fill**: a block of pixels at 255 in all colour bands touching the image edge, ≥ 0.2 %); dark (all-band 0 ≥ 5 %); burnt (pixels clipped white in all RGB bands, excluding white fill, ≥ 1 %, or a connected clipped block ≥ 0.2 %; single-band clipping on scattered bright pixels is reported, not a defect); clouds / cloud shadow (low-saturation HSV blob ≥ 0.5 % of the image, fill pixels excluded; needs 3 colour bands). DEMs are tested only for missing data.

**14 invalid, 14 valid.**

| File | Status | Defect type |
|---|---|---|
| A_L_013_F_0006.jp2 | Invalid | Missing data (white fill 13 %: border around the rotated frame) |
| A_L_013_F_0010.jp2 | Invalid | Missing data (white fill 7 %: border around the rotated frame) |
| AFQ_2617.tif | Invalid | Missing data (white fill 9 %, east strip) |
| ARD_2147.jp2 | Valid | – (0.4 % at 255 on sunlit slopes, scattered) |
| ARL_1827.jp2 | Valid* | – *visual remark: strong green colour cast (band means R 77 / G 135 / B 87); not covered by the automatic tests, needs colour correction |
| BSM_1111.tif | Valid | – |
| BSM_2222.tif | Valid | – (integer heights, whole metres) |
| ELT_1110.jp2 | Invalid | Missing data (white fill 66 %) |
| G_L_030_F_0137.jp2 | Invalid | Clouds / cloud shadow |
| G_L_030_F_0139.jp2 | Invalid | Clouds / cloud shadow |
| G_L_034_F_0067.jp2 | Invalid | Clouds / cloud shadow |
| G_L_034_F_0070.jp2 | Valid | – (largest cloud-like blob 0.1 %) |
| HDR_1234.tif | Valid* | – *no georeference (see Q1) |
| HDR_2580.tif | Invalid | Missing data (empty red band, black NoData block); dark areas |
| HEF_1000.tif | Valid | – |
| HEF_2000.tif | Valid | – |
| KSB_1626.jp2 | Valid | – (band 3 reaches 255 on ~1.5 % scattered bright roofs; only 0.05 % clipped in all RGB bands, not a defect) |
| Nadir_L_018_F_0044.jpg | Valid | – |
| Nadir_L_138_F_0102.jpg | Valid | – |
| NTN_0912.tif | Invalid | Missing data (NoData area, 77 %) |
| NZR_0408.jp2 | Invalid | Missing data (white fill 65 %) |
| QGT_0408.tif | Valid* | – *no georeference (see Q1) |
| QLY_2437.jp2 | Invalid | Missing data (white fill 18 %, stepped edge along the coast) |
| RSH_0932.jp2 | Invalid | Missing data (white fill 41 %; the rest is open sea) |
| SHF_0117.tif | Invalid | Missing data (NoData area, 21 %) |
| TLY_1008.jp2 | Invalid | Missing data (white fill 17 %, north edge) |
| TLY_1226.jp2 | Valid | – |
| YRS_2137.jp2 | Valid | – |

## Question 4 – Extract the data from the e-mails and display it as HTML
**Pipeline:** `מיילים/*.msg` → read (`extract-msg`; stdlib `email` for .eml) → rule-based extraction (`re`, `beautifulsoup4`, `Pillow`) → one self-contained HTML page (`email_tasks/emails.html`).

**Extracted fields:** request date, subject, sender / recipient, addressed-to (greeting), requester (sign-off), requested information, structured fields (key: value lines, ITM coordinates, bounding box, years, resolution, scale, buffer, notes, network path), client / organisation, area, type (הצעת מחיר / הזמנה / קליטת חומר מרשות / בקשת מידע), product, attached file (real attachments are separated from signature images).

**The page:** a summary table of all the e-mails, a card per e-mail (requested information, extracted fields, attached file, e-mail details), and at the bottom the 5 original messages as they are in the files.

**Findings in the mail folder:** 5 files → 5 different e-mails (each file name matches its subject). All mails were forwarded with the original headers stripped, so requesters come from the sign-offs (תמר, גיא). In the RTA and Tira mails the signature is only an image; no OCR engine was used, so the names (אפי ג'יאן, לימור דור) were read from the banners, stored in `SIGNATURE_IMAGES` (notebook section 3.1) by the image's SHA-1, and are marked on the page as read from the signature image. The Tira mail reports material received from the municipality and gives its network path.
