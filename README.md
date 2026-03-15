# Edge TPU runtime library (libedgetpu)

This repo contains the source code for the userspace
level runtime driver for [Coral devices](https://coral.ai/products).
This software is distributed in the binary form at [coral.ai/software](https://coral.ai/software/).

## Building

There are three ways to build libedgetpu:

* Docker + Bazel: Compatible with Linux, MacOS and Windows (via Dockerfile.windows and build.bat), this method ensures a known-good build enviroment and pulls all external depedencies needed.
* Bazel: Supports Linux, macOS, and Windows (via build.bat). A proper enviroment setup is required before using this technique.
* Makefile: Supporting only Linux and Native builds, this strategy is pure Makefile and doesn't require Bazel or external dependencies to be pulled at runtime.

### Bazel + Docker [Recommended]

For Debian/Ubuntu, install the following libraries:
```
$ sudo apt install docker.io devscripts
```

Build Linux binaries inside Docker container (works on Linux and macOS):
```
DOCKER_CPUS="k8" DOCKER_IMAGE="ubuntu:22.04" DOCKER_TARGETS=libedgetpu make docker-build
DOCKER_CPUS="armv7a aarch64" DOCKER_IMAGE="debian:bookworm" DOCKER_TARGETS=libedgetpu make docker-build
DOCKER_CPUS="k8 armv7a aarch64" DOCKER_IMAGE="debian:trixie" DOCKER_TARGETS=libedgetpu make docker-build
```

All built binaries go to the `out` directory. Note that the bazel-* are not copied to the host from the Docker container.

To package a Debian deb for `arm64`,`armhf`,`amd64` respectively:
```
debuild -us -uc -tc -b -a arm64 -d
debuild -us -uc -tc -b -a armhf -d
debuild -us -uc -tc -b -a amd64 -d
```

### Bazel
The version of `bazel` needs to be the same as that recommended for the corresponding version of tensorflow. For example, it requires `Bazel 6.5.0` to compile TF 2.19.1.

Current version of tensorflow supported is `2.19.1`.

Build native binaries on Linux and macOS:
```
$ make
```

Required libraries for Linux:

```
$ sudo apt install python3-dev
```

Build native binaries on Windows:
```
$ build.bat
```

Cross-compile for ARMv7-A (32 bit), and ARMv8-A (64 bit) on Linux:
```
$ CPU=armv7a make
$ CPU=aarch64 make
```

To package a Debian deb:
```
debuild -us -uc -tc -b
```
NOTE for MacOS: Compilation with MacOS fails. Two requirements:
- install `flatbuffers` (via macports)
- after failure in compilation, add the following line to the temporary file that is created by bazel in `/var/tmp/_bazl_xxxxx/xxxxxxxxxxxxx/external/local_config_cc/BUILD` line 48:
```
"darwin_x86_64": ":cc-compiler-darwin",
```
Repeat compilation.

### Makefile

If only building for native systems, it is possible to significantly reduce the complexity of the build by removing Bazel (and Docker). This simple approach builds only what is needed, removes build-time dependency fetching, increases the speed, and uses upstream packages.

TF 2.19.1 requires **FlatBuffers 24.3.25**, which is newer than what most distros ship. Build and install it from source first:
```
git clone --depth=1 --branch v24.3.25 https://github.com/google/flatbuffers /tmp/flatbuffers
cmake -S /tmp/flatbuffers -B /tmp/flatbuffers/build \
  -DCMAKE_INSTALL_PREFIX=/usr/local \
  -DFLATBUFFERS_BUILD_TESTS=OFF \
  -DCMAKE_BUILD_TYPE=Release
cmake --build /tmp/flatbuffers/build -j$(nproc)
sudo cmake --install /tmp/flatbuffers/build
sudo cp -r /tmp/flatbuffers/include/flatbuffers /usr/local/include/
```

Then install the remaining dependencies (available on Debian Bookworm/Trixie or Ubuntu 22.04+):
```
sudo apt install libabsl-dev libusb-1.0-0-dev xxd
```

Next, clone the [TensorFlow repo](https://github.com/tensorflow/tensorflow) at the matching tag:
```
git clone --depth=1 --branch v2.19.1 https://github.com/tensorflow/tensorflow
```

To build the library:
```
TFROOT=<Directory of Tensorflow> make -f makefile_build/Makefile -j$(nproc) libedgetpu
```

#### Debian 13 (Trixie) / Ubuntu 24.04 notes

On these systems `libabsl_flags` is split into sub-libraries. The Makefile already handles this automatically. No extra steps are needed beyond the instructions above.

Built libraries are placed in:
- `out/direct/k8/libedgetpu.so.1.0` — maximum operating frequency
- `out/throttled/k8/libedgetpu.so.1.0` — reduced operating frequency

## Support

If you have question, comments or requests concerning this library, please
reach out to coral-support@google.com.

## License

[Apache License 2.0](LICENSE)

## Warning

If you're using the Coral USB Accelerator, it may heat up during operation, depending
on the computation workloads and operating frequency. Touching the metal part of the USB
Accelerator after it has been operating for an extended period of time may lead to discomfort
and/or skin burns. As such, if you enable the Edge TPU runtime using the maximum operating
frequency, the USB Accelerator should be operated at an ambient temperature of 25°C or less.
Alternatively, if you enable the Edge TPU runtime using the reduced operating frequency, then
the device is intended to safely operate at an ambient temperature of 35°C or less.

Google does not accept any responsibility for any loss or damage if the device
is operated outside of the recommended ambient temperature range.

Note: This issue affects only USB-based Coral devices, and is irrelevant for PCIe devices.
