# openssl-prebuilt

This repository builds and publishes self-contained OpenSSL prebuilt packages for Windows.

The initial target is Windows x64 with MSVC on GitHub Actions `windows-latest`. It does not use vcpkg. OpenSSL is built directly from an official upstream git tag.

Current default OpenSSL ref: `openssl-3.5.6`.

## Packages

Each build produces both package variants:

- `openssl-<version>-windows-x64-static.zip`
- `openssl-<version>-windows-x64-shared.zip`

The zip files contain a single top-level directory:

```text
openssl-windows-x64-static/
  include/
  lib/
```

```text
openssl-windows-x64-shared/
  bin/
  include/
  lib/
```

The static package is configured with:

```powershell
perl Configure VC-WIN64A no-shared no-module no-tests no-docs no-asm --prefix=<install-dir>
nmake
nmake install_sw
```

The shared package is configured with:

```powershell
perl Configure VC-WIN64A no-tests no-docs no-asm --prefix=<install-dir>
nmake
nmake install_sw
```

SHA256 files are generated next to each zip:

- `openssl-<version>-windows-x64-static.zip.sha256`
- `openssl-<version>-windows-x64-shared.zip.sha256`

## Build

The GitHub Actions workflow builds packages when:

- Changes are pushed to `main`.
- The workflow is started manually with `workflow_dispatch`.
- A release tag matching `openssl-*-msvc-x64-static-*` or `openssl-*-msvc-x64-shared-*` is pushed.

Manual builds accept an `openssl_ref` input, for example:

```text
openssl-3.5.6
```

The ref is passed to:

```powershell
git clone --depth 1 --branch <openssl_ref> https://github.com/openssl/openssl.git
```

## Releases

Push a tag to build and publish a GitHub Release:

```powershell
git tag openssl-3.5.6-msvc-x64-static-1
git push origin openssl-3.5.6-msvc-x64-static-1
```

The workflow parses the OpenSSL ref from the tag, builds both static and shared packages, creates the GitHub Release, and uploads all assets:

- `openssl-3.5.6-windows-x64-static.zip`
- `openssl-3.5.6-windows-x64-static.zip.sha256`
- `openssl-3.5.6-windows-x64-shared.zip`
- `openssl-3.5.6-windows-x64-shared.zip.sha256`

Download the asset that matches your desired linkage. Use the `.sha256` file to verify the zip.

## CMake Client Usage

Client projects can select a package with `XY_OPENSSL_LINKAGE=static|shared`, download the matching release asset with `FetchContent`, and then point CMake's OpenSSL package discovery at the extracted root.

```cmake
cmake_minimum_required(VERSION 3.24)

include(FetchContent)

set(XY_OPENSSL_LINKAGE "static" CACHE STRING "OpenSSL linkage: static or shared")
set_property(CACHE XY_OPENSSL_LINKAGE PROPERTY STRINGS static shared)

set(XY_OPENSSL_VERSION "3.5.6")
set(XY_OPENSSL_RELEASE_TAG "openssl-3.5.6-msvc-x64-static-1")

if(XY_OPENSSL_LINKAGE STREQUAL "static")
  set(XY_OPENSSL_SHA256 "<sha256-from-openssl-3.5.6-windows-x64-static.zip.sha256>")
elseif(XY_OPENSSL_LINKAGE STREQUAL "shared")
  set(XY_OPENSSL_SHA256 "<sha256-from-openssl-3.5.6-windows-x64-shared.zip.sha256>")
else()
  message(FATAL_ERROR "XY_OPENSSL_LINKAGE must be static or shared")
endif()

set(XY_OPENSSL_URL
  "https://github.com/<owner>/<repo>/releases/download/${XY_OPENSSL_RELEASE_TAG}/openssl-${XY_OPENSSL_VERSION}-windows-x64-${XY_OPENSSL_LINKAGE}.zip")

FetchContent_Declare(
  xy_openssl
  URL "${XY_OPENSSL_URL}"
  URL_HASH SHA256=${XY_OPENSSL_SHA256}
)
FetchContent_MakeAvailable(xy_openssl)

set(OPENSSL_ROOT_DIR "${xy_openssl_SOURCE_DIR}/openssl-windows-x64-${XY_OPENSSL_LINKAGE}")

if(XY_OPENSSL_LINKAGE STREQUAL "static")
  set(OPENSSL_USE_STATIC_LIBS TRUE)
endif()

find_package(OpenSSL REQUIRED)

target_link_libraries(your_target PRIVATE OpenSSL::SSL OpenSSL::Crypto)
```

For the shared package, make sure the extracted `bin/` directory is available at runtime, for example by copying its DLLs next to your executable during your application's install or packaging step.
