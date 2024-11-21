# Updating bundled gdal version for gdal-src

Perform the following steps:

```
git submodule init
git submodule update
cd gdal-src/source
git pull
git checkout v3.8.3 # corresponds to the tag you want to update to
cd ../../
git add gdal-src/source
git commit -m "Update bundled gdal version to 3.8.4"
```

These steps assume that there are no fundamental changes to the gdal build system.

# Generating Binding

```bash
docker run -it --rm -v ./gdal-sys:/gdal_sys ghcr.io/osgeo/gdal:ubuntu-full-$GDAL_VERSION /bash
# everything from now on is inside of the container

# install minimal rust version
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- --profile minimal --component rustfmt -y
. "$HOME/.cargo/env"

# install mingw toolchain for generating windows bindings
# install libclang for bindgen
# gcc as linker
# gcc-i686-linux-gnu to generate bindings for 32 bit linux
apt update
apt install -y libclang-dev mingw-w64 gcc gcc-i686-linux-gnu pkg-config 

# install bindgen
cargo install bindgen-cli

# create the output directory for the prebuild bindings if necessary
mkdir -p /gdal_sys/prebuilt-bindings/$GDAL_VERSION/

# if you update these command consider updating the command in gdal-sys/build.rs
# make sure to use the same bindgen flags (everything before wrapper) for
# all targets
#
# 64 bit linux/macos
bindgen --constified-enum-module ".*" --ctypes-prefix ::std::ffi --allowlist-function "(CPL|CSL|GDAL|OGR|OSR|OCT|VSI).*" /gdal_sys/wrapper.h > /gdal_sys/prebuilt-bindings/$GDAL_VERSION/gdal_x86_64-unknown-linux-gnu.rs
# 32 bit linux/macos
bindgen --constified-enum-module ".*" --ctypes-prefix ::std::ffi --allowlist-function "(CPL|CSL|GDAL|OGR|OSR|OCT|VSI).*" /gdal_sys/wrapper.h -- -target i686-unknown-linux-gnu --sysroot /usr/i686-linux-gnu/ -I /usr/include > /gdal_sys/prebuilt-bindings/$GDAL_VERSION/gdal_i686-unknown-linux-gnu.rs

# make sure we don't get the wrong system headers
rm /usr/include/stdio.h /usr/include/stdlib.h /usr/include/limits.h /usr/include/features-time64.h /usr/include/features.h /usr/include/malloc.h /usr/include/string.h /usr/include/ctype.h /usr/include/errno.h /usr/include/math.h /usr/include/stdint.h /usr/include/time.h

# 64 bit windows
bindgen --constified-enum-module ".*" --ctypes-prefix ::std::ffi --allowlist-function "(CPL|CSL|GDAL|OGR|OSR|OCT|VSI).*" /gdal_sys/wrapper.h -- -target x86_64-pc-windows-gnu -I /usr/include/ > /gdal_sys/prebuilt-bindings/$GDAL_VERSION/gdal_x86_64-pc-windows-gnu.rs
# 32 bit windows 
bindgen --constified-enum-module ".*" --ctypes-prefix ::std::ffi --allowlist-function "(CPL|CSL|GDAL|OGR|OSR|OCT|VSI).*" /gdal_sys/wrapper.h -- -target i686-pc-windows-gnu -I /usr/include/ > /gdal_sys/prebuilt-bindings/$GDAL_VERSION/gdal_i686-pc-windows-gnu.rs
```
