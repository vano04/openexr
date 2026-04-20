<!-- SPDX-License-Identifier: BSD-3-Clause -->
<!-- Copyright (c) Contributors to the OpenEXR Project -->

This is a fork adding some features to the PYBIND that already exist in C++ to expose them when using the python module as a library. The changes in this fork require a few changes (maybe soon) to be compliant with the style guide defined in [CONTRIBUTING.md](CONTRIBUTING.md).

## PyOpenEXR Wrapper Changes Summary

### 1. Added tile read APIs for tiled EXR files
Two new File methods were added:
- readTile(dx, dy, part_index=0, level_x=0, level_y=0, separate_channels=False)
- readTiles(dx1, dx2, dy1, dy2, part_index=0, level_x=0, level_y=0, separate_channels=False)

These methods let Python users read one native tile or a rectangular tile range without loading full pixel data.

### 2. Added runtime validation for safe tile access
The new tile-read path validates:
- file is opened from a filename
- part index is valid
- dx1 <= dx2 and dy1 <= dy2
- requested level exists
- tile range is inside level bounds
- part is a regular tiled image

This gives clearer failures for invalid tile requests.

### 3. Added packed and separate channel output modes
The new APIs support:
- packed mode (default): RGB or RGBA arrays when channels are compatible
- separate mode: independent channel arrays when separate_channels=True

This matches existing user expectations while enabling more explicit workflows.

### 4. Exposed OpenEXR internal thread controls to Python
Two module functions were added:
- setGlobalThreadCount(count)
- globalThreadCount()

The module also initializes internal thread count to 0 at import, meaning chunk compression/decompression runs on the calling thread unless changed.

### 5. Added automated unit test coverage
A new test validates:
- shape of single-tile output
- shape of tile-range output
- value correctness against full-image reads
- separate-channel behavior for tile reads

This confirms correctness of tile-region reads and channel layout behavior.

## Example Code:
```python
import OpenEXR
import numpy as np

# 1) Read a single tile (packed RGBA by default)
with OpenEXR.File("tiled.exr", header_only=True) as f:
    one_tile = f.readTile(1, 1)
    rgba_tile = one_tile["RGBA"].pixels
    print("single tile shape:", rgba_tile.shape)

# 2) Read a tile rectangle (inclusive tile coordinates)
with OpenEXR.File("tiled.exr", header_only=True) as f:
    tile_block = f.readTiles(0, 1, 0, 1)
    rgba_block = tile_block["RGBA"].pixels
    print("tile block shape:", rgba_block.shape)

# 3) Read separate channels
with OpenEXR.File("tiled.exr", header_only=True) as f:
    separate = f.readTile(0, 0, separate_channels=True)
    print("channels:", sorted(separate.keys()))
    r = separate["R"].pixels
    g = separate["G"].pixels
    b = separate["B"].pixels
    a = separate["A"].pixels
    print("R shape:", r.shape)

# 4) Compare a tile read with a full read slice
with OpenEXR.File("tiled.exr") as f:
    full = f.channels()["RGBA"].pixels

with OpenEXR.File("tiled.exr", header_only=True) as f:
    tile = f.readTile(1, 1)["RGBA"].pixels

# Example comparison if tile size is 32x32 and tile index is (1,1)
# adjust slicing based on your file tile dimensions
print("matches expected slice:", np.array_equal(tile, full[32:64, 32:64]))

# 5) Control OpenEXR internal worker threads
print("thread count before:", OpenEXR.globalThreadCount())
OpenEXR.setGlobalThreadCount(4)
print("thread count after:", OpenEXR.globalThreadCount())
```


# OpenEXR repo README:
___

[![License](https://img.shields.io/github/license/AcademySoftwareFoundation/openexr)](LICENSE.md)
[![CII Best Practices](https://bestpractices.coreinfrastructure.org/projects/2799/badge)](https://bestpractices.coreinfrastructure.org/projects/2799)
[![OpenSSF Scorecard](https://api.securityscorecards.dev/projects/github.com/AcademySoftwareFoundation/openexr/badge)](https://securityscorecards.dev/viewer/?uri=github.com/AcademySoftwareFoundation/openexr)
[![Build Status](https://github.com/AcademySoftwareFoundation/openexr/workflows/CI/badge.svg)](https://github.com/AcademySoftwareFoundation/openexr/actions?query=workflow%3ACI)
[![Analysis Status](https://github.com/AcademySoftwareFoundation/openexr/workflows/Analysis/badge.svg)](https://github.com/AcademySoftwareFoundation/openexr/actions?query=workflow%3AAnalysis)
[![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=AcademySoftwareFoundation_openexr&metric=alert_status)](https://sonarcloud.io/dashboard?id=AcademySoftwareFoundation_openexr)

# OpenEXR

<img align="right" src="docs/technical/images/windowExample1.png">

OpenEXR provides the specification and reference implementation of the
EXR file format, the professional-grade image storage format of the
motion picture industry.

The purpose of EXR format is to accurately and efficiently represent
high-dynamic-range scene-linear image data and associated metadata,
with strong support for multi-part, multi-channel use cases.

OpenEXR is widely used in host application software where accuracy is
critical, such as photorealistic rendering, texture access, image
compositing, deep compositing, and DI.

## OpenEXR Project Mission

The goal of the OpenEXR project is to keep the EXR format reliable and
modern and to maintain its place as the preferred image format for
entertainment content creation. 

Major revisions are infrequent, and new features will be carefully
weighed against increased complexity.  The principal priorities of the
project are:

* Robustness, reliability, security
* Backwards compatibility, data longevity
* Performance - read/write/compression/decompression time
* Simplicity, ease of use, maintainability
* Wide adoption, multi-platform support - Linux, Windows, macOS, and others

OpenEXR is intended solely for 2D data. It is not appropriate for
storage of volumetric data, cached or lit 3D scenes, or more complex
3D data such as light fields.

The goals of the Imath project are simplicity, ease of use,
correctness and verifiability, and breadth of adoption. Imath is not
intended to be a comprehensive linear algebra or numerical analysis
package.

## Project Governance

OpenEXR is a project of the [Academy Software
Foundation](https://www.aswf.io). See the project's [governance
policies](GOVERNANCE.md), [contribution guidelines](CONTRIBUTING.md), and [code of conduct](CODE_OF_CONDUCT)
for more information.

# Building OpenEXR

See the [Install instructions](https://openexr.com/en/latest/install.html) for instructions on how to build
OpenEXR and its required prerequisites.

# Quick Start

See the [technical documentation](https://openexr.readthedocs.io) for
complete details, but to get started, the "Hello, world" [`exrwriter.cpp`](https://raw.githubusercontent.com/AcademySoftwareFoundation/openexr/main/website/src/exrwriter/exrwriter.cpp) writer program is:

    #include <ImfRgbaFile.h>
    #include <ImfArray.h>
    #include <iostream>

    using namespace OPENEXR_IMF_NAMESPACE;

    int
    main()
    {
        int width =  100;
        int height = 50;

        Array2D<Rgba> pixels(height, width);
        for (int y=0; y<height; y++)
        {
            float c = (y / 5 % 2 == 0) ? (y / (float) height) : 0.0;
            for (int x=0; x<width; x++)
                pixels[y][x] = Rgba(c, c, c);
        }

        try {
            RgbaOutputFile file ("stripes.exr", width, height, WRITE_RGBA);
            file.setFrameBuffer (&pixels[0][0], 1, width);
            file.writePixels (height);
        } catch (const std::exception &e) {
            std::cerr << "error writing image file stripes.exr:" << e.what() << std::endl;
            return 1;
        }
        return 0;
    }

This creates an image 100 pixels wide and 50 pixels high with
horizontal stripes 5 pixels high of graduated intensity, bright on the
bottom of the image and dark towards the top. Note that ``pixel[0][0]``
is in the upper left:

![stripes](website/images/stripes.png)

The [`CMakeLists.txt`](https://raw.githubusercontent.com/AcademySoftwareFoundation/openexr/main/website/src/exrwriter/CMakeLists.txt) to build:

    cmake_minimum_required(VERSION 3.12)
    project(exrwriter)
    find_package(OpenEXR REQUIRED)

    add_executable(${PROJECT_NAME} exrwriter.cpp)
    target_link_libraries(${PROJECT_NAME} OpenEXR::OpenEXR)

To build:

    $ cmake -S . -B _build -DCMAKE_PREFIX_PATH=<path to OpenEXR libraries/includes>
    $ cmake --build _build

For more details, see [The OpenEXR
API](https://openexr.readthedocs.io/en/latest/API.html#the-openexr-api).

# Community

* **Ask a question:**

  - Email: openexr-dev@lists.aswf.io

  - Slack: [academysoftwarefdn#openexr](https://academysoftwarefdn.slack.com/archives/CMLRW4N73)

* **Attend a meeting:**

  - Technical Steering Committee meetings are open to the
    public, fortnightly on Thursdays, 1:30pm Pacific Time.

  - Calendar: https://zoom-lfx.platform.linuxfoundation.org/meetings/openexr

  - Meeting Notes: https://wiki.aswf.io/display/OEXR/TSC+Meetings

* **Report a bug:**

  - Submit an Issue: https://github.com/AcademySoftwareFoundation/openexr/issues

* **Report a security vulnerability:**

  - File a GitHub [security
    advisory](https://github.com/AcademySoftwareFoundation/openexr/security/advisories/new).
    Email security@openexr.com for private/secure discussion with the project
    maintainers.

* **Contribute a Fix, Feature, or Improvement:**

  - Read the [Contribution Guidelines](CONTRIBUTING.md) and [Code of Conduct](CODE_OF_CONDUCT.md)

  - Sign the [Contributor License
    Agreement](https://contributor.easycla.lfx.linuxfoundation.org/#/cla/project/2e8710cb-e379-4116-a9ba-964f83618cc5/user/564e571e-12d7-4857-abd4-898939accdd7)

  - Submit a Pull Request: https://github.com/AcademySoftwareFoundation/openexr/pulls

  - If you'd like to contribute and could use some ideas of what to
    do, browse "good first issues" [here](https://github.com/AcademySoftwareFoundation/openexr/contribute) or on [Clotributor](https://clotributor.dev/search?ts_query_web=openexr&page=1).

# Resources

- Website: http://www.openexr.com
- Technical documentation: https://openexr.readthedocs.io
- Porting help: [OpenEXR/Imath Version 2.x to 3.x Porting Guide](https://openexr.readthedocs.io/en/latest/PortingGuide.html)
- Reference images: https://github.com/AcademySoftwareFoundation/openexr-images
- Security policy: [SECURITY.md](SECURITY.md)
- Release notes: [CHANGES.md](CHANGES.md)
- Contributors: [CONTRIBUTORS.md](CONTRIBUTORS.md)  

# License

OpenEXR is licensed under the [BSD-3-Clause license](LICENSE.md).


---

![aswf](/ASWF/images/aswf.png)
