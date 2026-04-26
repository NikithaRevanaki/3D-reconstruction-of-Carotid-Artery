# Function Reference & API Documentation

Comprehensive documentation of all functions in the 3D Reconstruction of Carotid Artery project.

---

## Input/Output Functions

### `readImage(file)`

Reads and preprocesses ultrasound images.

**Parameters:**
- `file` (str): Path to image file (.png or .dcm)

**Returns:**
- `img` (numpy.ndarray): Grayscale image, dtype=uint8, shape=(256, 256)

**Supports:**
- PNG images (resized to 256×256)
- DICOM medical images (requires pydicom)

**Example:**
```python
img = readImage("path/to/image.png")
print(img.shape)  # (256, 256)
```

---

### `showImage(im, title=None, numWin=None, show=True)`

Displays an image (for debugging).

**Parameters:**
- `im` (numpy.ndarray): Image to display
- `title` (str, optional): Title for the image window
- `numWin` (int, optional): Subplot position (e.g., 221 for 2×2 grid)
- `show` (bool): Whether to display immediately

**Returns:** None

**Behavior:**
- If DISPLAY=True, shows the image
- Blocks execution until user presses a button
- Auto-closes after interaction

**Example:**
```python
DISPLAY = True
showImage(img, "Ultrasound Image", numWin=111)
```

---

## Template Generation

### `LoGDisc(radedge, sigma=3, plot=True)`

Generates a Laplacian of Gaussian (LoG) template for blob detection.

**Parameters:**
- `radedge` (int): Edge radius for circular mask
- `sigma` (float): Gaussian standard deviation (default: 3)
- `plot` (bool): Whether to visualize intermediate steps

**Returns:**
- `template` (numpy.ndarray): LoG filter masked circularly

**Mathematical Background:**
- Creates a Gaussian blob: exp(-(x² + y²)/(2σ²))
- Applies Laplacian: ∇²G = d²G/dx² + d²G/dy²
- Masks with circle of radius `radedge`

**Example:**
```python
template = LoGDisc(radedge=16, sigma=3)
print(template.shape)  # (49, 49) - odd dimension
```

---

## Center Detection Functions

### `templateMatching(img, tpl)`

Cross-correlates image with template using pixel-wise dot product.

**Parameters:**
- `img` (numpy.ndarray): Input image, shape (H, W)
- `tpl` (numpy.ndarray): Template, usually odd dimensions

**Returns:**
- `corIm` (numpy.ndarray): Correlation image, same shape as img

**Algorithm:**
```
For each pixel (r, c):
    Extract sub-image centered at (r, c)
    corIm[r, c] = sum(sub_image * template)
```

**Complexity:** O(H × W × Tpl_H × Tpl_W)

**Example:**
```python
correlation = templateMatching(img, lpl_template)
peak_value = np.max(correlation)
peak_location = np.unravel_index(np.argmax(correlation), correlation.shape)
```

---

### `findMaxRegion(cimg)`

Finds the location and value of maximum correlation.

**Parameters:**
- `cimg` (numpy.ndarray): Correlation image

**Returns:**
- `(max_loc, max_value)` tuple:
  - `max_loc` (tuple): (x, y) coordinates of peak
  - `max_value` (float): Correlation value at peak

**Example:**
```python
loc, max_corr = findMaxRegion(correlation_image)
print(f"Center at: ({loc[0]}, {loc[1]}), Correlation: {max_corr}")
```

---

### `findMaxCorrelation(img, rho, sigma)`

Complete center detection for a single image.

**Parameters:**
- `img` (numpy.ndarray): Input ultrasound image
- `rho` (int): Expected artery radius (e.g., 16)
- `sigma` (int): LoG filter spread (e.g., 4)

**Returns:**
- `(cimg, maxloc, maxs)` tuple:
  - `cimg`: Correlation image
  - `maxloc`: (x, y) center coordinates
  - `maxs`: Actual radius offset used

**Algorithm:**
1. For each radius variation from rho to rho+sigma:
   - Generate LoG template
   - Perform template matching
   - Track maximum correlation
2. Return best match

**Example:**
```python
corr_img, center, radius_offset = findMaxCorrelation(img, 16, 4)
print(f"Detected center: {center}")
```

---

## Boundary Extraction Functions

### `getStrip(img, xc, yc)`

Extracts a radial strip around center (polar to Cartesian conversion).

**Parameters:**
- `img` (numpy.ndarray): Input image
- `xc`, `yc` (int): Center coordinates

**Returns:**
- `strip` (numpy.ndarray): Polar image, shape (360, RMAX-RMIN)

**Description:**
- Samples image at 360 angles (0-359°)
- For each angle, samples from RMIN to RMAX radius
- Creates "unwrapped" view of circular region

**Example:**
```python
strip = getStrip(img, center_x=128, center_y=128)
print(strip.shape)  # (360, 75) if RMIN=5, RMAX=80
plt.imshow(np.transpose(strip), cmap='gray')
```

---

### `getEdgeMap(strip)`

Detects edges in the radial strip using a difference mask.

**Parameters:**
- `strip` (numpy.ndarray): Polar image from getStrip()

**Returns:**
- `edgeIm` (numpy.ndarray): Edge strength at each angle and radius

**Algorithm:**
```
For each angle and radius:
    edge = sum(strip[angle, r-MASK_SIZE:r+MASK_SIZE] × mask)
    edgeIm[angle, radius] = edge
```

**Mask:** `[-1, -1, -1, 1, 1, 1] / MASK_SIZE` (difference filter)

**Example:**
```python
edge_map = getEdgeMap(strip)
max_edge_per_angle = np.argmax(edge_map, axis=1)
```

---

### `verifyLoc(img, loc)`

Evaluates quality of detected center using boundary smoothness.

**Parameters:**
- `img` (numpy.ndarray): Input image
- `loc` (tuple): (x, y) center coordinates

**Returns:**
- `score` (float): Quality score based on boundary continuity

**Higher score** = Better quality detection

**Example:**
```python
score = verifyLoc(img, detected_center)
if score > threshold:
    print("Good detection")
```

---

## Boundary Refinement Functions

### `getBoundary(im, loc)`

Complete boundary detection pipeline.

**Parameters:**
- `im` (numpy.ndarray): Input image
- `loc` (tuple): Center coordinates (x, y)

**Returns:**
- `smoothBPts` (numpy.ndarray): Smoothed boundary points, shape (360,)

**Pipeline:**
1. Extract radial strip
2. Detect edges
3. Find max edge per angle
4. Remove outliers
5. Adjust discontinuities
6. Apply Savitzky-Golay smoothing

**Example:**
```python
boundary = getBoundary(img, (128, 128))
print(f"Boundary range: {boundary.min()}-{boundary.max()}")
```

---

### `getEdgeMap(strip)`

See [Boundary Extraction](#boundary-extraction-functions) section above.

---

### `removeOutliers(maxEdge)`

Removes discontinuous boundary points using interpolation.

**Parameters:**
- `maxEdge` (numpy.ndarray): Raw edge positions, shape (360,)

**Modifies:** `maxEdge` in-place

**Returns:** None

**Algorithm:**
1. Find longest continuous segment
2. Set endpoints to segment average
3. Interpolate gaps using linear interpolation

**Example:**
```python
boundary = np.array([...])
removeOutliers(boundary)  # Modifies boundary in-place
```

---

### `longestRun(maxEdge)`

Finds the longest continuous segment in boundary data.

**Parameters:**
- `maxEdge` (numpy.ndarray): Edge positions

**Returns:**
- `avg` (float): Average value of longest continuous segment

**Helper Function:** Used by removeOutliers()

**Example:**
```python
avg_edge = longestRun(boundary)
print(f"Longest run average: {avg_edge}")
```

---

### `adjustBoundary(strip, maxIDs)`

Fixes boundary discontinuities by local optimization.

**Parameters:**
- `strip` (numpy.ndarray): Radial strip
- `maxIDs` (numpy.ndarray): Edge positions per angle (modified in-place)

**Modifies:** `maxIDs` in-place

**Returns:** None

**Algorithm:**
For each large jump between consecutive angles:
1. Find the best matching edge in neighborhood
2. Replace discontinuous point with best match

**Example:**
```python
boundary = np.array([...])
strip = getStrip(img, loc[0], loc[1])
adjustBoundary(strip, boundary)
```

---

### `smoothBoundary(pts)`

Applies Savitzky-Golay smoothing to boundary points.

**Parameters:**
- `pts` (numpy.ndarray): Raw boundary points, shape (360,)

**Returns:**
- `smooth_intpts` (numpy.ndarray): Smoothed boundary points, shape (360,)

**Algorithm:**
1. Extend boundary with wrapped overlap (cyclic padding)
2. Apply Savitzky-Golay filter (window=101, order=3)
3. Extract original region

**Example:**
```python
raw_boundary = np.array([...])
smooth_boundary = smoothBoundary(raw_boundary)
```

---

## 3D Reconstruction Functions

### `getCarotidPts(img)`

Extracts the carotid artery point set from within the detected boundary.

**Parameters:**
- `img` (numpy.ndarray): Input ultrasound image

**Returns:**
- `carotidPts4` (list): Points within artery region

**Algorithm:**
1. Detect boundary positions
2. Create circular region from boundary
3. Sample points within circle
4. Filter by pixel intensity (very selective)

**Example:**
```python
points = getCarotidPts(img)
print(f"Found {len(points)} points in artery")
```

---

### `show3dImage()`

Main function for 3D reconstruction and visualization.

**Parameters:** None

**Returns:** None

**Output:** Generates interactive HTML file: `Ultrasound_DD_MM_YYYY_HH_MM.html`

**Algorithm:**
1. Process each frame (0-299):
   - Detect artery center
   - Extract boundary
   - Extract point cloud
2. Filter outliers using z-score (threshold=0.1)
3. Convert from polar to Cartesian coordinates
4. Create 3D surface and point cloud visualization
5. Export as interactive Plotly HTML

**Configuration (global variables used):**
- `path`: Image directory
- `RADIUS`, `SPREAD`: Center detection
- `RMIN`, `RMAX`: Boundary search range

**Example:**
```python
# Configure globally
path = "path/to/frames/"
RADIUS = 16
SPREAD = 4

# Run reconstruction
show3dImage()
# Output: Ultrasound_26_04_2026_10_30.html
```

---

## Utility Functions

### `progress(value, max=300)`

Creates a progress bar for Jupyter notebooks.

**Parameters:**
- `value` (int): Current progress
- `max` (int): Maximum value (default: 300)

**Returns:**
- HTML widget for progress display

**Example:**
```python
out = display(progress(0, 300), display_id=True)
for i in range(300):
    out.update(progress(i, 300))
```

---

### `plotLineHisto(line, width=None)`

Plots a 1D histogram/line graph.

**Parameters:**
- `line` (array-like): Data to plot
- `width` (int): Optional width label

**Returns:** None

**Side Effects:** Opens matplotlib window

---

### `deleteLocMax(img, loc)`

Sets a local region around a point to zero.

**Parameters:**
- `img` (numpy.ndarray): Image (modified in-place)
- `loc` (tuple): (x, y) center of region to delete

**Modifies:** `img` in-place, sets region around `loc` to 0

**Region Size:** ±LOCAL pixels (LOCAL=20 by default)

**Example:**
```python
deleteLocMax(img, (128, 128))
# Pixels in img[108:148, 108:148] are now 0
```

---

## Global Constants

```python
LOCAL = 20           # Region size for deleteLocMax
RADIUS = 16          # Expected artery radius in pixels
SPREAD = 4           # Uncertainty in radius
RMIN = 5             # Minimum search radius
RMAX = 80            # Maximum search radius
MASK_SIZE = 3        # Edge detection kernel size
MAX_NODE = 3         # Maximum nodes (unused)
DISPLAY = False      # Debug visualization flag
THRESHOLD = 7        # Boundary continuity threshold
OVERLAP = 10         # Smoothing window overlap
```

---

## Data Flow Example

```python
# 1. Read image
img = readImage("US0003_0000.png")  # shape: (256, 256)

# 2. Detect center
corr_img, center, max_s = findMaxCorrelation(img, RADIUS, SPREAD)
# center = (128, 115), max_s = 0

# 3. Extract boundary
boundary = getBoundary(img, center)  # shape: (360,)
# boundary values: [10, 11, 12, ..., 9]  # 360 points around circle

# 4. Convert to Cartesian coordinates
x_coords = []
y_coords = []
z_coords = []
for theta in range(360):
    th = theta * 2 * np.pi / 360
    x = center[0] + (boundary[theta] + RMIN) * np.cos(th)
    y = center[1] + (boundary[theta] + RMIN) * np.sin(th)
    z = 0  # Frame number
    x_coords.append(x)
    y_coords.append(y)
    z_coords.append(z)

# 5. Repeat for all frames and create 3D plot
```

---

## Performance Reference

| Function | Time | Calls/Run |
|----------|------|-----------|
| readImage | 1-5 ms | 300 |
| findMaxCorrelation | 200-500 ms | 300 |
| getBoundary | 50-100 ms | 300 |
| getCarotidPts | 100-200 ms | 300 |
| 3D Plotting | 1-2 s | 1 |
| **Total** | **2-5 min** | |

---

## Error Handling

Functions do minimal error handling. Recommended practice:

```python
try:
    img = readImage(filepath)
    if img is None:
        print(f"Failed to load: {filepath}")
        continue
    
    center = findMaxCorrelation(img, RADIUS, SPREAD)
    if center[0] == 0 and center[1] == 0:
        print(f"No artery detected in frame")
        continue
    
    boundary = getBoundary(img, center)
    
except Exception as e:
    print(f"Error processing frame: {e}")
    continue
```

---

## Tips & Tricks

1. **Batch Processing:**
   ```python
   for frame_num in range(300):
       filepath = f"{path}US0003_{frame_num:04d}.png"
       if os.path.exists(filepath):
           process_frame(filepath)
   ```

2. **Memory Efficiency:**
   ```python
   # Process frames in chunks instead of keeping all in memory
   for chunk_start in range(0, 300, 30):
       chunk_end = min(chunk_start + 30, 300)
       process_chunk(chunk_start, chunk_end)
   ```

3. **Visualization During Processing:**
   ```python
   # Save debug images every N frames
   if frame_num % 10 == 0:
       visualize_boundary(img, boundary, frame_num)
   ```

---

**Last Updated**: April 2026  
**Version**: 1.0  
**API Stability**: Stable

