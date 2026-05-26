# Vessel Branch Feature Extraction in Voreen

## Overview

The main implementation is in the `vesselnetworkanalysis` module. Its input is
a segmented vessel volume. Foreground voxels represent vessels. Background
voxels represent surrounding tissue or empty space.

Voreen does not measure branches from a skeleton alone. It uses three related
representations:

| Prerequisite | How it is produced | Why it is needed |
| --- | --- | --- |
| Vessel volume | `VesselGraphCreator` binarizes the input segmentation into a temporary `LZ4SliceVolume<uint8_t>`. An optional sample mask can limit the valid region. | It preserves vessel thickness and surface position. Radius is measured from it. |
| Vessel skeleton | A `VolumeMask` is made from the binary vessel volume. `VolumeMask::skeletonize(...)` thins it to a one-voxel-wide centerline. | It supplies branch paths. Length and curveness use these paths. |
| Vessel graph | Skeleton voxels are classified and grouped into a `ProtoVesselGraph`. Original vessel voxels are then assigned to proto-graph edges. This creates the final `VesselGraph`. | It supplies edges as vessel branches and nodes as endpoints or junctions. Node degree is taken from graph connectivity. |

The practical pipeline is:

```text
segmented vessel volume
  -> binary vessel mask
  -> skeleton centerline
  -> proto-graph from skeleton topology
  -> label segmentation voxels by nearest branch
  -> final VesselGraph with branch measurements
```

### How each requested feature uses the prerequisites

| Feature | Object measured | Prerequisites used | Meaning in this code |
| --- | --- | --- | --- |
| Length | Graph edge / vessel branch | Skeleton path and node positions | Sum of world-space distances from the first node, along the smoothed centerline voxels, to the second node. |
| Radius | Graph edge / vessel branch | Skeleton path and labeled vessel volume surface | Surface voxels assigned to an edge are measured against their nearest centerline voxel. The edge stores mean and standard deviation summaries of minimum, average, and maximum surface distance. |
| Curvature | Graph edge / vessel branch | Length and the two graph node positions | The implementation calls this `curveness`: `branch length / straight endpoint distance`. It is a tortuosity measure, not pointwise geometric curvature. |
| Node degree | Graph node | Vessel graph topology | Number of graph edges attached to the node. |

Lengths and radii are in world-space coordinates because the code transforms
voxel positions with the volume's voxel-to-world matrix. This includes input
voxel spacing.

## Step-By-Step Workflow

### 1. Read and binarize the vessel volume

`VesselGraphCreator::prepareComputeInput()` reads the segmented volume. It
also reads an optional sample mask and optional fixed foreground points.

`VesselGraphCreatorProcessedInput` calls `binarizeVolume(...)`. This converts
the supplied volume to an on-disk binary volume. Voxels above the selected
threshold are treated as vessel foreground.

The graph extractor therefore expects a segmentation, or an input that can be
thresholded as a segmentation. It does not extract branch features directly
from an unsegmented intensity image.

### 2. Create a mask and skeletonize it

`createInitialVesselGraph(...)` creates a `VolumeMask` from the binary
segmentation. Fixed foreground points can be marked before thinning.

It then calls:

```cpp
mask.skeletonize<VolumeMask::IMPROVED>(...);
```

`VolumeMask::skeletonize(...)` repeatedly removes removable surface voxels.
The result is a thin centerline that retains connected vessel paths.

### 3. Turn the skeleton into graph topology

`createGraphFromMask(...)` passes the skeleton to
`NeighborCountVoxelClassifier`. For each centerline voxel,
`SkeletonClassReader::getClass(...)` examines its 26-neighborhood:

| Skeleton neighbors | Voxel class | Graph role |
| --- | --- | --- |
| 0 or 1 | End | Becomes an endpoint node. |
| 2 | Regular | Becomes part of an edge path. |
| More than 2 | Branch | Becomes part of a junction node. |

Connected voxels of the same class are grouped by the streaming connected
component code. `MetaDataCollector::createProtoVesselGraph(...)` then creates:

- proto-nodes from endpoint components and junction components
- proto-edges from sequences of regular centerline voxels

The proto-edge centerline positions are converted to world space and smoothed
in `ProtoVesselGraphEdge::ProtoVesselGraphEdge(...)`. The smoothing uses
neighboring control points before later path measurements are made.

### 4. Associate thick vessel voxels with branches

A skeleton provides centerlines and connectivity, but it does not preserve
branch width. Voreen therefore maps the binary segmentation back to the
proto-graph.

`createGraphFromMask(...)` runs these operations:

1. `createClosestIDVolume(...)` assigns foreground vessel voxels near a
   proto-edge to that edge.
2. `createCCAVolume(...)` separates connected labeled regions.
3. `mapEdgeIds(...)` changes temporary component labels to proto-edge IDs.
4. `collectUnfinishedRegions(...)` finds unassigned foreground regions.
5. `UnfinishedRegions::floodAllRegions(...)` fills those remaining regions
   from nearby assigned regions.

The resulting volume holds a branch ID for each assigned vessel voxel. It is
the link between centerline branches and the original thick vessel geometry.

### 5. Measure centerline voxels against the vessel surface

`ProtoVesselGraph::createVesselGraph(...)` creates one list of
`VesselSkeletonVoxel` values for every proto-edge. Each item represents a
smoothed centerline position.

It scans vessel voxels in the branch-ID volume. For each assigned vessel
voxel, it finds the nearest centerline voxel of that edge. It also identifies
surface voxels: a vessel voxel is on the surface when at least one of its six
axis-aligned neighbors is background.

For every surface voxel assigned to a centerline voxel, it records the surface
distance:

```text
distance(centerline point, surface voxel center)
  + half of the average voxel spacing
```

Each `VesselSkeletonVoxel` accumulates:

- `minDistToSurface_`
- `avgDistToSurface_`
- `maxDistToSurface_`
- `numSurfaceVoxels_`
- assigned voxel `volume_`

These surface distances are the local radius observations used for the branch
radius statistics.

The same scan estimates a radius for a graph junction node. A surface voxel
near two edges can belong to their shared node region. Its distance to the
shared node position can raise that node's stored `radius_`.

### 6. Create final nodes and edges

At the end of `ProtoVesselGraph::createVesselGraph(...)`,
`VesselGraphBuilder` inserts final `VesselGraphNode` objects and
`VesselGraphEdge` objects.

A final edge contains:

- two endpoint or junction nodes
- an ordered path of `VesselSkeletonVoxel` centerline samples
- properties calculated from that path

`VesselGraphEdge::finalizeConstruction()` calls
`VesselGraphEdgePathProperties::fromPath(...)` to calculate the edge
measurements.

### 7. Extract length

`VesselGraphEdgePathProperties::fromPath(...)` calculates `length_`.

For a normal edge, it adds:

1. distance from node 1 to the first centerline sample
2. distance between every neighboring pair of centerline samples
3. distance from the last centerline sample to node 2

For an edge with no path samples, it uses the direct distance between the two
nodes.

The public access method is `VesselGraphEdge::getLength()`.

### 8. Extract radius

`VesselGraphEdgePathProperties::fromPath(...)` summarizes radius observations
along an edge. Only centerline samples with valid surface data contribute.

The branch exposes three forms of radius:

| Public access method | What it summarizes |
| --- | --- |
| `getMinRadiusAvg()` | Average of each centerline sample's nearest surface distance. |
| `getAvgRadiusAvg()` | Average of each centerline sample's mean surface distance. |
| `getMaxRadiusAvg()` | Average of each centerline sample's farthest surface distance. |
| `getMaxRadiusMax()` | Largest farthest-surface distance seen anywhere on the branch. |

The matching standard deviation getters report variation along the branch.
For a general branch-radius summary, the code and CSV export expose
`getAvgRadiusAvg()` as the average-radius value.

Nodes also store a separate junction radius through
`VesselGraphNode::getRadius()`. That value is not the same as an edge's
radius statistics.

### 9. Extract curvature as curveness

The feature named closest to curvature in this module is:

```cpp
VesselGraphEdge::getCurveness()
```

It calculates:

```text
curveness = edge length / straight distance between its two nodes
```

A straight branch is near `1`. A winding branch is greater than `1`. If both
ends have the same position, the method returns infinity.

This is a whole-branch bending or tortuosity score. The extractor does not
calculate a curvature value at each centerline sample.

### 10. Extract node degree

Whenever `VesselGraphBuilder::insertEdge(...)` connects an edge, that edge ID
is stored with both incident nodes.

`VesselGraphNode::getDegree()` returns `edges_.size()`. In other words:

- degree `1` normally means a vessel endpoint
- degree `2` means a node connected through two edges
- degree `3` or more represents a junction

A node touching the sample boundary is tracked separately with
`isAtSampleBorder_`. This prevents a cropped branch end from automatically
being treated as a biological end node.

### 11. Optional refinement and output

`VesselGraphCreator::compute(...)` can run refinement iterations after the
first graph is built. `refineVesselGraph(...)` removes weak ending branches,
rebuilds a mask, skeletonizes again, and extracts all features again.

`VesselGraphGlobalStats::exportToFile(...)` demonstrates the feature output.
Its edge CSV writes `length`, `curveness`, radius statistics, and two endpoint
degrees. Despite their column names, `node1_degree` receives the smaller
endpoint degree and `node2_degree` receives the larger endpoint degree. Its
node CSV writes each node's `degree`.

## Exact Functions And Classes

Paths below are relative to the inner Voreen source directory, `voreen/`.

| Function / class | File | Responsibility |
| --- | --- | --- |
| `VesselGraphCreator` | `modules/vesselnetworkanalysis/processors/vesselgraphcreator.h` | Processor that drives graph and feature extraction from a segmented volume. |
| `VesselGraphCreator::prepareComputeInput()` | `modules/vesselnetworkanalysis/processors/vesselgraphcreator.cpp` | Reads volume inputs, sample mask, points, and threshold settings. |
| `VesselGraphCreatorProcessedInput` | `modules/vesselnetworkanalysis/processors/vesselgraphcreator.cpp` | Binarizes the segmentation and optional sample mask. |
| `createInitialVesselGraph(...)` | `modules/vesselnetworkanalysis/processors/vesselgraphcreator.cpp` | Creates the initial mask, skeleton, and graph. |
| `createGraphFromMask(...)` | `modules/vesselnetworkanalysis/processors/vesselgraphcreator.cpp` | Builds a proto-graph, assigns segmentation voxels to edges, and builds the final graph. |
| `createClosestIDVolume(...)` | `modules/vesselnetworkanalysis/processors/vesselgraphcreator.cpp` | Starts assigning segmented vessel voxels to their nearest proto-edge. |
| `createCCAVolume(...)` | `modules/vesselnetworkanalysis/processors/vesselgraphcreator.cpp` | Labels connected regions during branch-ID construction. |
| `mapEdgeIds(...)` | `modules/vesselnetworkanalysis/processors/vesselgraphcreator.cpp` | Replaces temporary region labels with proto-edge IDs. |
| `collectUnfinishedRegions(...)` | `modules/vesselnetworkanalysis/processors/vesselgraphcreator.cpp` | Locates foreground regions that still lack an edge ID. |
| `UnfinishedRegions::floodAllRegions(...)` | `modules/vesselnetworkanalysis/processors/vesselgraphcreator.cpp` | Extends nearby edge IDs into unfinished regions. |
| `refineVesselGraph(...)` | `modules/vesselnetworkanalysis/processors/vesselgraphcreator.cpp` | Optionally prunes branches and reruns extraction. |
| `binarizeVolume(...)` | `modules/bigdataimageprocessing/datastructures/lz4slicevolume.cpp` | Converts volume input to binary temporary storage. |
| `VolumeMask` | `modules/vesselnetworkanalysis/algorithm/volumemask.h` | Represents the binary mask used during thinning. |
| `VolumeMask::skeletonize(...)` | `modules/vesselnetworkanalysis/algorithm/volumemask.h` | Reduces vessel foreground to its centerline skeleton. |
| `SkeletonClassReader::getClass(...)` | `modules/vesselnetworkanalysis/algorithm/streaminggraphcreation.cpp` | Classifies skeleton voxels from their number of neighboring skeleton voxels. |
| `NeighborCountVoxelClassifier` | `modules/vesselnetworkanalysis/algorithm/streaminggraphcreation.h` | Provides neighbor-count classes for graph construction. |
| `MetaDataCollector::createProtoVesselGraph(...)` | `modules/vesselnetworkanalysis/algorithm/streaminggraphcreation.cpp` | Converts skeleton components into proto-nodes and proto-edges. |
| `ProtoVesselGraphEdge` | `modules/vesselnetworkanalysis/datastructures/protovesselgraph.cpp` | Stores and smooths a proto-edge centerline in world coordinates. |
| `ProtoVesselGraph::createVesselGraph(...)` | `modules/vesselnetworkanalysis/datastructures/protovesselgraph.cpp` | Measures labeled vessel surface voxels against centerlines and creates final graph objects. |
| `surfaceDistanceSq(...)` | `modules/vesselnetworkanalysis/datastructures/protovesselgraph.cpp` | Defines the distance used as a centerline-to-surface radius observation. |
| `VesselSkeletonVoxel` | `modules/vesselnetworkanalysis/datastructures/vesselgraph.h` | Stores local radius and volume observations for one edge centerline sample. |
| `VesselGraphBuilder` | `modules/vesselnetworkanalysis/datastructures/vesselgraph.h` | Inserts final nodes and edges and records connectivity. |
| `VesselGraphEdge::finalizeConstruction()` | `modules/vesselnetworkanalysis/datastructures/vesselgraph.cpp` | Selects the usable outer path and triggers edge property calculation. |
| `VesselGraphEdgePathProperties::fromPath(...)` | `modules/vesselnetworkanalysis/datastructures/vesselgraph.cpp` | Computes edge length and radius summaries from centerline samples. |
| `VesselGraphEdge::getLength()` | `modules/vesselnetworkanalysis/datastructures/vesselgraph.cpp` | Returns branch length. |
| `VesselGraphEdge::getAvgRadiusAvg()` | `modules/vesselnetworkanalysis/datastructures/vesselgraph.cpp` | Returns the mean of the local mean-radius observations. |
| `VesselGraphEdge::getMinRadiusAvg()` / `getMaxRadiusAvg()` | `modules/vesselnetworkanalysis/datastructures/vesselgraph.cpp` | Return lower and upper radius summaries for a branch. |
| `VesselGraphEdge::getCurveness()` | `modules/vesselnetworkanalysis/datastructures/vesselgraph.cpp` | Returns branch length divided by node-to-node distance. |
| `VesselGraphNode::getDegree()` | `modules/vesselnetworkanalysis/datastructures/vesselgraph.cpp` | Returns how many edges are incident on a graph node. |
| `VesselGraphNode::getRadius()` | `modules/vesselnetworkanalysis/datastructures/vesselgraph.cpp` | Returns the separately estimated radius of a graph node region. |
| `VesselGraphGlobalStats::exportToFile(...)` | `modules/vesselnetworkanalysis/processors/vesselgraphglobalstats.cpp` | Exports edge feature values and node degree values to CSV. |

## Feature Output Names

The edge CSV generated by `VesselGraphGlobalStats` includes these requested
feature fields:

| Requested concept | Exported field or getter |
| --- | --- |
| Length | `length`, `VesselGraphEdge::getLength()` |
| Radius | `avgRadiusAvg`, `minRadiusAvg`, `maxRadiusAvg`, and their standard deviations |
| Curvature-like branch value | `curveness`, `VesselGraphEdge::getCurveness()` |
| Degrees at branch ends | `node1_degree` (minimum endpoint degree), `node2_degree` (maximum endpoint degree), `VesselGraphNode::getDegree()` |

The node CSV also provides a single `degree` column for each graph node.
