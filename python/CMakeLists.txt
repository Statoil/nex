project(pynex)

# Configure setuptools setup.py
set(SETUP_PY_IN "${CMAKE_CURRENT_SOURCE_DIR}/setup.py.in")
set(SETUP_PY    "${CMAKE_CURRENT_BINARY_DIR}/setup.py")
configure_file(${SETUP_PY_IN} ${SETUP_PY})

pybind11_add_module(pynex nex/nex.cpp)
target_link_libraries(pynex PRIVATE nex)
target_include_directories(pynex PUBLIC ${NEX_INCLUDE_DIR})
set_target_properties(pynex PROPERTIES
    LIBRARY_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/nex
)

pybind11_add_module(pynex_no_rpath nex/nex.cpp)
target_link_libraries(pynex_no_rpath PRIVATE nex)
target_include_directories(pynex_no_rpath PUBLIC ${NEX_INCLUDE_DIR})
set_target_properties(pynex_no_rpath PROPERTIES
    LIBRARY_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/build/install/nex
    SKIP_BUILD_RPATH TRUE
    OUTPUT_NAME pynex
)

add_custom_target(
    nex-python ALL
    COMMENT "Building python library with setup.py"
    VERBATIM
    COMMAND ${CMAKE_COMMAND} -E
            copy_directory ${CMAKE_CURRENT_SOURCE_DIR}/test
                           ${CMAKE_CURRENT_BINARY_DIR}/test

    # Install python library to build folder for ctest
    COMMAND ${PYTHON_EXECUTABLE} ${SETUP_PY}
                                 build_ext --rpath $<TARGET_FILE_DIR:nex>
                                 install_lib -d ${CMAKE_CURRENT_BINARY_DIR}

    # Install python library to a predicable location for the final install
    COMMAND ${PYTHON_EXECUTABLE} ${SETUP_PY} install_lib -d build/install
)
add_dependencies(nex-python pynex pynex_no_rpath)

# Install pynex into the directory that is to be installed by python
install(CODE "
if (DEFINED ENV{DESTDIR})
    get_filename_component(abs-destdir \"\$ENV{DESTDIR}\" ABSOLUTE)
    set(root_destdir --root=\${abs-destdir})
endif()
execute_process(COMMAND
    ${PYTHON_EXECUTABLE} ${SETUP_PY}
        install_egg_info --install-dir ${CMAKE_CURRENT_BINARY_DIR}
        egg_info --egg-base ${CMAKE_CURRENT_BINARY_DIR}
        install_lib --build-dir build/install
        install --prefix=${CMAKE_INSTALL_PREFIX}
                --single-version-externally-managed
                --record installed-files
                \${root_destdir}
WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
)")

if(NOT BUILD_TESTING)
    return()
endif()

file(GLOB plts "${testdata}/*.plt" "${testdata}/*.csv")
foreach (plt ${plts})
    get_filename_component(fl ${plt} NAME)
    configure_file(${plt} test-data/${fl} COPYONLY)
endforeach ()

set(py-test ${PYTHON_EXECUTABLE} -m unittest -v)
add_test(NAME python.historical COMMAND ${py-test} test.historical)
add_test(NAME python.load COMMAND ${py-test} test.load)
add_test(NAME python.nex2ecl COMMAND ${py-test} test.nex2ecl)
add_test(NAME python.units COMMAND ${py-test} test.units)
