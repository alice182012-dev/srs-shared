# 3Di Local Setup Guide (Windows + VS Code)

This guide walks through:
- Installing Miniconda
- Creating a conda environment
- Installing dependencies (GDAL, h5py, 3Di libraries)
- Connecting to VS Code
- Verifying everything works

---

## 1. Install Miniconda

Download Miniconda from the [official website]([(https://www.anaconda.com/docs/getting-started/miniconda/main)]) and run the installer.

During installation:
- Select "Just Me" (recommended)
- Choose a simple install path (e.g., `C:\Users\<your_user>\miniconda3`)
- Check **"Add Miniconda to PATH"** if available

Finish installation.

---

## 2. Restart Terminal

After installation:
- Close VS Code
- Reopen VS Code
- Open a new terminal

---

## 3. Verify Conda Installation

Run:

```bash
conda --version
```

Expected output:

```
conda x.x.x
```

If this fails:
- Restart your computer
- Or reinstall Miniconda with PATH enabled

---

## 4. Create Conda Environment

Create a new environment named `3di`:

```bash
conda create -n 3di python=3.10
```

Activate it:

```bash
conda activate 3di
```

---

## 5. Install Core Dependencies

Install GDAL and HDF5 stack:

```bash
conda install gdal h5py -y
```

**Important:**
- Always install `gdal` via conda (not pip)
- This avoids DLL and compatibility issues

---

## 6. Install 3Di Python Libraries

```bash
pip install threedi-api-client threedidepth threedigrid
```

---

## 7. Verify Installation

Run each command:

```bash
python -c "import threedi_api_client; print('API OK')"
python -c "import threedidepth; print('Depth OK')"
python -c "import threedigrid; print('Grid OK')"
python -c "import h5py; print('H5PY OK')"
```

Expected output:

```
API OK
Depth OK
Grid OK
H5PY OK
```

---

## 8. Verify GDAL

```bash
python -c "from osgeo import gdal; print(gdal.__version__)"
```

If this prints a version number, GDAL is working correctly.

---

## 9. Connect Environment to VS Code

Open Command Palette:

```
Ctrl + Shift + P
```

Select:

```
Python: Select Interpreter
```

Choose:

```
Python (conda env: 3di)
```

---

## 10. Confirm Correct Environment in VS Code

Add this to a Python file:

```python
import sys
print(sys.executable)
```

Expected output path:

```
.../conda/envs/3di/python.exe
```

---

## 11. Recommended Project Structure

```
D:\3Di\
│
├── run_simulation.py
├── postprocess_depth.py
├── Results\
└── latest.txt
```

---

## 12. Run Depth Processing Script

Activate environment:

```bash
conda activate 3di
```

Run:

```bash
python postprocess_depth.py
```

---

## 13. Common Issues

### Conda not recognized
- Restart terminal or machine
- Ensure Miniconda is added to PATH

### ModuleNotFoundError
- Wrong environment selected
- Re-run `conda activate 3di`

### GDAL / h5py errors
- Installed via pip instead of conda
- Fix by reinstalling with conda:

```bash
conda install gdal h5py
```

### DLL load errors
- Mixing QGIS Python and conda
- Keep environments separate

### Path issues
Avoid non-ASCII paths like:

```
D:\Bản đồ ngập lụt\
```

Use:

```
D:\3Di\
```

---

## 14. Result

You now have a working environment with:
- GDAL
- h5py
- threedi_api_client
- threedidepth
- threedigrid

This setup is stable for running depth calculations and post-processing workflows.
