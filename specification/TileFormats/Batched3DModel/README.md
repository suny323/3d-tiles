# Batched 3D Model

## Contributors

_This section is non-normative_

* Patrick Cozzi, [@pjcozzi](https://twitter.com/pjcozzi)
* Tom Fili, [@CesiumFili](https://twitter.com/CesiumFili)
* Sean Lilley, [@lilleyse](https://github.com/lilleyse)

## Contents

* [Overview](#overview)
* [Layout](#layout)
    * [Padding](#padding)
* [Header](#header)
* [Feature Table](#feature-table)
	* [Semantics](#semantics)
		* [Feature semantics](#feature-semantics)
		* [Global semantics](#global-semantics)
* [Batch Table](#batch-table)
* [Binary glTF](#binary-gltf)
   * [Coordinate system](#coordinate-system)
* [File extension and MIME type](#file-extension-and-mime-type)
* [Implementation example](#implementation-example)

## Overview

_Batched 3D Model_ allows offline batching of heterogeneous 3D models, such as different buildings in a city, for efficient streaming to a web client for rendering and interaction.  Efficiency comes from transferring multiple models in a single request and rendering them in the least number of WebGL draw calls necessary.  Using the core 3D Tiles spec language, each model is a _feature_.

Per-model properties, such as IDs, enable individual models to be identified and updated at runtime, e.g., show/hide, highlight color, etc. Properties may be used, for example, to query a web service to access metadata, such as passing a building's ID to get its address. Or a property might be referenced on the fly for changing a model's appearance, e.g., changing highlight color based on a property value.

A Batched 3D Model tile is a binary blob in little endian.

## Layout

A tile is composed of two sections: a header immediately followed by a body. The following figure shows the Batched 3D Model layout (dashes indicate optional fields):

![](figures/layout.png)

### Padding

A tile's `byteLength` must be aligned to an 8-byte boundary. The contained [Feature Table](../FeatureTable/README.md#padding) and [Batch Table](../BatchTable/README.md#padding) must conform to their respective padding requirement.

The [binary glTF](#binary-gltf) must start and end on an 8-byte alignment so that glTF's byte-alignment guarantees are met. This can be done by padding the Feature Table or Batch Table if they are present.

## Header

The 28-byte header contains the following fields:

|Field name|Data type|Description|
|----------|---------|-----------|
| `magic` | 4-byte ANSI string | `"b3dm"`.  This can be used to identify the content as a Batched 3D Model tile. |
| `version` | `uint32` | The version of the Batched 3D Model format. It is currently `1`. |
| `byteLength` | `uint32` | The length of the entire tile, including the header, in bytes. |
| `featureTableJSONByteLength` | `uint32` | The length of the Feature Table JSON section in bytes. |
| `featureTableBinaryByteLength` | `uint32` | The length of the Feature Table binary section in bytes. |
| `batchTableJSONByteLength` | `uint32` | The length of the Batch Table JSON section in bytes. Zero indicates there is no Batch Table. |
| `batchTableBinaryByteLength` | `uint32` | The length of the Batch Table binary section in bytes. If `batchTableJSONByteLength` is zero, this will also be zero. |

The body section immediately follows the header section, and is composed of three fields: `Feature Table`, `Batch Table`, and `Binary glTF`.

## Feature Table

Contains values for `b3dm` semantics.
More information is available in the [Feature Table specification](../FeatureTable/README.md).

The `b3dm` Feature Table JSON schema is defined in [b3dm.featureTable.schema.json](../../schema/b3dm.featureTable.schema.json).

### Semantics

#### Feature semantics

There are currently no per-feature semantics.

#### Global semantics

These semantics define global properties for all features.

| Semantic | Data Type | Description | Required |
| --- | --- | --- | --- |
| `BATCH_LENGTH` | `uint32` | The number of distinguishable models, also called features, in the batch. If the Binary glTF does not have a `batchId` attribute, this field _must_ be `0`. | :white_check_mark: Yes. |
| `RTC_CENTER` | `float32[3]` | A 3-component array of numbers defining the center position when positions are defined relative-to-center, (see [Coordinate system](#coordinate-system)). | :red_circle: No. |

## Batch Table

The _Batch Table_ contains per-model application-specific metadata, indexable by `batchId`, that can be used for [declarative styling](../../Styling/README.md) and application-specific use cases such as populating a UI or issuing a REST API request.  In the binary glTF section, each vertex has an numeric `batchId` attribute in the integer range `[0, number of models in the batch - 1]`.  The `batchId` indicates the model to which the vertex belongs.  This allows models to be batched together and still be identifiable.

See the [Batch Table](../BatchTable/README.md) reference for more information.

## Binary glTF

Batched 3D Model uses [glTF 2.0](https://github.com/KhronosGroup/glTF/tree/master/specification/2.0) to embed model data.

The [binary glTF](https://github.com/KhronosGroup/glTF/tree/master/specification/2.0#binary-gltf-layout) immediately follows the Feature Table and Batch Table.  It may embed all of its geometry, texture, and animations, or it may refer to external sources for some or all of these data.

As described above, each vertex has a `batchId` attribute indicating the model to which it belongs.  For example, vertices for a batch with three models may look like this:
```
batchId:  [0,   0,   0,   ..., 1,   1,   1,   ..., 2,   2,   2,   ...]
position: [xyz, xyz, xyz, ..., xyz, xyz, xyz, ..., xyz, xyz, xyz, ...]
normal:   [xyz, xyz, xyz, ..., xyz, xyz, xyz, ..., xyz, xyz, xyz, ...]
```
Vertices do not need to be ordered by `batchId`, so the following is also OK:
```
batchId:  [0,   1,   2,   ..., 2,   1,   0,   ..., 1,   2,   0,   ...]
position: [xyz, xyz, xyz, ..., xyz, xyz, xyz, ..., xyz, xyz, xyz, ...]
normal:   [xyz, xyz, xyz, ..., xyz, xyz, xyz, ..., xyz, xyz, xyz, ...]
```
Note that a vertex can't belong to more than one model; in that case, the vertex needs to be duplicated so the `batchId`s can be assigned.

The `batchId` parameter is specified in a glTF mesh [primitive](https://github.com/KhronosGroup/glTF/tree/master/specification/2.0#reference-primitive) by providing the `_BATCHID` attribute semantic, along with the index of the `batchId` [accessor](https://github.com/KhronosGroup/glTF/tree/master/specification/2.0#accessors). For example, 

```JSON
"primitives": [
    {
        "attributes": {
            "_BATCHID": 0
        }
    }
]
```

```JSON
{
    "accessors": [
        {
            "bufferView": 1,
            "byteOffset": 0,
            "componentType": 5125,
            "count": 4860,
            "max": [2],
            "min": [0],
            "type": "SCALAR"
        }
    ]
}
```

The `accessor.type` must be a value of `"SCALAR"`. All other properties must conform to the glTF schema, but have no additional requirements.

When a Batch Table is present or the `BATCH_LENGTH` property is greater than `0`, the `_BATCHID` attribute is required; otherwise, it is not.

### Coordinate system

By default embedded glTFs use a right handed coordinate system where the _y_-axis is up. For consistency with the _z_-up coordinate system of 3D Tiles, glTFs must be transformed at runtime. See [coordinate reference system](../../README.md#gltf) for more details.

Vertex positions may be defined relative-to-center for high-precision rendering, see [Precisions, Precisions](http://help.agi.com/AGIComponents/html/BlogPrecisionsPrecisions.htm). If defined, `RTC_CENTER` specifies the center position that all vertex positions are relative to after the coordinate system transform and glTF node hierarchy transforms have been applied.

## File extension and MIME type

Batched 3D Model tiles use the `.b3dm` extension and `application/octet-stream` MIME type.

An explicit file extension is optional. Valid implementations may ignore it and identify a content's format by the `magic` field in its header.

## Implementation example

_This section is non-normative_

Code for reading the header can be found in
[`Batched3DModelTileContent.js`](https://github.com/AnalyticalGraphicsInc/cesium/blob/master/Source/Scene/Batched3DModel3DTileContent.js)
in the Cesium implementation of 3D Tiles.