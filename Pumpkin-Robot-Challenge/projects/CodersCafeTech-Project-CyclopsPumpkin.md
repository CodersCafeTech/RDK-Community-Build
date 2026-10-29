# Cyclops Pumpkin Robot

![Cyclops Pumpkin Robot](https://raw.githubusercontent.com/CodersCafeTech/Cyclops-Pumpkin/main/assets/Spooky.jpg)

## Project overview

The Cyclops Pumpkin Robot is a 3D-printed pumpkin with a single moving eye on a round display. A USB camera and the RDK X5's BPU detect people in real time: someone far away gets a whispered "I see you...", and when a face comes up close the eye twitches, slams shut and opens again as a reptile eye while LEDs pulse with a ghost laugh and a relay sends mist drifting out of the pumpkin. Camera, AI, eye animation, sound, lights and relay all run on the RDK X5, fully offline.

## Links

- Repository: https://github.com/CodersCafeTech/Cyclops-Pumpkin
- Demo video: https://youtube.com/shorts/P1_9KhJj4cU
- Creator / team: CodersCafeTech
- Creator profile: https://github.com/CodersCafeTech
- Source license: [MIT](https://github.com/CodersCafeTech/Cyclops-Pumpkin/blob/main/LICENSE)

## Features

- **Two-stage reaction:** a whisper for someone in the distance, a full transformation for a face up close.
- **Real-time detection on the BPU:** bodies, heads and faces from one model pass, about 13 ms per frame.
- **A moving eye:** glances side to side, blinks, breathes its pupil and transforms on cue, drawn in code with NumPy and played at 20 fps on a 1.28" round display.
- **Light that follows the laugh:** five WS2812B LEDs driven from SPI1, pulsing with the laugh's loudness.
- **Smoke on cue:** a relay switches a humidifier on during the transformation, so mist drifts out of the pumpkin.
- **Fully offline:** nothing leaves the board.

## How RDK X5 is used

The RDK X5 is the only computer in the pumpkin and runs every part of it:

- **BPU (10 TOPS):** runs the TROS multitask body/head/face/hand model (`multitask_body_head_face_hand_kps_960x544.hbm`) through `hobot_dnn` from Python. Webcam frames are letterboxed to 960x544 and converted to NV12. Inference takes about 13 ms per frame. Bodies/heads trigger the whisper; a face at least 80 px tall (about 1.5 m) triggers the scare.
- **8x Cortex-A55 CPU:** three Python processes (spawned, not forked) run in parallel: eye rendering and display, camera + BPU detection, and LED animation. The CPU also pre-renders and caches 402 eye frames.
- **40-pin header:** SPI2 (`/dev/spidev2.0`, enabled with a small device-tree overlay) drives the GC9A01 display with a custom `spidev` + `Hobot.GPIO` driver. SPI1 MOSI generates WS2812B timing for the LEDs. GPIO handles display control and the active-low relay for the humidifier.
- **USB 3.0:** USB webcam at 640x480 via OpenCV.
- **Onboard 3.5 mm audio:** voice, laugh and growl through PulseAudio, with an ALSA fallback.
- **Ubuntu 22.04 + systemd:** starts the whole show on boot with `pumpkin.service`.

## System architecture

![System architecture](https://raw.githubusercontent.com/CodersCafeTech/Cyclops-Pumpkin/main/assets/architecture.svg)

The watcher process owns the camera and the BPU and answers two questions: is someone out there, and is a face close. The main process turns those answers into a show: it picks the eye phase (`normal`, `transform`, `scary`, `revert`), starts the sounds and switches the humidifier through the relay. The LED process keeps the strip in time with the laugh.

## Bill of materials / BOM

| Item | Quantity | Notes |
| --- | ---: | --- |
| RDK X5 | 1 | Runs camera, BPU detection, eye, sound, LEDs and relay |
| Waveshare 1.28" round LCD (GC9A01, 240x240) | 1 | The moving eye, on SPI2 |
| USB webcam (640x480 MJPG) | 1 | Watches for people |
| WS2812B LED strip | 5 LEDs | Yellow glow, on SPI1 MOSI |
| 5 V power supply | 1 | For the LEDs, GND shared with the board |
| 1-channel relay module (active-low) | 1 | Switches the humidifier |
| Humidifier / mist module | 1 | Smoke effect |
| Speaker with 3.5 mm plug | 1 | Voice and laugh |
| Jumper wires | ~17 | Keep SPI wires short |
| 3D-printed pumpkin | 1 | Face, back shells and lid |

## Build notes

- **Mechanical:** the pumpkin is printed in four parts. The display sits in a holder behind the eye hole with the webcam just below it, and the X5, relay and mist module sit inside the shell.
- **Electrical:** the display moved from SPI1 to SPI2 because SPI1's pads run at 1.8 V / 3 mA and caused glitches. The WS2812B signal on SPI1 uses a 4 MHz, 5-bit encoding to cope with the slow pad. Full wiring table is in the repository README.
- **Software:** a custom GC9A01 driver sends only changed regions and refreshes registers every frame. Eye phases are rendered with NumPy and cached as `.npy`. Detection runs in a spawned process to avoid GIL and BPU-runtime fork issues. The LED brightness follows the laugh's precomputed loudness curve.

The full build story, including every bug along the way, is in the [repository README](https://github.com/CodersCafeTech/Cyclops-Pumpkin#how-we-built-it).

## Setup and run

Requirements: RDK X5 with Ubuntu 22.04 and TROS (which provides `hobot_dnn` and the model), plus NumPy, OpenCV, spidev, Hobot.GPIO, ffmpeg and PulseAudio/ALSA. Enable SPI2 with the overlay in `overlay/` and add the two credited sound files as described in the README.

```bash
git clone https://github.com/CodersCafeTech/Cyclops-Pumpkin.git ~/Desktop/pumpkin
cd ~/Desktop/pumpkin
python3 src/pumpkin_eye.py --mode normal   # display test
python3 src/ws2812.py test                 # LED test
python3 src/relay.py pulse 3               # relay test
python3 src/pumpkin.py                     # wake the pumpkin
```

Full instructions: [Setup and Run](https://github.com/CodersCafeTech/Cyclops-Pumpkin#setup-and-run).

## Known limitations and safety

- Needs some light to see; a pitch-dark doorway leaves the pumpkin half-blind.
- The "close" threshold (`--near-face 80`) is tuned for our camera and must be retuned for others.
- The scary eye drops some frames, because SPI2 tops out around 14 full frames a second.
- The eye does not track faces yet.
- Movement comes from the animated eye and the relay-switched mist. There are no motors or pinch points.
- Power the LEDs from their own 5 V supply with a shared ground. Keep mains wiring enclosed and use a relay rated for the load. Point the mist away from the electronics.
- The laugh is loud up close; `--mute` is available.

## Future ideas

- An eye that follows the closest face, using the face positions the BPU already provides.
- A hidden MIPI CSI camera instead of the USB webcam.
- Different voice lines for repeat visitors or groups.

## License

The source code is released under the [MIT License](https://github.com/CodersCafeTech/Cyclops-Pumpkin/blob/main/LICENSE). The two sound effects keep their own licences and are not included in the repository.

---

I confirm that I have the right to submit this project profile and allow D-Robotics to use it, with creator attribution, for challenge review, showcase, and community promotion. The project source code remains governed by the license in the linked source repository.
