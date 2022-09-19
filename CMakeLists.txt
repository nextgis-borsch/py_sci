################################################################################
# Project:  Lib scipy
# Purpose:  CMake build scripts
# Author:   Dmitry Baryshnikov, dmitry.baryshnikov@nexgis.com
################################################################################
# Copyright (C) 2018, NextGIS <info@nextgis.com>
#
# Permission is hereby granted, free of charge, to any person obtaining a
# copy of this software and associated documentation files (the "Software"),
# to deal in the Software without restriction, including without limitation
# the rights to use, copy, modify, merge, publish, distribute, sublicense,
# and/or sell copies of the Software, and to permit persons to whom the
# Software is furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included
# in all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS
# OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
# THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
# FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
# DEALINGS IN THE SOFTWARE.
################################################################################
cmake_minimum_required(VERSION 2.8.12 FATAL_ERROR)

project(scipy)

if(NOT DEFINED PACKAGE_VENDOR)
    set(PACKAGE_VENDOR NextGIS)
endif()

if(NOT DEFINED PACKAGE_BUGREPORT)
    set(PACKAGE_BUGREPORT info@nextgis.com)
endif()


# some init settings
set(CMAKE_COLOR_MAKEFILE ON)
# set path to additional CMake modules
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

if(OSX_FRAMEWORK AND (BUILD_SHARED_LIBS OR BUILD_STATIC_LIBS))
  message(FATAL_ERROR "Only OSX_FRAMEWORK key or any or both BUILD_SHARED_LIBS
                       and BUILD_STATIC_LIBS keys are permitted")
endif()

if(OSX_FRAMEWORK)
    set(INSTALL_LIB_DIR "Library" CACHE INTERNAL "Installation directory for libraries" FORCE)
    set(CMAKE_MACOSX_RPATH ON)
else()
    set(INSTALL_LIB_DIR "." CACHE INTERNAL "Installation directory for libraries" FORCE)
endif()

find_package(Python3 COMPONENTS Interpreter)

if(NOT Python3_FOUND)
    message(FATAL_ERROR "Cannot find python")
endif()

set(PYTHON_EXECUTABLE ${Python3_EXECUTABLE})

file(DOWNLOAD "https://pypi.python.org/pypi/${PROJECT_NAME}/json" "${CMAKE_CURRENT_BINARY_DIR}/meta.json")

include(util)
get_compiler_version(COMPILER)
# get version and download url
execute_process(COMMAND ${PYTHON_EXECUTABLE}
    ${CMAKE_SOURCE_DIR}/cmake/pypi_process.py ${PLUGIN} ${CMAKE_CURRENT_BINARY_DIR}/meta.json ${COMPILER} ${LATEST_AVAILABLE_VERSION}
    OUTPUT_VARIABLE PACKAGE_DETAILS
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

list(GET PACKAGE_DETAILS 0 DOWNLOAD_URL)
list(GET PACKAGE_DETAILS 1 VERSION)
list(GET PACKAGE_DETAILS 2 FULL_VERSION)
list(GET PACKAGE_DETAILS 3 PROJECT_CPACK_FILENAME)

report_version(${PROJECT_NAME} ${VERSION})

if(OSX_FRAMEWORK)
    set(FLAGS "ARCHFLAGS=-arch x86_64") # -static -static-libgfortran -static-libgcc
elseif(WIN32)
    if(MSVC_VERSION EQUAL 1400)
        set(FLAGS ${FLAGS} "VS90COMNTOOLS=$ENV{VS80COMNTOOLS}")
    elseif(MSVC_VERSION EQUAL 1500)
        set(FLAGS ${FLAGS} "VS90COMNTOOLS=$ENV{VS90COMNTOOLS}")
    elseif(MSVC_VERSION EQUAL 1600)
        set(FLAGS ${FLAGS} "VS90COMNTOOLS=$ENV{VS100COMNTOOLS}")
    elseif(MSVC_VERSION EQUAL 1700)
        set(FLAGS ${FLAGS} "VS90COMNTOOLS=$ENV{VS110COMNTOOLS}")
    elseif(MSVC_VERSION EQUAL 1800)
        set(FLAGS ${FLAGS} "VS90COMNTOOLS=$ENV{VS120COMNTOOLS}")
    elseif(MSVC_VERSION EQUAL 1900)
        set(FLAGS ${FLAGS} "VS90COMNTOOLS=$ENV{VS140COMNTOOLS}")
    else() # 1912 - Need to copy vcvarsall.bat from VC\Auxiliary\Build to VC
        get_filename_component(_COMNTOOLS_PREFIX "${CMAKE_C_COMPILER}" PATH)
        get_filename_component(_COMNTOOLS_PREFIX "${_COMNTOOLS_PREFIX}" PATH)
        get_filename_component(_COMNTOOLS_PREFIX "${_COMNTOOLS_PREFIX}" PATH)
        get_filename_component(_COMNTOOLS_PREFIX "${_COMNTOOLS_PREFIX}" PATH)
        get_filename_component(_COMNTOOLS_PREFIX "${_COMNTOOLS_PREFIX}" PATH)
        get_filename_component(_COMNTOOLS_PREFIX "${_COMNTOOLS_PREFIX}" PATH)
        # get_filename_component(_COMNTOOLS_PREFIX "${_COMNTOOLS_PREFIX}" PATH)
        set(FLAGS ${FLAGS} "VS90COMNTOOLS=${_COMNTOOLS_PREFIX}")
    endif()
else()
    set(FLAGS "NextGIS=TRUE")
endif()

# Download tarbal
set(OUT_DIR_NAME ${PROJECT_NAME}-${FULL_VERSION})
set(OUT_NAME "${OUT_DIR_NAME}.tar.gz")

if(NOT EXISTS "${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}")
    status_message("Download ${DOWNLOAD_URL}")
    file(DOWNLOAD "${DOWNLOAD_URL}" "${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}"
        SHOW_PROGRESS
    )
endif()

# Extract archive
if(NOT EXISTS ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME})
    status_message("Extract ${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}")
    execute_process(COMMAND ${CMAKE_COMMAND} -E tar xzf ${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME})
endif()

include(FindAnyProject)
find_anyproject(Openblas REQUIRED)

if(WIN32)
    set(BUILD_INCLUDES --compiler=msvc --fcompiler=flang --library-dirs="C:\\Users\\root\\conda\\Library\\lib" --libraries=libflang,libpgmath)
    
    set(OPENBLAS_INCLUDE_DIRS ${OPENBLAS_INCLUDE_DIRS})
    get_target_property(OPENBLAS_LIBRARY_DIRS openblas IMPORTED_IMPLIB_RELEASE)
    get_filename_component(OPENBLAS_LIBRARY_DIRS ${OPENBLAS_LIBRARY_DIRS} DIRECTORY)
    
    configure_file(${CMAKE_SOURCE_DIR}/cmake/site.cfg.in ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}/site.cfg IMMEDIATE @ONLY)
endif()

# python setup.py install --prefix=/usr/local

macro(build_py)
    status_message("Configuring ... ${FLAGS} ${PYTHON_EXECUTABLE} setup.py build")
    execute_process(COMMAND ${CMAKE_COMMAND} -E env "${FLAGS}"
        ${PYTHON_EXECUTABLE} setup.py build_ext ${BUILD_INCLUDES}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}
        #OUTPUT_QUIET ERROR_QUIET
    )

    add_custom_target(build_py ALL
        COMMAND ${CMAKE_COMMAND} -E env "${FLAGS}"
        ${PYTHON_EXECUTABLE} setup.py build_ext ${BUILD_INCLUDES}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}
        VERBATIM
    )

    if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
        set(SETUP_OPTS ${SETUP_OPTS} --no-compile --single-version-externally-managed --record=rekord.txt --prefix=${CMAKE_CURRENT_BINARY_DIR}/inst_tmp)
    elseif(WIN32)
        string(REPLACE "/" "\\\\" TMP_PATH "${CMAKE_CURRENT_BINARY_DIR}/inst_tmp")
        set(SETUP_OPTS ${SETUP_OPTS} --no-compile --single-version-externally-managed --record=rekord.txt --prefix=${TMP_PATH})
    else()
        execute_process ( COMMAND ${PYTHON_EXECUTABLE} -c "from distutils.sysconfig import get_python_lib; print(get_python_lib())" OUTPUT_VARIABLE PY_SITE_PACKAGES OUTPUT_STRIP_TRAILING_WHITESPACE)

        set(TEST_CMD "try:
    import setuptools
    print(1)
except ImportError:
    pass")

        execute_process ( COMMAND ${PYTHON_EXECUTABLE} -c ${TEST_CMD} OUTPUT_VARIABLE PY_HAVE_SETUPTOOLS OUTPUT_STRIP_TRAILING_WHITESPACE)
        if(PY_HAVE_SETUPTOOLS)
            set(SETUP_OPTS "${SETUP_OPTS} --no-compile --single-version-externally-managed --record=rekord.txt")
        endif()

        string( LENGTH "$ENV{DESTDIR}" DESTDIRLEN )
        if ( ${DESTDIRLEN} GREATER 0 )
            set(SETUP_OPTS "${SETUP_OPTS} --root=$ENV{DESTDIR}")
        elseif(PYTHON_DESTDIR)
            set(SETUP_OPTS "${SETUP_OPTS} --root=${PYTHON_DESTDIR}")
        endif()

        # check debian
        string(FIND ${PY_SITE_PACKAGES} "dist-packages" PY_DIST_POS)
        if(${PY_DIST_POS} GREATER -1)
            set(SETUP_OPTS "${SETUP_OPTS} --install-layout=deb")
        endif()

        # TODO: do we need --install-lib=$(PYTHON_INSTALL_LIB) ?

        set(SETUP_OPTS "${SETUP_OPTS} --prefix=${CMAKE_CURRENT_BINARY_DIR}/inst_tmp")

    endif()
    add_custom_target(install_py ALL
        COMMAND ${CMAKE_COMMAND} -E env "${FLAGS}"
        ${PYTHON_EXECUTABLE} setup.py install ${SETUP_OPTS}
        DEPENDS build_py
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}
        VERBATIM
    )
endmacro()

build_py()

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL )
    install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/inst_tmp/ DESTINATION ${INSTALL_LIB_DIR})
endif()

# Archiving ====================================================================
set(PACKAGE_NAME ${PROJECT_NAME})

set(CPACK_PACKAGE_NAME "${PACKAGE_NAME}")
set(CPACK_PACKAGE_VENDOR "${PACKAGE_VENDOR}")
set(CPACK_PACKAGE_VERSION "${VERSION}")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "${PACKAGE_NAME} Installation")
set(CPACK_PACKAGE_RELOCATABLE TRUE)
set(CPACK_ARCHIVE_COMPONENT_INSTALL ON)
set(CPACK_GENERATOR "ZIP")
set(CPACK_MONOLITHIC_INSTALL ON)
set(CPACK_STRIP_FILES TRUE)

# Get cpack zip archive name
set(CPACK_PACKAGE_FILE_NAME ${PROJECT_CPACK_FILENAME})

include(CPack)
