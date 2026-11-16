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

Global options go before the subcommand:

| Option | Default | Meaning |
|---|---|---|
| `--camera-id N` | `0` | uEye camera ID. `0` means the first available camera. |
| `--mode` | `mono8` | One of `mono8`, `mono10`, `mono12`, `mono16` |
| `-e`, `--exposure-ms` | camera's current value | Exposure in milliseconds (`1000` = 1 s) |
| `-g`, `--gain` | camera's current value | Hardware master gain, 0–100 |

Subcommands:

- **`info`**: prints the model, serial number, sensor size, pixel size, bit depth, and current exposure and gain.
- **`snap -o PATH [-n N] [--imagetyp TYPE] [--cube]`**: captures `N` frames (default 1) and writes them to FITS. With one frame it writes to `PATH`. With more than one it writes `PATH_frame00.fits`, `PATH_frame01.fits`, and so on, or a single 3D cube if you pass `--cube`. `--imagetyp` sets the `IMAGETYP` header (default `LIGHT`). Mean, std, min, max and saturated-pixel count are printed for each frame.
- **`stream [-n N]`**: runs live video and prints the frame rate and stats for each frame. It stops after `N` frames, or when you press Ctrl+C.

## API reference

Everything below can be imported directly from `ueye_cam`.

### `UEyeCamera`

```python
UEyeCamera(camera_id=0, color_mode="mono8", exposure_ms=None, gain=None)
```

This is the main camera object. Constructing it does not touch the hardware. The camera is opened by `open()` or by entering a `with` block.

| Argument | Meaning |
|---|---|
| `camera_id` | uEye camera ID. `0` opens the first available camera. |
| `color_mode` | A key of `COLOR_MODES`: `"mono8"`, `"mono10"`, `"mono12"` or `"mono16"`. Any other value raises `ValueError`. |
| `exposure_ms` | Exposure to apply when the camera opens. `None` keeps whatever the camera is currently set to. |
| `gain` | Hardware master gain (0–100) to apply when the camera opens. `None` keeps the current value. |

#### Lifecycle

- **`open() -> UEyeCamera`**: initializes the camera, sets the color mode, turns off auto shutter and auto gain, reads the sensor size and allocates the capture buffer. It then applies the requested exposure and gain, or reads back the current values if none were given, and fills in `info`. If any step fails, the camera is closed again before the error is raised. Calling `open()` on a camera that's already open does nothing. Returns `self`.
- **`close()`**: stops any running stream, stops live video, frees the image buffers and releases the camera. It's safe to call more than once.
- **`with UEyeCamera(...) as cam:`**: calls `open()` on entry and `close()` on exit, including when an exception is raised. This is the recommended way to use the class.

#### Attributes and properties

- **`info: CameraInfo`**: model, serial number, sensor size and pixel size. It is `None` until the camera is opened.
- **`bit_depth: int`**: significant bits per pixel for the chosen color mode (8, 10, 12 or 16).
- **`exposure_ms: float`**: the exposure the driver actually applied, in milliseconds. Assigning to it calls `set_exposure_ms()`.
- **`gain: int`**: the hardware master gain that was actually applied. Assigning to it calls `set_gain()`.
- **`h_cam`**: the raw `pyueye` camera handle, for driver calls this wrapper doesn't cover. Raises `CameraError` if the camera isn't open.

#### Settings

- **`set_exposure_ms(exposure_ms) -> float`**: sets the exposure and returns the value the driver actually applied. The camera's maximum exposure is limited by the frame time, so this lowers the frame rate first. Without that step, the driver silently cuts long exposures down to a few milliseconds. If the applied value differs from the request by more than 1%, an `ExposureClampedWarning` is issued.
- **`set_gain(gain) -> int`**: sets the hardware master gain (0–100) and returns the value that was applied.

Changing exposure or gain marks the next `snap()` or `burst()` to throw away one frame first, because the frame already being exposed used the old settings.

#### Capture

- **`snap() -> Frame`**: captures a single frame. It's equivalent to `burst(1)[0]`.
- **`burst(n_frames) -> list[Frame]`**: captures `n_frames` frames one after another in single-frame (triggered) mode. Use it for dark, flat and bias series. Frame `index` values run from `0` to `n_frames - 1`. Raises `CameraError` if a stream is running.
- **`stream(n_frames=None, n_buffers=4, timeout_ms=None) -> FrameStream`**: starts continuous live capture through a ring of `n_buffers` image buffers. It stops after `n_frames` frames if given, otherwise it runs until closed. `timeout_ms` is how long to wait for each frame. The default is `3 × exposure + 1000` ms. Frames arrive at roughly one per exposure time. If your code reads them more slowly than that, buffers are reused and frames are dropped instead of queued. Only one stream can run at a time.

### `FrameStream`

This is the live-video iterator returned by `UEyeCamera.stream()`. Always use it as a context manager, so the camera goes back to single-frame mode afterwards:

```python
with cam.stream(n_frames=100) as frames:
    for frame in frames:
        ...
```

- **Iteration (`for frame in frames`)**: waits for each new frame and yields it as a `Frame`. Frame indexes start at `0` for each stream. Iteration ends after `n_frames` frames, or once the stream has been closed. Raises `CameraTimeout` if no frame arrives within `timeout_ms`.
- **`close()`**: stops live video, frees the stream's buffers and switches back to the single-frame buffer, so `snap()` and `burst()` work again. It's called automatically when the `with` block exits, when iteration ends, and by `UEyeCamera.close()`.

### `Frame`

A frozen dataclass holding one captured image and its metadata.

| Field | Type | Meaning |
|---|---|---|
| `data` | `np.ndarray` | Image of shape `(height, width)`. The dtype is `uint8` for `mono8` and `uint16` for the higher bit depths. |
| `exposure_ms` | `float` | Exposure the driver applied for this frame |
| `gain` | `int` | Hardware master gain (0–100) |
| `bit_depth` | `int` | Significant bits per pixel |
| `timestamp` | `datetime` | UTC time from the host clock when the frame was read out |
| `index` | `int` | Position within the `snap`, `burst` or `stream` call that produced it |
| `camera` | `CameraInfo` | The camera that took the frame |

- **`saturation_value -> int`** (property): the largest possible pixel value at this bit depth, `2**bit_depth - 1`.
- **`n_saturated -> int`** (property): the number of pixels at or above `saturation_value`.
- **`save_fits(path, imagetyp="LIGHT", extra=None)`**: writes this frame to a FITS file. It's shorthand for `save_fits(path, frame, ...)`.

### `CameraInfo`

A frozen dataclass describing the open camera: `model` (sensor name), `serial`, `width` and `height` in pixels, and `pixel_size_um` (pixel width in microns).

### FITS functions

- **`frame_header(frame, imagetyp="LIGHT", extra=None) -> astropy.io.fits.Header`**: builds the standard header for a frame. Keys in `extra` are added on top or override the defaults. The header contains:

  | Key | Value |
  |---|---|
  | `EXPTIME` | Exposure in seconds |
  | `DATE-OBS` | UTC timestamp from the host clock, ISO 8601 |
  | `IMAGETYP` | `LIGHT`, `DARK`, `FLAT`, `BIAS`, etc. |
  | `GAIN` | Hardware master gain (0–100) |
  | `BITDEPTH` | Significant bits per pixel |
  | `FRAMENUM` | Frame index within the capture |
  | `CAMERA`, `SERIALNO` | Sensor model and serial number |
  | `XPIXSZ`, `YPIXSZ` | Pixel size in microns |

- **`save_fits(path, frame, imagetyp="LIGHT", extra=None, overwrite=True)`**: writes one frame as a 2D image in its original dtype (`uint8` or `uint16`) with the header above.
- **`save_cube(path, frames, imagetyp="LIGHT", extra=None, overwrite=True)`**: writes a sequence of frames as a single 3D cube with shape `(frame, y, x)`. The header comes from the first frame, with `FRAMENUM` removed and two keys added: `NFRAMES` (the number of frames) and `DATE-END` (the last frame's timestamp). Raises `ValueError` if `frames` is empty.

### Constants

- **`COLOR_MODES`**: a dict mapping each supported mode name to `(uEye color mode, buffer bits per pixel, significant bits)`. Supported names are `mono8`, `mono10`, `mono12` and `mono16`. The 10, 12 and 16-bit modes are all stored in 16-bit buffers.

### Exceptions and warnings

- **`CameraError(RuntimeError)`**: raised when a uEye driver call fails, or when the camera is used in the wrong state (not open, or a stream is already running). `.ret` holds the uEye return code when there is one.
- **`CameraTimeout(CameraError)`**: raised by `FrameStream` when no frame arrives within the timeout.
- **`ExposureClampedWarning(UserWarning)`**: issued by `set_exposure_ms()` when the applied exposure differs from the requested value by more than 1%.

## License

MIT. See [LICENSE](LICENSE).
