# Borehole Monitoring System Specification (Final)

## 1. Purpose
This system provides an integrated borehole monitoring dashboard on Raspberry Pi 5, including:
- Dual wide-angle camera preview and still capture
- Video recording with metadata
- PWM-controlled LED lighting (2 channels)
- Magnetic heading measurement
- Single-point LiDAR distance measurement
- Ambient and internal temperature / humidity monitoring
- Dew point calculation and condensation risk warning

## 2. Data Storage
All runtime-generated data are stored under the data directory:
- Still images: data/cam/
- Videos: data/video/
- Sensor CSV logs: data/csv/

## 3. Orientation Handling
### Still Images
- Camera direction is stored in EXIF (GPSImgDirection, True North).

### Videos
- Camera direction is NOT stored in the video file.
- Direction and related metadata are stored in meta.json.

### Direction Definition
direction_deg_true = wrap360(
  heading_deg_mag
  + magnetic_declination_deg
  + camera_offset_deg[cam_id]
)

## 4. Condensation Risk
Condensation risk is evaluated from ambient dew point and internal temperature.
Thresholds are configurable via YAML.

## 5. UI
- Color-coded ambient/internal temperature
- Condensation warning with legend (ⓘ button)
- Configurable thresholds
