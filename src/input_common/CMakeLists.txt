# SPDX-FileCopyrightText: Copyright 2026 Eden Emulator Project
# SPDX-License-Identifier: GPL-3.0-or-later

# SPDX-License-Identifier: GPL-2.0-or-later

add_library(input_common STATIC
    drivers/camera.cpp
    drivers/camera.h
    drivers/keyboard.cpp
    drivers/keyboard.h
    drivers/mouse.cpp
    drivers/mouse.h
    drivers/tas_input.cpp
    drivers/tas_input.h
    drivers/touch_screen.cpp
    drivers/touch_screen.h
    drivers/udp_client.cpp
    drivers/udp_client.h
    drivers/virtual_amiibo.cpp
    drivers/virtual_amiibo.h
    drivers/virtual_gamepad.cpp
    drivers/virtual_gamepad.h
    helpers/stick_from_buttons.cpp
    helpers/stick_from_buttons.h
    helpers/touch_from_buttons.cpp
    helpers/touch_from_buttons.h
    helpers/udp_protocol.cpp
    helpers/udp_protocol.h
    input_engine.cpp
    input_engine.h
    input_mapping.cpp
    input_mapping.h
    input_poller.cpp
    input_poller.h
    main.cpp
    main.h
)

if (MSVC)
    target_compile_options(input_common PRIVATE
        /we4242 # 'identifier': conversion from 'type1' to 'type2', possible loss of data
        /we4254 # 'operator': conversion from 'type1:field_bits' to 'type2:field_bits', possible loss of data
        /we4800 # Implicit conversion from 'type' to bool. Possible information loss
    )
else()
    target_compile_options(input_common PRIVATE $<$<COMPILE_LANGUAGE:C,CXX>:-Wconversion>)
endif()

if (ANDROID)
    target_sources(input_common PRIVATE
        drivers/android.cpp
        drivers/android.h)

    target_link_libraries(input_common PRIVATE android)
else()
    target_sources(input_common PRIVATE
        drivers/joycon.cpp
        drivers/joycon.h
        drivers/sdl_driver.cpp
        drivers/sdl_driver.h
        helpers/dualsense_haptics.cpp
        helpers/dualsense_haptics.h
        helpers/joycon_driver.cpp
        helpers/joycon_driver.h
        helpers/rumble_mapping.cpp
        helpers/rumble_mapping.h
        helpers/switch2_protocol.cpp
        helpers/switch2_protocol.h
        helpers/joycon_protocol/calibration.cpp
        helpers/joycon_protocol/calibration.h
        helpers/joycon_protocol/common_protocol.cpp
        helpers/joycon_protocol/common_protocol.h
        helpers/joycon_protocol/subcommand_response.h
        helpers/joycon_protocol/generic_functions.cpp
        helpers/joycon_protocol/generic_functions.h
        helpers/joycon_protocol/joycon_types.h
        helpers/joycon_protocol/irs.cpp
        helpers/joycon_protocol/irs.h
        helpers/joycon_protocol/nfc.cpp
        helpers/joycon_protocol/nfc.h
        helpers/joycon_protocol/poller.cpp
        helpers/joycon_protocol/poller.h
        helpers/joycon_protocol/ringcon.cpp
        helpers/joycon_protocol/ringcon.h
        helpers/joycon_protocol/rumble.cpp
        helpers/joycon_protocol/rumble.h)

    target_link_libraries(input_common PRIVATE SDL3::SDL3)
    target_compile_definitions(input_common PRIVATE HAVE_SDL3)

    # C++/WinRT ships with the Windows SDK, not with MinGW.
    if (WIN32 AND MSVC)
        target_sources(input_common PRIVATE
            drivers/switch2_ble.cpp
            drivers/switch2_ble.h)
        target_link_libraries(input_common PRIVATE WindowsApp)
        target_compile_definitions(input_common PRIVATE HAVE_SWITCH2_BLE)

        # The bundled Windows ARM64 Qt was compiled with C++/WinRT 2.0.250303.1, and the link fails
        # (/failifmismatch "C++/WinRT version") against objects that used the Windows SDK's own,
        # newer copy. Generate headers of Qt's version from the installed SDK's metadata instead.
        if (LICHII_USE_BUNDLED_QT AND ARCHITECTURE_arm64)
            set(cppwinrt_version 2.0.250303.1)
            set(cppwinrt_dir ${CMAKE_BINARY_DIR}/cppwinrt-${cppwinrt_version})
            if (NOT EXISTS ${cppwinrt_dir}/include/winrt/base.h)
                file(DOWNLOAD
                    https://www.nuget.org/api/v2/package/Microsoft.Windows.CppWinRT/${cppwinrt_version}
                    ${cppwinrt_dir}/cppwinrt.zip
                    EXPECTED_HASH SHA256=955e3051b35db1c00177ed14ab6b7b995c3b32a53fdb6c184ea53b1dba66e439
                    TLS_VERIFY ON)
                file(ARCHIVE_EXTRACT INPUT ${cppwinrt_dir}/cppwinrt.zip DESTINATION ${cppwinrt_dir}/pkg
                     PATTERNS bin/cppwinrt.exe)
                # vcvarsall sets WindowsSDKVersion with a trailing backslash; "sdk" is the newest SDK.
                string(REPLACE "\\" "" cppwinrt_sdk "$ENV{WindowsSDKVersion}")
                if (NOT cppwinrt_sdk)
                    set(cppwinrt_sdk sdk)
                endif()
                execute_process(
                    COMMAND ${cppwinrt_dir}/pkg/bin/cppwinrt.exe -input ${cppwinrt_sdk}
                            -output ${cppwinrt_dir}/include
                    RESULT_VARIABLE cppwinrt_result)
                if (NOT cppwinrt_result EQUAL 0 OR NOT EXISTS ${cppwinrt_dir}/include/winrt/base.h)
                    message(FATAL_ERROR "cppwinrt ${cppwinrt_version} failed to generate headers from ${cppwinrt_sdk}")
                endif()
            endif()
            target_include_directories(input_common BEFORE PRIVATE ${cppwinrt_dir}/include)
        endif()
    endif()
endif()

if (ENABLE_LIBUSB)
    target_sources(input_common PRIVATE
        drivers/gc_adapter.cpp
        drivers/gc_adapter.h)

    target_link_libraries(input_common PRIVATE libusb::usb)
    target_compile_definitions(input_common PRIVATE ENABLE_LIBUSB)
endif()

create_target_directory_groups(input_common)
target_link_libraries(input_common PUBLIC hid_core PRIVATE common Boost::headers)
