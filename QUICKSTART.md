# Quick Start Guide

## 5-Minute Setup

### 1. Install Dependencies
```bash
pip install -r requirements.txt
```

### 2. Run the Main Script
```bash
python main.py
```

This will process the ultrasound images and generate an interactive 3D model.

### 3. View Results
- Check the console output for progress updates
- An HTML file will be generated with timestamp: `Ultrasound_DD_MM_YYYY_HH_MM.html`
- Open the HTML file in your web browser for interactive 3D visualization

---

## For Jupyter Users

### Open an Interactive Notebook
```bash
jupyter notebook final.ipynb
```

Then run the cells sequentially to see:
- Image processing steps
- Boundary detection
- 3D reconstruction
- Visualization

---

## Using Your Own Data

### Step 1: Prepare Images
Place your ultrasound image sequences in a directory:
```
your_data_folder/
├── US0001.png
├── US0002.png
├── US0003.png
└── ...
```

### Step 2: Update Image Path
Edit `main.py` and change line 58:
```python
path = "your_data_folder/"
```

### Step 3: Run Processing
```bash
python main.py
```

---

## Adjust Parameters for Your Data

Edit these parameters in `main.py` (lines 48-54):

```python
RADIUS = 16      # Try 10-20 for different artery sizes
SPREAD = 4       # Try 2-6 for template matching sensitivity
RMIN = 5         # Minimum radius to search
RMAX = 80        # Maximum radius to search
MASK_SIZE = 3    # Edge detection kernel size
```

**Tip**: Start with default values, then adjust based on your ultrasound device and image quality.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| "Path not found" | Update the `path` variable to your data directory |
| Poor 3D model quality | Adjust RADIUS and SPREAD parameters |
| Images not processing | Ensure images are PNG format and named correctly |
| Memory issues with large datasets | Process fewer images at a time |

---

## Next Steps

1. Read the full [README.md](README.md) for detailed documentation
2. Explore the Jupyter notebooks for interactive learning
3. Review the algorithm overview in README.md for technical details
4. Check function documentation in the code comments

---

## File Input Requirements

- **Format**: PNG images (or DICOM with pydicom installed)
- **Resolution**: Recommended 256x256 or will be resized
- **Sequence**: Consecutive frames in numeric order
- **Quality**: Clear ultrasound images with good contrast

---

**Enjoy your 3D reconstruction!** 🎉

