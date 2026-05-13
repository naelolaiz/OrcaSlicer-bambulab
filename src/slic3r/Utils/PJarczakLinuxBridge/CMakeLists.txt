set(PJARCZAK_LINUX_BRIDGE_COMMON_SOURCES
    ${CMAKE_CURRENT_LIST_DIR}/PJarczakLinuxBridgeConfig.cpp
    ${CMAKE_CURRENT_LIST_DIR}/PJarczakLinuxSoBridgeRpcProtocol.cpp
    ${CMAKE_CURRENT_LIST_DIR}/PJarczakLinuxSoBridgeRpcClient.cpp
    ${CMAKE_CURRENT_LIST_DIR}/PJarczakLinuxSoBridgeEventPump.cpp
    ${CMAKE_CURRENT_LIST_DIR}/PJarczakBambuNetworkForwarderState.cpp
    ${CMAKE_CURRENT_LIST_DIR}/PJarczakBambuNetworkForwarderExports.cpp
)

get_filename_component(PJARCZAK_REPO_ROOT "${CMAKE_CURRENT_LIST_DIR}/../../../.." ABSOLUTE)
set(PJARCZAK_LINUX_HOST_DIR "${PJARCZAK_REPO_ROOT}/tools/pjarczak_bambu_linux_host")
set(PJARCZAK_LINUX_BRIDGE_INCLUDE_DIRS
    ${PJARCZAK_REPO_ROOT}
    ${PJARCZAK_REPO_ROOT}/src
    ${PJARCZAK_REPO_ROOT}/deps_src
)

if (WIN32)
    add_library(pjarczak_bambu_networking_bridge SHARED
        ${PJARCZAK_LINUX_BRIDGE_COMMON_SOURCES}
        ${CMAKE_CURRENT_LIST_DIR}/PJarczakLinuxSoBridgeLauncher_win.cpp
    )
    target_include_directories(pjarczak_bambu_networking_bridge PRIVATE ${PJARCZAK_LINUX_BRIDGE_INCLUDE_DIRS})
    target_compile_definitions(pjarczak_bambu_networking_bridge PRIVATE PJARCZAK_LINUX_BRIDGE_LIGHTWEIGHT_TASKS=1 BAMBU_EXPORTS)
    target_link_libraries(pjarczak_bambu_networking_bridge PRIVATE boost_headeronly boost_libs OpenSSL::Crypto)
elseif (APPLE)
    add_library(pjarczak_bambu_networking_bridge SHARED
        ${PJARCZAK_LINUX_BRIDGE_COMMON_SOURCES}
        ${CMAKE_CURRENT_LIST_DIR}/PJarczakLinuxSoBridgeLauncher_mac.mm
    )
    target_include_directories(pjarczak_bambu_networking_bridge PRIVATE ${PJARCZAK_LINUX_BRIDGE_INCLUDE_DIRS})
    target_compile_definitions(pjarczak_bambu_networking_bridge PRIVATE PJARCZAK_LINUX_BRIDGE_LIGHTWEIGHT_TASKS=1)
    target_link_libraries(pjarczak_bambu_networking_bridge PRIVATE boost_headeronly boost_libs OpenSSL::Crypto)
else()
    add_library(pjarczak_bambu_networking_bridge SHARED
        ${PJARCZAK_LINUX_BRIDGE_COMMON_SOURCES}
        ${CMAKE_CURRENT_LIST_DIR}/PJarczakLinuxSoBridgeLauncher_linux.cpp
    )
    target_include_directories(pjarczak_bambu_networking_bridge PRIVATE ${PJARCZAK_LINUX_BRIDGE_INCLUDE_DIRS})
    target_link_libraries(pjarczak_bambu_networking_bridge PRIVATE boost_headeronly boost_libs OpenSSL::Crypto)
endif()

set_target_properties(pjarczak_bambu_networking_bridge PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_RUNTIME_OUTPUT_DIRECTORY}"
    LIBRARY_OUTPUT_DIRECTORY "${CMAKE_RUNTIME_OUTPUT_DIRECTORY}"
)

install(TARGETS pjarczak_bambu_networking_bridge
    RUNTIME DESTINATION "."
    LIBRARY DESTINATION "."
)

if (UNIX AND NOT APPLE AND EXISTS "${PJARCZAK_LINUX_HOST_DIR}/main.cpp" AND EXISTS "${PJARCZAK_LINUX_HOST_DIR}/LinuxPluginHost.cpp")
    set(PJARCZAK_LINUX_HOST_COMMON_SOURCES
        ${PJARCZAK_LINUX_HOST_DIR}/main.cpp
        ${PJARCZAK_LINUX_HOST_DIR}/LinuxPluginHost.cpp
        ${CMAKE_CURRENT_LIST_DIR}/PJarczakLinuxBridgeConfig.cpp
        ${CMAKE_CURRENT_LIST_DIR}/PJarczakLinuxSoBridgeRpcProtocol.cpp
    )

    find_package(nlohmann_json CONFIG QUIET)

    function(add_pjarczak_linux_host_variant target_name abi_value)
        add_executable(${target_name}
            ${PJARCZAK_LINUX_HOST_COMMON_SOURCES}
        )
        target_include_directories(${target_name} PRIVATE ${PJARCZAK_LINUX_BRIDGE_INCLUDE_DIRS})
        target_compile_definitions(${target_name} PRIVATE
            PJARCZAK_LINUX_BRIDGE_STANDALONE_HOST=1
            PJARCZAK_LINUX_BRIDGE_LIGHTWEIGHT_TASKS=1
            _GLIBCXX_USE_CXX11_ABI=${abi_value}
        )
        if (TARGET nlohmann_json::nlohmann_json)
            target_link_libraries(${target_name} PRIVATE nlohmann_json::nlohmann_json)
        endif()
        target_link_libraries(${target_name} PRIVATE boost_headeronly ${CMAKE_DL_LIBS} OpenSSL::Crypto)
        target_link_options(${target_name} PRIVATE -static-libstdc++ -static-libgcc)
        set_target_properties(${target_name} PROPERTIES
            RUNTIME_OUTPUT_DIRECTORY "${CMAKE_RUNTIME_OUTPUT_DIRECTORY}"
            BUILD_WITH_INSTALL_RPATH TRUE
            INSTALL_RPATH "$ORIGIN"
        )
    endfunction()

    add_pjarczak_linux_host_variant(pjarczak_bambu_linux_host_abi1 1)
    add_pjarczak_linux_host_variant(pjarczak_bambu_linux_host_abi0 0)

    add_custom_target(pjarczak_bambu_linux_host
        DEPENDS pjarczak_bambu_linux_host_abi1 pjarczak_bambu_linux_host_abi0
    )

    set(PJARCZAK_LINUX_RUNTIME_COMPONENT pjarczak_linux_host_runtime)
    set(PJARCZAK_LINUX_RUNTIME_DEST linux-x86_64)
    set(PJARCZAK_LINUX_RUNTIME_DEPENDENCY_SET pjarczak_linux_host_runtime_dependencies)

    install(PROGRAMS "${CMAKE_SOURCE_DIR}/tools/pjarczak_bambu_runtime/wsl/pjarczak_bambu_linux_host"
        DESTINATION "${PJARCZAK_LINUX_RUNTIME_DEST}"
        COMPONENT "${PJARCZAK_LINUX_RUNTIME_COMPONENT}"
        EXCLUDE_FROM_ALL
    )

    install(TARGETS pjarczak_bambu_linux_host_abi1 pjarczak_bambu_linux_host_abi0
        RUNTIME_DEPENDENCY_SET "${PJARCZAK_LINUX_RUNTIME_DEPENDENCY_SET}"
        RUNTIME DESTINATION "${PJARCZAK_LINUX_RUNTIME_DEST}"
        COMPONENT "${PJARCZAK_LINUX_RUNTIME_COMPONENT}"
        EXCLUDE_FROM_ALL
    )

    foreach(PJARCZAK_LINUX_RUNTIME_CERT
        "${CMAKE_SOURCE_DIR}/cert/ca-certificates.crt"
        "${CMAKE_SOURCE_DIR}/cert/slicer_base64.cer"
        "${CMAKE_SOURCE_DIR}/resources/cert/ca-certificates.crt"
        "${CMAKE_SOURCE_DIR}/resources/cert/slicer_base64.cer"
    )
        if (EXISTS "${PJARCZAK_LINUX_RUNTIME_CERT}")
            install(FILES "${PJARCZAK_LINUX_RUNTIME_CERT}"
                DESTINATION "${PJARCZAK_LINUX_RUNTIME_DEST}"
                COMPONENT "${PJARCZAK_LINUX_RUNTIME_COMPONENT}"
                EXCLUDE_FROM_ALL
            )
        endif()
    endforeach()

    set(PJARCZAK_LINUX_PAYLOAD_IMPORTED_TARGETS "")
    set(PJARCZAK_LINUX_PAYLOAD_NAMES libbambu_networking.so libBambuSource.so)
    foreach(PJARCZAK_LINUX_PAYLOAD_NAME IN LISTS PJARCZAK_LINUX_PAYLOAD_NAMES)
        foreach(PJARCZAK_LINUX_PAYLOAD_CANDIDATE
            "${CMAKE_SOURCE_DIR}/src/slic3r/Utils/PJarczakLinuxBridge/runtime/proprietary/${PJARCZAK_LINUX_PAYLOAD_NAME}"
            "${CMAKE_SOURCE_DIR}/tools/pjarczak_bambu_linux_host/runtime/linux-x86_64/${PJARCZAK_LINUX_PAYLOAD_NAME}"
            "${CMAKE_SOURCE_DIR}/build/${PJARCZAK_LINUX_PAYLOAD_NAME}"
            "${CMAKE_SOURCE_DIR}/${PJARCZAK_LINUX_PAYLOAD_NAME}"
        )
            if (EXISTS "${PJARCZAK_LINUX_PAYLOAD_CANDIDATE}")
                string(MAKE_C_IDENTIFIER "${PJARCZAK_LINUX_PAYLOAD_NAME}" PJARCZAK_LINUX_PAYLOAD_TARGET)
                set(PJARCZAK_LINUX_PAYLOAD_TARGET "pjarczak_payload_${PJARCZAK_LINUX_PAYLOAD_TARGET}")
                add_library("${PJARCZAK_LINUX_PAYLOAD_TARGET}" SHARED IMPORTED)
                set_target_properties("${PJARCZAK_LINUX_PAYLOAD_TARGET}" PROPERTIES IMPORTED_LOCATION "${PJARCZAK_LINUX_PAYLOAD_CANDIDATE}")
                list(APPEND PJARCZAK_LINUX_PAYLOAD_IMPORTED_TARGETS "${PJARCZAK_LINUX_PAYLOAD_TARGET}")
                break()
            endif()
        endforeach()
    endforeach()

    if (PJARCZAK_LINUX_PAYLOAD_IMPORTED_TARGETS)
        install(IMPORTED_RUNTIME_ARTIFACTS ${PJARCZAK_LINUX_PAYLOAD_IMPORTED_TARGETS}
            RUNTIME_DEPENDENCY_SET "${PJARCZAK_LINUX_RUNTIME_DEPENDENCY_SET}"
            LIBRARY DESTINATION "${PJARCZAK_LINUX_RUNTIME_DEST}"
            COMPONENT "${PJARCZAK_LINUX_RUNTIME_COMPONENT}"
            EXCLUDE_FROM_ALL
        )
    endif()

    install(RUNTIME_DEPENDENCY_SET "${PJARCZAK_LINUX_RUNTIME_DEPENDENCY_SET}"
        DESTINATION "${PJARCZAK_LINUX_RUNTIME_DEST}"
        COMPONENT "${PJARCZAK_LINUX_RUNTIME_COMPONENT}"
        EXCLUDE_FROM_ALL
        POST_EXCLUDE_REGEXES
            ".*/ld-linux[^/]*$"
            ".*/ld-musl-[^/]*$"
            ".*/linux-vdso\\.so\\..*$"
            ".*/libc\\.so\\..*$"
            ".*/libm\\.so\\..*$"
            ".*/libpthread\\.so\\..*$"
            ".*/librt\\.so\\..*$"
            ".*/libdl\\.so\\..*$"
            ".*/libresolv\\.so\\..*$"
            ".*/libnss_[^/]*\\.so.*$"
            ".*/libnsl\\.so\\..*$"
            ".*/libutil\\.so\\..*$"
            ".*/libanl\\.so\\..*$"
    )

    install(CODE "
        set(runtime_dir \"\$ENV{DESTDIR}\${CMAKE_INSTALL_PREFIX}/${PJARCZAK_LINUX_RUNTIME_DEST}\")
        if (EXISTS \"\${runtime_dir}/libbambu_networking.so\" AND EXISTS \"\${runtime_dir}/libBambuSource.so\")
            file(READ \"${CMAKE_SOURCE_DIR}/src/slic3r/Utils/bambu_networking.hpp\" bambu_networking_header)
            string(REGEX MATCH \"BAMBU_NETWORK_AGENT_VERSION[ \\t\\r\\n]+\\\"([^\\\"]+)\\\"\" _ \"\${bambu_networking_header}\")
            set(abi_version \"\${CMAKE_MATCH_1}\")
            if (abi_version STREQUAL \"\")
                message(FATAL_ERROR \"Could not extract BAMBU_NETWORK_AGENT_VERSION\")
            endif()
            file(SHA256 \"\${runtime_dir}/libbambu_networking.so\" network_sha)
            file(SHA256 \"\${runtime_dir}/libBambuSource.so\" source_sha)
            file(WRITE \"\${runtime_dir}/linux_payload_manifest.json\"
\"{
  \\\"files\\\": [
    {
      \\\"name\\\": \\\"libbambu_networking.so\\\",
      \\\"sha256\\\": \\\"\${network_sha}\\\",
      \\\"abi_version\\\": \\\"\${abi_version}\\\"
    },
    {
      \\\"name\\\": \\\"libBambuSource.so\\\",
      \\\"sha256\\\": \\\"\${source_sha}\\\"
    }
  ]
}
\")
        endif()
    " COMPONENT "${PJARCZAK_LINUX_RUNTIME_COMPONENT}" EXCLUDE_FROM_ALL)

    set(PJARCZAK_WSL_ROOTFS "${CMAKE_SOURCE_DIR}/tools/pjarczak_bambu_runtime/rootfs/windows-wsl2-rootfs.tar")

    add_custom_target(pjarczak_bambu_wsl_rootfs
        COMMAND bash tools/pjarczak_bambu_runtime/rootfs/build_windows_wsl_rootfs.sh tools/pjarczak_bambu_runtime/rootfs/windows-wsl2-rootfs.tar
        WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}"
        COMMENT "Build WSL rootfs"
        VERBATIM
    )

    install(FILES "${PJARCZAK_WSL_ROOTFS}"
        DESTINATION "."
        COMPONENT pjarczak_wsl_rootfs
        EXCLUDE_FROM_ALL
    )

    install(TARGETS pjarczak_bambu_linux_host_abi1 pjarczak_bambu_linux_host_abi0
        RUNTIME DESTINATION "."
    )
endif()

if (APPLE)
    set(PJARCZAK_MAC_WRAPPER_SCRIPT_SOURCE ${CMAKE_SOURCE_DIR}/tools/pjarczak_bambu_linux_host/pjarczak-bambu-linux-host-wrapper)
    set(PJARCZAK_MAC_INSTALLER_SOURCE ${CMAKE_SOURCE_DIR}/tools/pjarczak_bambu_runtime/macos/pjarczak_install_macos_runtime.sh)
    set(PJARCZAK_MAC_VERIFY_SOURCE ${CMAKE_SOURCE_DIR}/tools/pjarczak_bambu_runtime/macos/pjarczak_verify_macos_runtime.sh)
    set(PJARCZAK_MAC_INSTANCE_SOURCE ${CMAKE_SOURCE_DIR}/tools/pjarczak_bambu_runtime/macos/pjarczak_lima_instance.txt)
    add_custom_command(TARGET pjarczak_bambu_networking_bridge POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${PJARCZAK_MAC_WRAPPER_SCRIPT_SOURCE}
            $<TARGET_FILE_DIR:pjarczak_bambu_networking_bridge>/pjarczak-bambu-linux-host-wrapper
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${PJARCZAK_MAC_INSTALLER_SOURCE}
            $<TARGET_FILE_DIR:pjarczak_bambu_networking_bridge>/install_runtime_macos.sh
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${PJARCZAK_MAC_VERIFY_SOURCE}
            $<TARGET_FILE_DIR:pjarczak_bambu_networking_bridge>/verify_runtime_macos.sh
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${PJARCZAK_MAC_INSTANCE_SOURCE}
            $<TARGET_FILE_DIR:pjarczak_bambu_networking_bridge>/pjarczak_lima_instance.txt
    )
    install(FILES ${PJARCZAK_MAC_WRAPPER_SCRIPT_SOURCE}
        DESTINATION "."
        RENAME pjarczak-bambu-linux-host-wrapper
    )
    install(FILES ${PJARCZAK_MAC_INSTALLER_SOURCE}
        DESTINATION "."
        RENAME install_runtime_macos.sh
    )
    install(FILES ${PJARCZAK_MAC_VERIFY_SOURCE}
        DESTINATION "."
        RENAME verify_runtime_macos.sh
    )
    install(FILES ${PJARCZAK_MAC_INSTANCE_SOURCE}
        DESTINATION "."
        RENAME pjarczak_lima_instance.txt
    )
endif()

if (WIN32)
    set(PJARCZAK_WSL_HELPER_SOURCE ${CMAKE_SOURCE_DIR}/tools/pjarczak_bambu_runtime/wsl/pjarczak_wsl_run_host.sh)
    set(PJARCZAK_WSL_INSTALLER_SOURCE ${CMAKE_SOURCE_DIR}/tools/pjarczak_bambu_runtime/wsl/install_runtime.ps1)
    set(PJARCZAK_WSL_INSTALLER_CMD_SOURCE ${CMAKE_SOURCE_DIR}/tools/pjarczak_bambu_runtime/wsl/install_runtime.cmd)
    set(PJARCZAK_WSL_VERIFY_SOURCE ${CMAKE_SOURCE_DIR}/tools/pjarczak_bambu_runtime/wsl/verify_runtime.ps1)
    set(PJARCZAK_WSL_DISTRO_FILE_SOURCE ${CMAKE_SOURCE_DIR}/tools/pjarczak_bambu_runtime/wsl/pjarczak_wsl_distro.txt)
    set(PJARCZAK_RUNTIME_ASSEMBLER_SOURCE ${CMAKE_SOURCE_DIR}/tools/pjarczak_bambu_runtime/release/assemble_windows_runtime_bundle.ps1)
    set(PJARCZAK_PLUGIN_CACHE_SUBDIR_SOURCE ${CMAKE_SOURCE_DIR}/tools/pjarczak_bambu_runtime/wsl/pjarczak_plugin_cache_subdir.txt)
    add_custom_command(TARGET pjarczak_bambu_networking_bridge POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${PJARCZAK_WSL_HELPER_SOURCE}
            $<TARGET_FILE_DIR:pjarczak_bambu_networking_bridge>/pjarczak_wsl_run_host.sh
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${PJARCZAK_WSL_INSTALLER_SOURCE}
            $<TARGET_FILE_DIR:pjarczak_bambu_networking_bridge>/install_runtime.ps1
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${PJARCZAK_WSL_INSTALLER_CMD_SOURCE}
            $<TARGET_FILE_DIR:pjarczak_bambu_networking_bridge>/install_runtime.cmd
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${PJARCZAK_WSL_VERIFY_SOURCE}
            $<TARGET_FILE_DIR:pjarczak_bambu_networking_bridge>/verify_runtime.ps1
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${PJARCZAK_WSL_DISTRO_FILE_SOURCE}
            $<TARGET_FILE_DIR:pjarczak_bambu_networking_bridge>/pjarczak_wsl_distro.txt
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${PJARCZAK_RUNTIME_ASSEMBLER_SOURCE}
            $<TARGET_FILE_DIR:pjarczak_bambu_networking_bridge>/assemble_windows_runtime_bundle.ps1
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${PJARCZAK_PLUGIN_CACHE_SUBDIR_SOURCE}
            $<TARGET_FILE_DIR:pjarczak_bambu_networking_bridge>/pjarczak_plugin_cache_subdir.txt
    )
    install(FILES
        ${PJARCZAK_WSL_HELPER_SOURCE}
        ${PJARCZAK_WSL_INSTALLER_SOURCE}
        ${PJARCZAK_WSL_INSTALLER_CMD_SOURCE}
        ${PJARCZAK_WSL_VERIFY_SOURCE}
        ${PJARCZAK_WSL_DISTRO_FILE_SOURCE}
        ${PJARCZAK_RUNTIME_ASSEMBLER_SOURCE}
        ${PJARCZAK_PLUGIN_CACHE_SUBDIR_SOURCE}
        DESTINATION "."
    )
endif()
