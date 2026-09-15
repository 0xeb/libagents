cmake_minimum_required(VERSION 3.15)
project(libagents
    VERSION 0.1.1
    DESCRIPTION "Unified C++ library for AI agent providers"
    LANGUAGES CXX
)

# C++20 required by copilot-sdk-cpp
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Enable PIC for all targets (required when linking static libs into shared libs)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# =============================================================================
# Options
# =============================================================================
option(LIBAGENTS_BUILD_COPILOT "Build Copilot provider" ON)
option(LIBAGENTS_BUILD_CLAUDE "Build Claude provider" ON)

# Disable tests/examples in external SDKs
set(CLAUDE_BUILD_TESTS OFF CACHE BOOL "" FORCE)
set(CLAUDE_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
set(COPILOT_BUILD_TESTS OFF CACHE BOOL "" FORCE)
set(COPILOT_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
set(FASTMCPP_BUILD_TESTS OFF CACHE BOOL "" FORCE)
set(FASTMCPP_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
set(BUILD_TESTING OFF CACHE BOOL "" FORCE)  # General CMake testing flag

# =============================================================================
# External Dependencies (in external/)
# =============================================================================

# nlohmann_json - fetch once for all SDKs to use
find_package(nlohmann_json CONFIG QUIET)
if(NOT TARGET nlohmann_json::nlohmann_json)
    include(FetchContent)
    FetchContent_Declare(
        nlohmann_json
        GIT_REPOSITORY https://github.com/nlohmann/json.git
        GIT_TAG v3.11.3
    )
    FetchContent_MakeAvailable(nlohmann_json)
endif()

# FastMCPP (required by claude-agent-sdk-cpp)
# We set FASTMCPP_ROOT and let claude-agent-sdk-cpp add it as a subdirectory
# to avoid duplicate target issues
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/external/fastmcpp/CMakeLists.txt")
    set(FASTMCPP_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/external/fastmcpp" CACHE PATH "" FORCE)
    set(HAVE_FASTMCPP TRUE)
else()
    set(HAVE_FASTMCPP FALSE)
    message(WARNING "fastmcpp not found in external/, Claude provider may fail to build")
endif()

# Copilot SDK
if(LIBAGENTS_BUILD_COPILOT AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/external/copilot-sdk-cpp/CMakeLists.txt")
    add_subdirectory(external/copilot-sdk-cpp)
    set(HAVE_COPILOT_SDK TRUE)
else()
    set(HAVE_COPILOT_SDK FALSE)
    if(LIBAGENTS_BUILD_COPILOT)
        message(WARNING "copilot-sdk-cpp not found in external/, Copilot provider disabled")
    endif()
endif()

# Claude SDK (will add fastmcpp via FASTMCPP_ROOT)
if(LIBAGENTS_BUILD_CLAUDE AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/external/claude-agent-sdk-cpp/CMakeLists.txt")
    add_subdirectory(external/claude-agent-sdk-cpp)
    set(HAVE_CLAUDE_SDK TRUE)
else()
    set(HAVE_CLAUDE_SDK FALSE)
    if(LIBAGENTS_BUILD_CLAUDE)
        message(WARNING "claude-agent-sdk-cpp not found in external/, Claude provider disabled")
    endif()
endif()

# =============================================================================
# Library Target
# =============================================================================
set(LIBAGENTS_HEADERS
    include/libagents/libagents.hpp
    include/libagents/agent.hpp
    include/libagents/provider.hpp
    include/libagents/session.hpp
    include/libagents/config.hpp
    include/libagents/events.hpp
    include/libagents/errors.hpp
    include/libagents/providers/copilot_provider.hpp
    include/libagents/providers/claude_provider.hpp
)

set(LIBAGENTS_SOURCES
    src/agent.cpp
    src/config.cpp
    src/provider_factory.cpp
    src/session_base.cpp
)

# Add provider sources based on availability
if(HAVE_COPILOT_SDK)
    list(APPEND LIBAGENTS_SOURCES src/providers/copilot_provider.cpp)
endif()

if(HAVE_CLAUDE_SDK)
    list(APPEND LIBAGENTS_SOURCES src/providers/claude_provider.cpp)
endif()

add_library(libagents ${LIBAGENTS_SOURCES} ${LIBAGENTS_HEADERS})
add_library(libagents::libagents ALIAS libagents)

# Link nlohmann_json (required by config.hpp)
target_link_libraries(libagents PUBLIC nlohmann_json::nlohmann_json)

target_include_directories(libagents
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/src
)

# Compile definitions and link libraries for available providers
if(HAVE_COPILOT_SDK)
    target_compile_definitions(libagents PUBLIC LIBAGENTS_HAS_COPILOT)
    target_link_libraries(libagents PUBLIC copilot_sdk_cpp)
endif()

if(HAVE_CLAUDE_SDK)
    target_compile_definitions(libagents PUBLIC LIBAGENTS_HAS_CLAUDE)
    target_link_libraries(libagents PUBLIC claude_sdk)
endif()

# =============================================================================
# Installation (only when built as top-level or explicitly requested)
# =============================================================================
# Note: Full find_package() support will be added when libagentscpp becomes
# a standalone submodule. For now, just install headers and library.
if(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
    include(GNUInstallDirs)

    # Install headers
    install(DIRECTORY include/
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
        FILES_MATCHING PATTERN "*.hpp"
    )

    # Install library
    install(TARGETS libagents
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    )
endif()

# =============================================================================
# CLI Tool
# =============================================================================
option(LIBAGENTS_BUILD_CLI "Build libagents_cli diagnostic tool" ON)

if(LIBAGENTS_BUILD_CLI AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/tools/libagents_cli.cpp")
    add_executable(libagents_cli tools/libagents_cli.cpp)
    target_link_libraries(libagents_cli PRIVATE libagents)

    # Install CLI if we're the top-level project
    if(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
        install(TARGETS libagents_cli
            RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        )
    endif()
endif()

# =============================================================================
# Examples
# =============================================================================
option(LIBAGENTS_BUILD_EXAMPLES "Build libagents examples" ON)

if(LIBAGENTS_BUILD_EXAMPLES AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples")
    # Claude tool test (diagnostic)
    if(HAVE_CLAUDE_SDK AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/claude_tool_test.cpp")
        add_executable(claude_tool_test examples/claude_tool_test.cpp)
        target_link_libraries(claude_tool_test PRIVATE libagents)
    endif()

    # Basic agent example
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/basic_agent.cpp")
        add_executable(basic_agent examples/basic_agent.cpp)
        target_link_libraries(basic_agent PRIVATE libagents)
    endif()

    # Hosted query example (main thread tool dispatch)
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/hosted_query.cpp")
        add_executable(hosted_query examples/hosted_query.cpp)
        target_link_libraries(hosted_query PRIVATE libagents)
    endif()

    # Events example
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/events.cpp")
        add_executable(events examples/events.cpp)
        target_link_libraries(events PRIVATE libagents)
    endif()

    # Multi-tool example
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/multi_tool.cpp")
        add_executable(multi_tool examples/multi_tool.cpp)
        target_link_libraries(multi_tool PRIVATE libagents)
    endif()
endif()

# =============================================================================
# Summary
# =============================================================================
message(STATUS "libagents configuration:")
message(STATUS "  Version: ${PROJECT_VERSION}")
message(STATUS "  FastMCPP: ${HAVE_FASTMCPP}")
message(STATUS "  Copilot provider: ${HAVE_COPILOT_SDK}")
message(STATUS "  Claude provider: ${HAVE_CLAUDE_SDK}")
message(STATUS "  CLI tool: ${LIBAGENTS_BUILD_CLI}")
message(STATUS "  Examples: ${LIBAGENTS_BUILD_EXAMPLES}")
