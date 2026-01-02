<!-- ROUGH DRAFT: Work in progress... -->

# Appendix C: Attribute Domain Conversion

## Domain Conversion Rules

This section provides comprehensive documentation of how attributes convert between different geometry domains (Point, Edge, Face, Face Corner, Spline, Instance).

### Understanding Domain Conversion

**What is Domain Conversion?**
- The process of transforming attribute data from one domain to another
- Required when operations need data in a specific domain
- Uses aggregation methods (Average, Min, Max, Sum) to combine multiple values

**Why Domain Conversion is Needed**:
- Different operations require specific domains
- Attribute data may exist in wrong domain for current operation
- Visualization and analysis may require different domains
- Material/shader access may need specific domain (UVs in corner domain)

**Automatic vs Manual Conversion**:
- **Automatic**: Blender converts when domain mismatch detected (uses default aggregation)
- **Manual**: Use Evaluate on Domain node for explicit control over aggregation method
- **Recommendation**: Use manual conversion for predictable results

**Performance Impact**:
- Conversions involving aggregation are computationally moderate
- Multiple elements → single element (averaging) requires computation
- Single element → multiple elements (distribution) is faster
- Conversion happens per-element in target domain

### The Fundamental Domains

**Point Domain (Vertex Domain)**
- **Elements**: Vertices/points in the mesh
- **Count**: Number of points in geometry
- **Use Cases**: Position data, point attributes, per-vertex operations
- **Example Attributes**: Position, ID, custom point data

**Edge Domain**
- **Elements**: Edges connecting two vertices
- **Count**: Number of edges in mesh
- **Use Cases**: Edge properties, crease values, edge-based selection
- **Example Attributes**: Crease, edge smoothness, custom edge data

**Face Domain (Polygon Domain)**
- **Elements**: Faces/polygons in mesh
- **Count**: Number of faces in geometry
- **Use Cases**: Material assignment, face normals, per-face attributes
- **Example Attributes**: Material Index, Shade Smooth, face normals

**Face Corner Domain (Loop Domain)**
- **Elements**: Corners where face meets vertex (one per vertex per adjacent face)
- **Count**: Sum of vertex counts of all faces
- **Use Cases**: UV coordinates, split normals, per-face-vertex data
- **Example Attributes**: UV Map, split normals, vertex colors (modern)
- **Important**: A quad face has 4 corners; same vertex may have multiple corners

**Spline Domain (Curve-Specific)**
- **Elements**: Individual splines within curve object
- **Count**: Number of separate curves/splines
- **Use Cases**: Per-spline properties, curve attributes
- **Example Attributes**: Cyclic, Resolution, spline-specific data

**Instance Domain**
- **Elements**: Individual instances in instance collection
- **Count**: Number of instances
- **Use Cases**: Per-instance transforms, instance attributes
- **Example Attributes**: Instance rotation, scale, custom instance data

**Layer Domain (Grease Pencil-Specific)**
- **Elements**: Drawing layers within a Grease Pencil object
- **Count**: Number of layers
- **Use Cases**: Organizing strokes, layer-level masking, blending modes
- **Example Attributes**: Opacity, Blend Mode, Show/Hide
- **Note**: Unique to Grease Pencil; acts as a container for Splines. Attributes here interpolate down to all strokes in the layer.

**Control Point Domain (Grease Pencil-Specific)**
- **Elements**: Individual points defining a Grease Pencil stroke
- **Count**: Total points across all strokes
- **Use Cases**: Stylus pressure data, vertex color painting, varying thickness
- **Example Attributes**: Pressure, Strength, Vertex Color
- **Distinction**: While technically the "Point" domain, GP points lack "Edge" connectivity (no wireframe edges) and interpolate data along the Spline (Stroke) rather than across a surface.

### Domain Conversion Rules Table

This matrix shows how data gets converted between domains. Aggregation and distribution are applied judiciously to enable sensible implicit conversions between incompatible data when domains differ in level of detail or in value cardinality.

**Legend**

- ✓ = Direct conversion available
- AGG = Aggregation required (averaging, min, max, or sum)
- DIST = Distribution (one-to-many)
- N/A = Not applicable (different geometry types)
- — = No meaningful conversion

#### Point Domain Conversions

**Point → Point**
- **Conversion**: None needed (same domain)
- **Method**: Direct pass-through
- **Data Loss**: None
- **Example**: Reading position, already in point domain

**Point → Edge**
- **Conversion**: AGG (Aggregation required)
- **Method**: 
  - Each edge connects two points
  - Aggregation combines data from both endpoint vertices
  - Default: Average of two endpoint values
  - Options: Average, Min, Max, Sum
- **Data Loss**: Per-vertex variation becomes per-edge average
- **Example**: 
  - Point values: Vertex A = 1.0, Vertex B = 3.0
  - Edge value (Average): (1.0 + 3.0) / 2 = 2.0
  - Edge value (Min): 1.0
  - Edge value (Max): 3.0
  - Edge value (Sum): 4.0
- **Use Cases**: Converting vertex data to edge properties, edge-based operations

**Point → Face**
- **Conversion**: AGG (Aggregation required)
- **Method**:
  - Each face has multiple vertices
  - Aggregation combines data from all face vertices
  - Default: Average of all vertex values in face
  - Options: Average, Min, Max, Sum
- **Data Loss**: Per-vertex variation becomes single per-face value
- **Example**:
  - Quad face with vertices: 1.0, 2.0, 3.0, 4.0
  - Face value (Average): (1.0 + 2.0 + 3.0 + 4.0) / 4 = 2.5
  - Face value (Min): 1.0
  - Face value (Max): 4.0
  - Face value (Sum): 10.0
- **Use Cases**: Face-based coloring from vertex data, material assignment

**Point → Face Corner**
- **Conversion**: DIST (Distribution/Expansion)
- **Method**:
  - Each vertex may have multiple corners (one per adjacent face)
  - Point value distributed to all corners at that vertex
  - One point value → many corner values (same value)
- **Data Loss**: None (values duplicated, not aggregated)
- **Example**:
  - Vertex value: 5.0
  - If vertex has 4 adjacent faces: all 4 corners get value 5.0
- **Use Cases**: Preparing vertex data for UV operations, corner-based processing
- **Note**: Multiple corners per vertex will have identical values from this conversion

**Point → Spline**
- **Conversion**: AGG (Aggregation required for curves)
- **Method**:
  - All points in spline aggregated to single spline value
  - Default: Average of all point values in spline
  - Options: Average, Min, Max, Sum
- **Data Loss**: All per-point variation becomes single value
- **Example**:
  - Spline with 10 points, values 0.0 to 9.0
  - Spline value (Average): 4.5
- **Use Cases**: Per-spline properties from point data
- **Context**: Curve geometry only

**Point → Instance**
- **Conversion**: N/A (different geometry contexts)
- **Method**: Not directly convertible
- **Note**: Points and instances are separate concepts; no automatic conversion

#### Edge Domain Conversions

**Edge → Point**
- **Conversion**: DIST (Distribution to endpoints)
- **Method**:
  - Each edge has two endpoint vertices
  - Edge value distributed to both endpoint vertices
  - If vertex has multiple adjacent edges: aggregation of all edge values
  - Default aggregation at vertex: Average
  - Options: Average, Min, Max, Sum
- **Data Loss**: None for edge-to-endpoint, aggregation occurs at vertices with multiple edges
- **Example**:
  - Edge value: 10.0
  - Both endpoint vertices receive: influenced by 10.0
  - If vertex has 3 edges (10.0, 20.0, 30.0):
    - Average: 20.0
    - Min: 10.0
    - Max: 30.0
    - Sum: 60.0
- **Use Cases**: Edge-based data influencing vertices, vertex coloring from edges

**Edge → Edge**
- **Conversion**: None needed (same domain)
- **Method**: Direct pass-through
- **Data Loss**: None

**Edge → Face**
- **Conversion**: DIST then AGG (Distribution to adjacent faces, then aggregation)
- **Method**:
  - Each edge is adjacent to 1-2 faces (boundary edges: 1, internal edges: 2)
  - Edge value distributed to adjacent faces
  - If face has multiple edges: aggregation occurs
  - Default: Average of all edge values around face
  - Options: Average, Min, Max, Sum
- **Data Loss**: Edge-specific variation becomes face aggregate
- **Example**:
  - Quad face with 4 edges: values 1.0, 2.0, 3.0, 4.0
  - Face value (Average): 2.5
  - Face value (Sum): 10.0
- **Use Cases**: Face properties from edge data, crease influence on faces

**Edge → Face Corner**
- **Conversion**: DIST (Complex distribution)
- **Method**:
  - Each edge touches multiple face corners (2 corners per adjacent face)
  - Edge value distributed to corners along that edge
  - Aggregation if corner receives from multiple edges
- **Data Loss**: Minimal with proper aggregation
- **Use Cases**: Corner-based edge influence
- **Note**: Complex conversion, less commonly used

**Edge → Spline**
- **Conversion**: N/A (edges are mesh concept, splines are curve concept)
- **Method**: Not applicable
- **Note**: Different geometry types

**Edge → Instance**
- **Conversion**: N/A (different geometry contexts)
- **Method**: Not directly convertible

#### Face Domain Conversions

**Face → Point**
- **Conversion**: DIST then AGG (Distribution to vertices, then aggregation)
- **Method**:
  - Each face has multiple vertices
  - Face value distributed to all vertices of face
  - If vertex belongs to multiple faces: aggregation occurs
  - Default: Average of all adjacent face values
  - Options: Average, Min, Max, Sum
- **Data Loss**: Face-specific variation becomes vertex aggregate
- **Example**:
  - Vertex shared by 3 faces with values 10.0, 20.0, 30.0
  - Vertex value (Average): 20.0
  - Vertex value (Min): 10.0
  - Vertex value (Max): 30.0
  - Vertex value (Sum): 60.0
- **Use Cases**: Vertex coloring from face materials, smooth transitions

**Face → Edge**
- **Conversion**: DIST then AGG (Distribution to edges, then aggregation)
- **Method**:
  - Each face has multiple edges
  - Face value distributed to all edges of face
  - If edge belongs to multiple faces: aggregation occurs
  - Default: Average of adjacent face values (1-2 faces per edge)
  - Options: Average, Min, Max, Sum
- **Data Loss**: Face variation becomes edge aggregate
- **Example**:
  - Edge shared by 2 faces with values 5.0 and 15.0
  - Edge value (Average): 10.0
- **Use Cases**: Edge properties from face data, boundary detection

**Face → Face**
- **Conversion**: None needed (same domain)
- **Method**: Direct pass-through
- **Data Loss**: None

**Face → Face Corner**
- **Conversion**: DIST (Direct distribution)
- **Method**:
  - Each face has N corners (N = vertex count of face)
  - Face value distributed to all corners of that face
  - No aggregation needed (corners belong to single face)
- **Data Loss**: None (distribution only)
- **Example**:
  - Quad face (4 corners), face value: 7.0
  - All 4 corners receive value: 7.0
- **Use Cases**: Preparing face data for corner-based operations, UV setup

**Face → Spline**
- **Conversion**: N/A (faces are mesh concept, splines are curve concept)
- **Method**: Not applicable

**Face → Instance**
- **Conversion**: N/A (different geometry contexts)
- **Method**: Not directly convertible

#### Face Corner Domain Conversions

**Face Corner → Point**
- **Conversion**: AGG (Aggregation at vertices)
- **Method**:
  - Multiple corners may exist at same vertex (one per adjacent face)
  - All corners at vertex aggregated to single point value
  - Default: Average of all corner values at that vertex
  - Options: Average, Min, Max, Sum
- **Data Loss**: Corner-specific variation (e.g., UV seams, split normals) lost
- **Example**:
  - Vertex with 4 adjacent faces = 4 corners
  - Corner values: 1.0, 2.0, 3.0, 4.0
  - Point value (Average): 2.5
  - Point value (Max): 4.0
- **Use Cases**: Vertex colors from corner data, averaging split normals
- **Note**: This conversion loses UV seam information

**Face Corner → Edge**
- **Conversion**: AGG (Aggregation from corners on edge)
- **Method**:
  - Each edge has 2-4 adjacent corners (2 per adjacent face)
  - Corners on/around edge aggregated to single edge value
  - Default: Average of corner values
  - Options: Average, Min, Max, Sum
- **Data Loss**: Corner variation becomes edge aggregate
- **Use Cases**: Edge properties from corner data
- **Note**: Less commonly used conversion

**Face Corner → Face**
- **Conversion**: AGG (Aggregation within face)
- **Method**:
  - All corners within face aggregated to single face value
  - Default: Average of all corner values in face
  - Options: Average, Min, Max, Sum
- **Data Loss**: Corner variation within face lost
- **Example**:
  - Quad face with 4 corners: values 2.0, 4.0, 6.0, 8.0
  - Face value (Average): 5.0
  - Face value (Sum): 20.0
- **Use Cases**: Face properties from corner data, UV analysis

**Face Corner → Face Corner**
- **Conversion**: None needed (same domain)
- **Method**: Direct pass-through
- **Data Loss**: None

**Face Corner → Spline**
- **Conversion**: N/A (different geometry types)
- **Method**: Not applicable

**Face Corner → Instance**
- **Conversion**: N/A (different geometry contexts)
- **Method**: Not directly convertible

#### Spline Domain Conversions (Curve Geometry)

**Spline → Point**
- **Conversion**: DIST (Distribution to all points in spline)
- **Method**:
  - Spline value distributed to all points within that spline
  - Each point in spline receives same value
- **Data Loss**: None (pure distribution)
- **Example**:
  - Spline value: 3.5
  - Spline has 20 points: all receive 3.5
- **Use Cases**: Applying spline-wide properties to all points, per-spline effects
- **Context**: Curve geometry only

**Spline → Edge**
- **Conversion**: N/A (edges are mesh concept)
- **Method**: Not applicable for curves
- **Note**: Curve segments are not the same as mesh edges

**Spline → Face**
- **Conversion**: N/A (faces are mesh concept)
- **Method**: Not applicable for curves

**Spline → Face Corner**
- **Conversion**: N/A (corners are mesh concept)
- **Method**: Not applicable for curves

**Spline → Spline**
- **Conversion**: None needed (same domain)
- **Method**: Direct pass-through
- **Data Loss**: None

**Spline → Instance**
- **Conversion**: N/A (different geometry contexts)
- **Method**: Not directly convertible
- **Note**: Curve splines and instances are separate concepts

#### Instance Domain Conversions

**Instance → Point**

- **Conversion**: Special (Instance position extraction)
- **Method**: Instance positions become point positions
- **Requires**: Instances to Points node (not automatic domain conversion)
- **Data Loss**: Instance geometry and transforms (except position) lost
- **Use Cases**: Converting instance positions to point cloud

**Instance → Edge**

- **Conversion**: N/A (different geometry concepts)
- **Method**: Not directly convertible

**Instance → Face**

- **Conversion**: N/A (different geometry concepts)
- **Method**: Not directly convertible

**Instance → Face Corner**

- **Conversion**: N/A (different geometry concepts)
- **Method**: Not directly convertible

**Instance → Spline**

- **Conversion**: N/A (different geometry types)
- **Method**: Not directly convertible

**Instance → Instance**: No-op

- **Conversion**: None needed (same domain)
- **Method**: Direct pass-through
- **Data Loss**: None

#### Volume Grid Domain Conversions (Blender 5.0)

**Grid → Point**

- **Conversion**: SAMPLING (Trilinear Interpolation)
- **Method**: 
  - Grid is a spatial volume, not topologically connected to mesh points
  - The value is calculated by sampling the grid at the point's position
  - Uses trilinear interpolation between the 8 nearest voxels
- **Data Loss**: High-frequency detail smaller than the voxel size is smoothed out
- **Use Cases**: transferring smoke density to particles, volume-based displacement
- **Performance**: Fast (spatial lookup), but dependent on grid resolution

**Grid → Edge / Face / Corner**

- **Conversion**: SAMPLING (Spatial)
- **Method**: 
  - Samples the grid at the element's center (Edge/Face) or position (Corner)
  - Equivalent to "Sample Volume" node behavior
- **Note**: Does not use topological connectivity

**Point → Grid**

- **Conversion**: RASTERIZATION / VOXELIZATION
- **Method**: 
  - Points are rasterized into the volume grid
  - Requires "Mesh to Volume" or "Points to Volume" operations
  - Attributes are averaged or summed into the voxels they occupy
- **Data Loss**: Discrete point data becomes continuous volumetric field
- **Use Cases**: Creating density fields from particles, SDF generation

**Grid → Instance**

- **Conversion**: SAMPLING (at Instance Origin)
- **Method**: Samples grid value at the instance's transform origin

## Conversion Methods

### Aggregation/Value Selection Methods

When converting from many elements to fewer elements, aggregation is used to combine multiple values into one summary value that describes some aspect of or selection from the values as a group.

**First**
- **Calculation**: Takes the first encountered value in the element list
- **Use Cases**: When order matters and you want the initial value
- **Behavior**: Deterministic based on internal geometry order
- **Example**: Values (10, 20, 30) → First = 10
- **Best For**: Situations where first element has priority or special meaning
- **Note**: Internal geometry order may not match visual/logical order
- **Performance**: Very fast (no calculation, just selection)

**Last**
- **Calculation**: Takes the last encountered value in the element list
- **Use Cases**: When final value in sequence is important
- **Behavior**: Deterministic based on internal geometry order
- **Example**: Values (10, 20, 30) → Last = 30
- **Best For**: End-of-sequence data, final state capture
- **Note**: Like First, depends on internal geometry order
- **Performance**: Very fast (no calculation, just selection)

**Average (Mean)**
- **Calculation**: Sum of all values ÷ Count of values
- **Use Cases**: Default for most conversions, smooth transitions, color blending
- **Behavior**: Produces middle value
- **Example**: Values (10, 20, 30) → Average = 20
- **Best For**: General purpose, natural-looking results
- **Mathematical Formula**: (v₁ + v₂ + ... + vₙ) / n
- **Vector Behavior**: Averages each component independently (X, Y, Z)
- **Rotation Behavior**: Linear average of components (Quaternion/Euler).
  - **Warning**: This may result in non-normalized rotations (scaling artifacts) or improper interpolation paths. For precise rotation blending, specialized rotation nodes are preferred over simple domain averaging.
- **Color Behavior**: Averages RGB channels independently
- **Performance**: Moderate (requires sum and division)

**Mode**
- **Calculation**: The most frequently occurring value
- **Use Cases**: Integer properties, Material Indices, categorizing based on majority
- **Behavior**: "Majority rules" logic
- **Example**: Values (1, 1, 1, 2, 5) → Mode = 1
- **Best For**: Discrete data types (Integer, Boolean) where averaging is invalid
- **Performance**: Slow (requires sorting and counting)

**Median**
- **Calculation**: Middle value when all values are sorted
- **Use Cases**: Robust averaging that ignores outliers
- **Behavior**: 50th percentile value
- **Example**: 
  - Values (10, 20, 30) → Median = 20
  - Values (10, 20, 30, 1000) → Median = 25 (average of 20 and 30)
  - Odd count: middle value
  - Even count: average of two middle values
- **Best For**: Data with outliers, robust statistics
- **Advantage**: Not affected by extreme values (unlike Average)
- **Performance**: Moderate (requires sorting)

**Sum (Total)**
- **Calculation**: Addition of all values
- **Use Cases**: Counting, accumulation, area/volume calculations
- **Behavior**: Values add up
- **Example**: Values (10, 20, 30) → Sum = 60
- **Best For**: Totals, counts, cumulative properties
- **Mathematical Formula**: v₁ + v₂ + ... + vₙ
- **Vector Behavior**: Sums each component independently
- **Warning**: Can grow very large with many elements
- **Performance**: Fast (simple addition)

**Minimum (Min)**
- **Calculation**: Lowest value among all inputs
- **Use Cases**: Preserving minimum values, thresholds, conservative estimates
- **Behavior**: Smallest value wins
- **Example**: Values (10, 20, 30) → Min = 10
- **Best For**: Ensuring minimums, lower bounds, conservative selection
- **Vector Behavior**: Minimum of each component independently (not shortest vector)
- **Color Behavior**: Darkest value per channel
- **Performance**: Fast (simple comparison)

**Maximum (Max)**
- **Calculation**: Highest value among all inputs
- **Use Cases**: Preserving maximum values, peak detection, liberal estimates
- **Behavior**: Largest value wins
- **Example**: Values (10, 20, 30) → Max = 30
- **Best For**: Ensuring maximums, upper bounds, peak preservation
- **Vector Behavior**: Maximum of each component independently (not longest vector)
- **Color Behavior**: Brightest value per channel
- **Performance**: Fast (simple comparison)

**Range**
- **Calculation**: Difference between maximum and minimum values
- **Use Cases**: Measuring variation, spread of values
- **Behavior**: Max - Min
- **Example**: Values (10, 20, 30) → Range = 30 - 10 = 20
- **Best For**: Understanding value distribution, detecting variation
- **Mathematical Formula**: Max(values) - Min(values)
- **Vector Behavior**: Range of each component independently
- **Performance**: Fast (two comparisons, one subtraction)

**Standard Deviation**
- **Calculation**: Measure of value spread around the mean
- **Use Cases**: Statistical analysis, understanding variation
- **Behavior**: Square root of variance
- **Example**: Values (10, 20, 30)
  - Mean = 20
  - Variance = [(10-20)² + (20-20)² + (30-20)²] / 3 = 66.67
  - Std Dev = √66.67 ≈ 8.165
- **Best For**: Scientific/statistical applications, quality control
- **Mathematical Formula**: √[Σ(vᵢ - mean)² / n]
- **Vector Behavior**: Standard deviation of each component independently
- **Performance**: Expensive (requires mean calculation, then variance)

**Variance**
- **Calculation**: Average of squared differences from mean
- **Use Cases**: Statistical analysis, measuring dispersion
- **Behavior**: Square of standard deviation
- **Example**: Values (10, 20, 30)
  - Mean = 20
  - Variance = [(10-20)² + (20-20)² + (30-20)²] / 3 = 66.67
- **Best For**: Statistical calculations, variance analysis
- **Mathematical Formula**: Σ(vᵢ - mean)² / n
- **Vector Behavior**: Variance of each component independently
- **Performance**: Expensive (requires mean calculation first)

**Count**
- **Calculation**: Number of contributing elements
- **Use Cases**: Determining how many elements contributed to aggregation
- **Behavior**: Returns integer count of elements
- **Example**: 5 vertices in face → Count = 5
- **Best For**: Understanding element density, validation, debugging
- **Output Type**: Always integer
- **Note**: Useful for normalizing or understanding aggregation context
- **Performance**: Very fast (simple counter)

### Aggregation Method Comparison

#### Statistical Properties

Robustness to Outliers (Best to Worst):
1. Median - Completely ignores outliers
2. Min/Max - Identifies outliers
3. Average - Heavily influenced by outliers
4. Sum - Extremely influenced by outliers

Example with Outlier:
Values: (10, 20, 30, 1000)
- Median: 25 (robust, not affected)
- Average: 265 (heavily skewed by 1000)
- Min: 10 (identifies lower bound)
- Max: 1000 (identifies outlier)
- Sum: 1060 (dominated by outlier)

### Computational Cost

Performance Ranking (Fastest to Slowest):
1. First, Last, Count - O(1) or O(n) single pass
2. Min, Max, Sum - O(n) single pass
3. Average, Range - O(n) single pass with division
4. Median - O(n log n) requires sorting
5. Variance, Std Dev - O(2n) requires two passes

For n = 1000 elements:
- First/Last: ~1µs
- Min/Max/Sum: ~10µs
- Average: ~12µs
- Median: ~50µs
- Std Dev: ~25µs

### Use Case Decision Matrix

```
┌──────────────────────┬─────────────┬──────────────────────┐
│ Scenario             │ Best Method │ Reason               │
├──────────────────────┼─────────────┼──────────────────────┤
│ Smooth blending      │ Average     │ Natural gradients    │
│ Preserve dark areas  │ Min         │ Keeps darkest        │
│ Preserve bright      │ Max         │ Keeps brightest      │
│ Total area/volume    │ Sum         │ Accumulation         │
│ With outliers        │ Median      │ Robust to extremes   │
│ Element counting     │ Count       │ Direct count         │
│ Value spread         │ Range       │ Shows variation      │
│ Statistical analysis │ Std Dev     │ Appropriate measure  │
│ First occurrence     │ First       │ Order matters        │
│ Final state          │ Last        │ End result matters   │
└──────────────────────┴─────────────┴──────────────────────┘
```

### Practical Examples

#### Example 1: Terrain Height with Outliers

Problem: Few spikes in terrain (outliers) skew average height
Values per face: [10, 10, 11, 10, 50, 10, 11] (one spike at 50)

Average: (10+10+11+10+50+10+11)/7 = 16 (skewed by spike)
Median: 10 (robust, ignores spike)
Min: 10 (base terrain)
Max: 50 (identifies spike)

Best choice: Median for robust typical height
Alternative: Average for including all features

#### Example 2: Color Blending

Values: Red(1,0,0), Green(0,1,0), Blue(0,0,1)

Average: (0.33, 0.33, 0.33) - Gray (smooth blend)
Min: (0, 0, 0) - Black (darkest per channel)
Max: (1, 1, 1) - White (brightest per channel)
Median: (0, 0, 0) - Black (middle of sorted per channel)

Best choice: Average for natural color mixing

#### Example 3: Selection Propagation

Vertices selected: [True, True, False, False] → [1, 1, 0, 0]

Average: 0.5 (50% selected - threshold comparison)
Min: 0 (not all selected - strict)
Max: 1 (any selected - liberal)
Sum: 2 (count of selected)
Count: 4 (total vertices)

Best choice depends on desired strictness:
- Min: All must be selected
- Max: Any can be selected
- Average: Majority rule (with threshold)

#### Example 4: Material Index (Integer Data)

Vertex material indices: [0, 0, 1, 2]

Average: 0.75 (nonsensical - not an integer)
First: 0 (use first vertex's material)
Last: 2 (use last vertex's material)
Min: 0 (use lowest index)
Max: 2 (use highest index)
Mode: 0 (most common value; requires Mode aggregation)

Best choice: First or Mode (if available)
Avoid: Average (produces invalid indices)

#### Example 5: Edge Crease Values

Edges around face: [0.0, 0.5, 1.0, 0.0]

Average: 0.375 (soft overall crease)
Min: 0.0 (face has some smooth edges)
Max: 1.0 (face has some sharp edges)
Median: 0.25 (typical crease level)

Best choice: Max to preserve sharpest edge influence
Alternative: Average for overall sharpness feel

#### Example 6: Vector Aggregation (Component-wise)

Vectors: (1,2,3), (4,5,6), (7,8,9)

Average: ((1+4+7)/3, (2+5+8)/3, (3+6+9)/3) = (4, 5, 6)
Min: (1, 2, 3) - minimum per component
Max: (7, 8, 9) - maximum per component
Sum: (12, 15, 18) - sum per component
Median: (4, 5, 6) - median per component

Note: Operations are per-component, not on vector length
Vector length aggregation requires separate calculation

#### Example 7: Color Aggregation

Colors: Red(1,0,0,1), Green(0,1,0,1), Blue(0,0,1,1)

Average: (0.33, 0.33, 0.33, 1) - Gray
Min: (0, 0, 0, 1) - Black (darkest per channel)
Max: (1, 1, 1, 1) - White (brightest per channel)
Sum: (1, 1, 1, 3) - Over-bright (needs normalization)

Color-specific considerations:
- Alpha usually averaged separately
- HDR colors can exceed 1.0 with Sum
- Min/Max per channel creates extreme colors

### Special Considerations

#### Empty Element Sets

What happens when aggregation is performed on zero elements?

Average (Undefined/indeterminate due to division of a (vacuous) zero sum by (a quantity of) zero): Returns 0 or default
Sum: 0 (sum of zero elements is (the additive identity,) zero)
Min/Max (Undefined): Returns default value
Count: 0 (i.e., no elements are present)
Median (Undefined): Returns 0 or default

Recommendation: Always ensure geometry has elements before performing aggregate to single value conversion regardless of whether or not this conversion is performed implicitly

#### Aggregation of a Single Element

Most aggregation methods return the same value when only one input element is provided:

- Values: [42]
- First: 42
- Last: 42
- Average: 42
- Median: 42
- Min: 42
- Max: 42
- Sum: 42
- Range: 0
- Std Dev: 0
- Count: 1

## Automatic Domain Conversion

### When Blender Converts Automatically

**Field Evaluation Context**

When a field is evaluated, Blender automatically converts it to match the domain required by the operation:

```
Automatic Conversion Process:
1. Node determines required domain for operation
2. Input field domain detected
3. If mismatch: Automatic conversion triggered
4. Default aggregation method applied (usually Average)
5. Field evaluated in correct domain

Example 1: Set Material Node (Requires Face Domain)
Input: Point domain selection (boolean per vertex)
Process:
- Set Material operates in Face domain
- Point selection automatically converted to Face
- Aggregation: Average of vertex selections per face
- Face selected if average ≥ 0.5 (threshold)
Result: Materials assigned to faces based on vertex selection

Example 2: Delete Geometry with Face Domain Selected
Input: Point domain selection
Process:
- Delete Geometry domain parameter set to Face
- Point selection converted to Face domain
- Average aggregation applied
- Faces deleted where average vertex selection > threshold
Result: Face deletion based on vertex selection

Example 3: Set Position (Requires Point Domain)
Input: Face domain displacement vector
Process:
- Set Position operates in Point domain
- Face vectors distributed to vertices
- Each vertex receives average of adjacent face vectors
Result: Vertex positions modified based on face data
```

**Aggregation Rules in Automatic Conversion**

```
Default Aggregation by Conversion:
Point → Face: Average (smooth face values from vertices)
Point → Edge: Average (edge value from two endpoints)
Point → Corner: Distribution (each corner gets vertex value)

Face → Point: Average (vertex value from adjacent faces)
Face → Edge: Average (edge value from adjacent faces)
Face → Corner: Distribution (each corner gets face value)

Edge → Point: Average (vertex value from connected edges)
Edge → Face: Average (face value from boundary edges)

Corner → Point: Average (vertex value from all corners at that vertex)
Corner → Face: Average (face value from its corners)

Spline → Point: Distribution (all points in spline get spline value)
Point → Spline: Average (spline value from all its points)

General Rule:
- Many → Few: Average aggregation (combines multiple values)
- Few → Many: Distribution (duplicates single value)
```

**Implicit Conversion Examples**

```
Example 1: Position-Based Face Coloring
Setup:
- Position node (Point domain, vector)
- Separate XYZ → Z component (Point domain, float)
- Connect to Set Material node material index

What Happens:
- Set Material requires Face domain
- Z values (Point domain) automatically converted
- Each face gets average Z of its vertices
- Material index calculated per face
- Automatic, invisible conversion

Example 2: Instance Rotation from Field
Setup:
- Noise Texture based on position (Point domain, float)
- Connect to Rotate Instances node rotation

What Happens:
- Rotate Instances operates on Instance domain
- Noise values (Point domain) automatically converted
- Each instance samples noise at its position
- Rotation applied per instance
- Field evaluation handles conversion

Example 3: Selection Propagation
Setup:
- Random Value > 0.5 (Point domain boolean)
- Use with Extrude Mesh (Face domain operation)

What Happens:
- Random selection in Point domain
- Extrude Mesh needs Face domain selection
- Point selection converted to Face
- Average: Face selected if average vertex selection ≥ 0.5
- Extrusion applied to selected faces
```

**Field Context vs Domain**

```
Important Distinction:
Field Context: Where field is evaluated (per element)
Domain: Where resulting values are stored

Example:
Position node (Point domain) used in Face context:
- Field evaluated per face (at face center or average)
- Results stored in Face domain
- Evaluation context differs from source domain

This is NOT the same as domain conversion:
- Field evaluation: Sampling field at different locations
- Domain conversion: Aggregating/distributing stored values
```

**When Automatic Conversion Happens**

```
Trigger Conditions:
1. Socket Connection with Domain Mismatch
  - Input domain ≠ required domain
  - Automatic conversion inserted

2. Field Evaluation in Different Context
  - Field evaluated where needed
  - Context determines domain

3. Node with Domain-Specific Requirements
  - Set Material: Requires Face domain
  - Set Position: Requires Point domain
  - Delete Geometry: Uses specified domain

4. Captured Attribute Interpolation
  - An attribute captured in one domain (e.g., Point) and used in another context (e.g., Face)
  - The field interpolates automatically at the point of usage
  - Example: Capturing "Position" (Point) and plugging it into "Set Material" (Face) results in the average position of the face's vertices.

5. Selection Propagation
  - Selection used by operation in different domain
  - Converts to match operation domain

Does NOT Happen:
- Same domain connection (no conversion needed)
- Explicit Evaluate on Domain used (manual control)
- Direct attribute reading in correct domain
```

**Automatic vs Manual Conversion**

```
Automatic Conversion:
Characteristics:
- Happens invisibly
- Uses default aggregation (usually Average)
- No user control over method
- Convenient but potentially unpredictable

When Appropriate:
- Simple operations
- Default behavior acceptable
- Prototyping and iteration
- Non-critical conversions

Risks:
- May not use optimal aggregation method
- Averaging may not be semantically correct
- Hidden behavior (harder to debug)
- Unexpected results with some data types

Example Problem:
Material indices (Point domain) → Face domain
- Vertices have indices: 0, 1, 2, 3
- Automatic average: 1.5 (nonsensical material index)
- Should use explicit conversion with different method
```

```
Manual Conversion (Evaluate on Domain):
Characteristics:
- Explicit in node network
- User chooses aggregation method
- Visible, debuggable
- Predictable behavior

When Required:
- Non-average aggregation needed (Min, Max, Sum)
- Critical conversions
- Data type requires specific handling
- Debugging domain issues

Advantages:
+ Full control over aggregation
+ Explicit, visible conversion
+ Correct semantic handling
+ Easier debugging

Example Solution:
Material indices (Point domain) → Face domain
- Use Evaluate on Domain
- Method: First or Mode (most common)
- Result: Sensible face material index
- Explicit, correct behavior
```

**Performance Implications**

```
Automatic Conversion Performance:
Impact: Moderate
- Happens during evaluation
- Cached for repeated access
- Usually not a bottleneck

Optimization:
- Minimize conversions by working in correct domain
- Cache results if used multiple times
- Use explicit conversion for complex cases

Measurement:
- Automatic conversions hard to profile individually
- Total evaluation time includes all conversions
- Restructure network to reduce conversions

Manual Conversion Performance:
Impact: Same as automatic (computation identical)
- Explicit node adds no overhead
- Same aggregation computation
- Visibility aids optimization

Benefit:
- Easier to identify conversion locations
- Can optimize by removing/relocating
- Profiling shows node specifically
```

### Node Input Requirements

Different nodes require specific domains for their inputs. Understanding these requirements prevents unexpected conversions and ensures correct behavior.

**Geometry Output Nodes**:

```
Group Output:
- Required Domain: Any (passes through)
- Geometry socket accepts any domain
- No domain conversion forced
- Final output domain preserved

Viewer Node:
- Required Domain: Any (visualization)
- Displays geometry in any domain
- Spreadsheet shows selected domain
- No conversion applied
```

**Transformation Nodes**:

```
Set Position:
- Required Domain: Point
- Position attribute is Point domain by definition
- Offset vector converted to Point if needed
- Selection converted to Point if needed

Transform Geometry:
- Required Domain: Any (operates on geometry)
- Transforms entire geometry object
- Domain-agnostic operation
- No domain conversion needed

Translate/Rotate/Scale Instances:
- Required Domain: Instance
- Operates per instance
- Selection converted to Instance if needed
- Transform values evaluated per instance
```

**Material and Shading Nodes**:

```
Set Material:
- Required Domain: Face
- Material Index must be Face domain
- Selection converted to Face if needed
- Materials assign to polygons (Face level)

Set Shade Smooth:
- Required Domain: Face
- Shade Smooth attribute is Face domain
- Selection converted to Face if needed
- Shading is per-face property
```

**Deletion and Selection Nodes**:

```
Delete Geometry:
- Required Domain: User-specified (parameter)
- Domain parameter: Point, Edge, Face, or Instance
- Selection converted to chosen domain
- Deletes elements in specified domain

Example:
Domain: Face
Selection: Point domain boolean
→ Point selection converted to Face
→ Faces deleted based on converted selection
```

**Geometry Generation Nodes**:

```
Extrude Mesh:
- Required Domain: Face or Edge (user choice)
- Selection in chosen domain
- Offset vector evaluated per element
- Individual vs Group modes affect behavior

Subdivide Mesh:
- Required Domain: Edge or Face
- Selection specifies what to subdivide
- Domain affects subdivision pattern
- Converted if necessary

Merge by Distance:
- Required Domain: Point
- Distance threshold per vertex
- Selection in Point domain
- Point-based operation

Shortest Edge Path:
- Required Domain (Cost): Edge
- Required Domain (End Vertex): Point (Selection)
- Output Domain: Point (Next Vertex Index)
- Note: This node is unique; it calculates using Edge properties (Cost/Length) but stores the routing result on Points.
```

**Curve-Specific Nodes**:

```
Set Curve Radius:
- Required Domain: Point (on curves)
- Curve radius is per-point attribute
- Radius values evaluated per curve point
- Selection in Point domain

Set Spline Cyclic:
- Required Domain: Spline
- Cyclic is spline-level property
- Selection in Spline domain
- Per-spline boolean attribute

Curve to Mesh:
- Required Domain: Curve data
- Profile in Curve domain
- Output mesh (different geometry type)
- Domain change via geometry type change
```

**Instancing Nodes**:

```
Instance on Points:
- Required Domain: Point (for instance positions)
- Points define instance locations
- Selection in Point domain
- Instance geometry domain-independent

Realize Instances:
- Required Domain: Instance → Geometry
- Converts instance references to geometry
- Instance attributes → Vertex/Face attributes
- Domain shift from Instance to mesh domains
```

**Attribute Nodes**:

```
Store Named Attribute:
- Required Domain: User-specified (parameter)
- Domain parameter determines storage domain
- Value converted to chosen domain if needed
- Explicit domain selection required

Capture Attribute:
- Required Domain: Matches geometry context
- Captures at evaluation domain
- Anonymous attribute in that domain
- Domain determined by usage context

Named Attribute (Read):
- Required Domain: Stored domain (no conversion)
- Reads from attribute's native domain
- Returns values in that domain
- User must handle domain matching
```

**Field Math Nodes**:

```
Math, Vector Math, Boolean Math:
- Required Domain: Context-dependent (field evaluation)
- Evaluate in whatever domain they're used
- No inherent domain requirement
- Results in usage domain

Map Range, Clamp:
- Required Domain: Context-dependent
- Field operations
- Domain from evaluation context
- No forced conversion
```

**Special Domain Behavior**:

```
Join Geometry:
- Required Domain: Mixed (preserves each input's domains)
- Each input retains its domain structure
- Attributes merged across inputs
- No forced domain unification

Split:
- Domain handling varies by split type
- Selection domain determines split behavior
- Output retains appropriate domains
- Context-dependent
```

**Node Domain Reference Table**:

```
┌─────────────────────────┬──────────────┬───────────────────┐
│ Node                    │ Required     │ Auto-Converts     │
│                         │ Domain       │ From              │
├─────────────────────────┼──────────────┼───────────────────┤
│ Set Position            │ Point        │ Any → Point       │
│ Set Material            │ Face         │ Any → Face        │
│ Set Shade Smooth        │ Face         │ Any → Face        │
│ Delete Geometry         │ User Choice  │ Any → Choice      │
│ Extrude Mesh            │ Face/Edge    │ Any → Face/Edge   │
│ Set Curve Radius        │ Point        │ Any → Point       │
│ Set Spline Cyclic       │ Spline       │ Any → Spline      │
│ Store Named Attribute   │ User Choice  │ Any → Choice      │
│ Instance on Points      │ Point        │ Any → Point       │
│ Distribute Points       │ Face         │ Any → Face        │
│ Transform Geometry      │ None         │ N/A               │
│ Join Geometry           │ Mixed        │ N/A               │
└─────────────────────────┴──────────────┴───────────────────┘

Legend:
- Required Domain: Domain node operates in
- Auto-Converts From: Automatic conversion if mismatch
- User Choice: Domain parameter in node
- Mixed: Preserves multiple domains
- None: Domain-agnostic
- N/A: No conversion occurs
```

**Debugging Domain Mismatches**:

```
Common Issue: Unexpected Results from Automatic Conversion

Symptoms:
- Values averaging unexpectedly
- Selection not working as intended
- Attributes appear in wrong domain
- Operations affect wrong elements

Diagnosis Process:
1. Check node documentation for required domain
2. Verify input attribute domain (Spreadsheet)
3. Add Viewer node to see intermediate domain
4. Check if automatic conversion occurring

Solution:
1. Use Evaluate on Domain explicitly
2. Choose correct aggregation method
3. Work in appropriate domain from start
4. Verify results with Viewer + Spreadsheet

Example Debug:
Problem: Material indices averaging to 1.5
Diagnosis: Point domain indices auto-converted to Face with Average
Solution: Use Evaluate on Domain with First or Mode aggregation
Result: Correct integer material indices per face
```

### Selection Propagation

Selection fields automatically convert domains to match the operation they're controlling. Understanding this behavior is crucial for predictable selection logic.

**Selection Domain Conversion**:

```
Basic Principle:
Selection boolean field converts to match operation domain

Example Flow:
1. Create selection in Point domain
  - Compare Position.Z > 5.0
  - Result: Boolean per vertex

2. Use with Face operation (e.g., Set Material)
  - Selection needs Face domain
  - Automatic conversion: Point → Face
  - Aggregation: Average of vertex booleans per face
  - Face selected if average ≥ 0.5 (threshold)

3. Material applied to selected faces
  - Face domain operation
  - Uses converted selection
  - Correct domain matching
```

**Selection Conversion Modes**:

```
Strict Mode (All elements must be selected):
Implementation: Use Min aggregation

Point → Face with Min:
- Face selected only if ALL vertices selected
- Min(vertex selections) = 1.0 only if all = 1.0
- Strictest selection propagation
- Use case: Ensure complete face coverage

Example:
Quad face with vertex selections: [True, True, False, True]
Average: 0.75 → Selected (if threshold 0.5)
Min: 0.0 (False) → Not selected

Liberal Mode (Any element can trigger selection):
Implementation: Use Max aggregation

Point → Face with Max:
- Face selected if ANY vertex selected
- Max(vertex selections) = 1.0 if any = 1.0
- Most permissive selection
- Use case: Grow selection to adjacent faces

Example:
Quad face with vertex selections: [True, False, False, False]
Average: 0.25 → Not selected (if threshold 0.5)
Max: 1.0 (True) → Selected

Moderate Mode (Majority rule):
Implementation: Use Average with threshold 0.5

Point → Face with Average:
- Face selected if majority of vertices selected
- Average > 0.5 → More than half selected
- Balanced selection propagation
- Use case: Natural selection flow

Example:
Quad face with vertex selections: [True, True, True, False]
Average: 0.75 > 0.5 → Selected
```

**Multi-Step Selection Propagation**:

```
Selection Through Multiple Domains:
Point → Edge → Face

Example: Select faces connected to selected vertices

Step 1: Point selection
- Vertices marked True/False

Step 2: Point → Edge conversion
- Max: Edge selected if either endpoint selected
- Result: Edges touching selected vertices

Step 3: Edge → Face conversion
- Max: Face selected if any edge selected
- Result: Faces touching selected edges

Final: Faces adjacent to original vertex selection
```

**Threshold-Based Selection**:

```
Adjustable Selection Strictness:
Use Compare node after Average conversion

Workflow:
1. Point selection (Boolean)
2. Convert Point → Face (Average)
3. Compare: Average > Threshold
4. Adjust threshold for strictness

Threshold Values:
- 0.0: Any vertex selected → face selected (like Max)
- 0.5: Majority vertices → face selected (default)
- 0.75: Most vertices → face selected
- 1.0: All vertices → face selected (like Min)

Example:
Face with 4 vertices: [1, 1, 0, 0]
Average: 0.5

Threshold 0.4: Selected (0.5 > 0.4)
Threshold 0.5: Selected (0.5 ≥ 0.5)
Threshold 0.6: Not selected (0.5 < 0.6)
```

**Selection Inversion Across Domains**:

```
Problem: Inverted selection after conversion

Cause: Boolean NOT applied before or after conversion affects result

Example:
Point selection: [1, 1, 0, 0]

NOT then Average:
- NOT: [0, 0, 1, 1]
- Average: 0.5
- Result: Half selected (ambiguous)

Average then NOT:
- Average: 0.5
- NOT: 0.5 (if treated as continuous)
- Or: NOT (0.5 > 0.5) = NOT False = True
- Result: Inverted selection

Recommendation:
- Invert in final domain (after conversion)
- Clearer semantic meaning
- Predictable behavior
```

**Soft Selections Across Domains**:

```
Gradient Selection (0.0 to 1.0):
Allows partial selection through domain conversion

Example:
Point domain gradient: 0.0 (unselected) to 1.0 (selected)
Vertices: [1.0, 0.8, 0.2, 0.0]

Convert to Face (Average):
Face value: (1.0 + 0.8 + 0.2 + 0.0) / 4 = 0.5

Use as selection factor:
- 0.5 → 50% effect
- Partial extrusion, partial material, etc.
- Smooth falloff across geometry

Application:
- Smooth selection transitions
- Gradient-based effects
- Partial operations
- Natural-looking procedural selection
```

**Selection Validation**:

```
Verifying Selection Propagation:

Method 1: Viewer + Spreadsheet
1. Place Viewer after selection creation
2. Open Spreadsheet editor
3. Select domain dropdown (Point, Face, etc.)
4. Check boolean values
5. Verify expected elements selected

Method 2: Delete Geometry Test
1. Use selection with Delete Geometry
2. Set domain parameter
3. Observe what gets deleted
4. Confirms selection in that domain

Method 3: Color Coding
1. Use selection as color factor
2. Store as color attribute
3. Visual verification in viewport
4. Immediate feedback

Example Verification:
Goal: Verify face selection from point selection
1. Point selection: Position.Z > 5.0
2. Expected: Faces with vertices above Z=5
3. Test:
  - Delete Geometry (Face domain, selection)
  - Observe: Correct faces deleted
  - Confirm: Selection propagated correctly
```

### Attribute Interpolation

When mesh operations create new geometry, attributes are interpolated to provide values for the new elements. Understanding interpolation is key to maintaining attribute data through operations.

**Interpolation During Subdivision**:

```
Subdivision Surface Behavior:

Original Geometry:
- 4 vertices (corners of quad)
- Vertex colors: Red, Green, Blue, Yellow

After Subdivision (1 level):
- Original 4 vertices: Keep original colors
- New edge midpoint vertices: Average of edge endpoints
- New face center vertex: Average of face vertices

Example:
Original quad vertices:
- V0: Red (1.0, 0.0, 0.0)
- V1: Green (0.0, 1.0, 0.0)
- V2: Blue (0.0, 0.0, 1.0)
- V3: Yellow (1.0, 1.0, 0.0)

New edge midpoint (V0-V1):
- Color: (Red + Green) / 2 = (0.5, 0.5, 0.0) [Yellow-green]

New face center:
- Color: (Red + Green + Blue + Yellow) / 4 = (0.5, 0.5, 0.25) [Gray-yellowish]

Attribute Interpolation:
- Point domain: Interpolated at new vertices
- Face domain: Distributed to new faces
- Corner domain: Interpolated at new corners
- Edge domain: Interpolated at new edges
```

**Interpolation During Extrusion**:

```
Extrude Mesh Behavior:

Original:
- Selected face with attributes

After Extrusion:
- Original face (top): Retains original attributes
- Extruded face (bottom): Retains original attributes (copied)
- Side faces (connecting): Interpolated from edges

Example:
Original face: Material Index = 2

Extrusion creates:
- Top face: Material Index = 2 (original)
- Bottom face: Material Index = 2 (copied)
- Side faces: Material Index = 2 (inherited)

Point Attributes:
- Original points: Keep values
- Extruded points: Copy from original
- Creates duplicate attribute data

Corner Attributes (UVs):
- Original corners: Unchanged
- Extruded corners: Copied from original
- Side face corners: Seams may need adjustment
```

**Interpolation During Booleans**:

```
Mesh Boolean Behavior:

Complex Interpolation:
- New vertices at intersections
- New faces from cutting
- Attributes from both input meshes

Intersection Vertices:
- Interpolate from nearby vertices
- Weighted by distance
- May average attributes from both meshes

New Faces:
- Inherit from source face
- Or blend if on intersection
- Material index from dominant mesh

Attribute Preservation Challenges:
- Some attributes may be lost
- Interpolation approximate
- UVs may need regeneration
- Manual fixing often required

Recommendation:
- Booleans last in workflow
- Re-apply critical attributes after
- Verify UVs and materials
- Use Store Named Attribute to preserve
```

**Controlled Interpolation**:

```
Manual Attribute Control After Operations:

Problem: Automatic interpolation not desired

Solution Workflow:
1. Capture attributes before operation
  - Capture Attribute node
  - Store critical values

2. Perform geometry operation
  - Subdivision, extrusion, etc.
  - Automatic interpolation occurs

3. Selectively restore/recompute attributes
  - For new elements only
  - Or recalculate entirely
  - Override interpolated values

Example: Preserve Original Position
1. Capture Attribute: Position → "original_position"
2. Set Position: Displace geometry
3. Use "original_position" for effects
  - New vertices have interpolated positions
  - Original vertices have stored positions
  - Controlled attribute preservation
```

**UV Interpolation Specifics**:

```
UV Map Interpolation (Corner Domain):

Subdivision:
- New corners created
- UVs interpolated linearly
- Maintains UV continuity
- Seams preserved automatically

Extrusion:
- Copied UVs from original
- Side faces may need UV unwrap
- Seams at extrusion boundaries
- Manual UV adjustment often needed

Recommendation:
- Procedural UVs: Regenerate after operation
- Painted UVs: Preserve carefully
- Check seams after operations
- Use UV editing for critical models
```

**Attribute Data Types and Interpolation**:

```
Float Attributes:
Interpolation: Linear average
Result: Smooth gradients
Example: Height values blend smoothly

Integer Attributes:
Interpolation: Rounded average
Result: May create non-original values
Example: ID values may average to new IDs
Warning: Averaging IDs often not meaningful

Boolean Attributes:
Interpolation: Averaged as 0.0/1.0, then thresholded
Result: May become 0.5 (ambiguous)
Example: Selection may become partial
Recommendation: Recalculate booleans after operations

Vector Attributes:
Interpolation: Component-wise average
Result: Smooth vector field
Example: Normals blend (may need renormalization)

Color Attributes:
Interpolation: RGB averaged, alpha averaged
Result: Color blending
Example: Smooth color gradients
Good for: Vertex colors, painted attributes
```

## Manual Domain Conversion

### Evaluate on Domain Node

**Node Purpose**
- Explicitly evaluates a field in the context of a specific domain.
- Freezes the evaluation of a field to a specific geometry component.
- Used to optimize performance by calculating a field once in a lower-resolution domain.
- Overrides implicit automatic conversion for precise control.

**Target Domain Specification**
- Select the desired output domain (Point, Edge, Face, Corner, Spline, Instance).
- The input field is immediately evaluated and interpolated to this domain.
- **Important**: This node forces the field to resolve. It is effectively a "Cast" or "Convert" operation.

**Interpolation Behavior**
- **Standard Interpolation Only**: This node applies the default domain conversion rules (see "Domain Conversion Rules Table").
- **Aggregation (Many → Few)**: Applies a weighted **Average** (e.g., Point → Face averages the vertices).
- **Distribution (Few → Many)**: Applies **Duplication** (e.g., Face → Corner copies the face value).
- **Limitation**: This node does *not* support Min, Max, or Sum aggregation. For statistical aggregation (e.g., "Max value per face"), use the **Accumulate Field** node or topology-based sampling.

**Use Cases and Examples**

*Example 1: Smoothing Noisy Data (Blur)*
```
Use Case: Create a "blurred" version of a color attribute
Setup:
  - Input: Noisy Vertex Colors (Point Domain)
  - Step 1: Evaluate on Domain (Target: Face)
    - Effect: Averages vertices to faces (slight blur)
  - Step 2: Evaluate on Domain (Target: Point)
    - Effect: Averages faces back to vertices (more blur)
  - Result: A smooth, locally averaged color map
```

*Example 2: converting for Export*
```
Use Case: Preparing attributes for a format that only supports Face data
Setup:
  - Attribute: Weight Map (Point Domain)
  - Evaluate on Domain node:
    - Target: Face
  - Store Named Attribute (Face Domain)
  - Result: Data is explicitly averaged to faces, ready for export or face-based masking.
```

*Example 3: Explicit Coordinate Evaluation*
```
Use Case: Locking texture coordinates to vertices before deformation
Setup:
  - Input: Position Field (changes if mesh deforms)
  - Evaluate on Domain (Target: Point)
  - Result: The position is evaluated *at that specific moment* in the graph.
  - Note: This acts similarly to Capture Attribute but for immediate field usage.
```

### Evaluate at Index Node

**Node Purpose**
- Read specific element value by index
- Direct access without domain conversion
- Precise control over which element to sample
- Returns single value for single index

**Point-Specific Evaluation**
- Read specific point attribute by point index
- Index input: Integer specifying which point (0-based)
- Value output: Attribute value at that point
- Use case: Accessing neighbor points, specific vertex data

**Edge-Specific Evaluation**
- Read specific edge attribute by edge index
- Useful for edge properties
- Edge index order matches internal geometry order
- Use case: Edge crease reading, edge-based logic

**Face-Specific Evaluation**
- Read specific face attribute by face index
- Material index reading
- Face selection states
- Use case: Face property queries, material analysis

**Corner-Specific Evaluation**
- Read specific corner attribute by corner index
- UV coordinate reading
- Corner-specific data access
- Use case: UV analysis, corner property queries

**Reading Exact Domain Values**
- No aggregation performed
- Direct element access
- Index must be valid for geometry
- Out-of-range index behavior: undefined (may return 0 or error)

**Use Cases**

*Example 1: Neighbor Point Access*
```
Setup: Reading position of adjacent point
  - Current point index: Index node
  - Neighbor point index: Index + 1 (or topology query)
  - Evaluate at Index:
    - Index: Neighbor index
    - Domain: Point
    - Attribute: Position
  - Result: Position of neighbor point
```

*Example 2: Material Index Query*
```
Setup: Check specific face's material
  - Face index: Known or calculated
  - Evaluate at Index:
    - Index: Face index
    - Domain: Face
    - Attribute: Material Index
  - Result: Integer material slot index
```

*Example 3: UV Coordinate Reading*
```
Setup: Read UV at specific corner
  - Corner index: Calculated from topology
  - Evaluate at Index:
    - Index: Corner index
    - Domain: Corner
    - Attribute: UVMap
  - Result: UV coordinate vector
```

### Attribute Transfer via Sample Nodes

**Sample Nearest**
- Finds the index of the closest element in target geometry
- Works across different geometry objects
- Returns index of nearest element (use **Sample Index** to get value at index)
- Use cases:
  - Projecting data from one mesh to another
  - Snapping to nearest surface
  - Proximity-based data transfer

**Sample Index**
- Direct index-based sampling
- Fast and precise
- No proximity calculation (if applicable, use **Sample Nearest** for proximity aspect)
- Use cases:
  - When index is known or calculated
  - Topology-based access
  - Sequential data reading

**Domain Considerations**
- Sample Nearest respects domain
- Can sample Point, Face, Edge, Corner
- Source and target domains can differ
- Automatic conversion happens if needed
- Example: Sampling Face domain attribute from Point domain geometry
  - Point finds nearest face
  - Face attribute value returned

**Cross-Geometry Transfer**

*Example 1: Surface Projection*
```
Setup: Project point cloud onto surface
  - Source: Point cloud positions
  - Target: Mesh surface
  - Sample Nearest Surface:
    - Sample Position: Point cloud positions
    - Target: Mesh
    - Attribute: Normal
  - Result: Each point gets normal from nearest face
```

*Example 2: Color Transfer*
```
Setup: Transfer vertex colors between meshes
  - Source: Mesh A with vertex colors (Point domain)
  - Target: Mesh B (different topology)
  - Sample Nearest:
    - Sample Position: Mesh B positions
    - Source: Mesh A
    - Attribute: Color (Point domain)
  - Result: Mesh B vertices colored from nearest Mesh A vertex
```

*Example 3: Data from Collection*
```
Setup: Instance colors from collection object
  - Source: Collection object with color attribute
  - Target: Instance positions
  - Sample Nearest:
    - Sample Position: Instance positions
    - Source: Collection geometry
    - Attribute: Color
  - Result: Each instance gets color from nearest point
```

## Domain Visualization Guide

### Visualizing Domains in Viewport

**Using Viewer Node with Spreadsheet**

*Setup Process:*
1. Place Viewer node after operation you want to inspect
2. Connect geometry output to Viewer
3. Open Spreadsheet editor
4. Spreadsheet automatically shows viewed geometry
5. Use domain dropdown to switch between domains

**WARNING**: View Domain Interpolation
- The Spreadsheet displays data evaluated in the *selected* View Domain.
- If you view a Point attribute while the Spreadsheet is set to Face, you see the interpolated average, *not* the raw data.
- Make sure you set the Spreadsheet's domain dropdown to the attribute's native domain to verify actual values rather than the (often misleading) results of an (unexpected and undesirable) implicit conversion.

*What to Look For:*
- Element count in each domain
- Attribute values per element
- Verification of expected conversions
- Unexpected aggregation results

**Color-Coding by Domain**

*Technique 1: Store Color by Domain*
```
Setup: Visualize which domain has which values
  - Create color attribute in specific domain
  - Use domain-specific colors:
    - Point domain: Blue
    - Face domain: Green
    - Edge domain: Red
    - Corner domain: Yellow
  - Viewer + Spreadsheet to verify
  - Visual confirmation in viewport
```

*Technique 2: Index-Based Coloring*
```
Setup: Color by element index in domain
  - Read Index in target domain
  - Convert index to color (Index / Count → Hue)
  - Store as color attribute
  - Visual gradient shows element order
```

**Understanding Domain Distribution**

*Checking Point Domain:*
- Count should match vertex count
- Each vertex listed once
- Position attribute always present
- Index sequential from 0

*Checking Face Domain:*
- Count should match polygon count
- Each face listed once
- Material Index usually present
- Index may not match visual order

*Checking Face Corner Domain:*
- Count = sum of vertices of all faces
- Quad has 4 corners
- Triangle has 3 corners
- Same vertex appears multiple times (once per adjacent face)
- Critical for UV attributes

*Checking Edge Domain:*
- Count = number of edges
- Includes both visible and hidden edges
- Boundary edges vs internal edges
- Crease attribute may be present

**Debugging Domain Issues**

*Issue: Expected attribute not appearing*
- Check correct domain selected in Spreadsheet dropdown
- Verify attribute name spelling
- Confirm attribute exists in that domain
- Check if attribute was created/stored properly

*Issue: Unexpected value count*
- Verify domain matches expected element count
- Point count ≠ Face count ≠ Corner count
- Corner count = sum of face vertex counts
- Instance count separate from mesh element counts

*Issue: Values look wrong*
- Check if aggregation occurred (averaging)
- Verify correct domain for semantic meaning
- Confirm conversion method used (Average vs Min vs Max)
- Test with known simple geometry (cube, plane)

### Domain Indicators in Spreadsheet

**Point Data View**
- **Column Headers**: Attribute names
- **Rows**: One per vertex/point
- **Index Column**: Sequential 0-based point index
- **Position Column**: Always present (XYZ coordinates)
- **Custom Attributes**: Any point domain attributes
- **Count Display**: Shows total point count

**Edge Data View**
- **Vertex Index Columns**: Shows edge endpoints
  - Vertex 1: First endpoint index
  - Vertex 2: Second endpoint index
- **Index Column**: Edge index (0-based)
- **Edge Attributes**: Crease, custom edge data
- **Count Display**: Total edge count
- **Boundary Indicator**: Can infer from vertex connectivity

**Face Data View**
- **Index Column**: Face index (0-based)
- **Vertex Count Column**: How many vertices in face (3=tri, 4=quad, 5+=ngon)
- **Material Index Column**: Which material slot (if present)
- **Shade Smooth Column**: Boolean smooth/flat
- **Face Attributes**: Any face domain attributes
- **Count Display**: Total face count

**Face Corner Data View**
- **Index Column**: Corner index (0-based)
- **Vertex Index Column**: Which vertex this corner represents
- **Face Index Column**: Which face this corner belongs to
- **UV Columns**: UV coordinates (if UV map present)
- **Custom Attributes**: Corner domain attributes
- **Count Display**: Total corner count (usually highest)
- **Important**: Multiple corners per vertex visible

**Spline Data View** (Curves)
- **Index Column**: Spline index (0-based)
- **Point Count Column**: How many points in this spline
- **Cyclic Column**: Boolean closed/open
- **Resolution Column**: Evaluation points count
- **Spline Attributes**: Per-spline attributes
- **Count Display**: Number of splines in curve object

**Instance Data View**
- **Index Column**: Instance index (0-based)
- **Position Columns**: Instance location (XYZ)
- **Rotation Columns**: Instance orientation
- **Scale Columns**: Instance scale (XYZ)
- **Instance ID Column**: Stable identifier
- **Custom Attributes**: Instance domain attributes
- **Count Display**: Total instance count

**Interpreting Values**

*Position Attribute (Vector):*
- Three columns: Position X, Position Y, Position Z
- Values in Blender units
- World space coordinates (object transform applied)
- Useful for spatial queries

*Color Attribute (Color):*
- Four columns: Color R, Color G, Color B, Color A
- Values 0.0 to 1.0 (can exceed for HDR)
- Alpha usually 1.0 if not explicitly set
- Visualize in viewport with appropriate shading

*Boolean Attribute (Boolean):*
- Single column showing True/False
- Selection attributes typically boolean
- Used for masking operations
- Can be converted to float (0.0/1.0)

*Integer Attribute (Integer):*
- Single column showing whole numbers
- Index, ID, Material Index typically integer
- No decimal places
- Can be large positive or negative

*Float Attribute (Float):*
- Single column showing decimal numbers
- Most numerical data
- Can be very small or very large
- Precision depends on calculation

## Common Domain Conversion Scenarios

### Scenario 1: UV Map Storage (Face Corner Domain)

**Problem**: UV coordinates must be stored per-corner, not per-vertex
- **Reason**: Vertices at UV seams need different UV values per adjacent face
- **Solution**: Always use Face Corner domain for UV attributes

**Why This Matters**:
- UV mapping requires ability to have different texture coordinates at same vertex for different faces
- Corner domain provides per-face-vertex data storage
- Point domain would average UV values, destroying seams
- Material texture mapping reads UV from corner domain

**Example**:
```
Cube Corner Scenario:
- Cube vertex at corner touches 3 faces
- Each face needs different UV coordinate for that vertex
- 3 different corner values at same vertex position
- Face 1 corner: UV (0.0, 0.0)
- Face 2 corner: UV (1.0, 0.0)
- Face 3 corner: UV (0.0, 1.0)
- Same vertex, different UVs based on which face
```

**Conversion Implications**:
- **Point → Face Corner** (distribution): Works, copies point UV to all corners at vertex
  - Result: No UV seams (all corners at vertex have same UV)
  - Use case: When seamless UV unwrap desired
  
- **Face Corner → Point** (aggregation): Destroys seams
  - Result: Averaged UV values
  - All corners at vertex collapse to single value
  - UV seams disappear
  - Only acceptable if seams not needed

**Anti-Pattern: Point Domain for UVs**
```
Wrong Approach:
- Store UV in Point domain
- Result: Cannot have UV seams
- Texture stretches incorrectly at seams
- Material looks broken

Correct Approach:
- Store UV in Face Corner domain
- Each corner can have independent UV
- Seams work correctly
- Professional UV unwrapping possible
```

**Implementation**:
```
Proper UV Creation:
1. Calculate UV coordinates (can be in Point domain initially)
2. Convert Point → Face Corner (distribution)
3. Store Named Attribute "UVMap" in Corner domain
4. Modify specific corners for seams if needed
5. Use in material with UV Map node
```

### Scenario 2: Material Assignment (Face Domain)

**Problem**: Need to assign different materials to different faces
- **Data**: Material Index attribute must be per-face
- **Solution**: Store Material Index in Face domain

**Why This Matters**:
- Set Material node operates on Face domain
- Material slots indexed per face
- Entire face gets one material
- Cannot have different materials on same face (corners share material)

**Example**:
```
Multi-Material Object:
- Building with walls, windows, roof
- Wall faces: Material Index = 0
- Window faces: Material Index = 1
- Roof faces: Material Index = 2
- Each face has integer material index
```

**Conversion Pitfalls**:
- **Point → Face** with Average: Nonsensical
  - Vertices have material indices 1, 2, 3, 4
  - Face gets average: 2.5
  - Material Index 2.5 doesn't exist
  - Result: Incorrect material or error

**Correct Workflow**:
```
Material Assignment:
1. Create selection for each material zone (can be Point, Face, or other)
2. Convert selection to Face domain if needed
3. Use Compare/Boolean logic to create material index values
4. Store directly in Face domain as Material Index attribute
5. Use Set Material node (operates in Face domain)
```

**Anti-Pattern: Material Index in Point Domain**
```
Wrong:
- Calculate material index per vertex
- Store in Point domain
- Attempt to use with Set Material
- Automatic conversion averages nonsensically

Correct:
- Calculate material zone selection in any domain
- Convert to Face domain
- Calculate integer material index in Face domain
- Store Material Index in Face domain
- No averaging occurs
```

### Scenario 3: Vertex Colors from Face Data

**Problem**: Have per-face colors, need smooth vertex colors
- **Solution**: Face → Point conversion with Average aggregation

**Process**:
```
Workflow:
1. Face colors stored in Face domain
2. Convert to Point domain using Evaluate on Domain
3. Aggregation method: Average
4. Each vertex gets average of adjacent face colors
5. Result: Smooth color gradients at vertices
```

**Example**:
```
Simple Case:
- Red face and Blue face share vertex
- Vertex receives: (Red + Blue) / 2 = Purple
- Smooth color transition between faces

Complex Case:
- Vertex shared by 4 faces: Red, Green, Blue, Yellow
- Vertex color = (Red + Green + Blue + Yellow) / 4
- Mixed color based on all adjacent faces
```

**Why Average**:
- Smooth, natural-looking gradients
- No harsh boundaries
- Visually pleasing interpolation
- Standard for vertex color workflows

**Alternative Aggregation Methods**:
```
Using Min:
- Vertex gets darkest adjacent face color
- Preserves dark regions
- May create harsh transitions

Using Max:
- Vertex gets brightest adjacent face color
- Preserves bright regions
- May create harsh transitions

Using First:
- Unpredictable (depends on internal order)
- Not recommended for colors
```

### Scenario 4: Height-Based Selection (Point Domain)

**Problem**: Select geometry by Z height
- **Data**: Position.Z is in Point domain
- **Usage**: Compare in Point domain for vertex selection

**Workflow**:
```
Point Domain Selection:
1. Read Position (Point domain, vector)
2. Separate XYZ → Extract Z component
3. Compare Z > threshold (still Point domain)
4. Result: Boolean selection per vertex
5. Use directly for Point-based operations
```

**Converting to Face Selection**:
```
If Face Selection Needed:
1. Point selection (True/False per vertex)
2. Evaluate on Domain:
  - Source: Point
  - Target: Face
  - Method: Average
3. Face selected if average of vertex selections ≥ 0.5
4. Result: Face selected if most/all vertices selected
```

**Aggregation Behavior**:
```
Face with Mixed Vertex Selection:
- 4 vertices: True, True, False, False
- Average: (1 + 1 + 0 + 0) / 4 = 0.5
- Face selection = 0.5
- If using as boolean: 0.5 = True (non-zero)
- If comparing: (0.5 ≥ 0.5) = True

Face with Partial Selection:
- 4 vertices: True, False, False, False
- Average: 0.25
- Face selection = 0.25
- Boolean: True (non-zero)
- Compare (≥ 0.5): False
```

**Selection Threshold Strategy**:
```
Conservative (all vertices must be selected):
- Use Min aggregation
- Face selected only if all vertices True
- Strictest selection

Moderate (most vertices selected):
- Use Average aggregation
- Compare result ≥ 0.5
- Face selected if majority of vertices selected

Liberal (any vertex selected):
- Use Max aggregation
- Face selected if any vertex True
- Broadest selection
```

### Scenario 5: Edge Crease from Face Analysis

**Problem**: Determine edge sharpness from face angle
- **Data**: Edge Angle provides angle in Edge domain
- **Usage**: Store crease value in Edge domain

**Correct Workflow**:
```
Edge Crease Assignment:
1. Edge Angle node → reads edge data (Edge domain)
2. Compare angle > threshold (Edge domain)
3. Map Range to 0-1 crease value (Edge domain)
4. Store Named Attribute "crease" (Edge domain)
5. No conversion needed - all operations in Edge domain
6. Subdivision Surface reads crease from Edge domain
```

**Why No Conversion**:
- Edge Angle naturally in Edge domain
- Crease attribute naturally in Edge domain
- Operations match domains perfectly
- Most efficient workflow

**Example**:
```
Sharp Edge Detection:
1. Edge Angle > 30° → crease = 1.0 (sharp)
2. Edge Angle < 30° → crease = 0.0 (smooth)
3. Stored in Edge domain
4. Subdivision Surface preserves sharp edges
```

**Anti-Pattern: Converting to Other Domains**
```
Unnecessary Conversion:
- Edge Angle in Edge domain
- Convert to Face domain (why?)
- Convert back to Edge domain for crease
- Extra computation, same result
- Just stay in Edge domain
```

### Scenario 6: Per-Spline Effects (Curve Geometry)

**Problem**: Apply different effects to each curve in multi-curve object
- **Solution**: Create attribute in Spline domain

**Setup**:
```
Per-Spline Variation:
1. Random Value with Spline Index as seed (Spline domain)
2. Store color/value attribute (Spline domain)
3. Convert Spline → Point (distribution)
4. All points in spline get same value
5. Different splines have different values
```

**Example**:
```
5 Curves with Random Colors:
- Spline domain: 5 color values (one per curve)
  - Spline 0: Red
  - Spline 1: Green
  - Spline 2: Blue
  - Spline 3: Yellow
  - Spline 4: Purple
- Convert to Point domain (distribution)
- All points in Spline 0: Red
- All points in Spline 1: Green
- And so on...
- Result: Each curve uniformly colored, different from others
```

**Distribution Behavior**:
```
Spline → Point Conversion:
- Spline value: 5.0
- Spline has 20 points
- After distribution: all 20 points have value 5.0
- No aggregation (one-to-many)
- Pure value duplication
```

**Use Cases**:
- Hair strand variation (each strand different color)
- Cable bundle (each cable different thickness)
- Grass blades (each blade different height)
- Decorative curves (each curve different pattern)

### Scenario 7: Instance Variation

**Problem**: Each instance needs unique properties
- **Solution**: Store attributes in Instance domain

**Instance Attributes**:
```
Instance Domain Data:
- Position (XYZ location)
- Rotation (orientation)
- Scale (XYZ size)
- ID (stable identifier)
- Custom attributes (color, type, etc.)
```

**Example**:
```
Instance Color Variation:
1. Random Value with Instance ID seed (Instance domain)
2. Create color from random (Instance domain)
3. Store as instance attribute (Instance domain)
4. Use in shader via attribute node
5. Each instance renders with different color
```

**Conversion to Point Domain**:
```
Instances to Points:
- Instances to Points node
- Instance positions → Point positions
- Other instance attributes CAN transfer if specified
- Instance geometry lost (only positions remain)
- Use case: Converting instance scatter to point cloud
```

**Why Instance Domain**:
- Each instance independent
- Efficient for large counts
- Attributes travel with instance
- No geometry duplication
- Performance optimized

### Scenario 8: Smooth Normals (Corner vs Point)

**Problem**: Smooth vs sharp edges in rendering
- **Split Normals**: Stored in Face Corner domain (different normals per face at shared vertex)
- **Vertex Normals**: Point domain (averaged from adjacent faces)

**Corner Domain Normals (Split)**:
```
Sharp Edge Scenario:
- Vertex at 90° corner
- Face 1 normal points up
- Face 2 normal points right
- Corner 1 (Face 1 side): Normal = Up
- Corner 2 (Face 2 side): Normal = Right
- Different normals at same vertex position
- Result: Sharp edge in render
```

**Point Domain Normals (Smooth)**:
```
Smooth Edge Scenario:
- Same vertex at 90° corner
- Average all adjacent face normals
- Result: Normal pointing diagonal (Up+Right)/2
- All corners at vertex get same normal
- Result: Smooth shading across edge
```

**Conversion Effects**:
```
Corner → Point (averaging):
- Averages all split normals at vertex
- Unifies different corner normals
- Creates smooth shading
- Loses sharp edge information

Point → Corner (distribution):
- Distributes vertex normal to all corners
- All corners at vertex get same normal
- Enforces smooth shading
- Cannot create sharp edges
```

**Auto-Smooth Workflow**:
```
Angle-Based Sharp/Smooth:
1. Edge Angle calculation (Edge domain)
2. If angle > threshold: Mark edge sharp
3. Split normals at sharp edges (Corner domain)
4. Average normals at smooth edges (Point domain via corners)
5. Store in Corner domain for rendering
6. Result: Sharp where needed, smooth elsewhere
```

### Scenario 9: Painting Density Map (Point to Face)

**Problem**: Painted vertex weights need to control face-based scattering
- **Data**: Vertex colors/weights in Point domain (painted)
- **Solution**: Point → Face conversion with Average

**Workflow**:
```
Painted Density to Scattering:
1. Paint vertex weights (Point domain)
  - Dark areas: low density
  - Bright areas: high density
2. Convert Point → Face (Average aggregation)
  - Face density = average of vertex weights
3. Use as Distribute Points on Faces density
  - Distribute Points operates in Face domain
  - Density controls point distribution
4. Result: More points in painted areas
```

**Example**:
```
Grass Distribution:
1. Paint vertices:
  - Path areas: Black (weight 0.0)
  - Grass areas: White (weight 1.0)
  - Transition areas: Gray (0.0-1.0)
2. Quad face with vertices (0.0, 0.5, 1.0, 1.0)
  - Average: (0.0 + 0.5 + 1.0 + 1.0) / 4 = 0.625
3. Face gets 62.5% density
4. Gradual transition from path to grass
```

**Why Average**:
- Smooth density transitions
- No harsh boundaries
- Respects painting gradients
- Natural-looking distribution

**Alternative: Direct Face Painting**:
```
Face Domain Workflow (alternative):
1. Paint directly in Face domain
2. No conversion needed
3. Less smooth transitions (per-face steps)
4. Faster workflow but less control
5. No gradients within faces
```

### Scenario 10: Edge Selection from Vertex Selection

**Problem**: Selected vertices should select their edges
- **Data**: Selection in Point domain (boolean field)
- **Conversion**: Point → Edge

**Aggregation Behavior**:
```
Edge Selection Logic:
Edge selected if BOTH endpoints selected

Using Average:
- Vertex A selected (1.0), Vertex B selected (1.0)
- Edge average: (1.0 + 1.0) / 2 = 1.0 → selected
- Vertex A selected (1.0), Vertex B not (0.0)
- Edge average: (1.0 + 0.0) / 2 = 0.5 → partially selected
- If threshold at 0.5: edge selected if either vertex selected
- If threshold > 0.5: edge selected only if both vertices selected

Using Min:
- Edge gets minimum of endpoint selections
- Min = 1.0 only if both vertices = 1.0
- Ensures BOTH endpoints selected
- Strictest edge selection
- Recommended for "both endpoints" logic

Using Max:
- Edge gets maximum of endpoint selections
- Max = 1.0 if either vertex = 1.0
- Ensures AT LEAST ONE endpoint selected
- Most liberal edge selection
```

**Example**:
```
4 Vertices, 2 Selected:
- Vertex 0: True
- Vertex 1: True  
- Vertex 2: False
- Vertex 3: False

Edge 0-1 (both selected):
- Average: 1.0 → Selected
- Min: 1.0 → Selected
- Max: 1.0 → Selected

Edge 1-2 (one selected):
- Average: 0.5 → Selected if threshold ≤ 0.5
- Min: 0.0 → Not selected
- Max: 1.0 → Selected

Edge 2-3 (neither selected):
- Average: 0.0 → Not selected
- Min: 0.0 → Not selected
- Max: 0.0 → Not selected
```

**Best Method**:
```
For "Both Endpoints" Logic:
- Use Min aggregation
- Guarantees both vertices selected
- Clean, predictable behavior

For "Either Endpoint" Logic:
- Use Max aggregation
- Edge selected if any vertex selected
- Broader selection
```

## Domain-Specific Attributes

### Point Domain Attributes

**Position** ("position")
- **Data Type**: Vector (Float, 3D)
- **Description**: XYZ coordinates of each point in space
- **Always Present**: Yes, fundamental to point geometry
- **Read/Write**: Read-Write
- **Use Cases**: 
  - Reading locations for spatial operations
  - Moving geometry via Set Position
  - Creating position-dependent fields
  - Displacement and deformation
- **Notes**: Most fundamental attribute, required for geometry to exist

**Radius** ("radius")
- **Data Type**: Float
- **Description**: Point size/thickness
- **Always Present**: No (defaults if not set)
- **Read/Write**: Read-Write
- **Use Cases**:
  - Point cloud display size
  - Controlling instance size
  - Metaball-like blending radius
- **Default Value**: 0.05 (Blender units)
- **Notes**: Affects point cloud rendering and some operations

**ID** ("id")
- **Data Type**: Integer
- **Description**: Stable identifier that persists through operations
- **Always Present**: No (can be created)
- **Read/Write**: Read-Write
- **Use Cases**:
  - Stable random values per element
  - Tracking elements through transformations
  - Creating consistent variation
- **Notes**: Unlike Index, ID persists when geometry changes
- **Creation**: May need Store Named Attribute to create

**Custom Point Data**
- Any attribute stored in Point domain
- Typical types: Float, Vector, Color, Boolean
- Created via Store Named Attribute
- Accessible via Named Attribute node
- Examples: height, temperature, age, damage

**Vertex Group Weights (Legacy)**
- Old system for storing per-vertex weights
- Now handled as generic Float attributes
- Still supported for compatibility
- Modern workflow uses named attributes

### Edge Domain Attributes

**Crease** ("crease")
- **Data Type**: Float (0.0 to 1.0)
- **Description**: Edge crease value for Subdivision Surface modifier
- **Always Present**: No (defaults to 0.0 if not set)
- **Read/Write**: Read-Write
- **Use Cases**:
  - Hard edges in subdivision surfaces
  - Preserving sharp features
  - Angle-based crease assignment
- **Value Range**: 0.0 = smooth, 1.0 = sharp
- **Notes**: Only affects Subdivision Surface modifier

**Edge Smooth/Sharp**
- **Data Type**: Boolean
- **Description**: Whether edge is smooth or sharp for rendering
- **Always Present**: No (defaults to smooth)
- **Read/Write**: Read-Write
- **Use Cases**:
  - Manual sharp edge control
  - Hard surface modeling
  - Crease marking
- **Notes**: Affects shading, separate from crease value

**Custom Edge Data**
- Any attribute stored in Edge domain
- Less commonly used than Point or Face
- Typical types: Float, Boolean
- Examples: edge length, edge type, selection state

### Face Domain Attributes

**Material Index** ("material_index")
- **Data Type**: Integer
- **Description**: Index into object's material slots (0-based)
- **Always Present**: Yes (defaults to 0)
- **Read/Write**: Read-Write
- **Use Cases**:
  - Assigning different materials to different faces
  - Multi-material objects
  - Material-based selection
- **Value Range**: 0 to (number of material slots - 1)
- **Notes**: References material slot position, not material directly

**Shade Smooth** ("shade_smooth")
- **Data Type**: Boolean
- **Description**: Whether face uses smooth shading (True) or flat (False)
- **Always Present**: Yes (defaults based on object settings)
- **Read/Write**: Read-Write
- **Use Cases**:
  - Controlling shading per face
  - Procedural smooth shading
  - Angle-based auto-smooth
- **Notes**: Affects display only, not actual geometry

**Face Normals** (Calculated)
- **Data Type**: Vector (Float, 3D, normalized)
- **Description**: Direction perpendicular to face surface
- **Always Present**: Yes (auto-calculated)
- **Read/Write**: Read-only (recalculated from geometry)
- **Use Cases**:
  - Surface orientation detection
  - Lighting calculations
  - Selection by facing direction
- **Notes**: Automatically computed from vertex positions

**Custom Face Data**
- Any attribute stored in Face domain
- Common types: Float, Vector, Color, Boolean, Integer
- Created via Store Named Attribute
- Examples: face area, face type, selection, random ID

### Face Corner Domain Attributes

**UV Map** (default name: "UVMap", can have custom names)
- **Data Type**: Vector 2D (stored as 3D with Z=0)
- **Description**: UV coordinates for texture mapping
- **Always Present**: No (created when needed)
- **Read/Write**: Read-Write
- **Use Cases**:
  - Texture coordinate storage
  - Material UV mapping
  - Procedural UV generation
  - Multiple UV layers (with different names)
- **Notes**: MUST be in Corner domain for UV seams to work
- **Critical**: Most important corner attribute

**Split Normals** (Modern)
- **Data Type**: Vector (Float, 3D, normalized)
- **Description**: Per-face-vertex normals (allows sharp edges)
- **Always Present**: Can be created
- **Read/Write**: Read-Write
- **Use Cases**:
  - Sharp edges while keeping mesh smooth
  - Custom normal direction per face
  - Hard surface modeling
- **Notes**: Allows different normals at same vertex for different faces

**Vertex Colors (Modern Storage)**
- **Data Type**: Color (RGBA)
- **Description**: Color data stored per corner
- **Always Present**: No (created when painted/generated)
- **Read/Write**: Read-Write
- **Use Cases**:
  - Painting color onto geometry
  - Corner-based color variation
  - Masking and selection via color
  - UV seam-aware coloring
- **Notes**: Modern vertex color system uses corner domain
- **Legacy**: Old system used Point domain (less flexible)

**Custom Corner Data**
- Any attribute stored in Corner domain
- Critical for per-face-vertex data
- Typical types: Vector, Color, Float
- Examples: per-corner weights, per-corner IDs

### Spline Domain Attributes (Curves)

**Cyclic** ("cyclic")
- **Data Type**: Boolean
- **Description**: Whether spline forms a closed loop
- **Always Present**: Yes (defaults to False)
- **Read/Write**: Read-Write
- **Use Cases**:
  - Creating closed vs open curves
  - Detecting loops
  - Conditional operations on closed curves
- **Notes**: Affects curve connectivity and fill operations

**Resolution** ("resolution")
- **Data Type**: Integer
- **Description**: Number of evaluated points along spline
- **Always Present**: Yes (defaults to 12)
- **Read/Write**: Read-Write
- **Use Cases**:
  - Curve detail control
  - LOD for curves
  - Adaptive resolution
- **Value Range**: 1 to very large (practical limit ~100-200)
- **Notes**: Higher values = smoother curves but more computation

**Custom Spline Data**
- Any attribute stored in Spline domain
- Affects entire spline uniformly
- Typical types: Float, Color, Boolean
- Examples: spline type, spline age, random seed

**Note on Curve Point Attributes**:
- Curve points have Point domain attributes
- Position, Radius, Tilt are per-point
- Handle positions (Bézier) are per-point
- Different from Spline domain (per-curve)

### Instance Domain Attributes

**Instance Position**
- **Data Type**: Vector (Float, 3D)
- **Description**: Location of each instance
- **Always Present**: Yes
- **Read/Write**: Read-Write
- **Use Cases**: Instance placement, distribution

**Instance Rotation**
- **Data Type**: Rotation (Euler or Quaternion)
- **Description**: Orientation of each instance
- **Always Present**: Yes (defaults to no rotation)
- **Read/Write**: Read-Write
- **Use Cases**: Instance orientation, variation

**Instance Scale**
- **Data Type**: Vector (Float, 3D)
- **Description**: Size of each instance (XYZ)
- **Always Present**: Yes (defaults to 1.0, 1.0, 1.0)
- **Read/Write**: Read-Write
- **Use Cases**: Instance size variation, scaling

**Instance ID**
- **Data Type**: Integer
- **Description**: Stable identifier for each instance
- **Always Present**: Can be created
- **Read/Write**: Read-Write
- **Use Cases**: Stable random seeds, tracking instances

**Custom Instance Data**
- Any attribute stored in Instance domain
- Travels with instance through operations
- Typical types: Float, Vector, Color, Integer
- Examples: instance type, age, color, custom properties


## Performance Considerations

### Attribute Memory in Rendering

**Memory per Attribute**:
```
Point Domain Attribute Memory:
- Float: 4 bytes per point
- Vector: 12 bytes per point (3 floats)
- Color: 16 bytes per point (4 floats: RGBA)
- Integer: 4 bytes per point
- Boolean: 1 byte per point

Example: 1 million points
- Float attribute: 4 MB
- Vector attribute: 12 MB
- Color attribute: 16 MB
- 10 attributes: potentially 120+ MB
```

**Face Corner Domain Memory**:
```
Corner Attributes:
- Highest count domain (sum of all face vertex counts)
- Quad mesh: 4 corners per face
- Triangle mesh: 3 corners per face
- Memory scales with face count × average vertices per face

Example: 500k quad faces = 2 million corners
- UV Map (Vector 2D): 16 MB
- Vertex Color (Color): 32 MB
- Multiple UV maps multiply memory
```

**Optimization Strategies**:
```
Memory Reduction:
1. Remove unused attributes (Remove Named Attribute)
2. Use appropriate data types (Boolean < Integer < Float < Vector)
3. Store in lowest-count domain when possible
4. Avoid duplicate attributes
5. Clean up anonymous attributes
6. Use instancing instead of realized geometry
```

### Computation Cost of Auto-Calculated Attributes

**Face Normals**:
- Recalculated whenever geometry changes
- Fast computation (cross product of edges)
- Negligible performance impact
- Cannot be disabled (always present)

**Vertex Normals**:
- Averaged from adjacent face normals
- Recalculated on geometry change
- Slightly more expensive than face normals
- Still very fast

**Edge Angles**:
- Calculated from adjacent face normals
- Only when accessed via Edge Angle node
- Lazy evaluation (only computed if used)
- Moderate cost for dense meshes

**Performance Impact**:
```
Auto-Calculated Costs (relative):
- Position: No cost (fundamental data)
- Face Normals: Very low (always needed)
- Vertex Normals: Low (cached)
- Edge Angles: Moderate (on-demand)
- Topology Queries: Moderate to High (depends on complexity)
```

### When to Use vs Avoid Certain Attributes

**Use Attributes When**:
```
Good Use Cases:
1. Data needs to persist through operations
2. Data accessed in multiple places
3. Data needed in shaders
4. Data represents inherent property of geometry
5. Data changes infrequently
6. Explicit data flow improves clarity
```

**Avoid Attributes When**:
```
Better Alternatives:
1. Data used once immediately → Use direct connection (anonymous)
2. Temporary calculation → Use Capture Attribute (anonymous)
3. Data changes every frame → Calculate on-demand
4. Simple constant value → Use value node
5. Large intermediate result → Lazy evaluation better
```

**Attribute vs Calculation Trade-off**:
```
Store Attribute:
+ Computed once, used many times
+ Persists through operations
+ Can be saved with geometry
+ Accessible in shaders
- Uses memory
- Slower initial computation if complex

Calculate On-Demand:
+ No memory overhead
+ Always up-to-date with changes
+ Lazy evaluation (only if needed)
- Recomputed every use
- Cannot access in shaders easily
- Lost if geometry modified
```

**Performance Best Practices**:
```
Optimal Attribute Usage:
1. Named attributes for persistent data
2. Anonymous (Capture) for temporary data
3. Direct connections for immediate use
4. Remove unused attributes regularly
5. Use lowest-count domain possible
6. Choose appropriate data types
7. Cache expensive calculations
8. Profile and measure impact
```

**Memory vs Speed Trade-off**:

```
Memory-Optimized Approach:
Strategy: Calculate on-demand, minimal attribute storage

Characteristics:
- Fewer stored attributes
- Calculate values when needed
- Use direct connections instead of storing
- Smaller data types where possible

Advantages:
+ Lower memory footprint
+ Less data to manage
+ Cleaner attribute list
+ Better for memory-constrained systems

Disadvantages:
- Recalculation overhead if used multiple times
- Cannot access in shaders easily
- Lost if geometry modified
- More complex node networks

When to Use:
- Limited memory available
- Attributes used once or rarely
- Temporary calculations
- Simple, fast calculations

Example:
Instead of storing:
  Position → Noise Texture → Store "height_variation"
Calculate on-demand:
  Position → Noise Texture → Use directly
(No storage, recalculated each use)
```

```
Speed-Optimized Approach:
Strategy: Pre-calculate and store, faster evaluation

Characteristics:
- More stored attributes
- Cache expensive calculations
- Pre-compute complex operations
- Store intermediate results

Advantages:
+ Faster evaluation (compute once, use many times)
+ Accessible in shaders
+ Persistent through operations
+ Simpler downstream node networks

Disadvantages:
- Higher memory usage
- More attributes to manage
- Potential for stale data
- Slower initial computation

When to Use:
- Memory plentiful
- Attributes accessed multiple times
- Expensive calculations (complex noise, raycasting)
- Shader access needed
- Frequent parameter changes

Example:
Pre-calculate and store:
  Position → Complex Noise (5 octaves) → Store "terrain_height"
  → Use "terrain_height" in multiple places
  → No recalculation overhead
  → Fast evaluation
```

**Hybrid Approach (Recommended)**:

```
Balanced Strategy:
Store expensive calculations, compute cheap ones on-demand

Decision Matrix:
┌────────────────────────┬──────────┬──────────┐
│ Calculation Type       │ Action   │ Reason   │
├────────────────────────┼──────────┼──────────┤
│ Expensive + Used Many  │ Store    │ Optimize │
│ Expensive + Used Once  │ Compute  │ Memory   │
│ Cheap + Used Many      │ Either*  │ Flexible │
│ Cheap + Used Once      │ Compute  │ Simple   │
└────────────────────────┴──────────┴──────────┘

For cheap calculations used many times:
  - Store if shader access needed
  - Compute if only used in Geometry Nodes

Examples:

Expensive Calculations (Store):
- Multi-octave noise (5+ octaves)
- Raycasting operations
- Complex curve sampling
- Nearest neighbor searches
- Volume operations
- Repeated texture lookups

Cheap Calculations (Compute):
- Simple math (add, multiply)
- Single comparison
- Vector component extraction
- Color channel separation
- Index-based lookups

Medium Calculations (Context-Dependent):
- Single-octave noise
- Map Range operations
- Color mixing
- Vector math
```

**Practical Examples**:

```
Example 1: Terrain Generation (Speed-Optimized)
Store These (Expensive, Used Multiple Times):
- "base_height" - Multi-octave noise (5 octaves)
  - Used for: Displacement, material zones, erosion
  - Calculation expensive, reuse cheap
  
- "erosion_factor" - Slope-based calculation with curve
  - Used for: Material blending, detail addition
  - Complex calculation, multiple uses

- "ambient_occlusion" - Raycast-based AO approximation
  - Used for: Darkening, material variation
  - Very expensive, definitely store

Compute On-Demand (Cheap):
- Material zone selection: Simple height comparison
- Vertex color mixing: Basic math
- Selection masks: Boolean operations

Result: Fast evaluation, acceptable memory usage
```

```
Example 2: Instanced Forest (Memory-Optimized)
Store These (Needed in Shaders):
- "tree_color" - Per-instance color variation (Instance domain)
  - Used in shader, must persist
  - Small memory footprint (Instance domain)

- "tree_type" - Integer classification (Instance domain)
  - Used for material switching
  - Tiny memory cost

Compute On-Demand:
- Tree rotation: Calculate from position + noise
  - Not stored, just applied to instances
  - Cheap calculation

- Tree scale: Simple random value
  - Calculated and used immediately
  - No storage needed

Result: Minimal memory, fast enough
```

```
Example 3: Architectural Model (Balanced)
Store These:
- "material_index" - Face domain, required attribute
  - Must be stored (Face domain requirement)
  
- "window_mask" - Boolean selection for windows
  - Used multiple times in network
  - Moderate cost, worth storing

- "floor_number" - Integer per face
  - Used for variation, accessed in shader
  - Small memory cost, useful data

Compute On-Demand:
- UV coordinates: Calculated procedurally
  - Stored as "UVMap" (Corner domain requirement)
  - But calculation not stored, only final UVs

- Temporary selections: Height-based, angle-based
  - Used once for operations
  - Not stored, recalculated if needed

Result: Organized data, efficient workflow
```

**Profiling and Measurement**:

```
How to Determine if Attribute Storage is Worth It:

1. Measure Calculation Time:
  - Use Blender's System Console timing
  - Note time for complex operations
  - Calculations > 100ms good candidates for storage

2. Measure Memory Impact:
  - Check Statistics overlay
  - Note memory before and after attribute storage
  - Acceptable if < 10-20% increase for significant speed gain

3. Count Usage Frequency:
  - How many times is value accessed?
  - Once: Probably compute on-demand
  - 2-3 times: Context-dependent
  - 4+ times: Probably store

4. Consider Workflow:
  - Frequent parameter changes? → Store expensive calculations
  - Set-and-forget setup? → More flexible
  - Animation? → Store to avoid per-frame recalculation

Decision Formula:
Store if: (Calculation Cost × Usage Count) > (Memory Cost + Storage Overhead)
```

**Attribute Lifecycle Management**:

```
Best Practices for Attribute Management:

1. Creation Phase:
  - Only create when needed
  - Use descriptive names
  - Document purpose
  - Choose appropriate data type

2. Usage Phase:
  - Access attributes efficiently
  - Minimize domain conversions
  - Cache results when reused
  - Monitor performance

3. Cleanup Phase:
  - Remove unused attributes (Remove Named Attribute)
  - Clean up temporary calculations
  - Verify what's still needed
  - Optimize attribute list

4. Documentation:
  - Frame labels noting attribute purpose
  - Comments in complex sections
  - README for project attributes
  - Version notes for changes
```

**Anonymous vs Named Attributes Trade-offs**:

```
Anonymous Attributes (Capture Attribute):

Advantages:
+ Automatic cleanup (garbage collected)
+ No naming conflicts
+ Scoped to local use
+ Faster for temporary data
+ No global namespace pollution

Disadvantages:
- Cannot access from other parts of network
- Not accessible in shaders
- Lost after node tree execution
- Cannot be saved with geometry
- No Spreadsheet visibility

Best For:
- Temporary calculations
- Preserving state through operations
- Intermediate results
- Local scope data

Example:
Capture Attribute: Store position before displacement
→ Use captured position for coloring
→ Automatically cleaned up
→ No persistent storage needed
```

```
Named Attributes (Store Named Attribute):

Advantages:
+ Persistent across operations
+ Accessible in shaders
+ Visible in Spreadsheet
+ Can be saved with geometry
+ Global access in node network
+ Cross-network communication

Disadvantages:
- Must manage namespace (naming conflicts)
- Manual cleanup required
- Persists even if not needed
- Memory overhead continues
- Needs explicit removal

Best For:
- Shader-accessible data
- Cross-operation persistence
- Saved geometry attributes
- Global data sharing
- User-facing attributes

Example:
Store Named Attribute: "UVMap"
→ Persists with geometry
→ Accessible in materials
→ Standard attribute name
→ Required persistence
```

**Strategic Attribute Planning**:

```
Workflow Planning Questions:

1. Data Lifetime:
   Q: How long does this data need to exist?
   A (Temporary): Use Capture Attribute (anonymous)
   A (Permanent): Use Store Named Attribute

2. Data Accessibility:
   Q: Where will this data be accessed?
   A (This node tree only): Anonymous acceptable
   A (Shaders, other nodes): Named required

3. Data Persistence:
   Q: Should this data save with the file?
   A (No): Anonymous or temporary named
   A (Yes): Named attribute

4. Data Scope:
   Q: How widely is this data used?
   A (Local operation): Anonymous
   A (Multiple operations): Named
   A (Cross-network): Named

5. Data Cost:
   Q: What's the memory/computation trade-off?
   A (Cheap to recalculate): Compute on-demand
   A (Expensive): Store as attribute
   A (Used in shader): Store named
```

**Common Attribute Anti-Patterns**:

```
Anti-Pattern 1: Store Everything
Problem:
- Storing every intermediate calculation
- Excessive memory usage
- Cluttered attribute namespace
- Difficult to maintain

Solution:
- Store only expensive or frequently-used calculations
- Use direct connections for single-use values
- Anonymous attributes for temporary data

Anti-Pattern 2: No Attribute Storage
Problem:
- Recalculating expensive operations repeatedly
- Poor performance
- Complex node networks
- Shader cannot access data

Solution:
- Identify expensive calculations
- Store values used multiple times
- Cache results for shader access

Anti-Pattern 3: Inconsistent Naming
Problem:
- "temp", "temp2", "value", "data"
- Unclear purpose
- Naming conflicts
- Maintenance nightmare

Solution:
- Descriptive names: "terrain_height", "window_mask"
- Consistent conventions: "prefix_description"
- Document attribute purposes

Anti-Pattern 4: Wrong Domain Storage
Problem:
- UVs stored in Point domain (no seams)
- Materials in Point domain (averaging nonsense)
- Unnecessary Corner domain usage (memory waste)

Solution:
- Understand domain requirements
- Use correct semantic domain
- See domain-specific best practices sections

Anti-Pattern 5: No Cleanup
Problem:
- Attributes accumulate over development
- Unused attributes persist
- Memory bloat
- Confusion about what's needed

Solution:
- Regular attribute audits
- Remove Named Attribute for obsolete data
- Document currently-used attributes
- Clean before finalizing project
```

**Attribute Performance Testing**:

```
Testing Methodology:

1. Baseline Measurement:
  - Note current performance (FPS, eval time)
  - Record memory usage
  - Document attribute count

2. Add Attribute:
  - Implement attribute storage
  - Measure new performance
  - Calculate difference

3. Compare Alternatives:
   Test A: Store attribute
   Test B: Compute on-demand
   Test C: Hybrid approach

4. Real-World Testing:
  - Test with typical usage patterns
  - Animate parameters
  - Check viewport responsiveness
  - Measure render time

5. Decision:
  - Compare measured differences
  - Consider workflow preferences
  - Choose based on data, not assumptions
```

**Platform-Specific Considerations**:

```
Memory-Constrained Systems (Low RAM):
- Minimize attribute storage
- Use smaller data types (Boolean < Integer < Float)
- Clean up aggressively
- Prefer computation over storage
- Monitor memory usage closely

Performance-Critical Systems (Viewport responsiveness):
- Store expensive calculations
- Cache frequently-used values
- Minimize repeated operations
- Accept higher memory usage for speed

Production Rendering (Offline):
- Optimize for accuracy over speed
- Memory less critical
- Store data for consistency
- Accept longer evaluation for quality

Real-Time/Interactive (Games, VR):
- Balance critical
- Profile extensively
- Store shader-accessed data
- Compute cheap operations on-demand
- LOD systems essential
```

## Domain-Specific Performance Characteristics

**Point Domain Performance**:

```
Operations:
✓ Very Fast:
  - Position access and modification
  - Per-vertex calculations
  - Math operations on vertex data

✓ Fast:
  - Vertex-based selection
  - Field evaluation per vertex
  - Texture sampling at vertex positions

✗ Slower:
  - Conversion to other domains (aggregation)
  - Large point counts (millions of vertices)

Memory:
- Moderate: Vertex count usually reasonable
- Attributes: ~4-16 bytes per vertex per attribute
- Total: Vertices × Attributes × Bytes

Best For:
- Position-based effects
- Per-vertex variation
- Smooth gradients
- Most general calculations
```

**Face Domain Performance**:

```
Operations:
✓ Very Fast:
  - Material assignment
  - Face-based selection
  - Per-polygon operations

✓ Fast:
  - Face area calculations
  - Face normal access
  - Subdivision operations

✗ Slower:
  - Conversion from Point (aggregation of vertices)
  - Very high polygon counts

Memory:
- Lower than Point: Typically fewer faces than vertices
- Attributes: ~4-16 bytes per face per attribute
- Efficient for per-polygon data

Best For:
- Material assignment
- Polygon-based effects
- Face-level randomization
- Scattering (Distribute Points operates here)
```

**Face Corner Domain Performance**:

```
Operations:
✓ Fast:
  - UV operations (native domain)
  - Corner-specific modifications
  - Split normal calculations

✗ Moderate:
  - Conversion to Point (aggregation, many-to-one)
  - Large corner counts

✗ Slower:
  - Very high corner counts (dense meshes)
  - Repeated corner domain operations

Memory:
- Highest: Corner count = sum of face vertex counts
- Quad mesh: ~4× face count
- Attributes: ~4-16 bytes per corner per attribute
- Can be largest memory consumer

Best For:
- UV coordinates (MANDATORY domain)
- Vertex colors with seam support
- Split normals
- Any per-face-vertex data

Critical: Only use when corner-level granularity needed
```

**Edge Domain Performance**:

```
Operations:
✓ Fast:
  - Edge-based selection
  - Crease operations
  - Edge angle calculations

✓ Moderate:
  - Topology queries
  - Edge loop operations

Memory:
- Moderate: Edge count between Face and Point typically
- Attributes: ~4-16 bytes per edge per attribute
- Reasonable memory usage

Best For:
- Edge properties (crease)
- Wireframe effects
- Topology-based operations
- Edge loops and rings

Note: Less commonly used than Point or Face
```

**Spline Domain Performance (Curves)**:

```
Operations:
✓ Very Fast:
  - Per-curve property access
  - Spline-level modifications
  - Curve-wide operations

✓ Fast:
  - Distribution to points (duplication)
  - Spline property queries

Memory:
- Minimal: Lowest element count
- Attributes: ~4-16 bytes per spline per attribute
- Very efficient for per-curve data

Best For:
- Uniform properties across entire curves
- Per-curve variation in multi-curve objects
- Curve-level classifications
- Hair/cable/grass per-strand properties

Optimization: Use spline domain for properties that should be uniform within each curve
```

**Instance Domain Performance**:

```
Operations:
✓ Extremely Fast:
  - Instance attribute access
  - Instance transform operations
  - Instance-level variation

✓ Very Fast:
  - Large instance counts (thousands to millions)
  - Instance creation and manipulation

✗ Expensive:
  - Realize Instances (converts to geometry)

Memory:
- Minimal: Instance data tiny compared to geometry
- Attributes: ~4-16 bytes per instance per attribute
- Geometry: Not duplicated (referenced)
- Most memory-efficient for repeated elements

Best For:
- Repeated geometry (trees, grass, crowds)
- Per-instance variation
- Large-scale scattering
- Performance-critical scenes

Critical: Keep as instances as long as possible
Only realize when absolutely necessary
```

### Memory Optimization Strategies by Domain

**Point Domain Optimization**:

```
Reduce Memory:
1. Remove unused point attributes
  - Remove Named Attribute node
  - Clean up after operations

2. Use appropriate data types
  - Boolean (1 byte) instead of Float (4 bytes) for flags
  - Integer (4 bytes) instead of Vector (12 bytes) when possible

3. Avoid redundant attributes
  - Don't store calculated values that can be recomputed
  - Use Capture Attribute sparingly

4. Geometry simplification
  - Fewer vertices = less attribute memory
  - Decimate modifier before Geometry Nodes
  - Use LOD systems

Example Memory Impact:
1M vertices × 10 float attributes = 40 MB
1M vertices × 5 float attributes = 20 MB (50% reduction)
```

**Face Corner Domain Optimization**:

```
Critical Memory Management:
1. Corner domain has highest element count
  - Use ONLY when necessary (UVs, split normals)
  - Don't store generic data here

2. Multiple UV maps multiply memory
  - UVMap: 12 bytes per corner (Vector)
  - 2 UV maps: 24 bytes per corner
  - Consider: Do you need multiple UV maps?

3. Vertex colors in corner domain
  - 16 bytes per corner (Color RGBA)
  - Consider if Point domain sufficient (no seams needed)

4. Geometry topology affects corner count
  - Quad mesh: 4 corners per face
  - Triangle mesh: 3 corners per face
  - Triangulate reduces corner count but changes topology

Example Memory Impact:
500K quad faces = 2M corners
UVMap (12 bytes) + Color (16 bytes) = 28 bytes per corner
Total: 56 MB just for UV and vertex color
```

**Instance Domain Optimization**:

```
Maximize Instance Benefits:
1. Use instances instead of realized geometry
  - 1000 instances of tree: Tiny memory
  - 1000 realized trees: Massive memory
  - Instance attributes: 4-16 bytes × 1000 instances
  - Realized geometry: Full geometry × 1000

2. Defer realization as long as possible
  - Perform operations on instances when possible
  - Realize only when mesh operations required

3. Instance attributes are cheap
  - Add variation through attributes
  - Not through geometry duplication

Example Memory Impact:
Tree: 100K vertices, 200K faces = ~3 MB geometry
1000 instances: ~3 MB + (instance data) ≈ 3.1 MB
1000 realized: ~3 GB (1000× geometry duplication)
Savings: ~997× memory reduction by using instances
```

**Cross-Domain Memory Optimization**:

```
Strategic Domain Usage:
1. Store data in lowest-count domain when possible
  - Spline domain < Face domain < Point domain < Corner domain
  - If property uniform across domain, use higher-level domain

2. Convert to higher-count domain only when needed
  - Spline → Point: Distribution adds no memory (duplication)
  - Point → Corner: Adds memory (more elements)

3. Clean up after operations
  - Remove Named Attribute for temporary data
  - Anonymous attributes from Capture Attribute cleaned automatically

4. Profile memory usage
  - Use Blender's System Console
  - Monitor memory during development
  - Identify memory hotspots

Example Strategy:
Per-curve color:
- Store in Spline domain: 5 curves × 16 bytes = 80 bytes
- Store in Point domain: 100 points × 16 bytes = 1,600 bytes
- Save: ~95% memory by using correct domain
```

### Performance Optimization by Domain

**Point Domain Performance Optimization**:

```
Optimization Techniques:
1. Minimize field complexity
  - Simple math faster than complex
  - Reduce nested texture nodes
  - Cache expensive calculations with Capture Attribute

2. Reduce vertex count when possible
  - LOD systems
  - Distance-based simplification
  - Decimate for background elements

3. Efficient selection
  - Early filtering with Delete Geometry
  - Reduces downstream vertex count
  - Operations on fewer vertices faster

4. Avoid repeated conversions
  - Point → Face → Point loses information and wastes computation
  - Calculate in appropriate domain and stay there
```

**Face Domain Performance Optimization**:

```
Optimization Techniques:
1. Use face domain for scatter
  - Distribute Points on Faces operates here
  - Natural domain for surface scattering

2. Material assignment efficiency
  - Face domain is correct and fast
  - Set Material directly in Face domain

3. Polygon reduction
  - Fewer faces = faster operations
  - Subdivision Surface only when needed
  - LOD for distant objects
```

**Face Corner Domain Performance Optimization**:

```
Optimization Techniques:
1. Minimize corner domain usage
  - Use ONLY for UVs and split normals
  - Store other data in Point or Face domain

2. Reduce geometry complexity
  - Fewer faces = fewer corners
  - Corner count grows with face complexity

3. Limit corner attribute count
  - Each attribute expensive (highest element count)
  - Remove unused corner attributes

4. Triangulation consideration
  - Triangles: 3 corners per face
  - Quads: 4 corners per face
  - Triangulation reduces corner count ~25%
  - But may affect other operations
```

**Instance Domain Performance Optimization**:

```
Optimization Techniques:
1. Maximize instance usage
  - Instances are extremely fast
  - Thousands to millions performant

2. Delay realization
  - Realize Instances is expensive
  - Only realize when mesh operations needed

3. Instance LOD
  - Different instance geometry by distance
  - Switch node based on camera distance
  - Simpler geometry for far instances

4. Instance culling
  - Delete instances outside camera frustum
  - Reduce instance count before realization
```

### Domain Conversion Performance

**Conversion Cost Analysis**:

```
Fast Conversions (Distribution - One to Many):
✓ Point → Corner: Very fast (value duplication)
✓ Face → Corner: Very fast (value duplication)
✓ Spline → Point: Very fast (value duplication)

Why fast: Simple value copying, no aggregation computation

Moderate Conversions (Aggregation - Many to One):
≈ Point → Face: Moderate (average vertex values per face)
≈ Corner → Point: Moderate (average corner values per vertex)
≈ Point → Edge: Moderate (average two endpoint values)

Why moderate: Aggregation requires computation (sum + divide)

Slow Conversions (Complex):
✗ Corner → Face → Point: Slower (multiple steps)
✗ Point → Edge → Face: Slower (multiple steps)
✗ Any repeated back-and-forth conversions

Why slow: Multiple conversion steps, repeated aggregation
```

**Conversion Optimization**:

```
Best Practices:
1. Convert once, not repeatedly
  - Bad: Point → Face → Point → Face
  - Good: Point → Face (then stay in Face domain)

2. Choose correct starting domain
  - Start calculations in appropriate domain
  - Minimize conversions needed

3. Cache conversion results
  - Use Capture Attribute to store converted values
  - Reuse instead of reconverting

4. Batch operations in same domain
  - Complete all Point domain work
  - Then convert once to Face domain
  - Complete all Face domain work
  - Minimal conversions

Example Optimization:
Inefficient:
- Calculate A (Point), convert to Face
- Calculate B (Point), convert to Face
- Calculate C (Point), convert to Face
Total: 3 conversions

Efficient:
- Calculate A, B, C (all Point domain)
- Convert once to Face domain (combine A, B, C)
Total: 1 conversion
```

### Performance Considerations During Domain Choice

Deciding on an attribute domain involves balancing the nature of your data against performance factors like calculation speed and memory usage. The checklist below provides a practical framework for making this choice.

Use these questions to evaluate your specific scenario. Keep in mind that these are general guidelines, not rigid rules. Optimization often requires trade-offs, so if the best approach isn't obvious, test different methods to see which performs best for your specific task.

**Domain Decision Checklist**

Q: What is the semantic meaning of the data?
A (Material Assignment): Use **Face Domain**
A (UV Coordinates): Use **Face Corner Domain** (mandatory for seams)
A (Vertex Positions): Use **Point Domain**
A (Edge Properties): Use **Edge Domain**
A (Per-Curve Properties): Use **Spline Domain**
A (Per-Instance Properties): Use **Instance Domain**

Q: Does the data need seam support?
A (Yes): Use **Face Corner Domain** (allows different values per face at the same vertex)
A (No): Use **Point Domain** (if data is per-vertex)

Q: Is the data uniform across a higher-level element?
A (Uniform per Curve?): Use **Spline Domain**
A (Uniform per Face?): Use **Face Domain**
A (Varies per Vertex?): Use **Point Domain**

Q: What operations will use this data?
A (A): Check the operation's *Required Domain* (see Node Input Requirements)
A (Strategy): Store data in that domain, or one conversion away, to minimize overhead

Q: What are the memory constraints?
A (Tight Memory): Use the **lowest-count domain possible** (e.g., Spline < Face < Point < Corner)
A (Memory Plentiful): Prioritize the domain that offers the most **clarity and maintainability**

Q: What are the performance needs?
A (Performance Critical): Prioritize **minimizing conversions**
A (Acceptable Overhead): Prioritize the **clearest domain** for logical flow

**Common Decision Examples**

- Scenario: Per-vertex color variation
  - Decision: Point domain
  - Reason: Per-vertex data, no seams needed

- Scenario: UV mapping with seams
  - Decision: Face Corner domain
  - Reason: Seams required, per-face-vertex coordinates

- Scenario: Material zones by height
  - Decision: Calculate in Point, store in Face
  - Reason: Calculate easily in Point, use in Face

- Scenario: Edge crease from angle
  - Decision: Edge domain
  - Reason: Edge property, natural domain

- Scenario: Random color per grass blade
  - Decision: Spline domain (distributed to Point)
  - Reason: Uniform per blade, varies between blades

- Scenario: Instance color variation
  - Decision: Instance domain
  - Reason: Per-instance property, efficient

## Version Compatibility Notes

### Attributes New in Blender 5.0

**Grid-Related Attributes** (Volume/SDF System)
- **Grid Background**: Default value for voxels outside active region
- **Grid Transform**: Position, rotation, scale of volume grid in world space
- **Active Voxel Count**: Number of non-background voxels
- **Voxel Size**: Spacing between voxels in grid
- **Grid Bounds**: Minimum and maximum coordinates of active region

These attributes only exist on Grid/Volume geometry types introduced in Blender 5.0.

**Enhanced Rotation Attributes**
- **Rotation Socket Type**: Dedicated rotation data type (previously stored as Euler vectors)
- **Rotation Attribute Storage**: Improved internal representation
- **Quaternion Support**: Better handling of rotation data

**Bundle-Related Metadata**
- Not stored as traditional attributes
- Internal system for grouping multiple attributes
- Accessed via Combine/Separate Bundle nodes
- Does not appear in Spreadsheet as individual attributes

**Closure-Related Data**
- Not stored as attributes
- Represents stored operations, not geometry data
- Exists only during node network evaluation
- Cannot be saved with geometry

### Deprecated Attributes

**Blender 3.0 (Fields System)**
- **Legacy Attribute Nodes**: Removed entirely
  - Old "Attribute" node replaced by field system
  - "Point Distribute" attribute handling removed
  - Manual attribute creation/deletion changed
  
**Migration**: Files using 2.92 attribute system cannot automatically convert to 3.0+. Complete rebuild required using field paradigm.

**Blender 4.0**
- **Old Vertex Color System**: Partially deprecated
  - Legacy vertex colors stored in Point domain
  - Modern workflow uses Corner domain
  - Old attributes still readable for compatibility
  - New projects should use Corner domain

**Migration**: Use Store Named Attribute in Corner domain for new vertex color data. Legacy point-domain colors automatically converted when read.

**Blender 5.0**
- **No Major Deprecations**: Strong backward compatibility maintained
- Legacy systems still supported
- Gradual migration to new systems encouraged
- Old attributes continue to work

### Changed Attribute Behaviors

**Blender 3.0 → 3.1**
- **Curve Attribute Access**: Improved
  - Better handle position access
  - Clearer Spline vs Point domain distinction
  - Handle Type stored as integer enum

**Blender 3.4 → 3.5**
- **UV Map Handling**: Clarified
  - Explicit corner domain requirement documented
  - Better error messages for wrong domain
  - Improved automatic domain detection

**Blender 4.0 → 4.1**
- **Grease Pencil Integration**: New attributes
  - Grease Pencil layer attributes
  - 2D point attributes for GP
  - Separate from 3D mesh attributes

**Blender 4.2 → 5.0**
- **Volume Attributes**: Entirely new system
  - Grid attributes introduced
  - VDB metadata support
  - Voxel-level attribute storage
  - Different from mesh attribute system

### Migration from Older Versions

**From Blender 2.92 to 3.x+**

*Complete Rebuild Required:*
```
What Changed:
- Entire attribute system redesigned
- Manual attribute nodes removed
- Field system introduced
- No automatic migration possible

Migration Process:
1. Understand field concepts (not just attribute storage)
2. Rebuild node networks from scratch
3. Use Capture Attribute instead of storing constantly
4. Store Named Attribute only when persistence needed
5. Embrace field-based thinking

Common Patterns:
Old: Store attribute → Read attribute → Delete attribute
New: Calculate field → Use directly → No cleanup needed
```

**From Blender 3.x to 4.x**

*Mostly Compatible:*
```
What Changed:
- Minor UI improvements
- Some node additions
- Attribute handling refinements
- Generally backward compatible

Migration Process:
1. Open file in 4.x
2. Check for deprecation warnings (rare)
3. Test attribute operations
4. Update to use new features if desired
5. Generally works without changes

Vertex Color Update (Optional):
- Old Point domain colors still work
- Consider migrating to Corner domain for UV seam support
- Not required but recommended for new features
```

**From Blender 4.x to 5.0**

*Fully Compatible with New Features:*
```
What Changed:
- Volume/Grid system added (new, not replacement)
- Bundle system added (organizational tool)
- Closure system added (advanced feature)
- Existing attributes unchanged

Migration Process:
1. Open file in 5.0 - works immediately
2. Explore new features gradually
3. Consider using SDF workflow for booleans
4. Adopt bundles for complex node groups
5. No changes required to existing setups

New Capabilities:
- Volume attributes for fog/cloud effects
- SDF attributes for implicit surfaces
- Grid-based attribute storage
- Maintained compatibility with mesh attributes
```

### Compatibility Best Practices

**For Cross-Version Projects**:
```
Recommendations:
1. Document Blender version used
2. Avoid version-specific features for shared projects
3. Use LTS versions for long-term work
4. Test in target version before delivery
5. Keep backup of files in original version

Version Numbering in Files:
- Include version in filename: "project_v4.2.blend"
- Note minimum required version in README
- Warn collaborators about version-specific features
```

**For Studio/Team Workflows**:
```
Studio Standards:
1. Standardize on specific Blender version
2. Update together, not individually
3. Test new versions before studio-wide adoption
4. Maintain legacy version for old projects
5. Document attribute naming conventions

Attribute Standards:
- Define allowed attribute names
- Specify domains for common attributes
- Create attribute templates
- Document custom attribute purposes
- Version control attribute schemas
```

**For Educational Content**:
```
Teaching Considerations:
1. Clearly state Blender version in tutorials
2. Note if features are version-specific
3. Explain attribute system evolution
4. Show migration strategies
5. Provide version-specific examples

Version Specification:
- "This tutorial uses Blender 5.0+"
- "Attribute system as of 3.0 (Fields)"
- "Volume attributes require 5.0+"
- "Compatible with 4.x series"
```

## Domain-Specific Best Practices

### Point Domain Best Practices

**When to Use**:
```
Ideal for Point Domain:
✓ Position-based operations
  - Displacement
  - Transformations
  - Spatial queries

✓ Per-vertex attributes
  - Vertex colors (if no seams needed)
  - Vertex weights
  - Selection states

✓ Vertex-based selection
  - Height-based selection
  - Proximity selection
  - Random vertex selection

✓ Gradients and smooth variation
  - Color gradients across surface
  - Smooth property transitions
  - Interpolated values
```

**What to Avoid**:
```
Not Suitable for Point Domain:
✗ Material assignment → Use Face domain
✗ UV coordinates → Use Face Corner domain
✗ Edge properties → Use Edge domain
✗ Per-face variation → Use Face domain

Reason: Wrong semantic meaning
- Materials apply to faces
- UVs need seam support (corners)
- Edge data belongs on edges
- Face data should be per-face
```

**Optimization Tips**:
```
Point Domain Efficiency:
1. Most versatile calculation domain
2. Direct access to positions
3. Efficient for per-vertex operations
4. Convert to other domains only when required
5. Use Capture Attribute to preserve detail before conversion
6. Smallest element count on simple meshes

Performance Characteristics:
- Fast: Direct position access
- Fast: Vertex-based math
- Moderate: Conversion to Face (aggregation)
- Slow: Repeated conversion back and forth
```

**Common Patterns**:
```
Pattern 1: Position-Based Effects
Position → Math/Texture → Set Position
- Calculate in Point domain
- Apply directly to points
- No conversion needed

Pattern 2: Vertex Color Generation
Position → Noise/Gradient → Store Color (Point domain if no seams)
- Generate in Point domain
- Store in Point domain (simple cases)
- Or convert to Corner domain (if seams needed later)

Pattern 3: Selection for Deletion
Position → Compare → Selection → Delete Geometry (Point domain)
- Select in Point domain
- Delete directly in Point domain
- Efficient, no conversion
```

### Face Domain Best Practices

**When to Use**:
```
Ideal for Face Domain:
✓ Material assignment
  - Material Index attribute
  - Set Material node
  - Multi-material objects

✓ Face-based selection
  - Normal direction selection
  - Area-based selection
  - Random face selection

✓ Polygon-specific properties
  - Shade Smooth attribute
  - Face area
  - Face type identification

✓ Per-face variation
  - Different colors per face
  - Per-polygon effects
  - Face-level randomization
```

**What to Avoid**:
```
Not Suitable for Face Domain:
✗ UV coordinates → Use Face Corner domain
✗ Smooth gradients → Point domain better (averages vertices)
✗ Vertex-specific data → Point domain
✗ Edge properties → Edge domain

Reason: Lacks necessary granularity
- UV seams require corners
- Smooth color needs vertex interpolation
- Vertex data doesn't map cleanly to faces
```

**Optimization Tips**:
```
Face Domain Efficiency:
1. Material Index MUST be Face domain
2. Face selection useful for deletion, extrusion
3. Converting Point → Face averages vertex values
4. Face count typically lower than Point count (good for memory)
5. Shade Smooth affects rendering only

Performance Characteristics:
- Fast: Face operations (subdivision, extrusion)
- Fast: Material assignment
- Moderate: Conversion from Point (aggregation)
- Moderate: Conversion to Point (distribution + aggregation)

Memory Considerations:
- Typically fewer faces than points
- Attributes use less memory than Point domain
- Good for per-polygon data storage
```

**Common Patterns**:
```
Pattern 1: Material Assignment by Height
Position (Point) → Separate Z → Map Range → Convert to Face → Material Index
- Calculate in Point domain
- Convert to Face with Average
- Store Material Index in Face domain
- Use Set Material

Pattern 2: Face-Based Random Variation
Random Value with Face Index → Store in Face domain → Use for effects
- Generate random per face
- No conversion needed
- Efficient face-level variation

Pattern 3: Normal-Based Material Zones
Normal → Separate Z → Compare → Face Selection → Material Index
- Use face normals
- Select faces by direction
- Assign materials by selection
- All in Face domain (efficient)
```

### Face Corner Domain Best Practices

**When to Use**:
```
Ideal for Face Corner Domain:
✓ UV coordinates (MANDATORY)
  - UV Map attribute
  - Texture mapping
  - Any UV-based operations

✓ Vertex colors with seam support
  - Modern vertex color workflow
  - Per-face-vertex colors
  - Allows color seams

✓ Split normals
  - Hard edges with smooth faces
  - Custom normal direction per face
  - Hard surface modeling

✓ Any data needing UV seam support
  - Different values at same vertex for different faces
  - Per-face-vertex information
  - Topology-sensitive data
```

**What to Avoid**:
```
Not Suitable for Corner Domain:
✗ Simple per-vertex data → Point domain simpler
✗ Data that should be uniform at vertices
✗ When seams not needed
✗ Excessive memory usage concern

Reason: Unnecessary complexity
- Corner count highest of all domains
- More memory usage
- More complex to work with
- Only use when seam support actually needed
```

**Critical Understanding**:
```
Corner Domain Fundamentals:
1. One corner per vertex per adjacent face
2. Quad face: 4 corners
3. Triangle: 3 corners
4. Same vertex can have multiple corners
5. Each corner can have different attribute values

Visualization:
Vertex at cube corner:
- 1 vertex position
- 3 adjacent faces
- 3 corners at that vertex
- Each corner can have different UV, different color, etc.

This is the KEY to understanding corners.
```

**UV Map Requirements**:
```
Why Corner Domain for UVs:
Problem: UV seams require different texture coordinates at same vertex
Solution: Corner domain provides per-face-vertex storage

Example:
Cube UV unwrap:
- Vertex at corner touches 3 faces
- Each face maps to different texture region
- Same vertex needs 3 different UV coordinates
- Corner domain: 3 corners, 3 different UVs
- Point domain: 1 vertex, 1 UV (would average, destroy seams)

Conclusion: UV coordinates MUST be in Corner domain for correct seaming.
```

**Optimization Tips**:
```
Corner Domain Efficiency:
1. Highest element count (uses most memory)
2. Critical for UVs (no alternative)
3. Use sparingly for non-UV data
4. Converting to Point loses seam information (averaging)
5. Distributing from Point to Corner duplicates values

Memory Impact:
- Quad mesh: 4× face count corners
- Triangle mesh: 3× face count corners
- Can be largest memory consumer
- Only use when necessary

Performance:
- Moderate: Corner operations
- Expensive: Corner → Point conversion (many-to-one aggregation)
- Fast: Point → Corner distribution (duplication)
- Expensive: Large corner counts on dense meshes
```

**Common Patterns**:
```
Pattern 1: Procedural UV Generation
Position (Point) → UV Calculation (Point) → Convert to Corner → Store "UVMap"
- Calculate UV coordinates in Point domain (easier math)
- Convert Point → Corner (distributes to all corners at vertex)
- Store in Corner domain as "UVMap"
- Modify specific corners for seams if needed

Pattern 2: UV-Aware Vertex Colors
Color Calculation (Point) → Convert to Corner → Modify at Seams → Store Color
- Generate colors in Point domain
- Convert to Corner for seam capability
- Adjust colors at UV seams
- Store in Corner domain

Pattern 3: Split Normals for Hard Edges
Edge Angle → Mark Sharp Edges → Split Normals → Store in Corner
- Identify sharp edges (Edge domain)
- Create split normals (Corner domain)
- Different normals for same vertex across sharp edge
- Rendering uses Corner domain normals
```

### Edge Domain Best Practices

**When to Use**:
```
Ideal for Edge Domain:
✓ Edge crease values
  - Subdivision Surface control
  - Sharp edge preservation
  - Crease attribute

✓ Edge-based selection
  - Edge loops
  - Boundary edges
  - Seam detection

✓ Edge properties
  - Smooth/sharp marking
  - Edge length
  - Edge angle
  - Custom edge data

✓ Wireframe-style effects
  - Edge rendering
  - Wire effects
  - Topology visualization
```

**What to Avoid**:
```
Not Suitable for Edge Domain:
✗ General vertex data → Point domain better
✗ Face data → Face domain better
✗ Overly complex when Point domain works

Reason: Less commonly needed
- Most operations work in Point or Face domain
- Edge-specific operations are specialized
- Convert only when edge semantics required
```

**Optimization Tips**:
```
Edge Domain Efficiency:
1. Less commonly used than Point or Face
2. Edge Angle node provides useful data
3. Crease attribute MUST be Edge domain
4. Edge count typically between Point and Face count
5. Useful for topology-based operations

Performance:
- Fast: Edge-based operations
- Moderate: Conversion from Point (aggregation)
- Moderate: Conversion to Face (distribution + aggregation)
- Specialized: Edge topology queries

Memory:
- Edge count: approximately 3× face count on typical meshes
- Lower than Corner, higher than Face
- Moderate memory usage
```

**Common Patterns**:
```
Pattern 1: Auto-Crease by Angle
Edge Angle → Compare > Threshold → Map to Crease Value → Store "crease"
- Edge Angle in Edge domain
- Compare in Edge domain
- Store crease in Edge domain
- No conversion needed (efficient)

Pattern 2: Edge Loop Selection
Topology Navigation → Edge Loop Detection → Selection → Operations
- Identify edge loops (Edge domain)
- Select edges in loop
- Extrude, bevel, etc. on edges
- Edge-specific workflow

Pattern 3: Wireframe with Variation
Edge → Random per Edge → Thickness Variation → Wireframe
- Random value per edge
- Variable edge thickness
- Wireframe rendering
- All in Edge domain
```

### Spline Domain Best Practices (Curves)

**When to Use**:
```
Ideal for Spline Domain:
✓ Per-curve properties
  - Different property for each curve in multi-curve object
  - Uniform property across entire curve

✓ Cyclic status
  - Open vs closed curves
  - Loop detection

✓ Curve resolution
  - Detail control per curve
  - LOD per spline

✓ Differentiating between curves
  - Random color per curve
  - Type identification
  - Curve classification
```

**What to Avoid**:
```
Not Suitable for Spline Domain:
✗ Per-point variation → Point domain on curves
✗ Mesh operations → Not applicable
✗ Gradual variation along curve → Point domain better

Reason: Wrong granularity
- Spline domain is entire curve
- Point domain provides per-point control
- Choose based on whether variation should be uniform or gradual
```

**Understanding Spline vs Point on Curves**:
```
Critical Distinction:
Spline Domain: Entire curve gets one value
Point Domain: Each point on curve can have different value

Example:
5 curves, each with 20 points:
- Spline domain: 5 values (one per curve)
- Point domain: 100 values (one per point, all curves)

Distribution:
Spline → Point: Each curve's 20 points get same value
Point → Spline: Aggregate 20 points to 1 spline value (Average)
```

**Optimization Tips**:
```
Spline Domain Efficiency:
1. Lowest count domain for multi-curve objects
2. Distributes same value to all points in curve
3. Useful for assigning properties to entire curves
4. Memory efficient for per-curve data

Performance:
- Fast: Spline property access
- Fast: Distribution to points (duplication)
- Moderate: Aggregation from points
- Efficient: Per-curve variation with low count

Use Cases:
- Hair strands (each strand different color)
- Cable bundles (each cable different thickness)
- Grass blades (each blade different length)
- Tree branches (each branch different behavior)
```

**Common Patterns**:
```
Pattern 1: Random Color per Curve
Random with Spline Index Seed → Color → Store in Spline → Distribute to Points
- Generate random per spline (5 colors for 5 curves)
- Store in Spline domain
- Convert to Point domain for rendering
- All points in each curve get same color

Pattern 2: Curve Type Identification
Spline Length → Compare → Type Classification → Store in Spline
- Calculate property per spline
- Classify entire curves
- Store classification in Spline domain
- Use for conditional operations

Pattern 3: Resolution per Curve
Spline Property → Map to Resolution → Set Spline Resolution
- Calculate desired resolution per curve
- Set in Spline domain
- Each curve can have different detail level
- LOD system for curves
```

### Instance Domain Best Practices

**When to Use**:
```
Ideal for Instance Domain:
✓ Per-instance variation
  - Each instance different color
  - Each instance different size
  - Each instance different rotation

✓ Instance transforms
  - Position, rotation, scale
  - Transform variation
  - Procedural placement

✓ Instance-specific attributes
  - Instance type
  - Instance ID
  - Custom properties per instance

✓ Random per-instance effects
  - Using Instance ID for stable randomization
  - Variation that persists
```

**What to Avoid**:
```
Not Suitable for Instance Domain:
✗ Confusing with Point domain (different concepts)
✗ Converting to other domains (limited options)
✗ After realization (becomes mesh, loses instance info)

Reason: Instances are references, not geometry
- Instances reference geometry
- Point clouds are actual points
- Different systems, different purposes
```

**Critical Understanding**:
```
Instance vs Point Distinction:
Instance:
- Reference to geometry
- Has position, rotation, scale
- Efficient (geometry not duplicated)
- Instance domain attributes

Point:
- Actual geometry element
- Only position (and attributes)
- No referenced geometry
- Point domain attributes

Not Interchangeable:
- Cannot directly convert domains
- Instances to Points node: extracts positions, loses geometry
- Points to Vertices: creates mesh, not instances
```

**Optimization Tips**:
```
Instance Domain Efficiency:
1. Most memory-efficient for repeated geometry
2. Instance attributes travel with instance
3. Realize Instances converts to actual geometry (expensive)
4. Keep instances as long as possible
5. Use Instance domain for variation before realization

Performance:
- Very Fast: Instance operations (reference-based)
- Fast: Instance attribute access
- Expensive: Realize Instances (duplicates geometry)
- Efficient: Large instance counts

Memory:
- Minimal: Instances reference geometry (not duplicated)
- Small: Instance attributes (per instance, not per vertex)
- Massive increase: After realization (geometry duplicated)

Best Practice:
- Use instances for repeated elements
- Vary with instance attributes
- Realize only when absolutely necessary
- Consider if operation can work on instances
```

**Common Patterns**:
```
Pattern 1: Instance Color Variation
Random Value with Instance ID → Color → Store in Instance → Use in Shader
- Generate random per instance (stable with ID)
- Store color in Instance domain
- Shader reads instance attribute
- Each instance renders different color

Pattern 2: Instance Transform Variation
Instance Index → Math → Rotation/Scale → Store → Apply to Instances
- Calculate variation per instance
- Store transforms in Instance domain
- Apply with Rotate/Scale Instances nodes
- Variation persists through operations

Pattern 3: Conditional Instance Realization
Instance Attribute → Selection → Realize Instances (on selection only)
- Only realize instances that need mesh operations
- Keep others as instances
- Optimization for mixed workflows
- Minimize realization overhead
```

### Cross-Domain Best Practices

**Domain Conversion Strategy**:
```
General Principles:
1. Start in most natural domain for data
2. Keep data in that domain as long as possible
3. Convert once, not repeatedly
4. Use appropriate aggregation method
5. Understand data loss from conversion

Conversion Order Matters:
Point → Face → Point ≠ Point (averaging causes data loss)
Face → Corner → Face ≠ Face (if corners modified)
Always consider: Is conversion reversible?

Minimize Conversions:
Bad: Point → Face → Point → Edge → Face
Good: Point → Face (or Point → Edge), then stay there
```

**Choosing Aggregation Methods**:
```
Semantic Meaning Matters:
- Colors: Average (smooth blending)
- IDs: First (averaging IDs nonsensical)
- Selections: Average + threshold or Min/Max
- Counts: Sum (totals make sense)
- Measurements: Average usually (but context-dependent)

Think About Purpose:
"What does this attribute represent?"
"What makes sense when combining values?"
"Will averaging destroy meaningful information?"
```

**Memory vs Computation Trade-offs**:
```
Store in Multiple Domains:
Pros:
+ Fast access in each domain
+ No conversion overhead
+ Redundancy for safety

Cons:
- Multiple memory copies
- Must keep synchronized
- Memory usage multiplied

Calculate On-Demand with Conversion:
Pros:
+ Single data copy
+ Less memory usage
+ Automatic synchronization

Cons:
- Conversion overhead each use
- Repeated computation
- May be slower

Choose Based On:
- How often accessed?
- How many domains needed?
- Memory constraints?
- Performance requirements?
```

## Workflow Patterns for Common Tasks

### Pattern 1: Height-Based Material Assignment

**Goal**: Assign different materials based on geometry height (Z position)

**Step-by-Step**:
```
1. Read Position (Point domain)
  - Position node → Vector output
  - Contains XYZ for each vertex

2. Extract Z Component (Point domain)
  - Separate XYZ node
  - Connect Position to input
  - Use Z output (height)

3. Map Height to Material Index Values (Point domain)
  - Map Range node
  - Input: Z height
  - From Min: Lowest Z in geometry
  - From Max: Highest Z in geometry
  - To Min: 0 (first material)
  - To Max: N-1 (where N = number of materials)
  - Result: Float value 0.0 to N-1

4. Convert to Integer (Point domain)
  - Math node: Round, Floor, or Ceiling
  - Ensures whole number material indices
  - Still in Point domain

5. Convert Point → Face (with Average aggregation)
  - Evaluate on Domain node
  - Source Domain: Point
  - Target Domain: Face  
  - Method: Average
  - Result: Face gets average of its vertex material indices

6. Round Again (Face domain, if needed)
  - Math node: Round
  - Ensures integer after averaging
  - Now definitively in Face domain

7. Store as Material Index (Face domain)
  - Store Named Attribute node
  - Attribute Name: "material_index"
  - Domain: Face
  - Value: Rounded material index

8. Apply Materials
  - Material Index automatically used by Set Material
  - Or read Material Index in subsequent operations
  - Face-based material assignment complete

Alternative: Calculate Directly in Face Domain
- Read Position in Face domain (averaged vertex positions)
- Separate Z, map range, round
- All in Face domain from start
- Fewer conversions but different semantics (face center vs vertex average)
```

**Domain Flow Diagram**:
```
Point → Point → Point → Point → Face → Face → Face
(Pos) (Z)   (Map) (Round) (Avg) (Round)(Store)

Total Conversions: 1 (Point to Face)
```

**Considerations**:
- **Averaging Material Indices**: Can produce non-integer values
  - Face with vertices material indices [0, 0, 1, 1] → average 0.5
  - Must round after averaging
  - Consider if this creates desired transition zones
  
- **Sharp Transitions**: For crisp material boundaries
  - Convert Point → Face with Min or Max instead of Average
  - Min: Face gets lowest vertex material index
  - Max: Face gets highest vertex material index
  - No intermediate values

- **Number of Materials**: Must exist in material slots
  - Material Index 2 requires at least 3 materials (0, 1, 2)
  - Out-of-range indices may cause errors or default material

**Example Parameters**:
```
Terrain Material Zones:
- Water: Z < 0.0 → Material Index 0
- Beach: 0.0 ≤ Z < 0.5 → Material Index 1  
- Grass: 0.5 ≤ Z < 3.0 → Material Index 2
- Rock: 3.0 ≤ Z < 5.0 → Material Index 3
- Snow: Z ≥ 5.0 → Material Index 4

Map Range Setup:
- From Min: -1.0 (below water)
- From Max: 6.0 (above snow)
- To Min: 0
- To Max: 4
- Clamped: Yes (prevents out-of-range)
```

### Pattern 2: Procedural UV Generation

**Goal**: Create UV coordinates procedurally based on geometry

**Step-by-Step**:
```
1. Read Position (Point domain)
  - Position node
  - XYZ coordinates of each vertex

2. Calculate UV Coordinates (Point domain)
  - Various methods:
     a) Planar Projection: Use X and Y directly as UV
     b) Box Projection: Different UV per face based on normal
     c) Cylindrical: Angle and height
     d) Spherical: Latitude and longitude

3. Combine into UV Vector (Point domain)
  - Combine XYZ node (using only X and Y components)
  - U → X component of vector
  - V → Y component of vector
  - Z → 0.0 (UV is 2D)
  - Result: Vector (U, V, 0.0) per vertex

4. Convert Point → Face Corner (distribution)
  - Evaluate on Domain node
  - Source: Point
  - Target: Face Corner
  - Method: Not aggregation (one-to-many distribution)
  - Each corner at vertex gets that vertex's UV value

5. Store as "UVMap" (Corner domain)
  - Store Named Attribute node
  - Attribute Name: "UVMap" (standard name)
  - Domain: Face Corner
  - Value: UV vector
  - Critical: MUST be Corner domain

6. Optional: Modify for Seams
  - Select specific corners at seams
  - Modify UV values for those corners only
  - Creates UV seams where needed
  - This is why Corner domain is required

7. Use in Materials
  - UV Map node in shader
  - Reads "UVMap" from Corner domain
  - Automatically handles seams
  - Textures map correctly
```

**Planar Projection Example**:
```
Simple XY Projection:
1. Position → Separate XYZ
2. Combine XYZ:
  - X → U (X component)
  - Y → V (Y component)
  - Z → 0.0
3. Optional: Scale/Offset
  - Vector Math: Multiply (scale)
  - Vector Math: Add (offset)
4. Convert Point → Corner
5. Store "UVMap" in Corner domain

Result: Geometry's XY position becomes UV coordinates
```

**Box Projection Example**:
```
Different Projection per Face Based on Normal:
1. Read Face Normal (Face domain)
2. Determine dominant axis (X, Y, or Z):
  - Separate Normal XYZ
  - Math: Absolute value each component
  - Compare to find largest (dominant axis)
3. Based on dominant axis, project accordingly:
  - X-dominant: Use YZ for UV
  - Y-dominant: Use XZ for UV
  - Z-dominant: Use XY for UV
4. Calculate UV per face
5. Convert Face → Corner (distribute)
6. Store "UVMap" in Corner domain

Result: Automatic box-like UV unwrap
```

**Domain Flow**:
```
Point → Point → Point → Corner → Corner
(Pos)  (Calc)  (UV)   (Dist)  (Store)

Total Conversions: 1 (Point to Corner)
```

**Critical Point: Why Corner Domain**:
```
UV Seam Scenario:
- Cube edge connects two faces
- Each face maps to different texture region
- Vertex on edge needs TWO UV coordinates:
  - UV for Face 1 side
  - UV for Face 2 side
- Point domain: Only one UV per vertex (seam breaks)
- Corner domain: Two corners at vertex, two UVs (seam works)

Conclusion: UV maps MUST be in Corner domain for seams.
```

**Considerations**:
- **Seam Handling**: May need manual adjustment of corner UVs at seams
- **Scale**: UV coordinates typically 0-1 range for texture repeat
- **Distortion**: Simple projections may cause stretching
- **Multiple UV Maps**: Can store multiple with different names ("UVMap", "UVMap2", etc.)

### Pattern 3: Vertex Color from Multiple Attributes

**Goal**: Create vertex colors by combining multiple geometry properties

**Step-by-Step**:
```
1. Calculate Red Channel (Point domain)
  - Example: Height (Position Z)
  - Map Range: Normalize height to 0-1
  - Result: Red value per vertex

2. Calculate Green Channel (Point domain)
  - Example: Curvature or Noise
  - Map Range: Normalize to 0-1
  - Result: Green value per vertex

3. Calculate Blue Channel (Point domain)
  - Example: Ambient Occlusion approximation or Random
  - Map Range: Normalize to 0-1
  - Result: Blue value per vertex

4. Combine RGB (Point domain)
  - Combine Color node or Combine XYZ (as color)
  - R → Red channel value
  - G → Green channel value
  - B → Blue channel value
  - A → 1.0 (fully opaque)
  - Result: RGBA color per vertex

5. Convert Point → Face Corner (distribution)
  - Evaluate on Domain node
  - Source: Point
  - Target: Face Corner
  - Why: Allows color seams if needed (like UV seams)

6. Optional: Modify at Seams
  - Select specific corners
  - Adjust colors for artistic control
  - Create color seams where desired

7. Store as Color Attribute (Corner domain)
  - Store Named Attribute node
  - Attribute Name: "Color" (or custom name)
  - Data Type: Color (RGBA)
  - Domain: Face Corner
  - Modern vertex color workflow

8. Use in Materials
  - Attribute node in shader
  - Attribute Name: "Color"
  - Fac/Color output to shader
  - Renders with vertex colors
```

**Example: Terrain Color**:
```
Red Channel: Height
- Position Z → Map Range (0 to 10 → 0 to 1)
- Low elevation: Dark red (0.0)
- High elevation: Bright red (1.0)

Green Channel: Slope
- Normal Z component (vertical = 1.0, horizontal = 0.0)
- Flat areas: Bright green (1.0)
- Steep slopes: Dark green (0.0)

Blue Channel: Noise Variation
- Noise Texture based on Position
- Adds organic variation
- 0.0 to 1.0 range

Combined Result:
- Flat low areas: Dark red, bright green, varied blue
- Steep high areas: Bright red, dark green, varied blue
- Visual terrain analysis through color
```

**Domain Flow**:
```
Point → Point → Point → Point → Corner → Corner
(R)    (G)    (B)   (Combine)(Dist)  (Store)

Total Conversions: 1 (Point to Corner)
```

**Alternative: Point Domain Storage**:
```
If No Seams Needed:
1-4: Same as above (calculate color in Point domain)
5: Skip conversion to Corner
6: Store directly in Point domain
7: Use Color attribute from Point domain

Pros:
+ Less memory (fewer elements in Point than Corner)
+ Simpler workflow
+ Faster

Cons:
- Cannot have color seams
- Less flexibility
- Modern workflow prefers Corner domain
```

**Considerations**:
- **Color Range**: Keep 0-1 for standard colors, can exceed for HDR
- **Alpha Channel**: Usually 1.0 unless transparency needed
- **Seam Capability**: Corner domain allows color seams (like different colors on adjacent faces at same vertex)
- **Shader Access**: Color attribute readable in all shader contexts

### Pattern 4: Edge Crease from Face Angle

**Goal**: Automatically assign edge crease based on angle between faces

**Step-by-Step**:
```
1. Read Edge Angle (Edge domain)
  - Edge Angle node
  - Outputs angle between adjacent faces (in radians)
  - Automatically in Edge domain
  - Range: 0 (flat) to π (180°, folded)

2. Convert Radians to Degrees (Optional, for clarity)
  - Math node: Multiply
  - Multiply by 180/π (≈ 57.2958)
  - Now in degrees (0 to 180)
  - Still Edge domain

3. Compare to Threshold (Edge domain)
  - Compare node
  - Angle > Threshold (e.g., 30°)
  - Result: Boolean (True = sharp, False = smooth)
  - Still Edge domain

4. Convert Boolean to Crease Value (Edge domain)
  - Map Range node
  - Input: Boolean (0.0 or 1.0)
  - From Min: 0.0, From Max: 1.0
  - To Min: 0.0 (smooth), To Max: 1.0 (sharp)
  - Or use Boolean directly (automatically converts)

5. Optional: Smooth Crease Transition
  - Map Range with smooth falloff
  - Input: Angle
  - From Min: 25° (start creasing)
  - From Max: 35° (full crease)
  - To Min: 0.0, To Max: 1.0
  - Interpolation: Smooth Step
  - Result: Gradual crease based on angle

6. Store as "crease" Attribute (Edge domain)
  - Store Named Attribute node
  - Attribute Name: "crease"
  - Data Type: Float
  - Domain: Edge
  - Value: 0.0 (smooth) to 1.0 (sharp)

7. Subdivision Surface Reads Crease
  - Subdivision Surface modifier
  - Automatically reads "crease" attribute
  - Preserves sharp edges based on crease value
  - No additional setup needed
```

**Domain Flow**:
```
Edge → Edge → Edge → Edge → Edge
(Angle)(Convert)(Compare)(Map)(Store)

Total Conversions: 0 (all operations in Edge domain)
```

**This is Optimal: No domain conversions needed**

**Example Threshold Values**:
```
Soft Auto-Smooth (preserve only hard edges):
- Threshold: 60-80°
- Only very sharp edges get crease
- Most edges remain smooth

Medium Auto-Smooth (balanced):
- Threshold: 30-45°
- Moderate angles get crease
- Balance between hard surface and organic

Hard Auto-Smooth (preserve detail):
- Threshold: 15-30°
- Even slight angles get crease
- Preserves more surface detail
- Good for hard surface modeling
```

**Smooth Transition Example**:
```
Gradual Crease:
Map Range settings:
- From Min: 20° (start)
- From Max: 40° (full)
- To Min: 0.0 (no crease)
- To Max: 1.0 (full crease)
- Interpolation: Smooth Step

Result:
- Angle < 20°: Crease = 0.0 (fully smooth)
- Angle 20-40°: Crease gradual 0.0 to 1.0 (transition)
- Angle > 40°: Crease = 1.0 (fully sharp)

Benefits:
- Smoother visual result
- Less harsh transitions
- More organic look while preserving edges
```

**Considerations**:
- **Angle Units**: Edge Angle outputs radians; convert for intuitive thresholds
- **Domain Match**: Edge Angle and Crease both Edge domain (no conversion overhead)
- **Subdivision**: Crease only affects Subdivision Surface modifier
- **Overlapping Creases**: Can combine angle-based with manual crease

### Pattern 5: Density Painting for Scattering

**Goal**: Use painted vertex weights to control point distribution density

**Step-by-Step**:
```
1. Paint Vertex Weights (External to Geometry Nodes)
  - Weight Paint mode in Blender
  - Paint weights on vertices
  - Dark (0.0): Low/no density
  - Bright (1.0): High density
  - Gradients for smooth transitions
  - Stored automatically as vertex attribute

2. Read Painted Weights (Point domain in Geometry Nodes)
  - Named Attribute node
  - Attribute Name: Name of vertex group/weight attribute
  - Output: Float value per vertex (0.0 to 1.0)
  - Automatically in Point domain

3. Optional: Adjust Density Curve (Point domain)
  - Float Curve node or Map Range
  - Reshape density response
  - Example: Steeper curve for more contrast
  - Still Point domain

4. Convert Point → Face (Average aggregation)
  - Evaluate on Domain node
  - Source: Point
  - Target: Face
  - Method: Average
  - Face density = average of vertex weights

5. Use as Distribute Points Density (Face domain)
  - Distribute Points on Faces node
  - Density input: Converted face values
  - Distribute Points operates in Face domain
  - More points where density higher

Alternative: Density Mode
  - If using Density (not Distance or Count)
  - Face density directly controls point count
  - Higher values = more points on that face

6. Result
  - Painted areas get more points
  - Unpainted areas get fewer points
  - Smooth transitions from gradients
  - Artistic control over scattering
```

**Domain Flow**:
```
Point → Point → Face → Face
(Paint) (Adjust)(Avg) (Scatter)

Total Conversions: 1 (Point to Face)
Necessary: Distribute Points on Faces requires Face domain density
```

**Why This Conversion**:
```
Point Domain: Vertex weights (painted)
Face Domain: Distribution operates per face

Distribute Points on Faces needs:
- Density value per face
- Determines how many points on each face

Conversion Method (Average):
- Face density = average of its vertex weights
- Quad with vertices [0.0, 0.5, 1.0, 1.0]: Density = 0.625
- Gradual transitions between painted and unpainted regions
```

**Example Workflow**:
```
Grass Distribution on Terrain:
1. Paint terrain vertices:
  - Paths: Black (weight 0.0)
  - Open grass: White (weight 1.0)
  - Transition zones: Gray (0.0-1.0 gradient)

2. In Geometry Nodes:
  - Read weight attribute (Point domain)
  - Convert Point → Face (Average)
  - Use as Distribute Points density

3. Result:
  - No grass on paths (density 0.0)
  - Full grass in fields (density 1.0)
  - Gradual density transition (smooth edges)
  - Natural-looking distribution
```

**Density Adjustment Example**:
```
Making Density More Contrast:
Float Curve node after reading weights:
- Input: 0.0 to 1.0
- Output curve: S-curve
  - 0.0 → 0.0 (low stays low)
  - 0.5 → 0.2 (midtones darker)
  - 1.0 → 1.0 (high stays high)
- Effect: Sharper transition, less mid-density
```

**Considerations**:
- **Vertex Weight Painting**: External to Geometry Nodes
- **Attribute Name**: Must match painted vertex group name
- **Density Units**: Understand Distribute Points density parameter
- **Performance**: Dense scattering can be expensive (use Poisson Disk with distance limit)

### Pattern 6: Per-Curve Variation

**Goal**: Apply different properties to each curve in a multi-curve object

**Step-by-Step**:
```
1. Generate Random Value per Spline (Spline domain)
  - Random Value node
  - Seed: Spline Index (or custom seed)
  - Result: One random value per spline
  - Spline Index varies per curve
  - Stable randomization

2. Create Color from Random (Spline domain)
  - ColorRamp node or Combine Color
  - Input: Random value
  - Output: Color per spline
  - Example: Random hue, full saturation
  - Still Spline domain

3. Store in Spline Domain (Spline domain)
  - Store Named Attribute node
  - Attribute Name: "CurveColor" (custom name)
  - Data Type: Color
  - Domain: Spline
  - One color value per entire curve

4. Convert Spline → Point (Distribution)
  - Evaluate on Domain node
  - Source: Spline
  - Target: Point
  - Method: Distribution (no aggregation, one-to-many)
  - All points in spline receive same color

5. Use for Rendering/Further Operations (Point domain)
  - Now each point has color
  - All points in Curve 0: Color 0
  - All points in Curve 1: Color 1
  - And so on...
  - Use in shaders, or further processing
```

**Domain Flow**:
```
Spline → Spline → Spline → Point → Point
(Random)(Color) (Store) (Dist) (Use)

Total Conversions: 1 (Spline to Point, distribution)
```

**Example: 5 Curves with Different Colors**:
```
Setup:
- 5 splines in one curve object
- Each spline has 20 points
- Random Value with Spline Index seed

Spline Domain:
- Spline 0: Random 0.123 → Red
- Spline 1: Random 0.456 → Green
- Spline 2: Random 0.789 → Blue
- Spline 3: Random 0.234 → Yellow
- Spline 4: Random 0.567 → Purple

After Distribution to Point Domain:
- Points 0-19 (Spline 0): All Red
- Points 20-39 (Spline 1): All Green
- Points 40-59 (Spline 2): All Blue
- Points 60-79 (Spline 3): All Yellow
- Points 80-99 (Spline 4): All Purple

Total: 100 points, 5 different colors
Each curve uniformly colored, different from others
```

**Why This Works**:
```
Spline Domain Distribution:
- Spline attribute: 1 value per curve
- Point domain: Many values (one per point)
- Distribution: Copies spline value to all its points
- No aggregation (one-to-many, not many-to-one)
- All points in same spline get identical value

Perfect for:
- Uniform properties across entire curve
- Different curves having different properties
- Per-strand variation in hair/grass
- Per-cable variation in bundles
```

**Use Cases**:
```
Hair Strands:
- Each strand different color/thickness
- Spline domain: Hair strand properties
- Point domain: After distribution for rendering

Cable Bundles:
- Each cable different thickness
- Spline: Cable type
- Point: Thickness along cable (uniform within cable)

Grass Blades:
- Each blade different height/color
- Spline: Blade properties
- Point: Applied uniformly per blade

Tree Branches:
- Each branch different behavior
- Spline: Branch type
- Point: Distribute for operations
```

**Considerations**:
- **Spline Index**: Built-in index for each spline (0-based)
- **Stable Randomness**: Using index as seed gives consistent random per curve
- **Distribution vs Aggregation**: Spline → Point is distribution (duplication), Point → Spline is aggregation (averaging)
- **Memory Efficient**: Storing in Spline domain uses less memory than storing per-point

### Pattern 7: Instance Color Variation

**Goal**: Give each instance a unique random color

**Step-by-Step**:
```
1. Create Instances (Instance domain)
  - Instance on Points node
  - Points from Distribute Points or other source
  - Instance geometry connected
  - Result: Instances at each point
  - Instance domain now exists

2. Generate Random per Instance (Instance domain)
  - Random Value node
  - ID: Instance ID (or Instance Index)
  - Seed: Custom seed for variation
  - Result: One random value per instance
  - Stable (same instance = same random)

3. Create Color from Random (Instance domain)
  - ColorRamp or Hue/Saturation conversion
  - Random Value → Hue (0-1 → full color wheel)
  - Full Saturation, Full Value
  - Result: Bright random color per instance

4. Store as Instance Attribute (Instance domain)
  - Store Named Attribute node
  - Attribute Name: "InstanceColor"
  - Data Type: Color
  - Domain: Instance
  - Each instance now has color attribute

5. Use in Shader (Instance domain)
  - Material on instanced geometry
  - Attribute node in Shader Editor
  - Attribute Name: "InstanceColor"
  - Connect to Base Color or other properties
  - Shader automatically reads per-instance

6. Result
  - Each instance renders with different color
  - Color stable per instance (ID-based)
  - Instances update if instance changes
  - Efficient (no geometry duplication)
```

**Domain Flow**:
```
Instance → Instance → Instance → Instance → Shader
(Create) (Random) (Color)  (Store)   (Read)

Total Conversions: 0 (all Instance domain)
Shader reads Instance domain directly
```

**Why Instance ID**:
```
Instance ID vs Instance Index:
- Index: Sequential 0, 1, 2, 3... (changes if instances added/removed)
- ID: Stable identifier (persists through changes)

For Random Seed:
- Use ID for stable randomization
- Same instance always gets same color
- Even if instances added/removed/reordered
- Recommended for consistent variation
```

**Example Shader Setup**:
```
Shader Node Tree:
1. Attribute Node:
  - Name: "InstanceColor"
  - Type: Color
  - Output: Color (per instance)

2. Connect to Principled BSDF:
  - Attribute Color → Base Color
  - Each instance gets its stored color

Result:
- 1000 instances, 1000 different colors
- Only one material
- Attribute drives color variation
- Efficient rendering
```

**Advanced: Multiple Instance Attributes**:
```
Per-Instance Variation:
1. Random Color (as above)
2. Random Roughness:
  - Random Value → Float
  - Store as "InstanceRoughness"
  - Use in shader Roughness input

3. Random Scale:
  - Random Value → Scale Factor
  - Store as "InstanceScale"
  - Scale Instances node reads it

Result:
- Each instance: unique color, roughness, scale
- All from instance attributes
- Highly varied scene from repeated geometry
```

**Considerations**:
- **Attribute Names**: Must match between GN and Shader
- **Data Types**: Color for colors, Float for numbers
- **Instance Domain**: Attributes travel with instances
- **Realization**: After Realize Instances, becomes per-vertex data (no longer per-instance)
- **Performance**: Extremely efficient (attributes tiny compared to geometry)

### Pattern 8: Smooth Normals (Corner vs Point)

**Goal**: Control smooth vs sharp shading through normal manipulation

**Step-by-Step for Split Normals (Sharp Edges)**:

```
Sharp Edge Creation:
1. Identify Sharp Edges (Edge domain)
  - Edge Angle node
  - Compare Angle > Threshold (e.g., 30°)
  - Result: Boolean per edge (True = sharp)

2. Mark Edges for Normal Splitting
  - Store edge selection
  - Edges marked as "sharp" or "seam"
  - Basis for split normal creation

3. Generate Split Normals (Face Corner domain)
  - For each corner:
     - If adjacent edge is sharp: Use face normal
     - If adjacent edge is smooth: Average with neighbor face normals
  - Result: Different normals for same vertex across sharp edge

4. Store Split Normals (Corner domain)
  - Store Named Attribute node
  - Attribute Name: Custom or use Blender's split normal system
  - Data Type: Vector (normalized)
  - Domain: Face Corner (CRITICAL)

5. Rendering Uses Split Normals
  - Renderer reads corner normals
  - Sharp edges: Discontinuous normals = visible edge
  - Smooth edges: Averaged normals = smooth appearance
```

**Step-by-Step for Smooth Normals (Vertex Averaging)**:

```
Smooth Edge Creation:
1. Calculate Face Normals (Face domain)
  - Automatically calculated from geometry
  - One normal vector per face
  - Points perpendicular to face

2. Convert Face → Face Corner (Distribution)
  - Each corner gets its face's normal
  - All corners of same face have same normal
  - Different faces have different normals

3. Convert Corner → Point (Aggregation with Average)
  - All corners at vertex aggregated
  - Average of all adjacent face normals
  - Result: Smooth normal at vertex

4. Convert Point → Corner (Distribution)
  - Smoothed vertex normal distributed back to corners
  - All corners at same vertex now have same normal
  - Creates smooth shading

5. Store Unified Normals (Corner domain)
  - Store as corner attribute
  - All corners at vertex have identical normal
  - Result: Smooth shading across edges
```

**Domain Flow Comparison**:

```
Sharp Edges (Split Normals):
Edge → Corner → Corner
(Sharp) (Split)  (Store)
- Different normals at same vertex for different faces
- Corner domain preserves differences

Smooth Edges (Averaged Normals):
Face → Corner → Point → Corner → Corner
(Norm) (Dist)  (Avg)  (Dist)  (Store)
- Averaged normals unified at vertex
- All corners at vertex get same normal
```

**Practical Example - Cube**:

```
Sharp Cube (Split Normals):
Vertex at corner touches 3 faces:
- Face Top: Normal = (0, 0, 1) [Up]
- Face Front: Normal = (0, -1, 0) [Forward]
- Face Right: Normal = (1, 0, 0) [Right]

Corner Normals (Corner domain):
- Corner on Top face: Normal = (0, 0, 1)
- Corner on Front face: Normal = (0, -1, 0)
- Corner on Right face: Normal = (1, 0, 0)

Result: Three different normals at same vertex
Rendering: Sharp edges visible (discontinuous normals)

Smooth Cube (Averaged Normals):
Same vertex, same 3 faces, but:
Average normal = [(0,0,1) + (0,-1,0) + (1,0,0)] / 3
               = (0.333, -0.333, 0.333)
Normalized: (0.577, -0.577, 0.577) [Diagonal direction]

Corner Normals (Corner domain):
- All 3 corners at vertex: Normal = (0.577, -0.577, 0.577)

Result: Same normal at all corners of vertex
Rendering: Smooth shading (continuous normals)
```

**Auto-Smooth Workflow (Hybrid)**:

```
Angle-Based Sharp/Smooth:
1. Edge Angle → Boolean selection (Edge domain)
  - Angle > 30°: Sharp (True)
  - Angle ≤ 30°: Smooth (False)

2. For Sharp Edges:
  - Corners adjacent to edge: Use face normal
  - No averaging across this edge
  - Creates split normal

3. For Smooth Edges:
  - Corners adjacent to edge: Average face normals
  - Smooth transition
  - Unified normal

4. Store in Corner Domain:
  - Mixed: Some corners split, some unified
  - Sharp where needed, smooth elsewhere
  - Optimal visual result

Result: Hard surface detail preserved, smooth surfaces smooth
```

**Considerations**:

- **Corner Domain Requirement**: Split normals MUST be in corner domain
  - Point domain cannot store multiple normals per vertex
  - Corner domain allows different normals per face at same vertex
  
- **Normalization**: Normal vectors must be unit length (length = 1.0)
  - Use Vector Math: Normalize after any normal math
  - Rendering expects normalized normals
  
- **Performance**: Split normals slightly more expensive than smooth
  - More data (potentially different at each corner)
  - Rendering must handle discontinuities
  - Usually negligible impact
  
- **Visual Quality**: Split normals essential for hard surface modeling
  - Preserves crisp edges
  - Allows smooth faces with sharp features
  - Industry standard for mechanical/architectural models

### Pattern 9: Painting Density Map (Point to Face) - Extended

**Goal**: Use painted vertex weights with advanced control for sophisticated scattering

**Extended Workflow with Controls**:

```
1. Paint Base Density (External, Point domain)
  - Weight Paint mode
  - Paint vertex weights
  - Base density map

2. Read Weights in Geometry Nodes (Point domain)
  - Named Attribute node
  - Attribute Name: Vertex group name
  - Float output per vertex

3. Apply Density Curve (Point domain)
  - Float Curve node
  - Reshape density response
  - Example curve types:
     - Linear: 1:1 response
     - Exponential: Emphasis on high values
     - Logarithmic: Emphasis on low values
     - S-Curve: Contrast enhancement

4. Add Procedural Variation (Point domain)
  - Noise Texture based on Position
  - Mix with painted weights
  - Adds organic variation to painting
  - Prevents too-uniform appearance

5. Multiply by Global Density Control (Point domain)
  - Math: Multiply
  - Input: Adjusted weights
  - Value: Global density slider (0-1)
  - Allows easy overall density adjustment

6. Clamp Values (Point domain)
  - Clamp node
  - Min: 0.0 (no negative density)
  - Max: 1.0 (or higher if needed)
  - Ensures valid density range

7. Convert Point → Face (Average aggregation)
  - Evaluate on Domain node
  - Source: Point
  - Target: Face
  - Method: Average
  - Face density = average of vertex densities

8. Optional: Face-Level Adjustment (Face domain)
  - Additional procedural variation in Face domain
  - Face-based noise
  - Random per-face modulation

9. Use in Distribute Points on Faces (Face domain)
  - Density input: Processed face values
  - Distribution: More points where density higher
  - Poisson Disk for even distribution

10. Result: Sophisticated Density-Controlled Scattering
    - Artist-painted base control
    - Procedural variation for organic look
    - Global controls for easy adjustment
    - Smooth transitions from painting gradients
```

**Domain Flow**:

```
Point → Point → Point → Point → Point → Face → Face → Scatter
(Paint)(Curve)(Noise)(Global)(Clamp)(Avg) (Adjust)(Points)

Total Conversions: 1 (Point to Face)
Multiple Point domain operations before conversion
```

**Advanced Density Curve Examples**:

```
Exponential Curve (Emphasis on High):
Input:  0.0  0.25  0.5  0.75  1.0
Output: 0.0  0.06  0.25 0.56  1.0

Effect:
- Low painted values: Very low density
- High painted values: Amplified density
- Creates dramatic contrast
- Good for: Sparse → very dense transitions

Logarithmic Curve (Emphasis on Low):
Input:  0.0  0.25  0.5  0.75  1.0
Output: 0.0  0.44  0.75 0.94  1.0

Effect:
- Low painted values: Amplified density
- High painted values: Compressed
- More subtle density variation
- Good for: Ensuring minimum coverage

S-Curve (Contrast Enhancement):
Input:  0.0  0.25  0.5  0.75  1.0
Output: 0.0  0.15  0.5  0.85  1.0

Effect:
- Darkens low values
- Brightens high values
- Enhances mid-tone contrast
- Good for: Clear density zones
```

**Noise Variation Example**:

```
Adding Organic Variation:
1. Painted density: Smooth gradients
2. Noise Texture:
  - Scale: 5.0 (feature size)
  - Output: 0.0 to 1.0
3. Mix:
  - Mode: Multiply or Add
  - Factor: 0.3 (30% noise influence)
  - Painted × (1.0 + Noise × 0.3)

Result:
- Base density from painting
- Organic variation from noise
- Prevents too-perfect appearance
- Natural clustering and gaps
```

**Multi-Layer Density System**:

```
Complex Density Control:
1. Base Layer: Painted vertex weights (Point)
  - Primary control by artist
   
2. Variation Layer: Noise (Point)
  - Organic variation
  - Mix: Multiply with factor 0.3

3. Exclusion Layer: Painted exclusion weights (Point)
  - Areas to completely exclude
  - Mix: Multiply (0.0 = exclude, 1.0 = allow)

4. Global Control: Slider (Point)
  - Overall density multiplier
  - Easy animation/adjustment

5. Combine all layers (Point):
  - Base × (1 + Noise × 0.3) × Exclusion × Global

6. Convert to Face (Average)

7. Use for scattering

Result: Maximum control and flexibility
```

**Considerations**:

- **Painting Resolution**: More vertices = finer control
  - Subdivide mesh for detailed painting
  - Or paint on high-res, transfer to low-res

- **Conversion Averaging**: Face gets average of vertices
  - Quad with vertices [0.0, 0.0, 1.0, 1.0]: Face = 0.5
  - Consider: Does this give desired distribution?

- **Performance**: Complex density calculations vs. simple painting
  - Trade-off: Artist time vs. computation time
  - Procedural variation adds overhead
  - May need optimization for large scenes

- **Animation**: Density can be animated
  - Animate global multiplier
  - Animate noise offset
  - Growing/receding vegetation
  - Dynamic scatter patterns

### Pattern 10: Edge Selection from Vertex Selection - Extended

**Goal**: Convert vertex selection to edge selection with different logical rules

**Complete Workflow with Multiple Methods**:

```
Method 1: Both Endpoints Selected (Strict)
1. Vertex Selection (Point domain)
  - Boolean selection per vertex
  - True = selected, False = not selected

2. Convert Point → Edge with Min Aggregation
  - Evaluate on Domain node
  - Source: Point
  - Target: Edge
  - Method: Min (minimum of two endpoint values)

3. Result (Edge domain)
  - Edge selected (1.0) only if BOTH endpoints selected
  - Edge = Min(Vertex1, Vertex2)
  - Strict selection: Both required

Method 2: Either Endpoint Selected (Liberal)
1. Vertex Selection (Point domain)
  - Same as above

2. Convert Point → Edge with Max Aggregation
  - Evaluate on Domain node
  - Source: Point
  - Target: Edge
  - Method: Max (maximum of two endpoint values)

3. Result (Edge domain)
  - Edge selected (1.0) if EITHER endpoint selected
  - Edge = Max(Vertex1, Vertex2)
  - Liberal selection: One sufficient

Method 3: Majority/Threshold (Moderate)
1. Vertex Selection (Point domain)
  - Same as above

2. Convert Point → Edge with Average Aggregation
  - Evaluate on Domain node
  - Source: Point
  - Target: Edge
  - Method: Average

3. Compare to Threshold (Edge domain)
  - Compare node
  - Average ≥ 0.5 (adjustable threshold)
  - Result: Boolean edge selection

4. Result (Edge domain)
  - Edge selected if average ≥ threshold
  - Threshold 0.5: Edge selected if both endpoints (avg = 1.0)
  - Threshold > 0.5: Stricter (closer to "both")
  - Threshold < 0.5: More liberal (closer to "either")
```

**Domain Flow Comparison**:

```
Method 1 (Min - Both):
Point → Edge
(Select)(Min)
Conversions: 1 (Point to Edge)
Result: Strictest

Method 2 (Max - Either):
Point → Edge
(Select)(Max)
Conversions: 1 (Point to Edge)
Result: Most liberal

Method 3 (Average + Threshold):
Point → Edge → Edge
(Select)(Avg) (Compare)
Conversions: 1 (Point to Edge)
Additional: Threshold comparison
Result: Adjustable
```

**Concrete Example - 4 Vertices**:

```
Vertex Selection:
- Vertex 0: True (1.0)
- Vertex 1: True (1.0)
- Vertex 2: False (0.0)
- Vertex 3: False (0.0)

Edges:
- Edge 0-1: Between two selected vertices
- Edge 1-2: Between selected and unselected
- Edge 2-3: Between two unselected vertices
- Edge 0-3: Between selected and unselected

Method 1 (Min - Both endpoints):
- Edge 0-1: Min(1.0, 1.0) = 1.0 → Selected
- Edge 1-2: Min(1.0, 0.0) = 0.0 → Not selected
- Edge 2-3: Min(0.0, 0.0) = 0.0 → Not selected
- Edge 0-3: Min(1.0, 0.0) = 0.0 → Not selected

Result: Only edge between two selected vertices

Method 2 (Max - Either endpoint):
- Edge 0-1: Max(1.0, 1.0) = 1.0 → Selected
- Edge 1-2: Max(1.0, 0.0) = 1.0 → Selected
- Edge 2-3: Max(0.0, 0.0) = 0.0 → Not selected
- Edge 0-3: Max(1.0, 0.0) = 1.0 → Selected

Result: All edges touching any selected vertex

Method 3 (Average, Threshold 0.5):
- Edge 0-1: Avg(1.0, 1.0) = 1.0 ≥ 0.5 → Selected
- Edge 1-2: Avg(1.0, 0.0) = 0.5 ≥ 0.5 → Selected (borderline)
- Edge 2-3: Avg(0.0, 0.0) = 0.0 < 0.5 → Not selected
- Edge 0-3: Avg(1.0, 0.0) = 0.5 ≥ 0.5 → Selected (borderline)

Result: Edges with average ≥ 0.5 (both or equal split)

Method 3 (Average, Threshold 0.75):
- Edge 0-1: 1.0 ≥ 0.75 → Selected
- Edge 1-2: 0.5 < 0.75 → Not selected
- Edge 2-3: 0.0 < 0.75 → Not selected
- Edge 0-3: 0.5 < 0.75 → Not selected

Result: Only edges where both endpoints selected
```

**Use Case Scenarios**:

```
Scenario 1: Edge Loop from Selected Vertices
Goal: Select only edges entirely within selection
Method: Min aggregation (both endpoints)
Reason: Want complete edge loops, no partial edges
Result: Clean edge loop selection

Scenario 2: Grow Selection to Adjacent Edges
Goal: Select all edges touching selected vertices
Method: Max aggregation (either endpoint)
Reason: Want to expand selection to connected edges
Result: Broader edge selection, includes boundary

Scenario 3: Partial Edge Selection for Beveling
Goal: Select edges that are mostly selected
Method: Average with threshold 0.6-0.8
Reason: Want edges strongly connected to selection
Result: Majority-selected edges, excluding weak connections
```

**Advanced: Weighted Edge Selection**:

```
Vertex Selection with Weights:
1. Vertex selection values: 0.0 to 1.0 (not just boolean)
  - Fully selected: 1.0
  - Partially selected: 0.5
  - Not selected: 0.0

2. Convert to Edge with Average:
  - Edge gets average of endpoint weights

3. Use as edge selection factor:
  - No threshold: Gradient edge selection
  - With threshold: Binary edge selection

Example:
- Vertex A: 1.0 (fully selected)
- Vertex B: 0.3 (slightly selected)
- Edge A-B: Average = 0.65
- Use as factor: 65% selected
- Good for: Soft selection, partial operations
```

**Boundary Edge Detection**:

```
Finding Selection Boundary Edges:
Goal: Edges between selected and unselected vertices

Method:
1. Vertex Selection (Point domain)
2. Convert Point → Edge (Average)
3. Compare: 0.0 < Average < 1.0
  - Average = 0.0: Both unselected
  - Average = 1.0: Both selected
  - 0.0 < Average < 1.0: Mixed (boundary)

Result: Boolean boundary edge selection
- True: Edge on boundary
- False: Edge fully inside or outside selection

Use case: Edge extrusion, selection refinement
```

**Considerations**:

- **Selection Logic**: Choose aggregation based on desired behavior
  - Min: Strictest (both required)
  - Max: Most liberal (either sufficient)
  - Average + threshold: Adjustable strictness

- **Partial Selections**: Non-boolean selections (0.0 to 1.0) create gradients
  - Allow soft selections
  - Good for procedural falloffs
  - More flexible than binary

- **Performance**: All methods similar performance
  - Single domain conversion
  - Simple aggregation operation
  - Negligible overhead

- **Chaining**: Edge selection can convert to Face selection
  - Edge → Face: Faces that have selected edges
  - Multiple conversion steps possible
  - Each step applies aggregation logic

### Pattern 11: Shortest Path Weights (Edge Domain)

- **Problem**: Defining traversal costs for the "Shortest Edge Path" node
  - **Data**: Users often have "weight" or "cost" data on Vertices (Point domain)
  - **Requirement**: The "Edge Cost" input expects data in the **Edge Domain**
  - **Implicit Behavior**: Point data connected to Edge Cost is automatically converted (Point → Edge)

- **The Conversion Trap**:
  - **Input**: "Danger" attribute on Vertices (0.0 = Safe, 1.0 = Dangerous)
  - **Automatic Conversion**: Average (default)
  - **Resulting Edge Cost**:
    - Edge connecting two Safe vertices (0, 0) = 0.0
    - Edge connecting two Dangerous vertices (1, 1) = 1.0
    - Edge connecting Safe to Dangerous (0, 1) = 0.5

- **Why This Matters**:
  - The pathfinding algorithm minimizes the *sum of edge costs*.
  - Using vertex data implicitly creates a gradient cost on edges connecting different zones.
  - **Topology logic**: The `Shortest Edge Path` node propagates data *backwards* from End to Start, but calculates costs based on *Edges*.

**Correct Workflow for Vertex-Based Costs**

If you want a path to avoid specific vertices, you must ensure the edges *entering/leaving* those vertices have high costs.

- **Method**: Max Aggregation (Strictest Avoidance)
  1. Vertex "Danger" attribute (Point domain)
  2. Evaluate on Domain:
    - Source: Point
    - Target: Edge
    - Method: Max
  3. Result: Any edge touching a dangerous vertex becomes fully dangerous (1.0).
  4. Connect to Edge Cost.

- **Anti-Pattern**:
  - Connecting Point attribute directly to Edge Cost.
  - Result: The solver might traverse a "Dangerous" vertex if the adjacent "Safe" vertex lowers the averaged edge cost enough to make it the mathematical shortest path.
