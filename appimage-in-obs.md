# Building AppImages in OBS

See https://github.com/jirimaier/DataPlotter/pull/13#issuecomment-2763012734

I assume that you already have a project and a package within it.
You have committed the application sources to OBS and you want AppImages to be built.

You need to do four things:
* Define a repo at the project level that includes an `OBS:AppImage` path and some base OS path.

  A meta snippet like this seems to do the trick:
  ```xml
  <repository name="AppImage">
    <path project="OBS:AppImage" repository="toolchain.15.5"/>
    <path project="openSUSE:Leap:15.5:Update" repository="standard"/>    <arch>x86_64</arch>
  </repository>
  ```
* Define a custom "project config" that forces an AppImage build
  in the relevant repo. This can be seen in https://build.opensuse.org/projects/OBS:AppImage:Templates:Leap:15.1/prjconf .
  For DataPlotter, the following worked:
  ```
  %if "%_repository" == "AppImage"

  Type: appimage
  Repotype: staticlinks
  Patterntype: none

  FileProvides: /sbin/mkinitrd mkinitrd

  Required: build-pkg2appimage
  Conflict: build-pkg2appimage:libudev-mini1
  Conflict: build-pkg2appimage:systemd-mini
  Required: cmake-full
  Conflict: cmake-mini
  Ignore: !gpg2:pinentry
  Ignore: !systemd:kbd
  Ignore: !systemd:kmod
  Ignore: !systemd:systemd-presets-branding
  Ignore: !systemd:dbus-1
  Ignore: !systemd:pam-config
  Ignore: !systemd:udev
  Ignore: !libgio-2_0-0:dbus-1-x11
  Ignore: !logrotate:cron
  Ignore: !cracklib:cracklib-dict

  # build-compare ignore appimages. Means we would not get maintenance updates
  Support: !build-compare

  # the checks are usually not valid for AppImage files, disable all of them for now
  Support: !post-build-checks
  Support: !brp-check-suse
  Support: !rpmlint-Factory

  %endif
  ```
  The `cmake-full` and `cmake-mini` dance resolves an error saying that nothing provides the package `this-is-only-for-build-envs` required by `cmake-mini`.
  The rest of the stuff is copied from the template.

  The format of this file is described in https://openbuildservice.org/help/manuals/obs-user-guide/cha-obs-prjconfig
* Create a file `_service` within the package repo with the following contents:
  ```xml
  <services>
    <service name="appimage"/>
  </services>
  ```
* Createa file `appimage.yml` within the package repo. This will depend on your package, but QtRvSim and DataPlotter use something like
  ```yml
  app: data-plotter

  build:
    packages:
      - linuxdeployqt
      - pkgconfig(Qt5Widgets)
      - pkgconfig(Qt5Core)
      - pkgconfig(Qt5Gui)
      - pkgconfig(Qt5SerialPort)
      - pkgconfig(Qt5PrintSupport)
      - pkgconfig(Qt5OpenGL)
      - pkgconfig(Qt5Qml)
      - pkgconfig(Qt5QuickWidgets)
      - pkgconfig(Qt5Quick)
      - pkgconfig(Qt5QuickControls2)
      - pkgconfig(Qt5Network)
      - libqt5-linguist
      - libqt5-linguist-devel


  script:
    - cd $BUILD_SOURCE_DIR
    - tar xf data-plotter_3.3.2.orig.tar.xz
    - cd data-plotter-3.3.2
    - mkdir build
    - cd build
    - export CMAKE_BUILD_PARALLEL_LEVEL=$(nproc)
    - cmake .. -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr
    - cmake --build .
    - DESTDIR="$BUILD_APPDIR" cmake --install .
    - unset QTDIR; unset QT_PLUGIN_PATH; unset LD_LIBRARY_PATH
    - linuxdeployqt $BUILD_APPDIR/usr/share/applications/*.desktop -bundle-non-qt-libs -verbose=2 -no-strip
  ```

## Other links

* (incomplete) upstream docs: https://docs.appimage.org/packaging-guide/hosted-services/opensuse-build-service.html
* Repo where this seems to be working: https://build.opensuse.org/project/show/home:linuxtardis:sdi_linux_pkgs
* Another repo (this one was used for reverse engineering what needs to be done): https://build.opensuse.org/repositories/home:hawkeye116477:waterfox/waterfox-g-appimage