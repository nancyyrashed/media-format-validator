# Festival Video Format Compliance Checker — Automated Validation & Conversion with FFmpeg

A metadata-driven pipeline (no re-encoding guesswork) that inspects, validates, and — when needed — transcodes film submissions to meet a festival's required delivery format, using **ffprobe** for inspection and **ffmpeg** for standardized conversion. Built as a single exercise: define a format spec, check every submitted film against it, convert whatever fails, then re-validate the converted output and generate a full compliance report.

## Contents

| File | Description |
|---|---|
| `ex3.ipynb` | Task 3 — full pipeline: ffprobe/ffmpeg setup, festival spec definition, format checking, conversion, re-validation, and report generation. |
| `Exercise_3.pdf` | Write-up describing the installation/configuration of ffprobe/ffmpeg and the logic and structure of the notebook. |
| `format_report.txt` | Generated compliance report for the 5 submitted films (issues found, conversions performed). |
| `Cosmos_War_of_the_Planets.mp4`, `Last_man_on_earth_1964.mov`, `The_Gun_and_the_Pulpit.avi`, `The_Hill_Gang_Rides_Again.mp4`, `Voyage_to_the_Planet_of_Prehistoric_Women.mp4` | Original submitted film files used to test the system. |
| `Cosmos_War_of_the_Planets_formatOK.mp4`, `Last_man_on_earth_1964_formatOK.mp4`, `The_Gun_and_the_Pulpit_formatOK.mp4`, `The_Hill_Gang_Rides_Again_formatOK.mp4`, `Voyage_to_the_Planet_of_Prehistoric_Women_formatOK.mp4` | Converted, festival-compliant output files produced by the pipeline. |

## How It Works

The notebook follows a deterministic pipeline: **inspect → validate → convert (if necessary) → re-validate → report**. All requirements live in a single source of truth, `FESTIVAL_SPEC` (container: mp4, video codec: h264, audio codec: aac, frame rate: 25.0, resolution: 640x360, aspect ratio: 16:9, video bitrate: 2–5 Mb/s, audio bitrate: up to 256 kb/s, channels: stereo).

1. **Setup** — `shutil.which('ffmpeg')` and `shutil.which('ffprobe')` dynamically resolve the binaries at runtime, so the notebook doesn't hard-code paths and works across operating systems as long as FFmpeg is installed and on the system PATH.
2. **Inspection (`get_video_info`)** — runs ffprobe with `-print_format json` and targeted `-show_entries` to pull container name/bitrate plus per-stream fields (codec, width/height, frame rate, display aspect ratio, bitrate, channels). Output is parsed into a dictionary, with missing fields handled defensively and the first valid video/audio streams selected.
3. **Validation (`check_format`)** — compares probed fields against `FESTIVAL_SPEC`. Frame rate is computed from `r_frame_rate` by parsing rational values (e.g. `30000/1001`). Aspect ratio is read from `display_aspect_ratio` when present. Bitrates are parsed to integers (`parse_bitrate`) for consistent threshold comparisons. Any mismatch — wrong container, codec, frame rate, resolution, aspect ratio, or bitrate out of range — is collected into a human-readable list of errors (with warnings for borderline frame-rate deviations that could affect playback smoothness).
4. **Conversion (`convert_video`)** — builds a single ffmpeg command that normalizes only what's non-compliant: compliant streams are passed through with `-c:v copy` / `-c:a copy` to preserve quality and save processing time, while non-compliant ones are re-encoded to H.264/AAC with `-r 25`, resized to 640x360, bitrate-capped to the 2–5 Mb/s video / ≤256 kb/s audio range, and forced to stereo (`-ac 2`). The video bitrate target is chosen deterministically by clamping the source's nominal bitrate into the allowed range rather than picking an endpoint at random. Output files are saved with a `_formatOK.mp4` suffix.
5. **Re-validation (`check_converted_format`)** — re-runs the same checks against each `_formatOK` file to confirm the conversion actually brought it into compliance.
6. **Reporting (`generate_report`)** — iterates over all submitted files, writes a per-file section to `format_report.txt` (measured specs, issues found, conversion trigger and outcome), and closes with a summary of how many files passed vs. were converted.

## Results

| Film File | Issues Found | Conversion Required |
|---|---|---|
| `Cosmos_War_of_the_Planets.mp4` | Frame rate, resolution, aspect ratio, audio bitrate | Yes |
| `Last_man_on_earth_1964.mov` | Container, video codec, frame rate, video bitrate, audio codec, audio bitrate | Yes |
| `The_Gun_and_the_Pulpit.avi` | Container, video codec, resolution, aspect ratio, video bitrate, audio codec, audio bitrate | Yes |
| `The_Hill_Gang_Rides_Again.mp4` | Video bitrate | Yes |
| `Voyage_to_the_Planet_of_Prehistoric_Women.mp4` | Video codec, frame rate, video bitrate, audio codec, audio bitrate | Yes |

All 5 submitted films failed initial compliance and were converted; all 5 `_formatOK` outputs passed re-validation. See `format_report.txt` for the full per-file breakdown and `Exercise_3.pdf` for the write-up on setup and pipeline design.

## Usage

1. Install FFmpeg (provides both `ffmpeg` and `ffprobe`) and ensure it's available on your system PATH — Windows: download from ffmpeg.org and add the `bin` folder to PATH; Linux: `sudo apt install ffmpeg`; Mac: `brew install ffmpeg`.
2. Place the film files listed in `film_files` (inside the notebook) in the working directory, or update the list with your own file names/paths.
3. Open `ex3.ipynb` and run all cells. This will:
   - Print original specifications for each file.
   - Check each file against `FESTIVAL_SPEC` and convert any non-compliant file, saving it as `<name>_formatOK.mp4`.
   - Write a full report to `format_report.txt`.
   - Re-check every converted file and print pass/fail status for each.
4. Review `format_report.txt` for the detailed compliance report, and use the printed pass/fail summary at the end of the notebook to confirm all converted files meet spec.

## Skills Demonstrated

- Programmatic media inspection with ffprobe (JSON output parsing, stream/format field extraction)
- Rule-based format validation against a centralized specification
- Selective transcoding with ffmpeg (stream copy vs. re-encode, bitrate control, frame rate/resolution/aspect ratio correction, audio channel remapping)
- Defensive parsing of real-world metadata (missing fields, rational frame rates, string bitrates)
- Automated, cross-platform batch processing and report generation
