# Configuration Guide

This document provides detailed information about all configurable parameters in the 3D Reconstruction project.

## Core Parameters

### Region Processing
```python
LOCAL = 20  # Size of local region around detected center
```
- **Default**: 20 pixels
- **Range**: 10-50 pixels
- **Effect**: Controls deletion area around found maxima to prevent redetection
- **Recommendation**: Keep default unless you have very small arteries

---

### Template Matching Parameters

#### RADIUS and SPREAD
```python
RADIUS = 16    # Expected radius of artery cross-section
SPREAD = 4     # Uncertainty in radius
```

**RADIUS** - Expected artery diameter in terms of pixel radius
- **Default**: 16 pixels (diameter ~32 pixels)
- **Range**: 8-35 pixels
- **How to determine**: 
  - Measure artery diameter in your ultrasound images
  - Convert to pixels (measure in image coordinates)
  - Use RADIUS = diameter / 2
- **For small arteries**: Decrease to 8-12
- **For large arteries**: Increase to 20-30

**SPREAD** - Template matching uncertainty
- **Default**: 4 pixels
- **Range**: 1-10 pixels
- **Effect**: Tests templates from RADIUS to RADIUS+SPREAD
- **Recommendation**: 
  - Solid arteries: Use 2-3
  - Variable size arteries: Use 4-6
  - High noise images: Use 6-10

---

### Search Range

```python
RMIN = 5       # Minimum search radius
RMAX = 80      # Maximum search radius
```

**RMIN** (Minimum Radius)
- **Default**: 5 pixels
- **Effect**: Ignores detections closer than RMIN pixels from center
- **Use case**: Filter out noise and very small artifacts

**RMAX** (Maximum Radius)
- **Default**: 80 pixels
- **Effect**: Only searches up to RMAX pixels from center
- **Recommendation**: Set to slightly larger than your largest artery diameter

---

### Edge Detection

#### MASK_SIZE
```python
MASK_SIZE = 3  # Kernel size for edge detection
```
- **Default**: 3 pixels
- **Options**: 3 (recommended), 5, 7
- **Effect**: Larger masks smooth edges more but may lose detail
- **For sharp edges**: Use 3
- **For noisy images**: Use 5 or 7

#### Custom Mask
```python
mask = np.array([-1] * MASK_SIZE + [1] * MASK_SIZE) / MASK_SIZE
```
- **Purpose**: Detects edges by differencing left and right pixels
- **Current**: [-1, -1, -1, 1, 1, 1] normalized by MASK_SIZE
- **Modified detection**: Adjust the mask pattern if needed

---

## Boundary Detection Parameters

### Continuity Threshold
```python
THRESHOLD = 7  # Maximum allowed jump in edge position
```
- **Default**: 7 pixels
- **Range**: 3-15 pixels
- **Effect**: Points with larger jumps to neighbors are considered discontinuous
- **Solid boundaries**: Use smaller values (3-5)
- **Irregular arteries**: Use larger values (10-15)

---

### Boundary Smoothing

#### OVERLAP
```python
OVERLAP = 10   # Number of points for cyclic boundary smoothing
```
- **Default**: 10 points
- **Range**: 5-20 points
- **Effect**: Wraps boundary endpoints for continuous smoothing
- **For smoother results**: Increase to 15-20
- **For sharper features**: Decrease to 5-8

#### Savitzky-Golay Filter
```python
# In smoothBoundary() function, line 277:
smooth_intpts = np.array(savgol_filter(extEdges, 101, 3), int)
```
- **Window length**: 101 (must be odd)
  - Larger values → More smoothing
  - Recommended range: 51-151
- **Polynomial order**: 3
  - Higher values → Preserves details better
  - Recommended range: 2-5

---

## 3D Reconstruction Parameters

### Processing Parameters
```python
MAX_NODE = 3   # Maximum nodes for processing (currently unused)
```

### Frame Processing
- **Default processing**: Every frame processed
- **To sample frames**: Modify loop in `show3dImage()` at line 310:
  ```python
  for i in range(0, 300, 5):  # Process every 5th frame
  ```

### Outlier Removal for Points

In `show3dImage()` function, lines 337-348:
```python
threshold = 0.1  # Z-score threshold for outliers
```
- **Default**: 0.1 (very strict)
- **Range**: 0.05-1.0
- **Effect**: Filters points that deviate from mean
- **Less filtering**: Increase to 0.5-1.0
- **More filtering**: Decrease to 0.05-0.08

---

## File I/O Parameters

### Image Path
```python
path = "/content/sample_data/patient 3/"
```
- Set to your data directory
- Use forward slashes or double backslashes for Windows paths

### Image Naming Format
```python
name = path+"US0003_{:04d}.png".format(i)  # Line 316
```
- Current format: `US0003_0000.png`, `US0003_0001.png`, etc.
- Modify the format string to match your naming convention

### Output Configuration
```python
fig.update_layout(title='Artries visualization', autosize=True,)
# Line 379: Plotly layout settings
```
- Customize title, size, colors, etc.

---

## Advanced Tuning Guide

### For Noisy Images

```python
SPREAD = 6           # More forgiving template matching
THRESHOLD = 10       # Allow larger jumps
MASK_SIZE = 5        # Smoother edge detection
# In smoothBoundary: increase window length to 151
```

### For High-Quality Images

```python
SPREAD = 2           # Precise template matching
THRESHOLD = 4        # Strict continuity
MASK_SIZE = 3        # Preserve fine details
# In smoothBoundary: use window length of 51
```

### For Different Artery Sizes

**Small arteries (diameter < 10mm / < 15 pixels)**
```python
RADIUS = 8
RMAX = 30
```

**Medium arteries (diameter 10-20mm / 15-30 pixels)**
```python
RADIUS = 16         # Default
RMAX = 80           # Default
```

**Large arteries (diameter > 20mm / > 30 pixels)**
```python
RADIUS = 25
RMAX = 150
```

---

## Performance Optimization

### Memory Optimization
```python
# Process in batches instead of all 300 frames
for i in range(0, 300, 10):  # Process every 10th frame
```

### Speed Optimization
```python
# In findMaxCorrelation() - reduce template variations
for s in range(0, 1):  # Default
# Change to skip every other radius variation:
for s in range(0, 2):  # Tests two radius values
```

---

## Visualization Parameters

### Plotly Figure Settings
```python
fig = go.Figure(data=[...])
fig.update_layout(
    title='Artries visualization',
    autosize=True,
    # Add these for more control:
    width=1200,
    height=800,
    # Camera angle:
    scene=dict(
        xaxis=dict(title='X'),
        yaxis=dict(title='Y'),
        zaxis=dict(title='Z'),
        camera=dict(
            eye=dict(x=1.5, y=1.5, z=1.5)
        )
    )
)
```

### Color Schemes
```python
# Change colorscale in line 377:
colorscale="Reds_r"        # Current
# Options: "Blues", "Greens", "Viridis", "Plasma", etc.
```

---

## Debugging Parameters

### Display Mode
```python
DISPLAY = False  # Set to True for debug visualization
```
When enabled, shows intermediate images and plots.

### Progress Tracking
- Progress bar shown in lines 309-311
- Modify update frequency if needed

---

## Example Configurations

### Toshiba Ultrasound System
```python
RADIUS = 14
SPREAD = 3
RMIN = 4
RMAX = 70
MASK_SIZE = 3
THRESHOLD = 6
```

### Ultrasonix Device
```python
RADIUS = 18
SPREAD = 5
RMIN = 5
RMAX = 85
MASK_SIZE = 3
THRESHOLD = 7
```

### Research-grade Images
```python
RADIUS = 16
SPREAD = 2
RMIN = 5
RMAX = 80
MASK_SIZE = 3
THRESHOLD = 5
```

---

## Troubleshooting Parameters

**Symptom**: "Artery center not detected"
- Increase SPREAD to 6-8
- Increase LOCAL to 30-40
- Check image path correctness

**Symptom**: "Jagged 3D surface"
- Decrease THRESHOLD to 3-4
- Increase smoothing filter window length to 151
- Increase OVERLAP to 15-20

**Symptom**: "Missing frames/gaps in 3D model"
- Decrease THRESHOLD (too strict)
- Adjust RADIUS to better match artery size
- Check image quality in those frames

**Symptom**: "Over-smoothed details"
- Decrease smoothing filter window length to 51
- Increase MASK_SIZE to 5 only if noise is severe
- Increase polynomial order in savgol_filter

---

## Configuration File Best Practices

1. **Document your changes** with comments
2. **Keep backups** of working configurations
3. **Test incrementally** - change one parameter at a time
4. **Record results** to find optimal settings
5. **Create presets** for different ultrasound devices

---

**Last Updated**: April 2026  
**Version**: 1.0

