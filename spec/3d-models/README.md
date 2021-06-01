# A GeoPackage extension for 3D Models

Extension name: ***`gpkgext-3dmodels`***

This is a draft extension for storing 3D models inside GeoPackages.
Rows are added to the `gpkgext_extensions` table to identify all tables set up for this extension.

For all of these tables, the `extension_name` is configured to be `ecere_3d_models`, the definition to be http://github.com/ecere/gpkgext-3dmodels, and the `scope` to be `read-write`.

The tables registered with this extension are:

- `gpkgext_3d_models` (only used for the reference points approach A)
- the individual tiles tables for batched 3D models (only used for approach B)
- `gpkgext_textures` for shared textures

## A) Referenced 3D models with placement information

In this approach, best suited for geo-typical models, a single table per GeoPackage (`gpkgext_3d_models`) is used to define one model per row, in a blob within the `model` field.
A `format` field allows to specify the format. Both glTF and E3D have been used in the experiments.
The `name` field allows to specify a name for the model, andt the `lod` field can optionally be used to distinguish between multiple level of details for the model, or left NULL if only a single version exists. The combination of `name`, `lod` and `format` must be unique.

The `model::id` field of the attributes table for the 3D models referencing points (which would also contain the point geometry in a non-tiled approach) references the `id` (primary key) of the `gpkgext_3d_models` table.
The attributes table may also contain additional fields for scaling and orienting the models:

- `model::scale`; or `model::scaleX`, `model::scaleY` and `model::scaleZ` for non-uniform scaling
- `model::yaw`, `model::pitch` and `model::roll`

These attributes duplicate the CDB fields `AO1`, `SCALx`, `SCALy`, `SCALz` (and in a sense `MODL` as well), but are intended to be defined in a non-CDB specific manner within a generic 3D Models extension for GeoPackage.

The following SQL is used to create the `gpkgext_3d_models` table:

```sql
CREATE TABLE gpkgext_3d_models (
   id INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL,
   name TEXT NOT NULL,
   lod INTEGER,
   format TEXT NOT NULL,
   model BLOB,
   CONSTRAINT unique_models UNIQUE(name, lod, format));
```

Example `gpkgext_3d_models`:

```
id  name               lod  format  model
--  -----------------  ---  ------  -----
1   coniferous_tree01       glb     glTF
2   palm_tree01             glb     glTF
```

Sample SQL table creation for attributes table referencing the 3D models:

```sql
CREATE TABLE attributes_Trees (
   id INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL,
   AO1 REAL, CNAM TEXT, RTAI INTEGER,
   SCALx REAL, SCALy REAL, SCALz REAL,
   AHGT TEXT, BBH REAL, BBL REAL, BBW REAL, BSR REAL,
   CMIX INTEGER, FACC TEXT, FSC INTEGER, HGT REAL,
   MODL TEXT,
   `model::id` INTEGER,
   `model::yaw` REAL,
   `model::scale` REAL)
```

See an [example GeoPackage](https://portal.ogc.org/files/?artifact_id=95351) using reference points as Mapbox Vector Tiles and glTF models

See an [example GeoPackage](https://portal.ogc.org/files/?artifact_id=95340) using reference points as GNOSIS Map Tiles and E3D models

## B) Batched 3D Models tiles

In this approach, best suited for geo-specific models, a single model covers a whole tile, batching all 3D models from the data layer found within that tile, and is stored in a tiles table much like raster or vector tiles (as a glTF blob in the `tile_data` field).
It is closer to the 3D Tiles / One World Terrain approach, and could potentially also combine both 3D Terrain and 3D Models (though ideally keeping them as distinct nodes within the model). Such an approach may facilitate transition between CDBX and OWT.

Because GeoPackage does not define a generic mechanism to specify the encoding of `tile_data` (it has previously been suggested that this would be a good field to add to the `gpkg_contents` table), the encoding of the 3D model must be deducted from the content of the blob. Fortunately, both glTF and E3D feature a header signature which facilitates this. The `3d-models` type is introduced to specify in the `data_type` column of the `gpkg_contents` table.

The translation origin of the model, as well as its orientation, is implied from the center of the tile (from the tile matrix / tile matrix set) for which it defined. The model is defined in the 3D cartesian space where (0, 0, 0) lies at that center, sitting directly on the WGS84 ellipsoid, and oriented so that by default it appears upright with its X axis pointing due East, its Z axis pointing North, and its Y axis pointing away from the center of the Earth. In other words, it would be equivalent to having a single point situated at the center of the tile in the referenced 3D points approach.

The height of the individual features (e.g. buildings) within the batched models tile models has already been adjusted match the elevation model. However, each separate feature from CDB is encoded in the model as a separate node to facilitate re-adjusting it to new elevation.

See an [example GeoPackage](https://portal.ogc.org/files/?artifact_id=95337) using batched 3D models encoded as glTF

See an [example GeoPackage](https://portal.ogc.org/files/?artifact_id=95334) using batched 3D models encoded as E3D

## Textures table

The textures table has the following fields:

- `id`: integer primary key
- `name`: The filename used in the model to refer to the texture
- `width`: width of the texture
- `height`: height of the texture
- `format`: e.g. "png"
- `texture`: blob containing the texture data

The combination of `name`, `width`, `height` and `format` must be unique.

The following SQL statement is used to create the table:

```sql
CREATE TABLE gpkgext_textures (
   id INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL,
   name TEXT NOT NULL,
   width INTEGER NOT NULL,
   height INTEGER NOT NULL,
   format TEXT NOT NULL,
   texture BLOB,
   CONSTRAINT unique_textures UNIQUE(name, width, height, format));
```

```
id  name   width  height  format  texture
--  -----  -----  ------  ------  -------
1   1.png  512    512     png     �PNG
2   1.png  256    256     png     �PNG
3   2.png  512    512     png     �PNG
4   2.png  256    256     png     �PNG
```

See also the [CDB X Discussion Paper](https://github.com/sofwerx/cdb2-eng-report/blob/master/11-tiling-coverages.adoc#tiled-3d-models) describing the experiments in which this extension was developed, along with the resulting sample GeoPackages.
