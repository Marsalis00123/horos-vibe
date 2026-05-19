# Horos MPR Fusion Fix

This fork exists to test and preserve a targeted fix for a Horos 4.0.1 crash when opening a 3D MPR reconstruction while PET or SPECT fusion is enabled.

The observed crash happened on Apple Silicon macOS while creating the MPR window from a fused PET/SPECT study. The crash report consistently ended in the OpenGL/VTK shader display path:

```text
vtkShader::Compile()
vtkOpenGLShaderCache::ReadyShaderProgram(...)
vtkOpenGLRayCastImageDisplayHelper::RenderTexture(...)
vtkHorosFixedPointVolumeRayCastMapper::Render(...)
VRView renderBlendedVolume
MPRDCMView updateViewMPROnLoading
MPRController showWindow
```

Deleting Horos preferences made the workflow work once, then the crash returned.

## What Changed

The main fix is in `Horos/Sources/vtkHorosFixedPointVolumeRayCastMapper.cxx`.

Horos uses a hidden volume renderer while building blended MPR views. In that hidden path the renderer has drawing disabled, but the mapper still entered VTK's OpenGL texture display helper. On current macOS/OpenGL/Metal-backed systems that could crash during shader compilation.

This fork skips the final OpenGL display step when the renderer is not drawing:

```cpp
if ( ren->GetDraw() )
  {
  this->DisplayRenderedImage( ren, vol );
  }
```

The ray-cast volume buffer is still generated, so MPR fusion reconstruction continues to receive the image data it needs.

## Test Build

The first release is:

`v4.0.1-mpr-fusion-fix.1`

It contains a local, ad-hoc-signed Apple Silicon build named `Horos-patched.app`. It is intended for validation of this crash fix, not for broad distribution or notarized production use.

To test the fix:

1. Quit any running Horos.
2. Launch `Horos-patched.app`.
3. Open a PET/SPECT fusion study.
4. Leave fusion enabled.
5. Create the 3D MPR reconstruction.

The fixed behavior is that the reconstruction opens without crashing in the `vtkShader::Compile` path.

## Known Notes

- The bundled test app is ad-hoc signed and not notarized.
- The toolbar is readable in the patched macOS 26/Xcode-built app, but modern AppKit lays out some legacy Horos toolbar controls differently from the original distributed Horos build.
- 3D MPR with PET/SPECT fusion is still heavier than regular MPR. Performance tuning of the hidden blended-volume render path is a likely next step.

## Building

This repo still follows the Horos Xcode build structure, but current macOS/Xcode needs several compatibility fixes for old bundled dependencies.

Prerequisites:

- Xcode with macOS SDK
- Homebrew on Apple Silicon
- `cmake`
- `pkg-config`
- `git-lfs`

Initialize dependencies:

```sh
git lfs install --local
git submodule update --init --recursive
```

Build the app:

```sh
env -u CC -u CXX xcodebuild \
  -project Horos.xcodeproj \
  -scheme Horos \
  -configuration Release \
  -derivedDataPath build \
  CODE_SIGN_STYLE=Manual \
  CODE_SIGN_IDENTITY=- \
  DEVELOPMENT_TEAM= \
  COMPILER_INDEX_STORE_ENABLE=NO
```

The built app will be at:

```text
build/Build/Products/Release/Horos.app
```

## Upstream

Horos is an open-source medical image viewer derived from OsiriX. This fork is narrowly focused on validating the MPR fusion crash fix above.

For general Horos project information, see [horosproject.org](https://horosproject.org/get-involved/).
