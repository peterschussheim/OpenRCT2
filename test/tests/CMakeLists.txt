cmake_minimum_required(VERSION 2.6)

option(SYSTEM_GTEST "Use the googletest library provided by the system.")

if (SYSTEM_GTEST)
    find_package(GTest REQUIRED)

    set(GTEST_LIBRARIES ${GTEST_BOTH_LIBRARIES})

    message(WARNING "Gtest strongly advices against using a system installation, see https://github.com/google/googletest/blob/master/googletest/docs/FAQ.md#why-is-it-not-recommended-to-install-a-pre-compiled-copy-of-google-test-for-example-into-usrlocal for detailed information. If errors occur please double-check without the SYSTEM_GTEST flag.")
else (SYSTEM_GTEST)

        # Bootstrap GoogleTest
        INCLUDE(ExternalProject)

        # Use googletest with https://github.com/google/googletest/pull/1045 applied
        ExternalProject_Add(
                googletest-distribution
                URL https://github.com/google/googletest/archive/ad49eaa0466b98f21449efaaaae817683daa8b99.tar.gz
                URL_HASH SHA1=e468bfdffe7c46c14ce06bfb48a29b6201908504
                TIMEOUT 10
                CONFIGURE_COMMAND ""
                BUILD_COMMAND ""
                INSTALL_COMMAND ""
        )

        # Specify include dir
        ExternalProject_Get_Property(googletest-distribution SOURCE_DIR)
        set(GOOGLETEST_DISTRIB_SOURCE_DIR "${SOURCE_DIR}")

        ExternalProject_Add(
                googletest
                DEPENDS googletest-distribution
                DOWNLOAD_COMMAND ""
                SOURCE_DIR "${GOOGLETEST_DISTRIB_SOURCE_DIR}/googletest"
                CMAKE_ARGS "-DCMAKE_CXX_FLAGS=${TARGET_M}"
                BUILD_BYPRODUCTS "googletest-prefix/src/googletest-build/${CMAKE_STATIC_LIBRARY_PREFIX}gtest${CMAKE_STATIC_LIBRARY_SUFFIX}"
                BUILD_BYPRODUCTS "googletest-prefix/src/googletest-build/${CMAKE_STATIC_LIBRARY_PREFIX}gtest_main${CMAKE_STATIC_LIBRARY_SUFFIX}"
                # Disable install step
                INSTALL_COMMAND ""
                # Wrap download, configure and build steps in a script to log output
                LOG_DOWNLOAD ON
                LOG_CONFIGURE ON
                LOG_BUILD ON)


        # Specify include dir
        set(GTEST_INCLUDE_DIRS ${GOOGLETEST_DISTRIB_SOURCE_DIR}/googletest/include)

        # Library
        ExternalProject_Get_Property(googletest BINARY_DIR)
        set(GOOGLETEST_BINARY_DIR "${BINARY_DIR}")
        set(GTEST_LIBRARY_PATH ${GOOGLETEST_BINARY_DIR}/${CMAKE_STATIC_LIBRARY_PREFIX}gtest${CMAKE_STATIC_LIBRARY_SUFFIX})
        set(GTEST_MAIN_LIBRARY_PATH ${GOOGLETEST_BINARY_DIR}/${CMAKE_STATIC_LIBRARY_PREFIX}gtest_main${CMAKE_STATIC_LIBRARY_SUFFIX})
        set(GTEST_LIBRARY gtest)
        set(GTEST_MAIN_LIBRARY gtest_main)
        add_library(${GTEST_LIBRARY} STATIC IMPORTED)
        add_library(${GTEST_MAIN_LIBRARY} STATIC IMPORTED)
        set_property(TARGET ${GTEST_LIBRARY} PROPERTY IMPORTED_LOCATION ${GTEST_LIBRARY_PATH})
        set_property(TARGET ${GTEST_MAIN_LIBRARY} PROPERTY IMPORTED_LOCATION ${GTEST_MAIN_LIBRARY_PATH})
        add_dependencies(${GTEST_LIBRARY} googletest)
        add_dependencies(${GTEST_MAIN_LIBRARY} ${GTEST_LIBRARY})

        set(GTEST_LIBRARIES gtest gtest_main pthread)

endif (SYSTEM_GTEST)

include_directories(SYSTEM ${GTEST_INCLUDE_DIRS})
include_directories("${ROOT_DIR}/src")
include_directories(SYSTEM ${SDL2_INCLUDE_DIRS})

# Some most common files required in tests
set(COMMON_TEST_SOURCES
    "${ROOT_DIR}/src/openrct2/core/Console.cpp"
    "${ROOT_DIR}/src/openrct2/core/Diagnostics.cpp"
    "${ROOT_DIR}/src/openrct2/core/Guard.cpp"
    "${ROOT_DIR}/src/openrct2/core/String.cpp"
    "${ROOT_DIR}/src/openrct2/Diagnostic.cpp"
    "${ROOT_DIR}/src/openrct2/localisation/ConversionTables.cpp"
    "${ROOT_DIR}/src/openrct2/localisation/Convert.cpp"
    "${ROOT_DIR}/src/openrct2/localisation/FormatCodes.cpp"
    "${ROOT_DIR}/src/openrct2/localisation/UTF8.cpp"
    "${ROOT_DIR}/src/openrct2/util/Util.cpp"
    "${ROOT_DIR}/src/openrct2/Version.cpp"
    )

# Create a re-usable library to save some compilation time
add_library(test-common STATIC ${COMMON_TEST_SOURCES})
set_target_properties(test-common PROPERTIES COMPILE_DEFINITIONS DISABLE_HTTP)

# Setup testdata. It should be fine here, as the only way to reach here is by explicitly requesting tests.
execute_process(COMMAND ${CMAKE_COMMAND} -E create_symlink "${CMAKE_CURRENT_LIST_DIR}/testdata" "${CMAKE_CURRENT_BINARY_DIR}/testdata")
install(CODE "execute_process(COMMAND \"${CMAKE_COMMAND}\" -E create_symlink \"\$ENV{DESTDIR}${CMAKE_INSTALL_FULL_DATADIR}/openrct2\" \"${CMAKE_CURRENT_BINARY_DIR}/data\")")

if (NOT MINGW AND NOT MSVC)
    # For unicode code page conversion (required for ini and string tests)
    find_package(ICU 59.0 REQUIRED COMPONENTS uc)
    target_link_libraries(test-common ${ICU_LIBRARIES})
    target_include_directories(test-common SYSTEM PUBLIC ${ICU_INCLUDE_DIRS})
endif ()

# Start of our tests

# sawyercoding test

set(SAWYERCODING_TEST_SOURCES
        "${CMAKE_CURRENT_LIST_DIR}/sawyercoding_test.cpp"
        )
add_executable(test_sawyercoding ${SAWYERCODING_TEST_SOURCES})
target_link_libraries(test_sawyercoding ${GTEST_LIBRARIES} libopenrct2 ${LDL} z)
target_link_platform_libraries(test_sawyercoding)
add_test(NAME sawyercoding COMMAND test_sawyercoding)

# LanguagePack test
set(LANGUAGEPACK_TEST_SOURCES
        "${CMAKE_CURRENT_LIST_DIR}/LanguagePackTest.cpp"
        )
add_executable(test_languagepack ${LANGUAGEPACK_TEST_SOURCES})
if (UNIX AND NOT ${CMAKE_SYSTEM_NAME} MATCHES "BSD")
    # Include libdl for dlopen
    set(LDL dl)
endif ()
target_link_libraries(test_languagepack ${GTEST_LIBRARIES} libopenrct2 ${LDL} z)
target_link_platform_libraries(test_languagepack)
add_test(NAME languagepack COMMAND test_languagepack)

# INI test
set(INI_TEST_SOURCES
        "${CMAKE_CURRENT_LIST_DIR}/IniWriterTest.cpp"
        "${CMAKE_CURRENT_LIST_DIR}/IniReaderTest.cpp"
        )
add_executable(test_ini ${INI_TEST_SOURCES})
SET_CHECK_CXX_FLAGS(test_ini)
target_link_libraries(test_ini ${GTEST_LIBRARIES} libopenrct2 ${LDL} z)
target_link_platform_libraries(test_ini)
add_test(NAME ini COMMAND test_ini)

# Platform
add_executable(test_platform ${CMAKE_CURRENT_LIST_DIR}/Platform.cpp)
SET_CHECK_CXX_FLAGS(test_platform)
target_link_libraries(test_platform ${GTEST_LIBRARIES} test-common ${LDL} z libopenrct2)
target_link_platform_libraries(test_platform)
add_test(NAME platform COMMAND test_platform)

# String test
set(STRING_TEST_SOURCES
        "${CMAKE_CURRENT_LIST_DIR}/StringTest.cpp"
        )
add_executable(test_string ${STRING_TEST_SOURCES})
SET_CHECK_CXX_FLAGS(test_string)
target_link_libraries(test_string ${GTEST_LIBRARIES} libopenrct2 ${LDL} z)
target_link_platform_libraries(test_string)
add_test(NAME string COMMAND test_string)

# Formatting tests
set(STRING_TEST_SOURCES "${CMAKE_CURRENT_LIST_DIR}/FormattingTests.cpp")
add_executable(test_formatting ${STRING_TEST_SOURCES})
SET_CHECK_CXX_FLAGS(test_formatting)
target_link_libraries(test_formatting ${GTEST_LIBRARIES} libopenrct2 ${LDL} z)
target_link_platform_libraries(test_formatting)
add_test(NAME formatting COMMAND test_formatting)

# Localisation test
set(STRING_TEST_SOURCES "${CMAKE_CURRENT_LIST_DIR}/Localisation.cpp")
add_executable(test_localisation ${STRING_TEST_SOURCES})
SET_CHECK_CXX_FLAGS(test_localisation)
target_link_libraries(test_localisation ${GTEST_LIBRARIES} libopenrct2 ${LDL} z)
target_link_platform_libraries(test_localisation)
add_test(NAME localisation COMMAND test_localisation)

if (NOT DISABLE_NETWORK)
    # Crypt tests
    add_executable(test_crypt "${CMAKE_CURRENT_LIST_DIR}/CryptTests.cpp"
                              "${CMAKE_CURRENT_LIST_DIR}/TestData.cpp")
    SET_CHECK_CXX_FLAGS(test_crypt)
    target_link_libraries(test_crypt ${GTEST_LIBRARIES} libopenrct2)
    target_link_platform_libraries(test_crypt)
    add_test(NAME Crypt COMMAND test_crypt)
endif ()

# ImageImporter tests
add_executable(test_imageimporter "${CMAKE_CURRENT_LIST_DIR}/ImageImporterTests.cpp"
                                  "${CMAKE_CURRENT_LIST_DIR}/TestData.cpp")
SET_CHECK_CXX_FLAGS(test_imageimporter)
target_link_libraries(test_imageimporter ${GTEST_LIBRARIES} libopenrct2)
target_link_platform_libraries(test_imageimporter)
add_test(NAME ImageImporter COMMAND test_imageimporter)

# Ride ratings test
set(RIDE_RATINGS_TEST_SOURCES "${CMAKE_CURRENT_LIST_DIR}/RideRatings.cpp"
                              "${CMAKE_CURRENT_LIST_DIR}/TestData.cpp")
add_executable(test_ride_ratings ${RIDE_RATINGS_TEST_SOURCES})
SET_CHECK_CXX_FLAGS(test_ride_ratings)
target_link_libraries(test_ride_ratings ${GTEST_LIBRARIES} libopenrct2 ${LDL} z)
target_link_platform_libraries(test_ride_ratings)
add_test(NAME ride_ratings COMMAND test_ride_ratings)

# Multi-launch test
set(MULTILAUNCH_TEST_SOURCES "${CMAKE_CURRENT_LIST_DIR}/MultiLaunch.cpp"
                             "${CMAKE_CURRENT_LIST_DIR}/TestData.cpp")
add_executable(test_multilaunch ${MULTILAUNCH_TEST_SOURCES})
SET_CHECK_CXX_FLAGS(test_multilaunch)
target_link_libraries(test_multilaunch ${GTEST_LIBRARIES} libopenrct2 ${LDL} z)
target_link_platform_libraries(test_multilaunch)
add_test(NAME multilaunch COMMAND test_multilaunch)

# Tile element test
set(TILE_ELEMENT_TEST_SOURCES "${CMAKE_CURRENT_LIST_DIR}/TileElements.cpp"
                              "${CMAKE_CURRENT_LIST_DIR}/TestData.cpp")
add_executable(test_tile_elements ${TILE_ELEMENT_TEST_SOURCES})
SET_CHECK_CXX_FLAGS(test_tile_elements)
target_link_libraries(test_tile_elements ${GTEST_LIBRARIES} libopenrct2 ${LDL} z)
target_link_platform_libraries(test_tile_elements)
add_test(NAME tile_elements COMMAND test_tile_elements)

# Replay tests
set(REPLAY_TEST_SOURCES "${CMAKE_CURRENT_LIST_DIR}/ReplayTests.cpp"
							  "${CMAKE_CURRENT_LIST_DIR}/TestData.cpp")
add_executable(test_replays ${REPLAY_TEST_SOURCES})
SET_CHECK_CXX_FLAGS(test_replays)
target_link_libraries(test_replays ${GTEST_LIBRARIES} libopenrct2 ${LDL} z)
target_link_platform_libraries(test_replays)
add_test(NAME replay_tests COMMAND test_replays)

# Play tests
set(PLAY_TEST_SOURCES "${CMAKE_CURRENT_LIST_DIR}/PlayTests.cpp"
                      "${CMAKE_CURRENT_LIST_DIR}/TestData.cpp")
add_executable(test_plays ${PLAY_TEST_SOURCES})
SET_CHECK_CXX_FLAGS(test_plays)
target_link_libraries(test_plays ${GTEST_LIBRARIES} libopenrct2 ${LDL} z)
target_link_platform_libraries(test_plays)
add_test(NAME play_tests COMMAND test_plays)

# Pathfinding test
set(PATHFINDING_TEST_SOURCES  "${CMAKE_CURRENT_LIST_DIR}/Pathfinding.cpp"
                              "${CMAKE_CURRENT_LIST_DIR}/TestData.cpp")
add_executable(test_pathfinding ${PATHFINDING_TEST_SOURCES})
SET_CHECK_CXX_FLAGS(test_pathfinding)
target_link_libraries(test_pathfinding ${GTEST_LIBRARIES} libopenrct2 ${LDL} z)
target_link_platform_libraries(test_pathfinding)
add_test(NAME pathfinding COMMAND test_pathfinding)

# S6 Import/Export test
set(S6IMPORTEXPORT_TEST_SOURCES "${CMAKE_CURRENT_LIST_DIR}/S6ImportExportTests.cpp"
                                 "${CMAKE_CURRENT_LIST_DIR}/TestData.cpp")
add_executable(test_s6importexporttests ${S6IMPORTEXPORT_TEST_SOURCES})
SET_CHECK_CXX_FLAGS(test_s6importexporttests)
target_link_libraries(test_s6importexporttests ${GTEST_LIBRARIES} libopenrct2 ${LDL} z)
target_link_platform_libraries(test_s6importexporttests)
add_test(NAME s6importexporttests COMMAND test_s6importexporttests)

# EnumMap Test
set(ENUMMAP_TEST_SOURCES "${CMAKE_CURRENT_LIST_DIR}/EnumMapTest.cpp.cpp"
                                 "${CMAKE_CURRENT_LIST_DIR}/TestData.cpp")
add_executable(test_enummap ${S6IMPORTEXPORT_TEST_SOURCES})
SET_CHECK_CXX_FLAGS(test_enummap)
target_link_libraries(test_enummap ${GTEST_LIBRARIES} libopenrct2 ${LDL} z)
target_link_platform_libraries(test_enummap)
add_test(NAME enummaptests COMMAND test_enummap)
