# FishCounting

credit to Charlie for the following env configuration for Macos

# Introduction
Instructions for installing [`openeb-4.5.2`](https://docs.prophesee.ai/stable/installation/linux_openeb.html) on an `arm64` Mac. Does not install ML modules (yet).

# Prerequisites

## Install package managers
1. Install [`homebrew`](https://brew.sh)
2. Install [`Miniforge3-MacOSX-arm64`](https://github.com/conda-forge/miniforge/)
3. Install [`macports`](https://www.macports.org/install.php)

## Install dependencies
```bash
brew install opencv boost libusb protobuf@3 hdf5 glew glfw pybind11 pck-config sophus
sudo port install ogre
conda create -n openeb-4.5.2-arm64 python=3.9
pip install "opencv-python==4.5.5.64" "sk-video==1.1.10" "fire==0.4.0" "numpy==1.23.4" "h5py==3.7.0" pandas scipy matplotlib "ipywidgets==7.6.5" pytest command_runner
```
Note: if you've already installed `protobuf` via homebrew, run the following to fix the version:
```bash
brew unlink protobuf
brew link --force protobuf@3
```

# Prepare source files
This assumes `$INSTALL` is the folder on your computer where all these files will live. To make things simple, cd to your preferred `$INSTALL` location, then run the following:
```bash
export INSTALL=$(pwd)
```

## Download `openeb`
```bash
cd $INSTALL
git clone https://github.com/prophesee-ai/openeb.git --branch 4.5.2 openeb-4.5.2-arm64
cd openeb-4.5.2-arm64
mkdir build
```

## Install SDKs
Download files from [Google Drive](https://drive.google.com/drive/folders/1Qnh3ywzRDvOSUx4vRcN-__Rd9F3zuOLt?usp=sharing)

```bash
cd ~/Downloads
tar -xvf metavision_sdk_sources_standalone_apps_x.y.z.tar -C $INSTALL/openeb-4.5.2-arm64
tar -xvf metavision_sdk_sources_analytics_x.y.z.tar -C $INSTALL/openeb-4.5.2-arm64
tar -xvf metavision_sdk_sources_calibration_x.y.z.tar -C $INSTALL/openeb-4.5.2-arm64
tar -xvf metavision_sdk_sources_cv3d_x.y.z.tar -C $INSTALL/openeb-4.5.2-arm64
tar -xvf metavision_sdk_sources_cv_x.y.z.tar -C $INSTALL/openeb-4.5.2-arm64
tar -xvf metavision_sdk_sources_ml_x.y.z.tar -C $INSTALL/openeb-4.5.2-arm64
```

# Apply fixes

## Fixing `OpenGL` errors
1. `open $INSTALL/openeb-4.5.2-arm64/sdk/modules/ui/cpp/include/metavision/sdk/ui/utils/opengl_api.h`
  - Change `#include <GL/gl.h>` to `#include <OpenGL/gl.h>`
2. `open $INSTALL/openeb-4.5.2-arm64/sdk/modules/ui/cpp/src/base_glfw_window.cpp`
  - Replace `void set_glfw_windows_hints()` with the following:
  ```bash
  void set_glfw_windows_hints() {
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
    glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
    glfwWindowHint(GLFW_CLIENT_API, GLFW_OPENGL_API);
  }
  ```
3. `open $INSTALL/openeb-4.5.2-arm64/sdk/modules/ui/cpp/src/base_window.cpp`
  - Change line 31 to `const char *vertex_shader_str = "#version 330\n"`
  - Change line 41 to `const char *fragment_shader_str = "#version 330\n"`

## Fixing `metavision_xyt` target 
1. Download [`imgui_1.90.4`](https://github.com/ocornut/imgui/releases/tag/v1.90.4)
2. Rename folder to `imgui` and move to `$INSTALL/openeb-4.5.2-arm64/sdk/modules/cv/cpp/samples/metavision_xyt/Polyfill/` 
3. `open $INSTALL/openeb-4.5.2-arm64/sdk/modules/cv/cpp/samples/metavision_xyt/Polyfill/OgreImGuiInputListener.cpp`
  - Change line 9 to `#include "imgui.h"`
  - Change line 79 to `io.KeyMap[ImGuiKey_KeypadEnter] = kc2sc(SDLK_KP_ENTER);`
4. `open $INSTALL/openeb-4.5.2-arm64/sdk/modules/cv/cpp/samples/metavision_xyt/Polyfill/OgreImGuiOverlay.h`
  - Change line 12 to `#include "imgui.h"`
5. `open $INSTALL/openeb-4.5.2-arm64/sdk/modules/cv/cpp/samples/metavision_xyt/Polyfill/OgreImGuiOverlay.cpp`
  - Change line 4 to `#include "imgui.h"`
  - Change line 5 to `#include "imgui/misc/freetype/imgui_freetype.h"`
6. `open $INSTALL/openeb-4.5.2-arm64/sdk/modules/cv/cpp/samples/metavision_xyt/Polyfill/imgui/misc/freetype/imgui_freetype.h`
  - Comment out line 46
  - Uncomment line 47
  - Comment out line 48

## Fixing `metavision_sdk_cv` targets
1. `open $INSTALL/openeb-4.5.2-arm64/sdk/modules/cv/cpp/src/triplet_matching_flow_algorithm_internal.cpp`
  - Comment line 124 and 131

## Fixing `metavision_sdk_cv3d` targets
1. `open $INSTALL/openeb-4.5.2-arm64/sdk/modules/cv3d/cpp/include/metavision/sdk/cv3d/algorithms/detail/edgelet_2d_tracking_algorithm_impl.h`
  - Change line 88 to `if (std::abs(matches_idx_[i] - median_idx) <= int(params_.median_outlier_threshold_)) {`

# Compile

## Generate `makefile`
```bash
cd $INSTALL/openeb-4.5.2-arm64/build
cmake .. -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTING=OFF -SOPHUS=ON
```
Confirm paths:
- `Found Python3...` is relative to `~/../miniforge3/envs/`
- `Found PkgConfig|LibUSB|Boost|pybind11|OpenCV|Protobuf|GLEW...` are relative to `/opt/homebrew/`
- `Found OpenGL...` is relative to `/Library/Developer/CommandLineTools`

## Build
```bash
cmake --build . --config Release -- -j 100

```

## Confirm
```bash
source $(readlink -f utils/scripts/setup_env.sh)
python -c "import metavision_sdk_core"
```
There should be no output and no errors.

## Persist environment variables
```bash
echo "source $(readlink -f utils/scripts/setup_env.sh)" >> ~/.zshrc
```

# Extra

1. Important, make sure when running `python`, the architecture is set to `arm64` (confirm via `file /path/to/python`).
2. There may be warnings throughout the build process, but these can (probably) be ignored.
