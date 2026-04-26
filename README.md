# 3D Reconstruction of Carotid Artery

A Python-based project for automatic 3D reconstruction and visualization of the carotid artery from ultrasound image sequences. This project processes 2D ultrasound images to detect artery boundaries and reconstructs them into an interactive 3D model.

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Project Structure](#project-structure)
- [Installation](#installation)
- [Usage](#usage)
- [Algorithm Overview](#algorithm-overview)
- [Data Information](#data-information)
- [Output](#output)
- [Requirements](#requirements)
- [Contributing](#contributing)
- [License](#license)

## Overview

This project automates the process of detecting and reconstructing 3D models of the carotid artery from transversal ultrasound image sequences. The algorithm uses advanced image processing techniques including:

- **Template Matching**: Utilizing Laplacian of Gaussian (LoG) filters to detect circular structures (artery cross-sections)
- **Edge Detection**: Identifying precise artery boundaries in each slice
- **3D Reconstruction**: Stacking detected boundaries from multiple frames to create a 3D model
- **Outlier Removal**: Statistical filtering to remove noise and anomalies
- **Boundary Smoothing**: Savitzky-Golay filtering for smooth curves

## Features

✓ **Automatic Artery Detection** - Template matching with adaptive LoG filters  
✓ **Edge Detection and Refinement** - Multi-step boundary detection with outlier removal  
✓ **3D Reconstruction** - Converts 2D slices into interactive 3D models  
✓ **Interactive Visualization** - Plotly-based 3D visualization with rotation and zoom  
✓ **Batch Processing** - Handles sequences of ultrasound images  
✓ **Statistical Outlier Removal** - Z-score based filtering for noise reduction  
✓ **Support for Multiple Imaging Devices** - Toshiba and Ultrasonix ultrasound systems  

## Project Structure

```
3D Reconstruction of Carotid Artery/
├── README.md                          # Project documentation
├── main.py                            # Main executable script
├── final.ipynb                        # Complete pipeline notebook
├── main_1.ipynb                       # Alternative implementation
├── 3D_plot.ipynb                      # 3D visualization notebook
├── Filtration.ipynb                   # Filtering techniques
├── dummy.py.ipynb                     # Testing notebook
├── data2.ply                          # Sample PLY format output
├── sample.xyz                         # Sample point cloud data
├── ARTERY_TRANSVERSAL/                # Training/test datasets
│   ├── Toshiba_test/                  # Toshiba ultrasound test images
│   ├── Ultrasonix_test/               # Ultrasonix ultrasound test images
│   └── Ultrasonix_train/              # Ultrasonix training images
├── data/                              # Additional data files
├── output/                            # Generated 3D model outputs
└── output_old/                        # Previous outputs archive
```

## Installation

### Prerequisites

- Python 3.7 or higher
- pip (Python package manager)

### Step 1: Clone or Download the Project

```bash
cd "C:\Users\Nikitha R\3D Reconstruction of Carotid Artery"
```

### Step 2: Install Required Dependencies

```bash
pip install -r requirements.txt
```

Or install manually:

```bash
pip install opencv-python numpy scipy matplotlib scikit-image plotly ipython ipympl
```

### Step 3: Optional Dependencies

For DICOM image support (medical imaging):

```bash
pip install pydicom
```

## Usage

### Method 1: Using the Main Python Script

```bash
python main.py
```

This will process ultrasound image sequences and generate a 3D model visualization.

### Method 2: Using Jupyter Notebooks

For interactive development and visualization:

```bash
jupyter notebook final.ipynb
```

Available notebooks:
- **final.ipynb** - Complete processing pipeline with explanations
- **3D_plot.ipynb** - Advanced 3D visualization techniques
- **Filtration.ipynb** - Image filtering and preprocessing
- **main_1.ipynb** - Alternative implementation approach

### Configuration

Key parameters in `main.py` (lines 48-54):

```python
LOCAL = 20              # Local region size for processing
RADIUS, SPREAD = 16, 4  # LoG filter parameters
RMIN, RMAX = 5, 80      # Minimum and maximum radii for artery detection
MASK_SIZE = 3           # Size of edge detection mask
THRESHOLD = 7           # Threshold for continuity checking
```

## Algorithm Overview

### Phase 1: Image Reading and Preprocessing
- Load ultrasound images (PNG, DICOM formats)
- Resize images to 256x256 for consistency

### Phase 2: Center Detection
- Apply Laplacian of Gaussian (LoG) template matching
- Find maximum correlation peaks to locate artery center
- `findMaxCorrelation()` - Detects center with highest correlation

### Phase 3: Boundary Detection
- Extract radial strip around detected center
- Apply edge detection using custom masks
- Detect edges along each radial direction (360 degrees)
- `getBoundary()` - Returns boundary coordinates

### Phase 4: Outlier Removal
- Remove discontinuous edge points using statistical analysis
- Interpolate missing or noisy edge points
- Verify location quality using boundary smoothness
- `removeOutliers()` - Cleans boundary data

### Phase 5: Boundary Refinement
- Smooth boundaries using Savitzky-Golay filtering
- Adjust discontinuous points in the boundary
- Ensure smooth transitions between frames

### Phase 6: 3D Reconstruction
- Stack boundaries from all frames (typically 300 slices)
- Convert cylindrical coordinates to Cartesian
- Create point cloud representation
- `show3dImage()` - Generates 3D visualization

### Phase 7: Visualization
- Generate interactive 3D surface plot using Plotly
- Overlay surface with point cloud scatter plot
- Export as interactive HTML visualization

## Data Information

### Supported Formats

- **PNG Images** - Primary input format for ultrasound sequences
- **DICOM (.dcm)** - Medical imaging standard (requires pydicom)
- **PLY Files** - 3D point cloud format for output
- **XYZ Files** - 3D coordinate format
- **HTML** - Interactive 3D visualization

### Dataset Structure

```
ARTERY_TRANSVERSAL/
├── Toshiba_test/         # Toshiba ultrasound system test data
├── Ultrasonix_test/      # Ultrasonix device test sequences
└── Ultrasonix_train/     # Ultrasonix device training data
```

### Image Naming Convention

Images are typically named as:
```
US0003_0000.png  (Frame 0)
US0003_0001.png  (Frame 1)
...
US0003_0299.png  (Frame 299)
```

## Output

### Generated Files

1. **3D Visualization HTML**
   - Interactive 3D surface plot
   - Filename format: `Ultrasound_DD_MM_YYYY_HH_MM.html`
   - Contains both surface and point cloud representations

2. **Point Cloud Data**
   - PLY format (Polygon File Format)
   - XYZ coordinate format
   - Can be visualized in 3D viewers

### Visualization Features

- **3D Surface Plot** - Reconstructed artery geometry
- **Point Cloud Overlay** - Detected boundary points color-coded by depth
- **Interactive Controls** - Rotate, zoom, and pan the model
- **Color Scaling** - Depth-based coloring for better visualization

## Key Functions

### Image Processing Functions

| Function | Purpose |
|----------|---------|
| `readImage()` | Load and preprocess ultrasound images |
| `templateMatching()` | Correlate image with LoG template |
| `LoGDisc()` | Generate Laplacian of Gaussian template |
| `findMaxCorrelation()` | Detect artery center location |
| `getStrip()` | Extract radial strip around center |

### Boundary Detection Functions

| Function | Purpose |
|----------|---------|
| `getBoundary()` | Extract complete boundary |
| `getEdgeMap()` | Generate edge detection map |
| `removeOutliers()` | Remove discontinuous points |
| `adjustBoundary()` | Fix boundary discontinuities |
| `smoothBoundary()` | Apply Savitzky-Golay filtering |

### 3D Reconstruction Functions

| Function | Purpose |
|----------|---------|
| `getCarotidPts()` | Extract carotid point set |
| `show3dImage()` | Main 3D reconstruction pipeline |
| `verifyLoc()` | Verify detection quality |

## Requirements

### Python Packages

- **opencv-python** (cv2) - Image processing
- **numpy** - Numerical computations
- **scipy** - Signal processing and filters
- **matplotlib** - 2D visualization and plotting
- **plotly** - Interactive 3D visualization
- **scikit-image** - Advanced image processing (optional)
- **pydicom** - DICOM medical image support (optional)

### System Requirements

- **RAM**: Minimum 4GB (8GB recommended)
- **Disk Space**: 500MB for dependencies + storage for datasets
- **Python**: 3.7 or higher

## Troubleshooting

### Common Issues

**Issue**: "ModuleNotFoundError: No module named 'cv2'"
```bash
Solution: pip install opencv-python
```

**Issue**: "No images found in path"
- Check the `path` variable in main.py points to correct directory
- Verify image naming convention matches code expectations
- Ensure images exist in the specified path

**Issue**: Poor artery detection
- Adjust `RADIUS` and `SPREAD` parameters for your ultrasound device
- Verify image quality and resolution
- Check image contrast and brightness

**Issue**: 3D model looks distorted
- Increase `OVERLAP` parameter for smoother boundaries
- Reduce outlier threshold if removing important points
- Verify image sequence continuity

## Performance Optimization

- **Batch Processing**: Process multiple patients in sequence
- **Parallel Processing**: Adapt code for multi-core processing
- **Memory Management**: Process large datasets in chunks

## Future Enhancements

- [ ] GPU acceleration for faster processing
- [ ] Deep learning-based center detection
- [ ] Automated parameter optimization
- [ ] Real-time visualization support
- [ ] Multi-vessel reconstruction
- [ ] Export to standard medical formats (NIFTI, NII)

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request with detailed description

## References

- Image processing techniques in medical imaging
- Laplacian of Gaussian filtering for blob detection
- Savitzky-Golay filtering for boundary smoothing
- 3D reconstruction from volumetric data

## License

This project is provided as-is for research and educational purposes.

## Contact & Support

For issues, questions, or suggestions, please create an issue in the project repository or contact the development team.

---

**Last Updated**: April 2026  
**Version**: 1.0  
**Status**: Active Development

