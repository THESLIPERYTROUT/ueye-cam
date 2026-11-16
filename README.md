# ueye-cam

A small Python wrapper around IDS uEye cameras. All the `pyueye`/`ctypes` handling is hidden behind a context-managed `UEyeCamera` class that returns numpy frames with the metadata needed to write calibrated FITS files.

- Single-frame snaps, back-to-back bursts and continuous ring-buffer streaming
- Mono 8/10/12/16-bit modes
- Auto shutter and auto gain are always off. After an exposure or gain change, one stale frame is thrown away.
- FITS output with `EXPTIME`, `DATE-OBS`, `IMAGETYP`, `GAIN`, camera model/serial and pixel size headers, as single images or 3D cubes
- A command-line tool for quick captures

## Requirements

- The IDS uEye driver / IDS Software Suite, installed so `pyueye` can find the uEye API DLL
- Python 3.9+, `numpy`, `pyueye`, `astropy`

## Install

```bash
pip install -e .
```

## Library usage

```python
from ueye_cam import UEyeCamera, save_cube

with UEyeCamera(exposure_ms=0.5, color_mode="mono12") as cam:
    print(cam.info)

    frame = cam.snap()
    frame.save_fits("light.fits")

    darks = cam.burst(20)
    save_cube("dark_cube.fits", darks, imagetyp="DARK")

    with cam.stream(n_frames=100) as frames:
        for f in frames:
            print(f.index, f.data.mean(), f.n_saturated)
```

## CLI

```bash
ueye-cam info
ueye-cam snap -e 1.0 -o light.fits
ueye-cam snap -e 0.5 -n 20 -o flats/flat.fits --imagetyp FLAT
ueye-cam snap -e 0.5 -n 20 -o flat_cube.fits --cube
ueye-cam stream -e 10 -n 100
```

`python -m ueye_cam ...` works the same way.

## License

MIT. See [LICENSE](LICENSE).
