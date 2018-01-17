# Poplar Andriod Status

## Andriod O build with 4.9 kernel (master)

| Features            | Status       |
| --------            | -------------|
| OpenGL              |     Y        |
| FB dev              |     Y        |
| HDMI Display        |     Y        |
| HDMI Audio          |     Y        |
| Audio Decoder - SW  |     Y        |
| Audio Decoder - HW  |     N        |
| Video Decoder - SW  |     Y        |
| Video Decoder - HW  |     Y        |
| Ethernet            |     Y        |
| USB Mouse/Keyboard  |     Y        |
| IR/Remote Control   |     Y        |
| Wifi                |     N        |
| BT                  |     N        |

## Known Issues

1. setting app crash when opening, need to cherry pick following patch if you want to use setting.

https://android-review.googlesource.com/#/c/platform/packages/apps/Settings/+/530416/

2. For v1 board (most of us are using), USB 2 port host mode will not work by default, enable it using following command. Alternatively, you can use USB3 port (for host mode) if you need to use keyboard/mouse.

```    
echo host > /sys/kernel/debug/hisi_inno_phy/role
```

For v2 board, USB2 port will be default to OTG mode.
