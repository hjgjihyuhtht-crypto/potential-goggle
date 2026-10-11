# Kinect 3D Studio — código nativo (Etapa 2: comunicação USB com Kinect v1)
cmake_minimum_required(VERSION 3.22.1)
project(kinect3dstudio C CXX)

set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(TP ${CMAKE_CURRENT_SOURCE_DIR}/third_party)

# ---------------------------------------------------------------- libusb (LGPL-2.1)
# Compilado como biblioteca compartilhada separada (libusb1.0.so) para cumprir a
# LGPL: o usuário pode substituir a .so sem recompilar o restante do app.
set(LIBUSB_DIR ${TP}/libusb)
add_library(usb1.0 SHARED
    ${LIBUSB_DIR}/libusb/core.c
    ${LIBUSB_DIR}/libusb/descriptor.c
    ${LIBUSB_DIR}/libusb/hotplug.c
    ${LIBUSB_DIR}/libusb/io.c
    ${LIBUSB_DIR}/libusb/sync.c
    ${LIBUSB_DIR}/libusb/strerror.c
    ${LIBUSB_DIR}/libusb/os/linux_usbfs.c
    ${LIBUSB_DIR}/libusb/os/events_posix.c
    ${LIBUSB_DIR}/libusb/os/threads_posix.c
    ${LIBUSB_DIR}/libusb/os/linux_netlink.c)
if(ANDROID)
    target_include_directories(usb1.0 PRIVATE ${LIBUSB_DIR}/android)
else()
    # Compilação de verificação no host Linux (scripts/check_native_host.sh)
    target_include_directories(usb1.0 PRIVATE ${K3D_HOST_LIBUSB_CONFIG_DIR})
endif()
target_include_directories(usb1.0
    PRIVATE ${LIBUSB_DIR}/libusb/os
    PUBLIC  ${LIBUSB_DIR}/libusb)
target_compile_options(usb1.0 PRIVATE -fvisibility=hidden -pthread -Wno-unused-parameter)
if(ANDROID)
    target_link_libraries(usb1.0 PRIVATE log)
else()
    target_link_libraries(usb1.0 PRIVATE pthread)
endif()

# ---------------------------------------------------------- libfreenect (Apache-2.0)
set(FN_DIR ${TP}/libfreenect)
add_library(freenect_static STATIC
    ${FN_DIR}/src/core.c
    ${FN_DIR}/src/tilt.c
    ${FN_DIR}/src/cameras.c
    ${FN_DIR}/src/flags.c
    ${FN_DIR}/src/usb_libusb10.c
    ${FN_DIR}/src/registration.c
    ${FN_DIR}/src/audio.c
    ${FN_DIR}/src/loader.c
    usb_bridge/freenect_android_open.c)
set_target_properties(freenect_static PROPERTIES POSITION_INDEPENDENT_CODE ON)
target_include_directories(freenect_static
    PUBLIC ${FN_DIR}/include ${CMAKE_CURRENT_SOURCE_DIR}/usb_bridge
    PRIVATE ${FN_DIR}/src)
target_compile_options(freenect_static PRIVATE -Wno-unused-parameter -Wno-sign-compare)
target_link_libraries(freenect_static PUBLIC usb1.0 m)

# ------------------------------------------------------------- ponte JNI do app
# --------------------------------------------- sensores (puro C++, testável no host)
add_library(k3d_sensors STATIC
    sensors/calibration.cpp
    sensors/colorize.cpp
    sensors/recording.cpp
    sensors/frame_source.cpp
    sensors/playback_source.cpp)
set_target_properties(k3d_sensors PROPERTIES POSITION_INDEPENDENT_CODE ON)
target_include_directories(k3d_sensors PUBLIC sensors)
target_compile_options(k3d_sensors PRIVATE -Wall -Wextra -Wno-unused-parameter -O2)
target_link_libraries(k3d_sensors PUBLIC freenect_static)

add_library(k3d_native SHARED
    usb_bridge/kinect_session.cpp
    usb_bridge/jni_bridge.cpp)
target_compile_options(k3d_native PRIVATE -Wall -Wextra -Wno-unused-parameter)
target_link_libraries(k3d_native PRIVATE k3d_sensors freenect_static)
if(ANDROID)
    target_sources(k3d_native PRIVATE usb_bridge/jni_bitmap.cpp)
    target_link_libraries(k3d_native PRIVATE log jnigraphics)
    # Páginas de 16 KB (Android 15+)
    target_link_options(k3d_native PRIVATE "-Wl,-z,max-page-size=16384")
    target_link_options(usb1.0 PRIVATE "-Wl,-z,max-page-size=16384")
else()
    find_package(JNI REQUIRED)
    target_include_directories(k3d_native PRIVATE ${JNI_INCLUDE_DIRS})
endif()

# ----------------------------------------------- testes nativos (somente host)
if(NOT ANDROID)
    enable_testing()
    add_executable(session_smoke_test tests/session_smoke_test.cpp usb_bridge/kinect_session.cpp)
    target_include_directories(session_smoke_test PRIVATE usb_bridge)
    target_link_libraries(session_smoke_test PRIVATE k3d_sensors freenect_static)
    add_test(NAME session_smoke_test COMMAND session_smoke_test)
    set_tests_properties(session_smoke_test PROPERTIES SKIP_RETURN_CODE 77)

    add_executable(sensors_test tests/sensors_test.cpp)
    target_link_libraries(sensors_test PRIVATE k3d_sensors)
    add_test(NAME sensors_test COMMAND sensors_test)
endif()
