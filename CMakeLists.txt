cmake_minimum_required(VERSION 3.24)
project(DAWindower VERSION 0.2.3 LANGUAGES CXX)

# Runtime contract: Windows x86, C++17 and a statically linked CRT.
if(NOT WIN32 OR NOT CMAKE_SIZEOF_VOID_P EQUAL 4)
  message(FATAL_ERROR "DAWindower requires Windows x86. Use -A Win32.")
endif()
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
add_compile_definitions(WIN32_LEAN_AND_MEAN NOMINMAX UNICODE _UNICODE _WIN32_WINNT=0x0A00)
add_compile_definitions(DAW_VERSION="${PROJECT_VERSION}")
if(MSVC)
  add_compile_options(/W4 /permissive- /EHsc)
endif()

include(CTest)
option(DAW_BUILD_PROXY_TESTS "Build the legacy development-only proxy comparison" OFF)
option(DAW_BUILD_X "Build DAWindower X with extra client controls" OFF)
set(DAW_SYSTEM_LIBRARIES dxguid user32 gdi32 comctl32 wtsapi32 dwmapi uxtheme msimg32)
set(DAW_DLL_NAME DAWindower)
set(DAW_EXPORTS src/DAWindower.def)
if(DAW_BUILD_X)
  set(DAW_DLL_NAME DAWindowerX)
  set(DAW_EXPORTS src/x/DAWindowerX.def)
  add_compile_definitions(DAW_X)
  if(DAW_BUILD_PROXY_TESTS)
    message(FATAL_ERROR "X does not support the historical development-only proxy target.")
  endif()
endif()
add_compile_definitions(DAW_TEST_DLL_A="${DAW_DLL_NAME}.dll" DAW_TEST_DLL_W=L"${DAW_DLL_NAME}.dll")

# Geometry is shared by the DLL and tests; it has no desktop dependency.
add_library(daw_geometry STATIC src/window/geometry.cpp src/window/aspect.cpp)
target_include_directories(daw_geometry PUBLIC include)
set(DAW_GRAPHICS
  src/graphics/draw.cpp
  src/graphics/surface.cpp
  src/graphics/gdi.cpp
  src/window/window.cpp
  src/window/icon.cpp
  src/window/frame.cpp
  src/window/caption.cpp
  src/window/sizing.cpp)
if(DAW_BUILD_X)
  set(DAW_X_SOURCES src/window/name.cpp src/x/launch.cpp)
  list(APPEND DAW_GRAPHICS ${DAW_X_SOURCES})
endif()
set(DAW_SOURCES
  src/main.cpp
  src/compatibility/bootstrap.cpp
  src/compatibility/imports.cpp
  src/compatibility/mutex.cpp
  src/compatibility/geometry.cpp
  src/cursor/diagnostics.cpp
  ${DAW_GRAPHICS})

# The only runtime distribution target.
add_library(DAWindower SHARED ${DAW_SOURCES} ${DAW_EXPORTS})
set_target_properties(DAWindower PROPERTIES OUTPUT_NAME ${DAW_DLL_NAME})
target_include_directories(DAWindower PRIVATE include src)
target_link_libraries(DAWindower PRIVATE daw_geometry ${DAW_SYSTEM_LIBRARIES})

if(BUILD_TESTING)
  enable_language(RC)
  # Direct source tests exercise recovery without injecting into a retail client.
  add_executable(geometry_tests tests/geometry_tests.cpp)
  target_link_libraries(geometry_tests PRIVATE daw_geometry)
  add_test(NAME geometry COMMAND geometry_tests)
  add_executable(aspect_tests tests/aspect_tests.cpp)
  target_link_libraries(aspect_tests PRIVATE daw_geometry)
  add_test(NAME aspect_geometry COMMAND aspect_tests)
  set_tests_properties(aspect_geometry PROPERTIES LABELS "unit")

  add_executable(recovery_tests tests/recovery_tests.cpp ${DAW_GRAPHICS})
  target_include_directories(recovery_tests PRIVATE include src)
  target_link_libraries(recovery_tests PRIVATE daw_geometry ddraw ${DAW_SYSTEM_LIBRARIES})
  add_test(NAME surface_recovery_contract COMMAND recovery_tests)

  add_executable(presentation_recovery_tests
    tests/presentation_recovery_tests.cpp
    src/window/window.cpp
    src/window/icon.cpp
    src/window/frame.cpp
    src/window/caption.cpp
    src/window/sizing.cpp
    src/graphics/gdi.cpp
    src/compatibility/imports.cpp)
  target_include_directories(presentation_recovery_tests PRIVATE include src)
  target_link_libraries(presentation_recovery_tests PRIVATE daw_geometry ${DAW_SYSTEM_LIBRARIES})
  if(DAW_BUILD_X)
    target_sources(presentation_recovery_tests PRIVATE ${DAW_X_SOURCES})
  endif()
  add_test(NAME presentation_recovery_contract COMMAND presentation_recovery_tests)
  add_test(NAME presentation_format_contract COMMAND presentation_recovery_tests --format)

  # LoadLibrary tests use the actual DLL beside their executable. The retail
  # helper is built for manual acceptance but is deliberately not a CTest case.
  add_executable(ddraw_harness tools/ddraw_harness.cpp)
  add_executable(retail_loadlibrary_test tools/retail_loadlibrary_test.cpp)
  target_link_libraries(retail_loadlibrary_test PRIVATE user32 gdi32)
  foreach(target ddraw_harness retail_render_tests gdi_render_tests resize_tests resize_cursor_tests borderless_tests)
    if(NOT TARGET ${target})
      add_executable(${target} tests/${target}.cpp)
    endif()
    target_include_directories(${target} PRIVATE include)
    target_link_libraries(${target} PRIVATE ddraw dxguid user32 gdi32)
    add_dependencies(${target} DAWindower)
  endforeach()

  target_link_libraries(borderless_tests PRIVATE dwmapi)
  add_test(NAME directdraw_contract COMMAND ddraw_harness --self-test)
  add_test(NAME loadlibrary_only_contract COMMAND ddraw_harness --load-only)
  add_test(NAME dormant_unload_contract COMMAND ddraw_harness --dormant-unload)

  # Preserve compatibility with launchers that keep the old dawnd.dll filename.
  add_custom_command(TARGET ddraw_harness POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E make_directory "${CMAKE_BINARY_DIR}/renamed/$<CONFIG>"
    COMMAND ${CMAKE_COMMAND} -E copy "$<TARGET_FILE:ddraw_harness>" "${CMAKE_BINARY_DIR}/renamed/$<CONFIG>/ddraw_harness.exe"
    COMMAND ${CMAKE_COMMAND} -E copy "$<TARGET_FILE:DAWindower>" "${CMAKE_BINARY_DIR}/renamed/$<CONFIG>/dawnd.dll")
  add_test(NAME renamed_loadlibrary_contract COMMAND "${CMAKE_BINARY_DIR}/renamed/$<CONFIG>/ddraw_harness.exe" --load-only-renamed)

  add_test(NAME retail_render_contract COMMAND retail_render_tests)
  add_test(NAME gdi_render_contract COMMAND gdi_render_tests)
  add_test(NAME retail_render_loadlibrary_contract COMMAND retail_render_tests --load-only)
  add_test(NAME gdi_render_loadlibrary_contract COMMAND gdi_render_tests --load-only)
  add_test(NAME native_aspect_resize_contract COMMAND resize_tests)
  add_test(NAME resize_cursor_contract COMMAND resize_cursor_tests)
  add_test(NAME borderless_window_contract COMMAND borderless_tests)
  add_test(NAME fullscreen_before_attach_contract COMMAND borderless_tests --before)
  add_test(NAME fullscreen_after_attach_contract COMMAND borderless_tests --after)
  add_test(NAME client_geometry_gate_contract COMMAND borderless_tests --gates)

  configure_file(tests/window_icon_tests.rc.in window_icon_tests.rc @ONLY)
  add_executable(window_icon_tests tests/window_icon_tests.cpp "${CMAKE_CURRENT_BINARY_DIR}/window_icon_tests.rc")
  target_link_libraries(window_icon_tests PRIVATE ddraw dxguid user32 gdi32 shell32 dwmapi)
  add_dependencies(window_icon_tests DAWindower)
  add_test(NAME window_icon_contract COMMAND window_icon_tests)
  set_tests_properties(window_icon_contract PROPERTIES LABELS "desktop" RUN_SERIAL TRUE)
  add_executable(titlebar_tests tests/titlebar_tests.cpp)
  target_include_directories(titlebar_tests PRIVATE include)
  target_link_libraries(titlebar_tests PRIVATE ddraw dxguid user32 gdi32 dwmapi)
  add_dependencies(titlebar_tests DAWindower)
  if(NOT DAW_BUILD_X)
    add_test(NAME standard_titlebar_contract COMMAND titlebar_tests "${DAW_DLL_NAME}.dll")
    set_tests_properties(standard_titlebar_contract PROPERTIES LABELS "desktop" RUN_SERIAL TRUE)
  endif()

  add_executable(character_name_tests tests/character_name_tests.cpp src/window/name.cpp)
  target_include_directories(character_name_tests PRIVATE src)
  add_test(NAME character_name_contract COMMAND character_name_tests)
  set_tests_properties(character_name_contract PROPERTIES LABELS "unit")
  if(DAW_BUILD_X)
    add_executable(x_launch_fixture tests/x_launch_fixture.cpp)
    target_include_directories(x_launch_fixture PRIVATE include)
    target_link_libraries(x_launch_fixture PRIVATE ddraw dxguid user32)
    add_executable(x_launch_tests tests/x_launch_tests.cpp src/x/launch.cpp)
    target_include_directories(x_launch_tests PRIVATE src)
    add_dependencies(x_launch_tests DAWindower x_launch_fixture)
    add_test(NAME x_launch_contract COMMAND x_launch_tests)
    set_tests_properties(x_launch_contract PROPERTIES LABELS "desktop" RUN_SERIAL TRUE)
    add_test(NAME x_titlebar_contract COMMAND titlebar_tests "${DAW_DLL_NAME}.dll")
    set_tests_properties(x_titlebar_contract PROPERTIES LABELS "desktop" RUN_SERIAL TRUE)
  endif()

  # Geometry is safe without a desktop. Native rendering tests share desktop
  # state and must not compete for visible windows/cursor state under ctest -j.
  set_tests_properties(geometry PROPERTIES LABELS "unit")
  set_tests_properties(
    surface_recovery_contract presentation_recovery_contract presentation_format_contract directdraw_contract
    loadlibrary_only_contract dormant_unload_contract renamed_loadlibrary_contract
    retail_render_contract gdi_render_contract retail_render_loadlibrary_contract
    gdi_render_loadlibrary_contract native_aspect_resize_contract resize_cursor_contract
    borderless_window_contract fullscreen_before_attach_contract fullscreen_after_attach_contract
    client_geometry_gate_contract
    PROPERTIES LABELS "desktop" RUN_SERIAL TRUE)

  # Optional historical comparison route. Never installed with the standalone DLL.
  if(DAW_BUILD_PROXY_TESTS)
    add_library(ddraw_proxy SHARED ${DAW_SOURCES} src/ddraw.def)
    target_compile_definitions(ddraw_proxy PRIVATE DAW_PROXY)
    target_include_directories(ddraw_proxy PRIVATE include src)
    target_link_libraries(ddraw_proxy PRIVATE daw_geometry ${DAW_SYSTEM_LIBRARIES})
    set_target_properties(ddraw_proxy PROPERTIES OUTPUT_NAME ddraw
      RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/proxy/$<CONFIG>"
      ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/proxy/$<CONFIG>")
    foreach(target ddraw_harness retail_render_tests gdi_render_tests)
      add_dependencies(${target} ddraw_proxy)
      add_custom_command(TARGET ${target} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy "$<TARGET_FILE:${target}>" "${CMAKE_BINARY_DIR}/proxy/$<CONFIG>/${target}.exe")
      if(target STREQUAL "ddraw_harness")
        set(proxy_arg --proxy-self-test)
      else()
        set(proxy_arg --proxy)
      endif()
      add_test(NAME ${target}_proxy_contract COMMAND "${CMAKE_BINARY_DIR}/proxy/$<CONFIG>/${target}.exe" ${proxy_arg})
      set_tests_properties(${target}_proxy_contract PROPERTIES LABELS "desktop" RUN_SERIAL TRUE)
    endforeach()
  endif()
endif()

# Installation is one file. Development tools, optional proxy, diagnostics,
# headers and documentation are not runtime distribution files.
install(TARGETS DAWindower RUNTIME DESTINATION .)
