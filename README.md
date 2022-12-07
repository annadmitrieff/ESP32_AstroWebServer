# CameraWebServerRecorder Astro Edition

https://github.com/paoloinverse/CameraWebServerRecordeR_AstroEdition



This is a modified version of James Zahary's awesome work at https://github.com/jameszah/CameraWebServerRecorder , I added a number of functions to turn his version of the  espressif ESP32-CAM CameraWebServer example into a functional Astrophotography / Timelapse / Dash camera device. 

First of all, a filemanager is available on tcp port 8080. 
If using the AP mode, just connect to http://192.168.4.1:8080 and the file manager will load. 
You Will be able to download, upload and delete anything on the onboad SD card. 
This will also help with uploading the secret.txt file with the optional wifi credentials. 
For the file manager I integrated this library from James: https://github.com/jameszah/ESPxWebFlMgr/tree/master/esp32_sd_file_manager
it's really well done.

Also, I modified the framebuffer capture functions to override the normal timeout limit and request frames with exposure times thousands of times longer than normal.

The second, and most important function I added, is the low level access to a few of the OV2640 configuration registers
that enable this sensor to perform very long exposures, way beyond is intended usage. 
There are 4 options: 
- Frame time: this is the most significant byte of the frame lenght configuration register, its main effect is to stabilize the auto-exposure loop within the sensor AND to limit the maximum exposure time. Unless required to stabilize the luminance under low light, this is best left untouched and set to zero at all times. 
- Add VSYNC lines: this is the most significant byte of the ADDVSYNC register (ADDVSH), it extends the frame duration by adding N * 256 lines before the VSYNC signal is transmitted. This extends the exposure time enormously, especially when the XCLOCK is set to 1 MHz. For daytime usage, leave it to zero. For low light photography you may raise it carefully. For full night time astrophotography you may set it to 255. 
- Target luminance high limit: this sets the luminance high limit for the auto exposure loop within the sensor. Raise it in case the autoexposure loop gets unstable in long exposures and low light. 
- Target luminance low limit: same as above, you may lower it in case of instability. This makes more sense in case of unattended timelapse cameras or dash cams in low light, however the best fix still remains raising the frame time. 

Tips for normal usage under low light: 
set XCLOCK to 5 or 10 MHz, if it's still not enough then raise the AddVSYNC register, then tweak Frametime to control any auto exposure instability

Tips for extremely low light and astro photography: 
- Set the frame size to HD  1280 x 720 or higher. Anything lower is not going to work this well.
- Set the picture quality to 1 (best!): you want to have no quantization nor compression artifacts.
- Disable the AWB, AEC, AEC DSP, then set Exposure to 1200. Maximize the exposure before you even touch the ADD VSYNC register.
- Disable AGC and reset the gain to 1. Anything over gain 1 will just introduce unneeded additional noise and amplify the dark current.
- Disable BPC and WPC (white pixel correction). WPC will work well under low light, but will fail under astrophotography and just introduce artifacts.
- disable LENC (Lens Correction). The lens correction digital filter MUST never be used under very long exposures as it worsens the dark current effect considerably. You want a background as flat and black as possible!
- Set the ADD VSYNC directly to 255
- Set XCLOCK to 1

Next, cover the objective lens, go to the default capture page ( http://192.168.4.1/capture in Access Point mode) and capture at least 3 dark frames, save the last one. You will need it to post process the astro pictures and subctract the hot pixels.

- I recommend cooling the sensor, if possible. The difference in dark current from 20C ambient temperature to 5C is DRAMATIC. 

See these sample pictures taken with a wide angle objective (XCLOCK 1MHz, ADDVSYNC 255 (maxed), AEC fully disabled and exposure set to 1200, AGC disabled and analog gain set to 1, prescaler left untouched, with these parameters the exposure time is in the order of one minute): 

20 Celsius ambient temperature dark frame, lens correction enabled:
![image](https://github.com/paoloinverse/CameraWebServerRecordeR_AstroEdition/blob/main/dark_frame_VSHMSB255_EXP1200_GAIN1_1Meg_LC_20deg.jpg)

20 Celsius ambient temperature dark frame, NO lens correction:
![image](https://github.com/paoloinverse/CameraWebServerRecordeR_AstroEdition/blob/main/dark_frame_VSHMSB255_EXP1200_GAIN1_1Meg_noLC_20deg.jpg)

5 Celsius ambient temperature dark frame, NO lens correction:
![image](https://github.com/paoloinverse/CameraWebServerRecordeR_AstroEdition/blob/main/dark_frame_VSHMSB255_EXP1200_GAIN1_1Meg_noLC_5deg.jpg)

3 Celsius ambient temperature dark frame, NO lens correction:
![image](https://github.com/paoloinverse/CameraWebServerRecordeR_AstroEdition/blob/main/dark_frame_VSHMSB255_EXP1200_GAIN1_1Meg_noLC_3deg.jpg)

3C ambient temperature, sample shot with a wide angle lens. The red light is from a tiny red led from the portable battery bank. The scene was shot on a 3/4 moon lit area during night time. It looks like day. Please don't mind the objective, it's a tiny lens of poor quality with the infrared filter removed.
See why it is important to save the dark frame and subtract it afterwards? Hot pixels can no longer be kept under control by the camera internal DSP, so it's best to subtact them externally during the post processing.
![image](https://github.com/paoloinverse/CameraWebServerRecordeR_AstroEdition/blob/main/sample_frame_VSHMSB255_EXP1200_GAIN1_1Meg_noLC_3deg.jpg)

Be aware that the OV2640 camera is not running under its normally intended limits, I find it surprising it can still work at such long exposures while keeping the noise and dark current at reasonable levels. 

#ROADMAP
- add a function to save the dark frame in RAM 
- add a second capture page that on load opens a canvas with the captured frame, load the dark frame and subtracts it automatically. The canvas content can be saved manually. 


# ORIGINAL README follows

Enhancement of @espressif CameraWebServer to add avi video recording to an SD Card 


CameraWebServerRecorder  
 
  https://github.com/jameszah/CameraWebServerRecorder  
  
  
  This is a modified version of the espressif ESP32-CAM CameraWebServer example for Arduino, which lets you fiddle
  with all the parameters on the ov2640 camera providing a web browser interface to make the changes, and view the stream,
  ot stills jpeg captures.  
    
  The modification is to add a video recorder facilty.  You can set the frame rate to record, the "speedup" to you can play a
  timelapse at 30 fps on your computer, and a avi segment length, so you end the movie from time to time, so you won't have a  2 GB
  avi file to edit later.  You can break it into chunks, each with its own index.  
  
  My original idea was add all the parameters to control the camera to https://github.com/jameszah/ESP32-CAM-Video-Recorder-junior,
  but I left this as primarily the espressif demo program, and added a simple sd recording facility.  No semaphores, and multitask
  communications, other than setting a global variable to start stop the recording task, using all the parameters you have already
  set in the espressif streaming/snapshot control panel.  
    
  If you put a file called "secret.txt" on your SD, with 2 lines giving your SSID name, and SSID password, then it will
  join your router and you can access from your phone or computer.  
  
  If there is no file, it will start in AP mode and you can connect your phone or computer the "ESP32-CAM", password "123456789",
  to access all the functionality.  It will create a secret.txt for you with the SSID name "AP ESP32-CAM".  You can edit that
  later to a different AP name a password.  
  
  This AP funtionality is useful it you are setting up a camera away from a router, and you can connect with your phone, choose
  your parameters, start the recording, and walk away.  When you return the SD will be full of great video!  
  
  The avi recordings seem quite amenable to haveing parameters changed during the recording, including framesize!  
  
  It turns off streaming when you start recording, but you can turn it back on, with some loss of sd recording speed.
  If you are timelapse recording like 1 fps, it doesn't really matter.  (The junior program below, can record and stream 2 channels without 
  reducing recording rate.)   
  
  The filenames of the recordings are just the date and time (2022-08-14_11.45.49.avi).  Date and time in AP mode is acheived by automatically 
  sending unix time when you click the Start Record button since your phone/computer should know the time.  Timezones have not been implemented.
  
  If you have no SD card installed, it will show 0 freespace on the SD, and continue with other functionality.  
  When the SD runs out of diskspce, recording will stop - no deleting old stuff implemented.  (my other programs have it)  
    
  You can also control this CamWebServer with simple urls, if you want to configure and start it with motioneye (or
  equivalent) as follows,  
  
  http://192.168.1.67/control?var=brightness&val=2  
  http://192.168.1.67/control?var=contrast&val=-2  
  http://192.168.1.67/control?var=saturation&val=2  
  http://192.168.1.67/control?var=special_effect&val=1  
  http://192.168.1.67/control?var=awb&val=1  
  http://192.168.1.67/control?var=awb_gain&val=1  
  http://192.168.1.67/control?var=ae_level&val=-1  
  http://192.168.1.67/control?var=agc&val=0  
  http://192.168.1.67/control?var=gainceiling&val=1  
  http://192.168.1.67/control?var=ae_level&val=-2  
  http://192.168.1.67/control?var=agc&val=1  
  http://192.168.1.67/control?var=bpc&val=0  
  http://192.168.1.67/control?var=raw_gma&val=0  
  http://192.168.1.67/control?var=lenc&val=0  
  http://192.168.1.67/control?var=hmirror&val=1  
  http://192.168.1.67/control?var=dcw&val=0  
  http://192.168.1.67/control?var=vflip&val=1  
  http://192.168.1.67/control?var=hmirror&val=0  
  http://192.168.1.67/control?var=dcw&val=1  
  http://192.168.1.67/control?var=colorbar&val=1  
  http://192.168.1.67/status   
  http://192.168.1.67/capture   
  http://192.168.1.67:81/stream - streaming using different port 81, so the web still responds on port 80  
    
  ... and CamWebServerRecorder adds these:  
  
  http://192.168.1.67/control?var=interval&val=1000      -- 1000 milliseconds between frames  
  http://192.168.1.67/control?var=seglen&val=1800        -- avi file is closed and new file started every 1800 seconds  
  http://192.168.1.67/control?var=speedup&val=30         -- play 1 fps recording at 30fps when you play the avi  
  http://192.168.1.67/startrecord  
  http://192.168.1.67/stoprecord   
    
    
  The avi.cpp part of this code is based on heavily modified and simplfied subset of:  
  https://github.com/jameszah/ESP32-CAM-Video-Recorder-junior  
  which is a heavily modifed and simplfied rework of:  
  https://github.com/jameszah/ESP32-CAM-Video-Recorder  
    
  Those two programs have various ways to control the recording, and view live streams, and download the video to you computer
  or phone, or upload snapshots and video to Telegram, etc.  
    
  The changes to the original files:  
  
  CamWebServerRecorder.ino - mount the SD card, the AP stuff, start the recorder task  
  
  app_httpd.cpp - the esp32 side code to handle the new paramters  
  
  camera_index.h - this has the gziped html and javascript from espressif expanded (and easily editable), and includes all the changes to the web display for new parameters and controls  
  
  avi.cpp - all the origninal simplfied code to record a mjpeg avi  
  
  camer_pins.h - no changes  
    
  Work-In-Progress .... but other things to do.  
  - only works for ov2640 camera  
  - webpage does report SD freespace, but no info on recordings, including whether it is recording (if you reload the page)  
  - One-Click-Installer not done -- maybe tomorrow  
  - didn't put in mdns - so you have to figure out your ip from serial monitor, or port authority (it should respond on 80 and 81, which would give you a clue, but has a defualt name like esp32-F41450)   
      
  James Zahary  
  Aug 14, 2022  
  Version 55.10 - in my mysterious numbering scheme  
  Free coffee https://ko-fi.com/jameszah/  
  jamzah.plc@gmail.com  
    
  jameszah/CameraWebServerRecorder is licensed under the GNU General Public License v3.0  
    
Arduino 1.8.19  
Arduino-ESP32 2.0.4  
Board: AI-Thinker ESP32-CAM  
Huge APP  

![image](https://user-images.githubusercontent.com/36938190/184581874-a0a66c24-0a92-4854-9117-76b3b94cfffc.png)


Here are the new controls. 
From the top:
1.  How much freespace on your SD card
2.  Debug parameters show   
2.1  framesize    
2.2  quality  
2.3  interval (milliseconds per frame)  
2.4  speedup (muliply record framerate for playing)   
2.5  avi segment length in seconds  
3.  set avi segment length  
4.  set frames per second to record (shows as "seconds per frame" for slow recording)  
5.  set speedup playback over record rate - this will adjust to playback all videos at 30 fps when you change framerate, but you can readjust it)  
6.  Start and Stop recording  
