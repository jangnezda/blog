---
title: "Electron uninstaller for macOS"
date: 2021-11-21T08:11:06+01:00
draft: false
toc: false
images:
tags: 
  - electron
  - macos
---

Recently I was working on an [Electron](https://www.electronjs.org/) app that needed to work with system audio and for MacOS the authors have opted to use [BlackHole audio driver](https://github.com/ExistentialAudio/BlackHole). The app was already in development for some time and already used in production. But there was a problem: when user deleted the app, the audio driver wasn't deleted from the system.

The problem is that during installation the app installed audio driver files into `/Library/Audio/Plug-Ins/HAL`. Normally, this wouldn't be an issue, just bundle an uninstaller that removes those files when a user runs it. But a common pattern for macOS applications is to not have any uninstaller. Users simply delete the app from `Applications`. Wouldn't it be nice, if I could - as an app developer - hook into a 'user deleted the app' event? Then our uninstall logic would run on that event. Unfortunately, I could not find such an option.

Ok, so I checked how other macOS apps do this. Apps that add some system wide functionality on top of audio, file system, etc. And the common approach I noticed is that they do in fact bundle uninstallers. For example, [My Cloud](https://www.mycloud.com/) NAS:

[![My Cloud - Applications](/posts/electron-macos-uninstall/1.small.png)](/posts/electron-macos-uninstall/1.png)

Yuck, all those sub-directories. But in reality it's not a big issue. Myself and many other macOS users don't regularly browse the `Applications` folder in Finder. Instead, I just use `CMD + Space` to bring up search bar and type the app name.

[![My Cloud - spotlight](/posts/electron-macos-uninstall/2.small.png)](/posts/electron-macos-uninstall/2.png)

The uninstaller is right there at the top. As an aside, I should point out that many discourage having a 'Windows uninstaller pattern' in macOS. There should be no uninstaller-type of thing in macOS, they say. It's unnecessary. However, I don't know of a way to clean system-wide stuff like drivers, when a user moves an app to a trash bin.

One alternative to having a separate app for uninstalling is to initiate uninstall right from the running app. For example, have a menu option to uninstall the app. However, this approach was immediately rejected by product owner. "We can't have the users uninstall our app too easily!", he said.

It was then agreed to have an uninstaller for our app as well. Now comes the second problem: Electron does not provide any out-of-the-box support for uninstallers in macOS. It does for [Windows](https://www.electron.build/configuration/nsis.html), but not for Mac. So I turned to Google again to see what people are doing in such cases. It may come to no surprise that quite a few decide to build another Electron app for uninstalling and bundle it with the main app. Ugh, I hated the approach, so wasteful. Both, time-wise (development) and resource-wise (users' hard drives).

Scratch Electron, what are people doing in general when it comes to uninstallers on macOS. There must be a sane way. Essentially, I needed just:

1. "Are you sure you want to uninstall?" prompt,
2. "Elevate to admin" prompt,
3. Run uninstall script

In the end, I stumbled upon this answer: [How to make a Mac OS X .app with a shell script?](https://apple.stackexchange.com/questions/224394/how-to-make-a-mac-os-x-app-with-a-shell-script). It's achingly simple and works in all macOS versions that I've tried with. Have an executable bash script and mimic the normal macOS app directory structure. Then it automagically works.

```
/Applications
  |
  `-- MyBash.app
        |
        `-- Contents
              |
              `-- MacOS
              |    |
              |    `-- MyBash
              |
              `-- Resources
                    |
                    `-- ... (any other files like icons, etc.)

```

`MyBash` is simply a bash script that is executable. Then, this app will show as `MyBash` in Finder and double clicking it will run the `MyBash` script	. We can take advantage of this for the uninstaller by bundling such structure in the main electron app as an extra resource. Then the installer copies it into `/Applications` directory and voila, uninstaller is there and ready to be run by the user.

Final piece of the puzzle was how to display prompts from bash scripts. This can be achieved by using Apple script that is in every macOS by means of [osascript](https://ss64.com/osx/osascript.html) command.

Alright, let's examine all the steps (assuming you are using [electron-builder](https://www.electron.build) for bundling/installer):

1. First, the uninstaller app. Should be quite simple, just follow the structure I outlined above. The script itself should be roughly: display couple of prompts, then remove some directories. For example:

    ```
    #!/bin/sh

    CHOICE=`osascript <<EOF
    button returned of (display dialog "This action will uninstall MyApp and associated audio driver. Are you sure you want to continue?" buttons {"Cancel", "Uninstall"} default button 2 with icon caution with title "MyApp Uninstall")
    EOF`

    if [[ "$CHOICE" == "Uninstall" ]]
    then
      APP_PATH="/Applications/Utilities/MyApp/MyAppUninstall.app/Contents"
      osascript -e 'do shell script "$APP_PATH/Resources/uninstall.sh" with administrator privileges'
    fi
    ```
    
    Then the `uninstall.sh` might look like:
    
    ```
    #!/bin/sh

    # Remove following directories
    Dirs=(
        "/Applications/MyApp.app"
        "/Applications/Utilities/MyApp"
        "/Library/Audio/Plug-Ins/HAL/MyApp	Audio.driver"
        "/Users/$SUDO_USER/Library/Caches/com.electron.myapp"
        "/Users/$SUDO_USER/Library/Caches/com.electron.myapp.ShipIt"
    )

    for Dir in ${Dirs[*]}
    do
        if [[ -d "$Dir" ]]
        then
            rm -rf "$Dir"
        fi
    done

    # Restart core audio
    launchctl kickstart -kp system/com.apple.audio.coreaudiod

    osascript <<EOF
    display dialog "MyApp successfully uninstalled." buttons {"Quit"} default button 1 with title "MyApp Uninstall"
    EOF
    ```
    
    Essentially it's remove some dirs, restart audio subsystem, display success prompt.

2. Add uninstaller app to your main Electron app, to the directory specified by [extraResources](https://www.electron.build/configuration/contents.html#extraresources) configuration param. That directory will be copied into `/Applications/MyApp.app/Contents/Resources` during installation.

3. Add a `postinstall` script in the main Electron app - by default in the `pkg-scripts` directory. More info about it [here](https://www.electron.build/configuration/pkg). This script should move the bundled uninstaller from `/Applications/MyApp.app/Contents/Resources/MyAppUninstaller.app` to `/Applications/Utilities/MyAppUninstaller.app` (or another directory of your choice, as long as it's within `/Applications`). For example:

    ```
    #!/bin/sh

    # Move uninstaller app, so it's accessible for the user in "Applications"
    UTILITIES_DIR="$2/Utilities/MyApp"
    mkdir -p "$UTILITIES_DIR"
    mv "$2/MyApp.app/Contents/Resources/extraResources/uninstall/MyAppUninstall.app" "$UTILITIES_DIR"
    ```

    Note that `$2` is the parent installation directory. `/Applications` by default, however the user may choose another location during installation.

That's it! Very straightforward once the right things fall in place. The main takeaway for me was that it was right to explore some more instead of going the way of another Electron app for uninstaller.