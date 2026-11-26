MetaSOP Live View

Real-time stress mapping with MSOP. The software grabs frames from the Basler camera, splits each frame into left and right views, computes disparity on the GPU, and displays disparity and stress maps at about 4 frames per second.

1. Requirements
A Windows PC with an OpenCL-capable GPU and its current graphics driver.
The Basler camera on USB. Close pylon Viewer before starting, because only one program can open the camera at a time.
Python 3.7 or later with these packages:
pip install numpy opencv-python matplotlib pyopencl pypylon

2. Folder layout
The code finds its data by searching the current working directory's path for a folder named StereoOpticalPalpation, so the folders must be arranged like this:
StereoOpticalPalpation\
├── SOP_CODE\SOP_Code_2026_Meta\        <- run from here \n
│   ├── sop_main_MetaLive.py               entry point for the live view
│   ├── sop_take_pic_Meta.py               captures single left/right images
│   ├── ximea\
│   └── sop\
│       ├── display\live_capture_Rhys_edit.py   camera, main loop, stress conversion, display
│       ├── logic\stereo_processor.py           GPU disparity calculation
│       └── logic\roi_selector.py               region-of-interest handling
└── SOP_Data\Probe_2026\                   results and data folder
    ├── Config.json                        camera settings (section 4.1)
    ├── LayerDispDist.txt                  stage readings from the disparity–displacement calibration
    └── Scans\
        ├── LayerAtWindow\disp_int.npy     reference disparity map (section 4.2)
        └── <date_time>\                   snapshots are saved here

3. Running the live view
Connect the camera, rest the stress-sensing layer against the window, and switch on the UV illumination.
Open a terminal in the code folder and start the program:
cd C:\...\StereoOpticalPalpation\SOP_CODE\SOP_Code_2025_Meta_C
python sop_main_MetaLive.py
A window opens. The top row shows the left and right views, with the analysed regions outlined in red. The bottom row shows the disparity map (pixels) and the stress map (kPa).
To save the current frame, click the Save image toolbar button (section 5 covers the Record button). Files are written to SOP_Data\Probe_2026\Scans\<start date_time>\.
Close the window to stop the program.

4. Settings
Options in sop_main_MetaLive.py:
--probe_id	2026	Selects the probe folder SOP_Data\Probe_<id>
--layer_thickness	Undeformed thickness of the stress-sensing layer (mm)
--speckle_height	Height of the microparticle plane above the bottom of the layer (mm)
--window_size	15	Correlation window size (pixels)
--sub_spacing	1	Step between correlation windows (pixels)

4.1 Probe folder: SOP_Data\Probe_2026\Config.json
Key	Current value	What it does
ROI	top 50, bottom 250, left 75, right 275	Analysed region in the left view, in pixels of the downsampled 413 × 325 view
OFFSET_VALUES	[8, 60]	Shift [vertical, horizontal] of the right-view region relative to the left. Re-measure this if the probe is rebuilt or refocused
SEARCH_RANGE	[[-5, 5], [-20, 20]]	Disparity search range around the offset [vertical, horizontal]. A wider range is slower
DISPLACEMENT_MODEL	[0.3, -0.05]	Linear disparity–displacement calibration: displacement (mm) = slope × disparity + intercept
RESOLUTION	[826, 325]	Size of the downsampled frame. Must equal the camera's Width/4 and Height/4 (section 4.3)

CONNECTION_DETAILS.CONNECTION_TYPE must stay at 5 (Basler).

4.2 Reference disparity map: Scans\LayerAtWindow\disp_int.npy
The program subtracts this map from every frame so that the displayed disparity shows only load-induced change. Its path is hard-coded in live_capture_Rhys_edit.py at line 360 as C:\Users\SOP Software\Documents\StereoOpticalPalpation\SOP_DATA\..., so edit that line on any other computer.
Rebuild the map whenever the ROI, offsets, window size or focus change. It must have the same size as the disparity map, which is 185 × 185 for the default ROI and window.

One way to rebuild it:
Temporarily change line 360 to DispAtWindow = 0.
Run the live view with the layer resting unloaded against the window, then click Save image.
Load the saved disp_map_image_1.npy, round it, save it as disp_int.npy in Scans\LayerAtWindow\, and restore line 360.

4.3 Code: sop\display\live_capture_Rhys_edit.py
Setting	Line	Current value	Notes
Exposure	154	100000.0 µs (100 ms)	The largest share of the frame time. Shorter exposures run faster but give a dimmer image
Gain	157	5.0 dB	
Gamma	159	1.5	
Sensor crop	166–169	Width 3304, Height 1300, OffsetX 250, OffsetY 290	Positions the crop so both channel images are captured. Width must be divisible by 8. If you change it, update RESOLUTION in Config.json
Strain-to-stress fit	251–255	A0 = 0, AA = −2.79709, BB = −3.52231, CC = 2.8478, DD = 1.98350	stress = A0 + AA·e^(BB·strain) + CC·e^(DD·strain), in kPa. Update this if the layer material changes
Colour scales	673–674	Disparity 0–10, stress 0–15 kPa	The values at lines 509–510 are overwritten on the first frame, so edit these lines instead
Disparity display	679	14-disp_map	This offset applies to the display only. Stress is calculated from disp_map itself

4.4 GPU selection: sop\logic\stereo_processor.py
Line 276 selects the OpenCL platform with cl.get_platforms()[0]. If the program picks the wrong GPU or reports that no GPU device was found, change the index to [1]. To list the platforms available:
python -c "import pyopencl as cl; print(cl.get_platforms())"
