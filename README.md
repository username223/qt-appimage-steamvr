# Deploying Linux Qt 5.12 SteamVR AppImages from CircleCI

In the process of creating a working AppImage for a Qt 5.12 application that heavily relies on the `openvr` API ([OpenVR Advanced Settings](https://github.com/OpenVR-Advanced-Settings/OpenVR-AdvancedSettings) there were some issues.
This document is intended to provide a reference for problems and fixes.

## Why AppImage?

It was not possible to decide which format was superior (Snap vs Flatpak vs AppImage) however the tooling for creating AppImages seemed significantly better.

The presence of an easy to use tool (`[linuxdeployqt](https://github.com/probonopd/linuxdeployqt)`) was a big factor in deciding on AppImage.

## Creating the AppImage

We used `[linuxdeployqt](https://github.com/probonopd/linuxdeployqt)` to create the AppImages.
The documentation on the Github page details how to set up the folder structure and where to copy files to.

The undocumented `-unsupported-allow-new-glibc` switch can be used to prototype on Ubuntu newer than 16.04 Xenial.
Without it `linuxdeployqt` will refuse to work on newer than 16.04 Xenial.

## Folder structure and Setup

We followed the suggested setup from the `linuxdeployqt` documentation.
```
└── usr
    ├── bin
    │   ├── AdvancedSettings
	│	├── res
	│	│	└── QML and various resource files
	│	├── default_action_manifests
	│	│	└── HMD/Controller specific input manifests
	│	├── action_manifest.json
	│	├── manifest.vrmanifest
	│	└── libopenvr_api.so
    ├── lib
    └── share
        ├── applications
        │   └── AdvancedSettings.desktop
        └── icons
            └── hicolor
                └── 256x256 
                    └── apps 
                        └── AdvancedSettings.png
```
Our application (AdvancedSettings) was set up to find everything in the binary directory. 
Since we're not polluting the real `bin` directory we didn't change it.
The input manifests are required for the new OpenVR input system.
`libopenvr_api.so` has been copied from the OpenVR repo into the binary directory because AdvancedSettings is compiled with `rpath=$ORIGIN` for simplicity.

The `.vrmanifest` is a SteamVR file. 
It looks like this:
```
{
	"source" : "builtin",
	"applications": [{
		"app_key": "OVRAS-Team.AdvancedSettings",
		"launch_type": "binary",
		"binary_path_windows": "AdvancedSettings.exe",
		"is_dashboard_overlay": true,

		"strings": {
			"en_us": {
				"name": "Advanced Settings",
				"description": "OpenVR Advanced Settings Overlay"
			}
		}
	}]
}
```

`AdvancedSettings.desktop` looks like this
```
[Desktop Entry]
Type=Application
Name=OpenVR Advanced Settings
Comment=OVRAS
Exec=AdvancedSettings
Icon=AdvancedSettings
Categories=Office;
```

## `linuxdeployqt` on CircleCI

If you just attempt to run an AppImage (`linuxdeployqt` ships as an AppImage) on the CircleCI server you'll get something like
```
dlopen(): error loading libfuse.so.2

AppImages require FUSE to run. 
You might still be able to extract the contents of this AppImage 
if you run it with the --appimage-extract option. 
See https://github.com/AppImage/AppImageKit/wiki/FUSE 
for more information
Exited with code 1
```
To circumvent this you can use the `--appimage-extract-and-run` switch on the appimage itself.
This works with whatever other arguments and switches you're supplying to the application.

## Module is not installed

You might get something like
`QML Error: file:///tmp/.mount_OpenVRtQNnT6/usr/bin/res/qml/common/mainwidget.qml:1:1: module "QtQuick" is not installed`

The fix for this is appending `-qmldir=/path/to/your/qml/dir` at the creation of the AppImage. Supposedly `linuxdeployqt` doesn't like relative paths, so you might need a full path.

## `QtQuick/Dialogs` and `QtQuick/LocalStorage` not installed

`module "QtQuick.Dialogs" is not installed`

This is a long term issue with `linuxdeployqt` detailed [here](https://github.com/probonopd/linuxdeployqt/issues/25).

To solve it, manually copy the `Dialogs` folder.
We used Stephan Binners [PPA](https://launchpad.net/~beineri/+archive/ubuntu/opt-qt-5.12.3-xenial) which installed it in `/opt/qt512/qml/QtQuick/Dialogs`.
Additionally, our CI also required the `qt512quickcontrols` package, otherwise the `QtQuick/Dialogs` folder did not exist.

## Unsupported image format

`Error decoding: qrc:/common/backarrow: Unsupported image format`

In our case we needed`libqsvg.so` because we used `.svg` files.
Manually copying the file into `/usr/plugins/imageformats/libqsvg.so` made it work.

It appears that later versions of `linuxdeployqt` automatically deploys all image formats if they are present. 
In that case the error would likely be a missing package on the CI server.
In our case, using the [Stephan Binner PPA](https://launchpad.net/~beineri/+archive/ubuntu/opt-qt-5.12.3-xenial) we needed to also get `qt512imageformats` and `qt512svg` for it to work.
