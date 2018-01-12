# Features status

## Andriod O build with 4.9 kernel (master)

| Features            | Status       |
| --------            | -------------|
| OpenGL              |     Y        |
| FB dev              |     Y        |
| HDMI Display        |     Y        |
| HDMI Audio          |     Y        |
| Audio Decoder (SW)  |     Y        |
| Video Decoder (SW)  |     Y        |
| Ethernet            |     Y        |
| USB Mouse/Keyboard  |     Y        |
| Wifi                |     N        |
| BT (audio)          |     N        |


Known Issues :

- setting app crash when opening, need to cherry pick following patch if you want to use setting.

https://android-review.googlesource.com/#/c/platform/packages/apps/Settings/+/530416/

- For v1 board (most of us are using), USB 2 port host mode will not work by default, enable it using following command. Alternatively, you can use USB3 port (for host mode) if you need to use keyboard/mouse.

```    
echo host > /sys/kernel/debug/hisi_inno_phy/role
```

For v2 board, USB2 port will be default to OTG mode.

## Andriod O build with 4.4 kernel

| Features            | Status       |
| --------            | -------------|
| OpenGL              |     Y        |
| FB dev              |     Y        |
| HDMI Display        |     Y        |
| HDMI Audio          |     Y        |
| Audio Decoder (SW)  |     Y        |
| Video Decoder (SW)  |     Y        |
| Ethernet            |     Y        |
| USB Mouse/Keyboard  |     Y        |
| Wifi                |     N        |
| BT (audio)          |     N        |


Known Issues :

- audio: audio line out isn't configured correctly at the moment, waiting kernel patch.
- setting app crash when opening, need to apply following patch if you want to use setting.

https://android-review.googlesource.com/#/c/platform/packages/apps/Settings/+/530416/

## Andriod N build


| Features            | Status       |
| --------            | -------------|
| OpenGL              |     Y        |
| FB dev              |     Y        |
| HDMI Display        |     Y        |
| HDMI Audio          |     Y        |
| Audio Decoder (SW)  |     Y        |
| Video Decoder (SW)  |     Y        |
| Ethernet            |     Y        |
| USB Mouse/Keyboard  |     Y        |
| Wifi                |     Y        |
| BT (audio)          |     Y        |

Known Issues :

- audio: audio line out isn't configured correctly at the moment, waiting kernel patch.
- setting app crash when click some items (say wifi)