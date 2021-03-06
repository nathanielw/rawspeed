file(STRINGS "${CMAKE_CURRENT_SOURCE_DIR}/all-fuzzers.txt" ALL_EXPECTED_FUZZERS)
list(SORT ALL_EXPECTED_FUZZERS)

set(ALL_FUZZERS "" CACHE INTERNAL "" FORCE)

if(NOT (CMAKE_CXX_COMPILER_ID STREQUAL "Clang" AND CMAKE_BUILD_TYPE STREQUAL "FUZZ"))
  message(WARNING "For fuzzets to function, compiler must be Clang, and build type must be FUZZ.")
  add_library(libFuzzer_dummy STATIC "${CMAKE_CURRENT_SOURCE_DIR}/libFuzzer_dummy_main.cpp")
  target_link_libraries(libFuzzer_dummy rawspeed)

  if(WITH_OPENMP AND OPENMP_FOUND AND TARGET OpenMP::OpenMP)
    target_link_libraries(libFuzzer_dummy OpenMP::OpenMP)
  endif()

  add_library(libFuzzer ALIAS libFuzzer_dummy)

  macro(add_fuzz_target fuzzer)
    list(APPEND ALL_FUZZERS "${fuzzer}")
    set(ALL_FUZZERS "${ALL_FUZZERS}" CACHE INTERNAL "" FORCE)

    add_test(NAME fuzzers/${fuzzer} COMMAND ${fuzzer})
  endmacro()
else()
  add_library(libFuzzer STATIC IMPORTED "${LIBFUZZER_ARCHIVE}")
  set_property(TARGET libFuzzer PROPERTY IMPORTED_LOCATION "${LIBFUZZER_ARCHIVE}")

  file(GLOB FUZZ_DICT "common.dict")

  macro(add_fuzz_target fuzzer)
    list(APPEND ALL_FUZZERS ${fuzzer})
    set(ALL_FUZZERS "${ALL_FUZZERS}" CACHE INTERNAL "" FORCE)

    install(TARGETS ${fuzzer} DESTINATION "${CMAKE_INSTALL_BINDIR}")
    install(FILES "${FUZZ_DICT}" DESTINATION "${CMAKE_INSTALL_BINDIR}" RENAME "${fuzzer}.dict")
  endmacro()
endif()

add_custom_target(fuzzers ALL)

add_subdirectory(librawspeed)
add_subdirectory(rawspeed)

add_subdirectory(corpora)

list(SORT ALL_FUZZERS)
if(NOT "${ALL_FUZZERS}" STREQUAL "${ALL_EXPECTED_FUZZERS}")
  message(SEND_ERROR "Actual list of fuzzers does not match the expected list: "
    "${ALL_FUZZERS} vs ${ALL_EXPECTED_FUZZERS}")
endif()
