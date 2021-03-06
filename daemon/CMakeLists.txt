
if(SHADOW)
    add_shadow_plugin(shadow-plugin main.cpp ${TRACY_ROOT}/TracyClient.cpp)
    target_link_libraries(shadow-plugin liblokinet)
    enable_lto(shadow-plugin)
else()
    add_executable(lokinet main.cpp)
    add_executable(lokinet-vpn lokinet-vpn.cpp)
    enable_lto(lokinet lokinet-vpn)

    if(TRACY_ROOT)
        target_sources(lokinet PRIVATE ${TRACY_ROOT}/TracyClient.cpp)
    endif()

    foreach(exe lokinet lokinet-vpn)
        if(WIN32 AND NOT MSVC_VERSION)
            target_sources(${exe} PRIVATE ../llarp/win32/version.rc)
            target_link_libraries(${exe} PRIVATE ws2_32 iphlpapi)
        elseif(CMAKE_SYSTEM_NAME MATCHES "FreeBSD")
            target_link_directories(${exe} PRIVATE /usr/local/lib)
        endif()
        target_link_libraries(${exe} PRIVATE liblokinet)
        if(WITH_JEMALLOC)
          target_link_libraries(${exe} PUBLIC jemalloc)
        endif()
        target_compile_definitions(${exe} PRIVATE -DVERSIONTAG=${GIT_VERSION_REAL})
        add_log_tag(${exe})
        install(TARGETS ${exe} RUNTIME DESTINATION bin COMPONENT lokinet)
    endforeach()

    if(WIN32)
        install(PROGRAMS ${CMAKE_SOURCE_DIR}/lokinet-bootstrap.ps1 DESTINATION bin COMPONENT lokinet)
    else()
        install(PROGRAMS ${CMAKE_SOURCE_DIR}/lokinet-bootstrap DESTINATION bin COMPONENT lokinet)
    endif()

    if(CMAKE_SYSTEM_NAME MATCHES "Linux")
        option(WITH_SETCAP "Enables setcap'ing the lokinet binary with the required capabilities during installation (requires root)" ON)
        if(WITH_SETCAP)
          find_program(setcap_bin NAMES setcap HINTS /sbin /usr/sbin REQUIRED)
          message(STATUS "found setcap: ${setcap_bin}")
          install(CODE "execute_process(COMMAND ${setcap_bin} cap_net_admin,cap_net_bind_service=+eip ${CMAKE_INSTALL_PREFIX}/bin/lokinet)")
        endif()
    endif()
    
endif()
