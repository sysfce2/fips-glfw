if (NOT FIPS_IMPORT)
    cmake_minimum_required(VERSION 2.8)
    get_filename_component(FIPS_ROOT_DIR "../fips" ABSOLUTE)
    include("${FIPS_ROOT_DIR}/cmake/fips.cmake")
    fips_setup()
    fips_project(fips-glfw)
endif()

option(FIPS_GLFW_WAYLAND "Enable GLFW Wayland support" OFF)

if ((FIPS_MACOS OR FIPS_WINDOWS OR FIPS_LINUX) AND NOT (FIPS_RASPBERRYPI OR FIPS_OSX_USE_ARC OR FIPS_UWP))

    fips_begin_lib(glfw3)

        include_directories(. glfw/src glfw/include)
        add_definitions(-D_GLFW_USE_CONFIG_H)
        if (FIPS_WINDOWS)
            add_definitions(-D_GLFW_USE_DWM_SWAP_INTERVAL)
        endif()

        fips_files(glfw_config.h)
        fips_dir(glfw/include/GLFW GROUP .)
        fips_files(glfw3.h glfw3native.h)
        fips_dir(glfw/src GROUP .)
        fips_files(
            internal.h
            context.c init.c input.c monitor.c vulkan.c window.c
        )
        if (FIPS_MACOS)
            add_definitions(-DFIPS_GLFW_MACOS=1)
            fips_files(
                cocoa_platform.h cocoa_joystick.h posix_thread.h
                cocoa_init.m cocoa_monitor.m cocoa_window.m
                cocoa_joystick.m cocoa_time.c posix_thread.c
                nsgl_context.h nsgl_context.m
                egl_context.h egl_context.c
                osmesa_context.h osmesa_context.c
            )
            fips_frameworks_osx(Cocoa CoreVideo OpenGL Carbon)
        elseif (FIPS_WINDOWS)
            add_definitions(-DFIPS_GLFW_WINDOWS=1)
            fips_files(
                win32_platform.h win32_joystick.h
                win32_init.c win32_monitor.c win32_time.c
                win32_thread.c win32_window.c win32_joystick.c
                wgl_context.h wgl_context.c
                egl_context.h egl_context.c
                osmesa_context.h osmesa_context.c
            )
            fips_libs(opengl32)
        elseif (FIPS_LINUX)
            add_definitions(-DFIPS_GLFW_LINUX=1)
            if (FIPS_GLFW_WAYLAND)
                add_definitions(-DFIPS_GLFW_WAYLAND=1)
                find_package(ECM REQUIRED NO_MODULE)
                list(APPEND CMAKE_MODULE_PATH "${ECM_MODULE_PATH}" "${CMAKE_CURRENT_LIST_DIR}/glfw/CMake/modules")
                find_package(Wayland REQUIRED Client Cursor Egl)
                find_package(WaylandScanner REQUIRED)
                find_package(WaylandProtocols 1.15 REQUIRED)

                include_directories("${Wayland_INCLUDE_DIRS}")
                fips_libs("${Wayland_LIBRARIES}" "${CMAKE_THREAD_LIBS_INIT}" EGL xkbcommon)

                find_package(XKBCommon REQUIRED)
                include_directories("${XKBCOMMON_INCLUDE_DIRS}")

                include(CheckIncludeFiles)
                include(CheckFunctionExists)
                check_include_files(xkbcommon/xkbcommon-compose.h HAVE_XKBCOMMON_COMPOSE_H)
                check_function_exists(memfd_create HAVE_MEMFD_CREATE)
                ecm_add_wayland_client_protocol(glfw_SOURCES
                    PROTOCOL
                    "${WAYLAND_PROTOCOLS_PKGDATADIR}/stable/xdg-shell/xdg-shell.xml"
                    BASENAME xdg-shell)
                ecm_add_wayland_client_protocol(glfw_SOURCES
                    PROTOCOL
                    "${WAYLAND_PROTOCOLS_PKGDATADIR}/unstable/xdg-decoration/xdg-decoration-unstable-v1.xml"
                    BASENAME xdg-decoration)
                ecm_add_wayland_client_protocol(glfw_SOURCES
                    PROTOCOL
                    "${WAYLAND_PROTOCOLS_PKGDATADIR}/stable/viewporter/viewporter.xml"
                    BASENAME viewporter)
                ecm_add_wayland_client_protocol(glfw_SOURCES
                    PROTOCOL
                    "${WAYLAND_PROTOCOLS_PKGDATADIR}/unstable/relative-pointer/relative-pointer-unstable-v1.xml"
                    BASENAME relative-pointer-unstable-v1)
                ecm_add_wayland_client_protocol(glfw_SOURCES
                    PROTOCOL
                    "${WAYLAND_PROTOCOLS_PKGDATADIR}/unstable/pointer-constraints/pointer-constraints-unstable-v1.xml"
                    BASENAME pointer-constraints-unstable-v1)
                ecm_add_wayland_client_protocol(glfw_SOURCES
                    PROTOCOL
                    "${WAYLAND_PROTOCOLS_PKGDATADIR}/unstable/idle-inhibit/idle-inhibit-unstable-v1.xml"
                    BASENAME idle-inhibit-unstable-v1)
                set(CurDir "")
                fips_files("${glfw_SOURCES}")
                include_directories("${CMAKE_CURRENT_BINARY_DIR}")
                fips_dir(glfw/src GROUP .)
                fips_files(
                    wl_platform.h wl_init.c wl_monitor.c wl_window.c 
                )
            else()
                fips_files(
                    x11_platform.h x11_init.c x11_monitor.c x11_window.c
                    glx_context.h glx_context.c
                )
                fips_libs(X11 Xrandr Xi Xinerama Xxf86vm Xcursor)
            endif()
            fips_files(
                osmesa_context.h osmesa_context.c
                egl_context.c egl_context.h
                posix_time.c posix_thread.c xkb_unicode.c linux_joystick.c
                linux_joystick.h xkb_unicode.h posix_time.h posix_thread.h
            )
            fips_libs(GL m ${CMAKE_DL_LIBS})
        endif()

    fips_end_lib()

    # hack to silence some warnings in GLFW code
    if (FIPS_CLANG)
        set_target_properties(glfw3 PROPERTIES COMPILE_FLAGS "-Wno-sign-compare -Wno-missing-field-initializers -Wno-deprecated-declarations")
    elseif(FIPS_GCC)
        set_target_properties(glfw3 PROPERTIES COMPILE_FLAGS "-Wno-sign-compare -Wno-missing-field-initializers")
    elseif(FIPS_MSVC)
        set_target_properties(glfw3 PROPERTIES COMPILE_FLAGS "/wd4152 /wd4204 /wd4242 /wd4244 /wd4668 /wd4996")
    endif()

endif()

if (NOT FIPS_IMPORT)
    fips_finish()
endif()
