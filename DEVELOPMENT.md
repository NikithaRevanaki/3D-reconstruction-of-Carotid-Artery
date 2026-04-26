# Development Guide

This guide is for developers interested in understanding, modifying, or contributing to the 3D Reconstruction of Carotid Artery project.

## Code Architecture

### High-Level Flow

```
Input Images (PNG/DICOM)
    ↓
[Preprocessing] → Image resizing and normalization
    ↓
[Center Detection] → LoG template matching
    ↓
[Boundary Extraction] → Radial strip analysis
    ↓
[Edge Detection] → Pixel-level edge finding
    ↓
[Outlier Removal] → Statistical cleaning
    ↓
[Boundary Smoothing] → Savitzky-Golay filtering
    ↓
[3D Stacking] → Combine boundaries from all frames
    ↓
[Visualization] → Plotly 3D rendering
    ↓
Output (HTML interactive 3D model)
```

---

## Module Organization

### Main Functions by Category

#### Image I/O
```python
readImage(file)                    # Load PNG or DICOM images
showImage(im, title, numWin, show) # Display intermediate results
```

#### Template Generation
```python
LoGDisc(radedge, sigma, plot)      # Create LoG template
```

#### Center Detection
```python
templateMatching(img, tpl)         # Correlate image with template
findMaxRegion(cimg)                # Find correlation peak
findMaxCorrelation(img, rho, sigma)# Complete center detection
```

#### Edge Detection
```python
getStrip(img, xc, yc)              # Extract radial strip
getEdgeMap(strip)                  # Apply edge detection
verifyLoc(img, loc)                # Verify detection quality
```

#### Boundary Refinement
```python
removeOutliers(maxEdge)            # Remove discontinuous points
longestRun(maxEdge)                # Find continuous segment
adjustBoundary(strip, maxIDs)      # Fix boundary jumps
smoothBoundary(pts)                # Apply Savitzky-Goyal filter
```

#### Complete Pipeline
```python
getBoundary(im, loc)               # End-to-end boundary detection
getCarotidPts(img)                 # Extract point cloud
show3dImage()                       # Main 3D reconstruction
```

---

## Key Data Structures

### Images
```python
img              # numpy.ndarray, shape (H, W), dtype uint8
                 # Grayscale image (0-255)
```

### Correlations
```python
corIm            # numpy.ndarray, shape (H, W), dtype int
                 # Template correlation at each pixel
maxloc           # tuple (x, y) - location of peak correlation
cuRMAX           # int - peak correlation value
```

### Boundary Data
```python
strip            # numpy.ndarray, shape (360, RMAX-RMIN), dtype uint8
                 # Polar coordinates: [angle, radius] = intensity
maxEdge          # numpy.ndarray, shape (360,), dtype int
                 # Edge position for each angle (0-360 degrees)
smoothBPts       # numpy.ndarray, shape (360,), after filtering
```

### 3D Points
```python
xl, yl, zl       # lists of lists
                 # xl[frame] = [x_0, x_1, ..., x_359]
                 # yl[frame] = [y_0, y_1, ..., y_359]
                 # zl[frame] = [z_0, z_1, ..., z_359] (all same value)

xs, ys, zs       # lists of filtered points
                 # Point cloud after outlier removal
```

---

## Algorithm Deep Dive

### 1. Laplacian of Gaussian (LoG) Template

```python
def LoGDisc(radedge, sigma=3, plot=True):
    # 1. Create Gaussian blob
    gaussian[center, center] = 1
    gaussian = gaussian_filter(gaussian, sigma)
    
    # 2. Apply Laplacian operator (edge detector)
    LoGTpl = Laplacian(gaussian, ...)
    
    # 3. Create circular mask
    circularEdge = zeros(...)
    for each pixel:
        if distance > radedge: circularEdge = 0
    
    # 4. Apply circular mask to LoG
    template = convolve(circularEdge, LoGTpl)
    
    return template
```

**Why LoG?**
- Detects circular structures (blob detection)
- Sensitive to edges with specific sizes
- Enhanced by circular masking for artery-like shapes

### 2. Template Matching

```python
def templateMatching(img, tpl):
    for each pixel (r, c):
        subimage = img[r-h:r+h, c-w:c+w]
        correlation = sum(subimage * tpl)
        corIm[r, c] = correlation
    
    return corIm  # High values = good matches
```

**Complexity**: O(H × W × Tpl_H × Tpl_W)  
**Optimization**: Can use FFT convolution for large templates

### 3. Radial Strip Extraction

```python
def getStrip(img, xc, yc):
    strip = zeros(360, RMAX-RMIN)
    for theta in 0..359:
        for radius in RMIN..RMAX:
            # Convert polar to Cartesian
            x = xc + r * cos(theta)
            y = yc + r * sin(theta)
            strip[theta, r-RMIN] = img[y, x]
    
    return strip  # Unwrapped polar image
```

**Visualization**: Think of "unwrapping" the circular region around center

### 4. Edge Detection in Polar Space

```python
def getEdgeMap(strip):
    for each angle in 0..359:
        for each radius in MASK_SIZE..nodes:
            # Apply mask: [-1, -1, -1, 1, 1, 1]
            edge = sum(strip[angle, r-3:r+3] * mask)
            edgeMap[angle, radius] = edge
    
    maxEdge[angle] = argmax(edgeMap[angle, :])
    return maxEdge  # Strongest edge per angle
```

**Effect**: Finds sharp transitions in intensity

### 5. Outlier Removal (Longest Run Algorithm)

```python
def removeOutliers(maxEdge):
    # 1. Find longest continuous run
    best, len = find_longest_continuous_segment(maxEdge)
    avg = mean(maxEdge[best:best+len])
    
    # 2. Set boundaries to average
    maxEdge[0] = avg
    maxEdge[-1] = avg
    
    # 3. Find and interpolate gaps
    for each discontinuity:
        # Linear interpolation between valid points
        maxEdge[gap] = interpolate(before, after)
    
    return maxEdge  # Cleaned boundary
```

**Why this works**: Removes noise while preserving structure

### 6. Boundary Smoothing

```python
def smoothBoundary(pts):
    # Wrap boundary for cyclic smoothing
    extended = [pts[-OVERLAP:], pts, pts[:OVERLAP]]
    
    # Apply Savitzky-Golay filter
    # (polynomial fit within window)
    smooth = savgol_filter(extended, window=101, order=3)
    
    # Extract original region
    return smooth[OVERLAP:len(pts)+OVERLAP]
```

**Why Savitzky-Golay?**
- Preserves edges while smoothing
- Polynomial fitting keeps local structure
- Better than simple averaging

### 7. 3D Reconstruction

```python
def show3dImage():
    for each frame i:
        center = findMaxCorrelation(img)
        boundary = getBoundary(img, center)
        
        # Convert from polar to Cartesian
        for angle in 0..359:
            x = center_x + (boundary[angle] + RMIN) * cos(angle)
            y = center_y + (boundary[angle] + RMIN) * sin(angle)
            z = frame_number
            
            xl, yl, zl append (x, y, z)
    
    # Filter outliers using z-score
    for point in points:
        zx = (point.x - mean_x) / std_x
        zy = (point.y - mean_y) / std_y
        if zx < threshold and zy < threshold:
            include in point cloud
    
    # Create 3D visualization
    fig = Surface(z=zl, x=xl, y=yl) + Scatter3D(points)
```

---

## Performance Characteristics

### Computational Complexity

| Function | Complexity | Time (280 frames) |
|----------|-----------|------------------|
| readImage | O(H·W) | < 1s |
| LoGDisc | O(r²) | < 0.1s per template |
| templateMatching | O(H·W·r²) | 0.2-0.5s |
| getStrip | O(360·R) | < 0.01s |
| getEdgeMap | O(360·R) | < 0.01s |
| removeOutliers | O(360) | < 0.01s |
| smoothBoundary | O(360·window) | < 0.01s |
| **Total per frame** | | **0.4-1.0s** |
| **Total batch (300 frames)** | | **2-5 minutes** |

### Memory Usage

| Data | Size (300 frames) |
|------|------------------|
| Images (cached) | ~50 MB |
| LoG templates | ~2 MB |
| Correlation maps | ~200 MB |
| 3D point arrays | ~10 MB |
| **Total** | **~260 MB** |

---

## Common Modifications

### 1. Add DICOM Support
```python
def readImage(file):
    if file.endswith('.dcm'):
        import pydicom
        dcm = pydicom.dcmread(file)
        img = dcm.pixel_array
        # Normalize to 0-255
        img = (img - img.min()) / (img.max() - img.min()) * 255
        return np.uint8(img)
    else:
        # Existing PNG code
        ...
```

### 2. Add Visualization During Processing
```python
# In show3dImage() after getBoundary():
if i % 10 == 0:  # Every 10 frames
    plt.subplot(2, 3, i//50 + 1)
    plt.imshow(img, cmap='gray')
    # Plot boundary
    for t in range(360):
        th = t * 2 * np.pi / 360
        x = loc[0] + (BPts[t] + RMIN) * np.cos(th)
        y = loc[1] + (BPts[t] + RMIN) * np.sin(th)
        plt.plot(x, y, 'r.')
```

### 3. Add Multi-threading
```python
from multiprocessing import Pool

def process_frame(frame_path):
    img = readImage(frame_path)
    center = findMaxCorrelation(img, RADIUS, SPREAD)
    boundary = getBoundary(img, center)
    return boundary

if __name__ == '__main__':
    with Pool(4) as p:  # 4 cores
        boundaries = p.map(process_frame, frame_paths)
```

### 4. Export to Different Formats
```python
# Export to PLY format
def export_ply(xl, yl, zl, filename):
    with open(filename, 'w') as f:
        # Write PLY header
        points = []
        for frame in range(len(xl)):
            for angle in range(360):
                points.append([xl[frame][angle], yl[frame][angle], zl[frame][angle]])
        
        f.write(f"ply\nformat ascii 1.0\n")
        f.write(f"element vertex {len(points)}\n")
        f.write(f"property float x\nproperty float y\nproperty float z\n")
        f.write(f"end_header\n")
        
        for p in points:
            f.write(f"{p[0]} {p[1]} {p[2]}\n")
```

### 5. Add Batch Processing
```python
def process_patient_batch(patient_dirs):
    results = {}
    for patient_dir in patient_dirs:
        path = patient_dir
        results[patient_dir] = show3dImage()  # Modify to return data
    return results
```

---

## Testing Recommendations

### Unit Tests
```python
def test_templateMatching():
    # Create synthetic image with known peak
    img = np.zeros((100, 100))
    img[50, 50] = 255  # Bright spot
    
    # Test correlation
    corr = templateMatching(img, simple_template)
    assert corr[50, 50] > corr[0, 0]  # Peak should be highest

def test_longestRun():
    maxEdge = np.array([0, 5, 6, 7, 0, 20, 21, 22, 23])
    avg = longestRun(maxEdge)
    assert 20 <= avg <= 23  # Should return center of longest run
```

### Integration Tests
```python
def test_complete_pipeline():
    img = readImage("test_image.png")
    center = findMaxCorrelation(img, 16, 4)
    boundary = getBoundary(img, center)
    assert len(boundary) == 360  # Should have 360 boundary points
    assert all(5 <= b <= 80 for b in boundary)  # Within expected range
```

### Validation Tests
```python
def test_output_quality():
    # Generate 3D model
    show3dImage()  # Generates HTML
    
    # Verify HTML was created
    assert os.path.exists("Ultrasound_*.html")
    
    # Verify it's valid JSON/HTML
    with open("Ultrasound_*.html") as f:
        html = f.read()
        assert "plotly" in html.lower()
```

---

## Debugging Tips

### Enable Debug Output
```python
# Add to main.py
import logging
logging.basicConfig(level=logging.DEBUG)

# In functions:
logging.debug(f"Processing frame {i}, center: {loc}")
logging.debug(f"Boundary stats: min={minE}, max={maxE}, mean={meanE}")
```

### Visualize Intermediate Results
```python
# In show3dImage() after finding boundary:
if i % 30 == 0:  # Every 30 frames
    plt.figure(figsize=(15, 5))
    
    plt.subplot(131)
    plt.imshow(img, cmap='gray')
    plt.title(f"Frame {i} - Raw Image")
    plt.plot(loc[0], loc[1], 'r+', markersize=20)
    
    plt.subplot(132)
    strip = getStrip(img, loc[0], loc[1])
    plt.imshow(np.transpose(strip), cmap='gray')
    plt.title("Radial Strip")
    
    plt.subplot(133)
    edgeIm = getEdgeMap(strip)
    plt.imshow(np.transpose(edgeIm), cmap='hot')
    plt.plot(BPts)
    plt.title("Edge Map + Boundary")
    
    plt.tight_layout()
    plt.savefig(f"debug_frame_{i}.png")
    plt.close()
```

### Profiling Code
```python
import cProfile
import pstats

profiler = cProfile.Profile()
profiler.enable()

show3dImage()  # Your function

profiler.disable()
stats = pstats.Stats(profiler)
stats.sort_stats('cumulative')
stats.print_stats(10)  # Top 10 functions
```

---

## Contributing Guidelines

1. **Create a feature branch**: `git checkout -b feature/my-feature`
2. **Add docstrings**: Document all new functions
3. **Add tests**: Include unit tests for new code
4. **Run linting**: Use PEP 8 style guide
5. **Submit PR**: With clear description of changes

---

## Resource Links

- **OpenCV Documentation**: https://docs.opencv.org/
- **Scipy Filtering**: https://docs.scipy.org/doc/scipy/reference/ndimage.html
- **Plotly 3D**: https://plotly.com/python/3d-plots/
- **Numpy Array Operations**: https://numpy.org/doc/stable/

---

**Last Updated**: April 2026  
**Version**: 1.0

