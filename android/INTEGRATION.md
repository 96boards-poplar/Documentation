# Poplar Android Integration

This page cover all the things related with the integration of Aspen project, mostly focus on kernel and user space HAL. It will over test cases, known issues, how to report issue, capture logs, etc.

It's not our intention here to provide a full set of test cases to validate a android product. Rather the goal here is provide minimal test cases and steps that can be used to 1) validate the basic kernel interfaces, when we doubt something might be wrong with the kernel; 2) validate the basic features from user's point of view such as play a youtube video.

We will keep it simple but useful.

## Smoke Test

Following test is required to run for any patch submission. 

TBD:

## Test Video/Video

### 1.1 Unit Test audio

1. (host)
Download any .wav file, call it `test_audio.wav`.
```
adb push test_audio.wav /sdcard/test.wav
```

2. on poplar console, after board booting up
```
# su
# tinyplay /sdcard/test.wav
Playing sample: 2 ch, 44100 hz, 16 bit
```

What to Expect:

You should hear the audio in both HDMI and audio line out interface.

### 1.2 Play mp3 file

1. (host)
Download [Wheels on the Bus](http://billysworld.biz/wp-content/uploads/2014/12/WheelsOn-the-Bus.1.mp3)

```
adb push WheelsOn-the-Bus.1.mp3 /sdcard/wheels.mp3
```

2. on poplar console, after board boot up

```
# su
# stagefright -a -o /sdcard/wheels.mp3 
```

If you don't have access the console, you can use apps to play it. Reboot the board, open "Downloads" app, click "Audio", go through the dir hierarchy : `Unknown` -> `0 `-> `wheels.mp3`, and double click it. 

What to Expect:

You should hear the audio in both HDMI and audio line out interface.

### 1.3 test local media playback

1. Download the one of mp4 video from [1], call it `test_video.mp4`

2. (host) `adb push test_video.mp4 /sdcard/test.mp4`

3. Reboot the poplar board to make sure the new pushed media will be picked by the media player apps.

4. Plug in usb mouse (make sure your kernel has patch made ehci controller builtin)

4. Open Gallery app, click the video

What to Expect:

You should see the video playing and hear the audio in both HDMI and audio line out interface.

[1]http://www.mobiles24.co/downloads/tag/the+simpsons/mp4-videos

### 1.3 Test web media playback

1. Open web browser, to make sure we're testing the same video, type https://goo.gl/o8ErmS, which is `Taylor Swift - Look What You Made Me Do` in youtube.

What to Expect:

You should see the video playing and hear the audio in both HDMI and audio lineout interface.

## Test Graphics

## Test Wifi

## Test BlueTooth

## Report Issues

### Debug & capture logs

General Preparation:

- power on the board, and it will stop at the u-boot console
- set up the serial tool, the instruction depend on the serial console tool you used. for minicom, `ctrl-A`, then `L`, then the log file name.
- boot android, either using `run setupa; run boota` or `run bootai`

For start up issues:

- Waiting android booting, *as soon as* you can get the console, type `logcat`, wait the log dumping (may take ~10 seconds), `ctrl+c` to kill the `logcat`
- type `ps` to get all the running process.

For specific issues:

- start logcat and make sure the log is captured
- reproduce the issue
- send the logcat information for analysis

It is always a good idea to provide the start up logs as well to make sure the issue isn't caused in early phase (case study: not able the get dhcp address is due to mounting failure in data partition).

### debug audio

Play the media as described in 1.2, 1.3. Make sure the media is keep playing during following log capture process. The youtube one is long enough (~4 minutes). 

When the media is playing, do following in the host. 

```
(host) 
adb shell ps                             >> log_no_audio
adb shell dumpsys media.audio_flinger    >> log_no_audio
adb shell dumpsys media.audio_flinger    >> log_no_audio
adb logcat                               >> log_no_audio
```

`ctrl+c` to kill logcat and send along the `log_no_audio` file.


## Misc

### adb

Currently the board doesn't support usb OTG, you will have to use adb over tcp ip, and here are the steps to set it up.

1. Plug an Ethernet cable to your board and make sure eth0 is getting its address

```
poplar:/ # ifconfig eth0 | grep "inet addr"
          inet addr:192.168.0.18  Bcast:192.168.0.255  Mask:255.255.255.0
```

2. Write down the ip address, 192.168.0.18 in this case

3. On you developer machine:

```
$adb connect  ${poplar_ip_addr}       #192.168.0.18
```

And, check with `adb devices`

```
$ adb devices
List of devices attached
192.168.0.18:5555   device
```

4. Now, adb is ready for you to use, use `adb help` for more information.

```
$adb remount
$adb push path/to/your/tools  /system/bin
```
