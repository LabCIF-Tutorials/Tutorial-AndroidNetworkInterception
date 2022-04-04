# Tutorial: Android Network Traffic Interception <!-- omit in toc -->
How to intercept network trafic on Android 

| Version | 2022.04.04 |
| :-:     | :--        |
| ![by-nc-sa](https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png) | This work is licensed under a [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-nc-sa/4.0/) |

## Table of Contents <!-- omit in toc -->

- [Requirements](#requirements)
- [Methods](#methods)
- [HttpTolkit](#httptolkit)
- [Bypass Certificate Pinning](#bypass-certificate-pinning)
  - [Install Frida on the PC](#install-frida-on-the-pc)
  - [Install Frida on Android](#install-frida-on-android)
  - [Intercept networt traffic from APPS with certificate pinning](#intercept-networt-traffic-from-apps-with-certificate-pinning)
- [Exercises](#exercises)
  - [Exercise 1](#exercise-1)
  - [Exercise 2](#exercise-2)

## Requirements

In order to implement this tutorial you need to use one of these Android devices:

- Android Virtual Device (AVD) -- see one of these tutorials: [Android Studio Emulator - GUI](https://labcif.github.io/AndroidStudioEmulator-GUIconfig/), or [Android Studio Emulator - command line](https://labcif.github.io/AndroidStudioEmulator-cmdConfig/) to learn how to set up an AVD;
  - Android 11 (API version 30), or older, is recommended for this tutorial
- or a physical smartphone with Android rooted. Rooting an Android device is beyond the scope of this tutorial, but you can read this [webpage](https://magiskmanager.com/) to learn more about it.

> ***NOTE***
>
> The Android emulator uses the `x86`, or `x86_64` CPU instruction set. However, some APPs are compiled only for `arm`, or `arm64` CPU architectures. 
> If the APP you are analysing does not provide a version for `x86`, or `x86_64`, you need to use **Android 9**, or **Android 11** on the emulator, because these versions include a translation mechanism from `arm` instructions to `x86`.

## Methods

To intercept the network traffic of an Android device we need a proxy. The proxy will act as Man-in-the-middle between the Android device and the servers it connects to. There are several ways to accomplish network traffic interception:

- using a proxy on a computer, like [mitmproxy](https://mitmproxy.org/), or [PolarProxy](https://www.netresec.com/?page=PolarProxy);
- using a fake VPN on Android to act like a proxy, like [Packet Capture](https://www.apkmirror.com/?s=packet+capture), or [HTTP Toolkit](https://httptoolkit.tech/).
  
**Using a proxy on a computer** -- this method is a bit more complex to setup, but is the one that generally guarantees more flexibility to analyse the captured traffic. The main disadvantage is that all Android traffic is routed through the proxy and it's more difficult to find the packects related to the app we want to study.

**Using a fake VPN on Android** -- this is the simplest way to intercept traffic, and it allows choosing just one app to be redirected and captured. On one hand, no root permission is required, on the other hand it might require extra steps to download the captured packets to a computer.

## HttpTolkit

For this tutorial we are going to use HTTP Toolkit that sets up a fake VPN service. Download [HTTP Toolkit](https://httptoolkit.tech/) (it's available for Linux, MacOS and Windows) and then install it on your computer.

Start Android Vistual Device (AVD) and open the HTTP Toolkit software. On the main window you'll see several options, select `Android Device via ADB`:

![](img/../imgs/httptoolkit.png)

When the option `Android Device via ADB` is selected, several things happen behind the scenes:

- the app `tech.httptoolkit.android.v1` is installed on the AVD
- the `HTTP Toolkit CA` digital certificate is added to the `Trusted credentials`:
  
  ![](imgs/AVD_trusted_ca.png)

- a fake VPN service is started on the AVD:
  
  ![](img/../imgs/AVD_httptoolkit.png)


By default, HTTP Toolkit will intercept the network traffic from **ALL** apps and services installed on the AVD. However, we are going to analyse just one app, so let's change HTTP Toolkit configurations on the AVD:

- click the button `All APPS`
- on the 3 vertical dots menu choose `Disable all apps`
- again on the 3 vertical dots menu, choose `Show system`
- now, on the search bar type `chrome` and enable the capture:

![](imgs/httptoolkit-chrome.png)

To generate some traffic, open the Chrome browser on the AVD and type `AFD2` (or something else) on the address bar and press enter. This will make a query to google search, and the HTTP Toolkit on the computer will show the captured network packets:

![](imgs/httptoolkit-view.png)

However, you might not be able to access any website due to the Certificate Pinning protection. Keep reading to learn how to bypass it.

> ***NOTE***
>
> The HTTP Toolkit is an open source project hosted on [https://github.com/httptoolkit/httptoolkit](https://github.com/httptoolkit/httptoolkit). However, there are some features that are reserved for the paying costumers, namely the ability to save the captures into a file. This can be overcome by copy/paste the packets contents. Alernativelly, you can use `mitmproxy`, but the setup process is more complex.

## Bypass Certificate Pinning

After the proxy is enabled and the digital certificates are properly configured, some APPS might still not work. That happens because they are able to detect that the digital certificate we are using is not the one they expect. This technique is called [certificate pinning](https://owasp.org/www-community/controls/Certificate_and_Public_Key_Pinning). Certificate pinning is an extra step to protect SSL/TLS network traffic from Man-in-the-middle attacks, which we are trying to do.

In order to bypass certificate pinning we need to dynammicly change the network traffic. We can use [Frida](https://frida.re/), an open source tool for dynamic interception and alteration of network traffic to bypass some certificate pinning security mechanisms.

### Install Frida on the PC

To [install Frida](https://frida.re/docs/installation/) we need to have the latest Python 3.x. Then install Frida via `pip` tool:

```Console
pip install frida-tools
```

Or grab the binaries from [Frida’s GitHub releases](https://github.com/frida/frida/releases) page.

> ***NOTE***
>
> On Windows, it is useful to add the frida-tools to the path on the system environment variables, go to: `Control Panel > System > Advanced System Settings > Environment Variables`. Then add the parent folder in which Frida is installed: `C:\Users\<username>\AppData\Local\Packages\PythonSoftwareFoundation.Python.3.9_qbz5n2kfra8p0\LocalCache\local-packages\Python39\Scripts\` (adapt accordingly to your  Python version)

> ***NOTE ***
>
> If you installed Frida with `pip` you may need to add its location to the PATH:
>
> ```Console
> user@linux:AFD2$ export $PATH:$HOME/.local/bin
> ```
>
> On Windows, go to: `Control Panel > System > Advanced System Settings > Environment Variables` and add the default `PATH` to `pip` installed tools:
>
> `C:\Users\<username>\AppData\Local\Packages\PythonSoftwareFoundation.Python.3.9_qbz5n2kfra8p0\LocalCache\local-packages\Python39\Scripts` (adapt according to your Python version)


### Install Frida on Android

To install Frida on Android, the device must be rooted first. For this tutorial we are going to use an Android Virtual Device (AVD) running Android 11 (API version 30).

> ***NOTE***
>
> Make sure `frida` already supports the Android version you're using.

Download the `frida-server` from [Frida’s GitHub releases](https://github.com/frida/frida/releases) page that **matches both**:

- The CPU architectutre of your Android device. If you are not sure check it by doing `adb shell` followed by `uname -m`;
- and the `frida` (client) version running on the desktop. If you are not sure check it by doing `frida --version`.

Then uncompress it with [**7zip**](https://www.7-zip.org/download.html), or on the Linux command line:

```Console
user@linux:AFD2$ unxz frida-server-15.1.16-android-x86_64.xz
```

> ***NOTE***
>
> Be aware that your emulator might be `x86` (32 bits) instead of the `x86_64` (64 bits) that is used in this tutorial.
>
> If your are using a physical Android device, the CPU architecture could be `armv8l`,
> in that case you should download the `arm64` version of the `frida-server`.

Now, make sure your Android device is connected, copy `frida-server` to your device and run it as root, as shown here:

```Console
user@linux:AFD2$ adb devices
List of devices attached
emulator-5554   device
user@linux:AFD2$ adb push ./frida-server-15.1.16-android-x86_64 /sdcard/Download/
./frida-server-15.1.16-android-x86_64/: 1 file pushed. 99.8 MB/s (41358640 bytes in 0.395s)
user@linux:AFD2$ adb shell 
generic_x86_64:/ $ su
generic_x86_64:/ # cd /data/local/tmp
generic_x86_64:/data/local/tmp # cp /sdcard/Download/frida-server-15.1.16-android-x86_64 .
generic_x86_64:/data/local/tmp # chmod 755 frida-server-15.1.16-android-x86_64
generic_x86_64:/data/local/tmp # ./frida-server-15.1.16-android-x86_64 &
[1] 6268
```

Open a new terminal and test if Frida is running:

```Console
user@linux:AFD2$ frida-ps -Uai
4914  Chrome                   com.android.chrome                     
1358  Google                   com.google.android.googlequicksearchbox
1358  Google                   com.google.android.googlequicksearchbox
4203  HTTP Toolkit             tech.httptoolkit.android.v1            
3792  Phone                    com.android.dialer                     
4784  Settings                 com.android.settings                   
...
```

> ***NOTE***
>
> If you need to terminate `frida-server` do (change `8888` to the actual PID):
> ```Console
> user@linux:AFD2$ adb shell
> a40:/ $ su
> a40:/ # ps -e | grep frida-server
> root          8888     1  139456   4416 poll_schedule_timeout 79c0dce088 S frida-server
> kill -9 8888
> ```

### Intercept networt traffic from APPS with certificate pinning

Suppose we want to bypass Google Chrome certificate pinning, the first step is to identify the package name:

```Console
user@linux:AFD2$ frida-ps -U | grep chrome
6061  com.android.chrome
6415  com.android.chrome:privileged_process0
6394  com.android.chrome:sandboxed_process0:org.chromium.content.app.SandboxedProcessService0:5
6384  com.android.chrome_zygote
```

Then apply the `javascript` that enables to bypass certificate pinning with Frida. In the computer run:

```Console
user@linux:~$ frida -U --codeshare akabe1/frida-multiple-unpinning -f <mobile-app-name> --no-pause
```

For the Google Chrome browser:

```Console
user@linux:~$ frida -U --codeshare akabe1/frida-multiple-unpinning -f com.android.chrome 
     ____
    / _  |   Frida 15.1.16 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://frida.re/docs/home/
   . . . .
   . . . .   Connected to Android Emulator 5554 (id=emulator-5554)
Spawned `com.android.chrome`. Use %resume to let the main thread start executing!
[Android Emulator 5554::com.android.chrome ]-> %resume
[Android Emulator 5554::com.android.chrome ]->
======
[#] Android Bypass for various Certificate Pinning methods [#]
======
[-] OkHTTPv3 {1} pinner not found
...
```

If everything is working as expected, you should now be able to read the content of the `SSL` packets.


> ***NOTE***
>
> Frida is able to avoid certificate pinning from many Android apps, **but not all of them**. For example, Tiktok is known to have implemented some technics against Frida and other similar tools.
>
> If the certificate pinning bypass is not working for your mobile app, try:
>
> - with an older version of the app itself,
> - or, use an older version of Android,
> - or both an older version of the app and older version of Android.

## Exercises

### Exercise 1

On the AVD open Chome and access to `https://ead.ipleiria.pt`

- then, on the login page type:
  - for the username: `Asdrubal`
  - for the password: `loves AFD2!!`
- go to the HTTP Toolkit interface on your computer and find the packet that contains the username and password.

### Exercise 2

Execute the following tutorials:

- mandatory
  - [Solving OWASP MSTG UnCrackable App for Android Level 1](https://nibarius.github.io/learning-frida/2020/05/16/uncrackable1)
- optional
  - [Solving OWASP MSTG UnCrackable App for Android Level 2](https://nibarius.github.io/learning-frida/2020/05/23/uncrackable2)
  - [Solving OWASP MSTG UnCrackable App for Android Level 3](https://nibarius.github.io/learning-frida/2020/06/05/uncrackable3)