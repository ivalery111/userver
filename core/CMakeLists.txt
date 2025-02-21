project(userver-core CXX)

include(GetUserverVersion)
message(STATUS "Userver version ${USERVER_VERSION}")

include(CheckFunctionExists)
check_function_exists("accept4" HAVE_ACCEPT4)
check_function_exists("pipe2" HAVE_PIPE2)

set(BUILD_CONFIG ${CMAKE_CURRENT_BINARY_DIR}/build_config.hpp)
if(${CMAKE_SOURCE_DIR}/.git/HEAD IS_NEWER_THAN ${BUILD_CONFIG})
  configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/build_config.hpp.in
    ${BUILD_CONFIG}
    ESCAPE_QUOTES @ONLY)
endif()

file(GLOB_RECURSE SOURCES
  ${CMAKE_CURRENT_SOURCE_DIR}/include/*pp
  ${CMAKE_CURRENT_SOURCE_DIR}/internal/*pp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/*pp
)

file(GLOB_RECURSE UNIT_TEST_SOURCES
  ${CMAKE_CURRENT_SOURCE_DIR}/*_test.cpp
)

file(GLOB_RECURSE LIBUTEST_SOURCES
  ${CMAKE_CURRENT_SOURCE_DIR}/testing/*.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/testing/*.hpp
)
list(REMOVE_ITEM LIBUTEST_SOURCES ${UNIT_TEST_SOURCES})

list(REMOVE_ITEM SOURCES ${UNIT_TEST_SOURCES})

file(GLOB_RECURSE BENCH_SOURCES
  ${CMAKE_CURRENT_SOURCE_DIR}/*_benchmark.cpp
)
file(GLOB_RECURSE LIBUBENCH_SOURCES
  ${CMAKE_CURRENT_SOURCE_DIR}/src/core_benchmark.cpp
)
list (REMOVE_ITEM SOURCES ${BENCH_SOURCES} ${LIBUBENCH_SOURCES})

file(GLOB_RECURSE INTERNAL_SOURCES
  ${CMAKE_CURRENT_SOURCE_DIR}/internal/*.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/internal/*.hpp
)

list (REMOVE_ITEM SOURCES ${INTERNAL_SOURCES})

find_package(Boost REQUIRED COMPONENTS
    program_options
    filesystem
    locale
    regex
    iostreams
)
find_package_required(ZLIB "zlib1g-dev")

find_package(Iconv REQUIRED)
find_package_required(OpenSSL "libssl-dev")

if (USERVER_CONAN)
    find_package(c-ares REQUIRED)
    find_package(CURL REQUIRED)
    find_package(cryptopp REQUIRED)
    find_package(spdlog REQUIRED)
    find_package(http_parser REQUIRED)
    find_package(libnghttp2 REQUIRED)
    find_package(libev REQUIRED)

    find_package(concurrentqueue REQUIRED)
else()
    include(SetupCAres)
    include(SetupCURL)
    include(SetupCryptoPP)
    include(SetupSpdlog)
    find_package(Http_Parser REQUIRED)
    find_package(Nghttp2 REQUIRED)
    find_package(LibEv REQUIRED)
endif()

add_library(${PROJECT_NAME} STATIC ${SOURCES})

target_compile_definitions(${PROJECT_NAME}
  PRIVATE
    SPDLOG_PREVENT_CHILD_FD
    SPDLOG_FMT_EXTERNAL
    [[SPDLOG_LEVEL_NAMES={"TRACE", "DEBUG", "INFO", "WARNING", "ERROR", "CRITICAL", "OFF" }]]
)

if (USERVER_DISABLE_PHDR_CACHE)
  target_compile_definitions(${PROJECT_NAME} PRIVATE USERVER_DISABLE_PHDR_CACHE)
endif()



target_link_libraries(${PROJECT_NAME}
  PUBLIC
    userver-universal
    Boost::locale
    CURL::libcurl
  PRIVATE
    Boost::filesystem
    Boost::program_options
    Boost::iostreams
    Boost::regex
    Iconv::Iconv
    OpenSSL::Crypto
    OpenSSL::SSL
    ZLIB::ZLIB
)

if (USERVER_FEATURE_UBOOST_CORO)
    add_subdirectory(${USERVER_THIRD_PARTY_DIRS}/uboost_coro uboost_coro_build)
    target_link_libraries(${PROJECT_NAME}
      PRIVATE
        userver-uboost-coro
    )
    target_include_directories(${PROJECT_NAME} PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/uboost_coro/include
    )
else()
    find_package(Boost REQUIRED COMPONENTS
        context
        coroutine
    )
    target_link_libraries(${PROJECT_NAME}
      PRIVATE
        Boost::context
    )
    target_include_directories(${PROJECT_NAME} PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/sys_coro/include
    )
endif()

if (USERVER_CONAN)
    target_link_libraries(${PROJECT_NAME}
      PUBLIC
        c-ares::cares
        concurrentqueue::concurrentqueue
      PRIVATE
        cryptopp::cryptopp
        http_parser::http_parser
        libev::libev
        libnghttp2::nghttp2
        spdlog::spdlog_header_only
    )
else()
    target_link_libraries(${PROJECT_NAME}
      PUBLIC
        c-ares::cares
      PRIVATE
        CryptoPP
        Http_Parser
        Nghttp2
        LibEv
        spdlog_header_only
    )

    target_include_directories(${PROJECT_NAME} SYSTEM PUBLIC
        ${USERVER_THIRD_PARTY_DIRS}/moodycamel/include
    )
endif()

if (NOT MACOS AND NOT ${CMAKE_SYSTEM_NAME} MATCHES "BSD")
  target_link_libraries(${PROJECT_NAME} PUBLIC atomic)
endif()

target_include_directories(${PROJECT_NAME} PUBLIC
  ${CMAKE_CURRENT_SOURCE_DIR}/include
  ${USERVER_THIRD_PARTY_DIRS}/pfr/include
)
target_include_directories(${PROJECT_NAME} SYSTEM BEFORE PUBLIC
  ${CMAKE_CURRENT_SOURCE_DIR}/libc_include_fixes
)

# The bug is only triggered with optimizations enabled -- TAXICOMMON-1729
set_property(
  SOURCE ${CMAKE_CURRENT_SOURCE_DIR}/src/engine/errno_test.cpp
  APPEND PROPERTY COMPILE_FLAGS -O2
)

target_include_directories(${PROJECT_NAME}
  PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/../universal/src/
    ${CMAKE_CURRENT_SOURCE_DIR}/src/
    ${CMAKE_CURRENT_BINARY_DIR}
)

if (USERVER_IS_THE_ROOT_PROJECT OR USERVER_FEATURE_UTEST)
  add_library(userver-core-internal STATIC ${INTERNAL_SOURCES})
  target_compile_definitions(userver-core-internal PUBLIC $<TARGET_PROPERTY:${PROJECT_NAME},COMPILE_DEFINITIONS>)
  target_include_directories(userver-core-internal PUBLIC
    $<TARGET_PROPERTY:${PROJECT_NAME},INCLUDE_DIRECTORIES>
    ${CMAKE_CURRENT_SOURCE_DIR}/internal/include
    ${CMAKE_CURRENT_SOURCE_DIR}/../universal/internal/include
  )
  target_link_libraries(userver-core-internal
    PUBLIC
      ${PROJECT_NAME}
  )
endif()

if (USERVER_FEATURE_UTEST)
    add_library(userver-utest STATIC ${LIBUTEST_SOURCES})
    target_compile_definitions(userver-utest PUBLIC $<TARGET_PROPERTY:${PROJECT_NAME},COMPILE_DEFINITIONS>)

    target_link_libraries(userver-utest
      PUBLIC
        ${PROJECT_NAME}
      PRIVATE
        userver-core-internal
        Boost::program_options
        userver-universal-internal-utest
    )

    target_include_directories(userver-utest PUBLIC
        ${CMAKE_CURRENT_SOURCE_DIR}/testing/include
    )
    target_include_directories(userver-utest PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/src/
        ${CMAKE_CURRENT_SOURCE_DIR}/testing/src/
    )
endif()

if (USERVER_IS_THE_ROOT_PROJECT)
    add_executable(${PROJECT_NAME}_unittest ${UNIT_TEST_SOURCES})
    target_include_directories (${PROJECT_NAME}_unittest PRIVATE
        $<TARGET_PROPERTY:${PROJECT_NAME},INCLUDE_DIRECTORIES>
    )
    target_link_libraries(${PROJECT_NAME}_unittest PRIVATE
      userver-utest
      userver-core-internal
    )

    target_compile_definitions(${PROJECT_NAME}_unittest PRIVATE
      DEFAULT_DYNAMIC_CONFIG_FILENAME="${CMAKE_SOURCE_DIR}/core/tests/dynamic_config_fallback.json"
    )

    # We keep testing deprecated functions, no need to warn about that
    target_compile_options(${PROJECT_NAME}_unittest PRIVATE "-Wno-deprecated-declarations")
    add_google_tests(${PROJECT_NAME}_unittest)
    add_subdirectory(functional_tests)
endif()

if (USERVER_FEATURE_UTEST)
    add_library(userver-ubench ${LIBUBENCH_SOURCES})
    target_include_directories(userver-ubench PUBLIC $<TARGET_PROPERTY:${PROJECT_NAME},INCLUDE_DIRECTORIES>)
    target_compile_definitions(userver-ubench PUBLIC $<TARGET_PROPERTY:${PROJECT_NAME},COMPILE_DEFINITIONS>)
    target_link_libraries(userver-ubench
      PUBLIC
        ${PROJECT_NAME}
      PRIVATE
        userver-core-internal
        userver-universal-internal-ubench
    )
endif()

if (USERVER_IS_THE_ROOT_PROJECT)
    add_executable(${PROJECT_NAME}_benchmark ${BENCH_SOURCES})
    #    find_package(absl REQUIRED)

    target_link_libraries(${PROJECT_NAME}_benchmark
      PUBLIC
        userver-ubench
      PRIVATE
      #        absl::raw_hash_set
        userver-core-internal
    )
    add_google_benchmark_tests(${PROJECT_NAME}_benchmark)
endif()
