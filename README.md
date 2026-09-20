# FullyMask Camera

FullyMask Camera is a Windows desktop application that exposes browser content or an integrated webcam through one persistent Windows virtual camera. Applications such as WhatsApp, Telegram, Windows Camera, OBS, Zoom, and browser-based meeting tools can remain connected to **FullyMask Camera** while the user changes the source inside the FullyMask control application.

The project does not require OBS, an OBS virtual-camera plugin, desktop screen sharing, or a kernel-mode camera driver at runtime.

## Download

[Download FullyMask Camera 0.1](https://github.com/adorts/fullymask-camera/releases/download/0.1/FullyMask-Camera-Setup.exe)

The installer is currently unsigned. Windows SmartScreen may display an unknown-publisher warning until the release binaries and installer are signed with a trusted Authenticode certificate.

## What the application does

FullyMask Camera provides two source modes without requiring the receiving application to change its selected camera:

- **FullyMask ON** renders a supplied web address in a dedicated background Chromium process and sends that rendered page to the virtual camera.
- **FullyMask OFF** passes a selected physical webcam through the same virtual-camera device.

The control interface also lets the user select portrait or landscape output. The virtual device remains named **FullyMask Camera (Windows Virtual Camera)** in Windows camera selectors.

## Current platform support

The current native virtual-camera implementation requires:

- Windows 11, build 22000 or later
- A 64-bit Intel or AMD Windows installation
- Microsoft Edge WebView2 Runtime
- Permission to install the virtual camera for all users

Windows 10 and 32-bit Windows are not supported by the current build. The implementation uses Microsoft's `MFCreateVirtualCamera` API, which was introduced for Windows 11. Supporting Windows 10 would require a separate virtual-camera implementation and installer rather than a simple rebuild of this project.

## Output formats

The Media Foundation source advertises six NV12 formats at 30 frames per second:

| Orientation | Resolution | Aspect ratio | Frame rate | Format |
|---|---:|---:|---:|---|
| Portrait | 1080 x 1920 | 9:16 | 30 FPS | NV12 |
| Portrait | 720 x 1280 | 9:16 | 30 FPS | NV12 |
| Portrait | 540 x 960 | 9:16 | 30 FPS | NV12 |
| Landscape | 1920 x 1080 | 16:9 | 30 FPS | NV12 |
| Landscape | 1280 x 720 | 16:9 | 30 FPS | NV12 |
| Landscape | 960 x 540 | 16:9 | 30 FPS | NV12 |

Windows or the receiving application may expose additional converted formats such as YUY2. That conversion is performed by the Windows camera pipeline; the FullyMask source itself advertises NV12.

The receiving application controls how a camera frame is displayed. A calling application may crop, letterbox, or place a landscape stream inside its own layout. FullyMask controls the video frames and their declared dimensions, but it cannot force another application's call interface to become portrait or full screen.

## How it was built

The project combines a Go desktop application, a web-based control interface, a native C++ Media Foundation camera source, a native camera lifecycle broker, and an NSIS installer.

### Desktop application

The application shell is built with [Wails v2](https://wails.io/). Wails embeds the HTML, CSS, and JavaScript frontend in a native Windows executable and exposes selected Go methods to JavaScript.

The Go controller in `app.go` is responsible for:

- validating the source URL;
- starting and stopping URL capture;
- selecting physical-camera mode;
- replacing a running publisher when the mode changes;
- starting the native virtual-camera broker;
- reporting camera state to the user interface;
- stopping the publisher and broker during a normal application shutdown.

The Windows entry point and branded Wails window configuration are in `main.go`. The interface source is under `frontend/src`, and Vite builds it into `frontend/dist` for embedding by Wails.

### Browser URL renderer

URL mode is implemented in `internal/frames/publisher_url_windows.go` with [chromedp](https://github.com/chromedp/chromedp). It launches a private headless Microsoft Edge or Chromium session with a controlled viewport, navigates to the supplied URL, captures the rendered page in memory, decodes the captured image, and publishes the result to the native frame ring.

This is not desktop screen capture. Covering or minimizing the FullyMask control window does not change the URL feed because the page is rendered by a separate background browser process. Overlapping windows are not included in the virtual-camera output.

The background browser uses a temporary user-data directory. A normal application shutdown cancels its context and removes its child processes. During development, forcibly terminating the desktop executable can leave a browser process behind; normal users should close the application with its Quit control.

### Physical-camera mode

The frontend uses WebView2 `getUserMedia` to enumerate and open standard Windows video-input devices. The FullyMask virtual device is excluded from the physical-camera selector to prevent the output from feeding back into itself.

The current physical-camera path renders the selected webcam in the application's WebView and publishes the configured video region. Because this path depends on the visible WebView region, it is not yet as independent of the control window as URL mode. A future native Media Foundation source-reader implementation should replace this path.

### Shared frame transport

The application and Media Foundation source communicate through:

```text
%ProgramData%\FullyMaskCamera\frames.bin
```

`frame-pipeline/FrameProtocol.h` defines the binary contract. The file is used as a cross-process, cross-session memory-mapped triple buffer. Triple buffering lets the producer write a new frame without changing the buffer currently being read by Windows Camera Frame Server.

The publisher writes BGRA frame data and metadata, marks completed slots, and advances the published sequence. The camera source reads the newest complete slot, converts or scales it for the media type requested by the consumer, and emits the sample with Media Foundation timestamps. If the publisher temporarily stalls, the source repeats the last complete frame. Before the first valid frame, it emits black video rather than returning an invalid sample.

The file is placed in ProgramData because the Windows Camera Frame Server may run outside the desktop application's local object namespace. The installer grants the permissions required for the application and camera service to access the frame ring.

### Windows virtual camera

The native source is based on Microsoft's open-source [Windows-Camera Virtual Camera sample](https://github.com/microsoft/Windows-Camera/tree/master/Samples/VirtualCamera). The customized source is under:

```text
virtual-camera/reference/Samples/VirtualCamera/VirtualCameraMediaSource
```

`FullyMaskCameraMediaSource.dll` is an in-process COM Media Foundation source loaded by Windows Camera Frame Server. It implements the media source and stream consumed by camera applications.

`FullyMaskCameraBroker.exe` is the lifecycle broker. It calls `MFCreateVirtualCamera`, supplies the stable FullyMask source CLSID, starts or removes the Windows software-camera registration, and keeps the registered camera available to consumers.

The installer registers the source under this COM class:

```text
{5C0A9EC8-2637-4C71-B6A9-E8BE438CA48A}
```

This is a user-mode Media Foundation design. It does not install a kernel driver.

### Installer

The distributable setup program is built with NSIS through Wails' Windows packaging process. The customized script is located at `build/windows/installer/project.nsi`.

The installer:

1. Installs the Wails desktop executable.
2. Installs `FullyMaskCameraBroker.exe` and `FullyMaskCameraMediaSource.dll`.
3. Registers the COM media source for all users.
4. Creates the ProgramData frame directory and assigns its access permissions.
5. Invokes the broker to create the Windows virtual-camera registration.
6. Creates application shortcuts and uninstall information.

The uninstaller asks the broker to remove the camera registration, removes the COM registration, and removes the application files.

## Architecture

```text
                           FullyMask desktop application
                         Wails, Go, WebView2, HTML/CSS/JS
                                      |
                   +------------------+------------------+
                   |                                     |
          URL capture mode                     Physical camera mode
      Headless Edge via chromedp              WebView2 getUserMedia
                   |                                     |
                   +------------------+------------------+
                                      |
                            BGRA triple-buffer frames
                  %ProgramData%\FullyMaskCamera\frames.bin
                                      |
                     FullyMaskCameraMediaSource.dll
                         Media Foundation COM source
                                      |
                       Windows Camera Frame Server
                                      |
           Windows Camera, WhatsApp, Telegram, OBS, Zoom, browsers

FullyMaskCameraBroker.exe creates and manages the Windows virtual-camera device.
```

## Repository layout

```text
.
|-- app.go                         Go application controller
|-- main.go                        Wails Windows entry point
|-- frontend/                      Control interface and branded assets
|-- internal/frames/               URL, WebView, and frame-ring publishers
|-- frame-pipeline/FrameProtocol.h Shared native frame protocol
|-- virtual-camera/broker/         C++ virtual-camera lifecycle broker
|-- virtual-camera/reference/      Microsoft sample and customized MF source
|-- virtual-camera/bin/            Built native camera binaries
|-- installer/                     Registration and development scripts
|-- build/windows/installer/       Customized NSIS installer project
|-- go.mod                         Go module and dependency versions
`-- wails.json                     Wails product and build configuration
```

## Development requirements

- Windows 11 x64, build 22000 or newer
- Git
- Go 1.23 or newer
- Node.js 20 or newer with npm
- Wails v2 CLI
- Visual Studio 2022 Build Tools
- Desktop development with C++ workload
- A Windows 11 SDK containing the Media Foundation virtual-camera API
- NSIS for the distributable installer
- Microsoft Edge WebView2 Runtime

Install and verify the Wails toolchain:

```powershell
go install github.com/wailsapp/wails/v2/cmd/wails@latest
wails doctor
go version
node --version
npm --version
```

## Building from source

Clone the repository and install the frontend dependencies:

```powershell
git clone https://github.com/adorts/fullymask-camera.git
cd fullymask-camera
cd frontend
npm install
cd ..
```

Fetch the Microsoft Windows-Camera reference source when setting up a new development tree:

```powershell
.\virtual-camera\bootstrap-reference.ps1
```

Build the native broker and Media Foundation source:

```powershell
.\virtual-camera\build-native.ps1
```

Run the tests and build the desktop executable:

```powershell
go test ./...
wails build -clean
```

The desktop executable is written to `build\bin\fullymask-camera.exe`.

Build the distributable installer after the native binaries have been copied to `virtual-camera/bin`:

```powershell
wails build -clean -nsis
```

The release installer is written to `build\bin\FullyMask-Camera-Setup.exe`.

## Development installation and removal

The PowerShell scripts under `installer` are intended for local development. Run an elevated PowerShell terminal when registering the machine-wide COM source and virtual camera.

```powershell
.\installer\install-camera.ps1
```

Remove the development registration with:

```powershell
.\installer\uninstall-camera.ps1
```

Use `installer/hot-upgrade.ps1` while replacing a native source build during development. Close applications that are using FullyMask Camera before upgrading because Windows camera consumers and Camera Frame Server may retain the existing COM source until their sessions end.

## Running in development

Start the Wails development environment:

```powershell
wails dev
```

For an end-to-end camera test:

1. Install the native source and virtual-camera registration.
2. Run FullyMask Camera.
3. Enter a reachable HTTP or HTTPS URL.
4. Choose portrait or landscape output.
5. Start FullyMask mode.
6. Open Windows Camera or OBS and select **FullyMask Camera**.
7. Switch to physical-camera mode inside FullyMask and confirm that the receiving application remains attached to the same device.

## Release process

Before creating a public release:

1. Update the product version in `wails.json` and the release notes.
2. Run `go test ./...`.
3. Build the x64 Release native binaries.
4. Build the clean Wails application and NSIS installer.
5. Test installation on a clean Windows 11 x64 machine.
6. Verify all six advertised camera formats in at least one Media Foundation consumer.
7. Test URL mode, physical mode, source switching, application shutdown, uninstall, and reinstall.
8. Authenticode-sign the executable, broker, media-source DLL, and final installer.
9. Generate and publish a SHA-256 checksum.
10. Upload `FullyMask-Camera-Setup.exe` to the matching GitHub release.

## Known limitations

- Windows 10 is not supported by the current Media Foundation virtual-camera architecture.
- The published installer is x64 only.
- The current installer is not Authenticode-signed.
- Physical-camera passthrough currently depends on the WebView-rendered camera preview.
- URL capture uses an in-memory screenshot and decode path rather than a Direct3D shared texture, which adds processing overhead.
- Protected video, DRM content, pages requiring an existing browser profile, and sites that block automated browsers may not render as expected.
- A receiving application decides whether to crop, rotate, letterbox, mirror, or resize the camera feed in its own interface.
- Force-terminating FullyMask during development can leave its private headless browser process running until it is stopped or Windows is restarted.

## Troubleshooting

### FullyMask Camera is not listed

Confirm that the computer is running Windows 11 build 22000 or later, the installer was allowed to elevate, the native DLL is registered, and the broker completed its install command. Fully close and reopen the receiving application because camera-device lists are commonly cached.

### The virtual camera is listed but shows black

Start a source inside FullyMask, confirm that the URL is reachable, and confirm that security software has not blocked the background Edge process. Close and reopen the receiving application after changing native builds.

### The integrated camera does not work

Close other applications that may be using it. On Lenovo and other laptops, also check the mechanical camera shutter or side-mounted electronic privacy switch. Windows can report the camera driver as healthy while the hardware privacy control intentionally returns no image.

### A browser or calling application selects the wrong camera

Open that application's camera settings and choose **FullyMask Camera** when using this project, or **Integrated Camera** when bypassing it. Installing a virtual camera may change the browser's remembered or default camera selection.

### The page works in a normal browser but not in FullyMask

The site may require authentication stored in another browser profile, may reject automation, or may use protected media. Check the URL, network access, certificate status, and page permissions.

## Security and privacy

URL content is rendered locally on the user's computer. Frames are passed through local process memory and the ProgramData frame ring; FullyMask does not provide a cloud video relay or upload captured frames to this repository.

The application launches a browser engine for URL mode and accesses a webcam in physical mode. Users should only load sites they trust and should review camera permissions in Windows. Production distributions should be signed so users can verify the publisher and file integrity.

## Main technologies

- Go 1.23
- Wails v2
- HTML, CSS, and JavaScript
- Vite
- Microsoft Edge WebView2
- chromedp and the Chrome DevTools Protocol
- C++ with Visual Studio 2022
- Windows Media Foundation
- `MFCreateVirtualCamera` and `IMFVirtualCamera`
- Memory-mapped files and a triple-buffer frame protocol
- NSIS

## Technical references

- [MFCreateVirtualCamera documentation](https://learn.microsoft.com/windows/win32/api/mfvirtualcamera/nf-mfvirtualcamera-mfcreatevirtualcamera)
- [IMFVirtualCamera documentation](https://learn.microsoft.com/windows/win32/api/mfvirtualcamera/nn-mfvirtualcamera-imfvirtualcamera)
- [Microsoft Windows-Camera repository](https://github.com/microsoft/Windows-Camera)
- [Wails documentation](https://wails.io/docs/introduction)
- [chromedp repository](https://github.com/chromedp/chromedp)
- `virtual-camera/DESIGN.md` for the native frame and camera design

## License

No project license has been declared in this repository. Unless a license file is added, the source code remains under the copyright of its owner and no open-source usage rights are granted automatically. The Microsoft reference code under `virtual-camera/reference` retains its original license and notices.
