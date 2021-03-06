cmake_minimum_required (VERSION 2.8)
project (mraa C CXX)

FIND_PACKAGE (Threads REQUIRED)

###############################################################################
# Detect supported warning flags
# Modified from work By Dan Liew (fpbench - MIT)
# https://github.com/delcypher/fp-bench/blob/master/CMakeLists.txt
###############################################################################
# Warnings for both the C and C++ compiler

set (MRAA_BOTH_WARNING_FLAGS
  -Wall
  -Werror=main
  -Wformat
  -Wmain
  -Wuninitialized
  -Winit-self
)

# Warning flags for the C compiler only
set (MRAA_C_WARNING_FLAGS
  -Werror=implicit
  -Werror=missing-parameter-type
)

# Warning flags for the C++ compiler only
set (MRAA_CXX_WARNING_FLAGS
  -Wnon-virtual-dtor
  -Woverloaded-virtual
  -Wreorder
)

include (CheckCCompilerFlag)
include (CheckCXXCompilerFlag)
function (MRAA_SANITIZE_FLAG_NAME OUTPUT_VAR FLAG)
  string (REPLACE "-" "_" SANITIZED_FLAG_NAME "${FLAG}")
  string (REPLACE "/" "_" SANITIZED_FLAG_NAME "${SANITIZED_FLAG_NAME}")
  string (REPLACE "=" "_" SANITIZED_FLAG_NAME "${SANITIZED_FLAG_NAME}")
  string (REPLACE " " "_" SANITIZED_FLAG_NAME "${SANITIZED_FLAG_NAME}")
  set (${OUTPUT_VAR} "${SANITIZED_FLAG_NAME}" PARENT_SCOPE)
endfunction ()

# Globally set C compiler warning flags that are supported and emit
# a warning about unsupported flags
foreach (flag ${MRAA_BOTH_WARNING_FLAGS} ${MRAA_C_WARNING_FLAGS})
  MRAA_SANITIZE_FLAG_NAME(SANITIZED_FLAG_NAME "${flag}")
  CHECK_C_COMPILER_FLAG("${flag}" HAS_C_${SANITIZED_FLAG_NAME})
  if (HAS_C_${SANITIZED_FLAG_NAME})
    set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${flag}")
  else ()
    message (WARNING "C compiler does not support flag \"${flag}\"")
  endif ()
endforeach ()

# Globally set C++ compiler warning flags that are supported and emit
# a warning about unsupported flags
foreach (flag ${MRAA_BOTH_WARNING_FLAGS} ${MRAA_CXX_WARNING_FLAGS})
  MRAA_SANITIZE_FLAG_NAME (SANITIZED_FLAG_NAME "${flag}")
  CHECK_CXX_COMPILER_FLAG ("${flag}" HAS_CXX_${SANITIZED_FLAG_NAME})
  if (HAS_CXX_${SANITIZED_FLAG_NAME})
    set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${flag}")
  else ()
    message (WARNING "C++ compiler does not support flag \"${flag}\"")
  endif ()
endforeach ()

# Set CMAKE_INSTALL_LIBDIR if not defined
include(GNUInstallDirs)

# Older cmake might not pick CMAKE_INSTALL_LIBDIR right
if (CMAKE_INSTALL_LIBDIR)
  set (LIB_INSTALL_DIR "${CMAKE_INSTALL_LIBDIR}" CACHE PATH "Installation path for libraries")
else ()
  set (LIB_INSTALL_DIR "lib" CACHE PATH "Installation path for libraries")
endif ()

# By default, build shared object libraries on linux
if (UNIX AND NOT APPLE)
  if (NOT DEFINED BUILD_SHARED_LIBS)
    set(BUILD_SHARED_LIBS ON)
  endif()
endif()

# Appends the cmake/modules path to MAKE_MODULE_PATH variable.
set (CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules ${CMAKE_MODULE_PATH})

# Make a version file containing the current version from git.
include (GetGitRevisionDescription)
git_describe (VERSION "--tags")
if ("x_${VERSION}" STREQUAL "x_GIT-NOTFOUND" OR "x_${VERSION}" STREQUAL "x_HEAD-HASH-NOTFOUND" OR "x_${VERSION}" STREQUAL "x_-128-NOTFOUND")
  message (WARNING " - Install git to compile a production libmraa!")
  set (VERSION "v1.4.0")
endif ()

message (STATUS "INFO - libmraa Version ${VERSION}")
message (STATUS "INFO - cmake Version ${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION}.${CMAKE_PATCH_VERSION}")

#parse the version information into pieces.
string (REGEX REPLACE "^v([0-9]+)\\..*" "\\1" VERSION_MAJOR "${VERSION}")
string (REGEX REPLACE "^v[0-9]+\\.([0-9]+).*" "\\1" VERSION_MINOR "${VERSION}")
string (REGEX REPLACE "^v[0-9]+\\.[0-9]+\\.([0-9]+).*" "\\1" VERSION_PATCH "${VERSION}")
string (REGEX REPLACE "^v[0-9]+\\.[0-9]+\\.[0-9]+\\-([0-9]+).*" "\\1" VERSION_COMMIT "${VERSION}")
string (REGEX REPLACE "^v[0-9]+\\.[0-9]+\\.[0-9]+-[0-9]+\\-(.*)" "\\1" VERSION_SHA1 "${VERSION}")
set (VERSION_SHORT "${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}")

if ("${VERSION_COMMIT}" MATCHES "^v.*")
  set (VERSION_COMMIT "")
endif()

configure_file (${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules/version.c.in
                ${CMAKE_CURRENT_BINARY_DIR}/src/version.c)

# this is the library version, independant of git revision
set (mraa_VERSION_MAJOR ${VERSION_MAJOR})
set (mraa_VERSION_MINOR ${VERSION_MINOR})
set (mraa_VERSION_PATCH ${VERSION_PATCH})
set (mraa_VERSION_STRING ${mraa_VERSION_MAJOR}.${mraa_VERSION_MINOR}.${mraa_VERSION_PATCH})

set (CMAKE_SWIG_FLAGS "")

find_path (SYSTEM_USR_DIR "stdlib.h")
include_directories (${SYSTEM_USR_DIR})

option (BUILDDOC "Build all doc." OFF)
option (BUILDSWIG "Build swig modules." ON)
option (BUILDSWIGPYTHON "Build swig python modules." ON)
option (BUILDSWIGNODE "Build swig node modules." ON)
option (BUILDSWIGJAVA "Build Java API." OFF)
option (USBPLAT "Detection USB platform." OFF)
option (FIRMATA "Add Firmata support to mraa." OFF)
option (ONEWIRE "Add Onewire support to mraa." ON)
option (JSONPLAT "Add Platform loading via a json file." ON)
option (IMRAA "Add Imraa support to mraa." OFF)
option (FTDI4222 "Build with FTDI FT4222 subplatform support." OFF)
option (IPK "Generate IPK using CPack" OFF)
option (RPM "Generate RPM using CPack" OFF)
option (ENABLEEXAMPLES "Disable building of examples" ON)
option (INSTALLGPIOTOOL "Install gpio tool" OFF)
option (INSTALLTOOLS "Install all tools" OFF)
option (BUILDTESTS "Override the addition of tests" ON)

set (BUILDARCH "" CACHE STRING "Override architecture to build for")

set (MRAAPLATFORMFORCE "ALL" CACHE STRING "Override platform to build for")

if (NOT BUILDSWIG)
  set (BUILDSWIGPYTHON OFF)
  set (BUILDSWIGNODE OFF)
  set (BUILDSWIGJAVA OFF)
endif()

if (NOT BUILDARCH)
  include (TargetArch)
  target_architecture (DETECTED_ARCH)
  message (STATUS "INFO - Target arch is ${DETECTED_ARCH}")
else ()
  set (DETECTED_ARCH ${BUILDARCH})
  message (STATUS "INFO - Override arch is ${DETECTED_ARCH}")
endif()

if (DETECTED_ARCH STREQUAL "i586" OR DETECTED_ARCH STREQUAL "x86_64"
    OR DETECTED_ARCH STREQUAL "i386")
  set (X86PLAT ON)
elseif (DETECTED_ARCH MATCHES "arm.*")
  set (ARMPLAT ON)
elseif (DETECTED_ARCH STREQUAL "MOCK")
  set (MOCKPLAT ON)
else ()
  message (FATAL_ERROR "Only x86, arm and mock platforms currently supported")
endif()

if (BUILDSWIGPYTHON OR BUILDTESTS)
  include (cmake/modules/OpenCVDetectPython.cmake)
endif ()

if (BUILDDOC)
  # add a target to generate API documentation with Doxygen
  find_package (Doxygen)
  if (DOXYGEN_FOUND)
    configure_file (${CMAKE_CURRENT_SOURCE_DIR}/Doxyfile.in ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile @ONLY)
    if (BUILDSWIGJAVA)
      configure_file (${CMAKE_CURRENT_SOURCE_DIR}/Doxyfile.java.in ${CMAKE_CURRENT_BINARY_DIR}/src/java/Doxyfile @ONLY)
    endif ()
    add_custom_target (doc
      ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile
      WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
      COMMENT "Generating API documentation with Doxygen" VERBATIM
    )
  endif (DOXYGEN_FOUND)
endif ()

if (IPK)
  # Get target package arch from Yocto ADT sysroot if set or host OS, mapping to Ubuntu name if necessary
  if (DEFINED ENV{OECORE_TARGET_SYSROOT})
    GET_FILENAME_COMPONENT (DETECTED_SYSROOT $ENV{OECORE_TARGET_SYSROOT} NAME)
    string (REGEX REPLACE "-poky-linux" "" TARGET_ARCH "${DETECTED_SYSROOT}")
  else ()
    # debian uses amd64 to denote x86_64
    if (DETECTED_ARCH STREQUAL "x86_64")
      set (TARGET_ARCH "amd64")
    else ()
      set (TARGET_ARCH ${DETECTED_ARCH})
    endif ()
  endif ()
  message (STATUS "INFO - Package arch is ${TARGET_ARCH}")

  set(CPACK_GENERATOR "DEB")
  set(OPKG_ARCH ${TARGET_ARCH})
  set(CPACK_BINARY_DIR ${CMAKE_BINARY_DIR})
  set(CPACK_DEBIAN_PACKAGE_MAINTAINER "Intel IoT-Devkit") #required
  set(mraa_PACKAGE_ON_TAG ".")
  if ("${VERSION_COMMIT}" STREQUAL "")
    set(mraa_PACKAGE_ON_TAG "")
  endif()
  set(CPACK_PACKAGE_VERSION
      "${mraa_VERSION_MAJOR}.${mraa_VERSION_MINOR}.${mraa_VERSION_PATCH}${mraa_PACKAGE_ON_TAG}${VERSION_COMMIT}")
  set(CPACK_PACKAGE_NAME "mraa")
  set(CPACK_DEBIAN_PACKAGE_SECTION "libs")
  set(CPACK_DEBIAN_PACKAGE_ARCHITECTURE ${TARGET_ARCH})
  set(CPACK_SYSTEM_NAME ${TARGET_ARCH})
  set(CPACK_DEBIAN_PACKAGE_PROVIDES "mraa-dev, mraa-dbg, mraa-doc")
  set(CPACK_DEBIAN_PACKAGE_REPLACES "${CPACK_DEBIAN_PACKAGE_PROVIDES}, libmraa, libmraa-dev, libmraa-doc")
  set(CPACK_DEBIAN_PACKAGE_CONFLICTS ${CPACK_DEBIAN_PACKAGE_PROVIDES})
  set(WDIR "${CPACK_TOPLEVEL_DIRECTORY}/${CPACK_PACKAGE_FILE_NAME}")
  include (CPack)
endif()

if (RPM)
  message (STATUS "INFO - Enabled RPM packaging for ${DETECTED_ARCH}")
  set(CMAKE_INSTALL_PREFIX "/usr")
  set(CPACK_PACKAGE_VERSION ${VERSION})
  set(CPACK_GENERATOR "RPM")
  set(CPACK_PACKAGE_NAME "libmraa${mraa_VERSION_MAJOR}")
  set(CPACK_PACKAGE_RELEASE 1)
  set(CPACK_PACKAGE_VERSION
      "${mraa_VERSION_MAJOR}.${mraa_VERSION_MINOR}.${mraa_VERSION_PATCH}${mraa_PACKAGE_ON_TAG}${VERSION_COMMIT}")
  set(CPACK_PACKAGE_CONTACT "Intel IoT-Devkit")
  set(CPACK_PACKAGE_VENDOR "Intel IoT-Devkit")
  set(CPACK_RPM_PACKAGE_PROVIDES "${CPACK_PACKAGE_NAME}-devel")
  # Get distro tag (e.g. 'fc20') by parsing output of rpm --showrc
  EXECUTE_PROCESS(
    COMMAND rpm --showrc
    COMMAND grep -w dist
    COMMAND sed -e "s/\\t./ /"
    COMMAND awk "{printf \"%s\", \$NF}"
    OUTPUT_VARIABLE DIST_TAG
  )
  set(CPACK_PACKAGE_FILE_NAME "${CPACK_PACKAGE_NAME}-${CPACK_PACKAGE_VERSION}-${CPACK_PACKAGE_RELEASE}.${DIST_TAG}.${DETECTED_ARCH}")
  include(CPack)
endif()

add_subdirectory (src)
if (ENABLEEXAMPLES)
  add_subdirectory (examples)
endif ()

if (IMRAA)
  add_subdirectory (imraa)
endif ()

if (BUILDTESTS AND PYTHON_DEFAULT_EXECUTABLE)
  enable_testing ()
  add_subdirectory (tests)
endif ()
