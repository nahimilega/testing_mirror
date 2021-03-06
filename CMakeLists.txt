################################################################################
# Project: RooUnfold
################################################################################

project(RooUnfold)

if (${RunningAtlasCI})
  message("Setup for CI in ATLAS analysis release")
  include (CMakeLists_forAtlasCI.txt)
endif()


cmake_minimum_required(VERSION 2.8)

find_package(AnalysisBase QUIET)
find_package(AthAnalysis QUIET)

if (${AnalysisBase_FOUND} OR ${AthAnalysis_FOUND})
  message ("Configuring for build within analysis release")
  set (foundAnalysisRelease  1)
else()
  set (foundAnalysisRelease  0)
endif()

# The project version number.
set(VERSION_MAJOR   1   CACHE STRING "Project major version number.")
set(VERSION_MINOR   12   CACHE STRING "Project minor version number.")
mark_as_advanced(VERSION_MAJOR VERSION_MINOR)

find_package( ROOT COMPONENTS Tree Unfold Matrix Hist RIO MathCore Physics RooFitCore RooFit Graf Postscript Gpad)

file(GLOB RooUnfoldLinkDef src/*_LinkDef.h)
file(GLOB RooUnfoldSources src/*.cxx)
file(GLOB RooUnfoldHeaders src/*.h)
list(REMOVE_ITEM RooUnfoldHeaders ${RooUnfoldLinkDef})
#file(GLOB RooUnfoldExecSources examples/*.cxx)
file(GLOB RooUnfoldExecSources test/src/RooUnfoldTest.cxx test/src/RooUnfoldTest2D.cxx test/src/RooUnfoldTest3D.cxx)
file(GLOB RooUnfoldUnitTests test/*.cxx)

# -fprofile-arcs -ftest-coverage
SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${EXTRA_FLAGS}  -ftest-coverage  -fprofile-arcs ")
set(CMAKE_CXX_OUTPUT_EXTENSION_REPLACE ON)
if(${foundAnalysisRelease})
  atlas_subdir( RooUnfold )

  atlas_add_root_dictionary( RooUnfold _dictSource
    ROOT_HEADERS
    ${RooUnfoldHeaders}
    ${RooUnfoldLinkDef})

  # Ensure that the 'RooUnfold' folder expected by atlas_add_library exists at source level
  # by linking the appropriate files from src/
  set( _RooUnfold_header_dir ${CMAKE_CURRENT_SOURCE_DIR}/RooUnfold )
  file(MAKE_DIRECTORY ${_RooUnfold_header_dir})
  execute_process( COMMAND ln -sf ${RooUnfoldHeaders} -t ${_RooUnfold_header_dir} )


  atlas_add_library( RooUnfold
    ${RooUnfoldHeaders} ${RooUnfoldSources} ${_dictSource}
    PUBLIC_HEADERS RooUnfold
    PRIVATE_INCLUDE_DIRS ${ROOT_INCLUDE_DIRS}
    PRIVATE_LINK_LIBRARIES ${ROOT_LIBRARIES})

  foreach(ExecSource ${RooUnfoldExecSources})
    get_filename_component(ExecName ${ExecSource} NAME_WE)
    atlas_add_executable( ${ExecName} ${ExecSource}
      INCLUDE_DIRS ${ROOT_INCLUDE_DIRS} RooUnfold ${CMAKE_CURRENT_SOURCE_DIR}/examples ${CMAKE_CURRENT_SOURCE_DIR}/src
      LINK_LIBRARIES ${ROOT_LIBRARIES} RooUnfold)
  endforeach()
else()

  execute_process( COMMAND ln -sf ${RooUnfoldHeaders} -t ${CMAKE_CURRENT_BINARY_DIR} )
  set(SETUP ${CMAKE_CURRENT_BINARY_DIR}/setup.sh)
  file(WRITE ${SETUP} "#!/bin/bash\n")
  file(APPEND ${SETUP} "# this is an auto-generated setup script\n" )


  # register all the files and directories
  include_directories ("${CMAKE_CURRENT_SOURCE_DIR}/src")
  include_directories ("${ROOT_INCLUDE_DIRS}")

  file(APPEND ${SETUP} "export PATH=\${PATH}:${CMAKE_CURRENT_BINARY_DIR}\n")
  file(APPEND ${SETUP} "export PYTHONPATH=\${PYTHONPATH}:${CMAKE_CURRENT_BINARY_DIR}\n")
  file(APPEND ${SETUP} "export LD_LIBRARY_PATH=\${LD_LIBRARY_PATH}:${CMAKE_CURRENT_BINARY_DIR}\n")

  # generate the dictionary source code
  include(${ROOT_USE_FILE})
  if(SKBUILD)
    # Relocatable installations need relative paths
    set(RelativeRooUnfoldHeaders "")
    foreach(path ${RooUnfoldHeaders})
      get_filename_component(barename ${path} NAME)
      list(APPEND RelativeRooUnfoldHeaders ${barename})
    endforeach()
    ROOT_GENERATE_DICTIONARY(G__RooUnfold ${RelativeRooUnfoldHeaders} LINKDEF ${RooUnfoldLinkDef} OPTIONS ${EXTRA_FLAGS})
    unset(barename)
    unset(RelativeRooUnfoldHeaders)
  else()
    ROOT_GENERATE_DICTIONARY(G__RooUnfold ${RooUnfoldHeaders} LINKDEF ${RooUnfoldLinkDef} OPTIONS ${EXTRA_FLAGS})
  endif()

  # register the shared object to include both sources and dictionaries
  add_library( RooUnfold SHARED ${RooUnfoldSources} G__RooUnfold.cxx)

  # link everything together at the end
  target_link_libraries( RooUnfold ${ROOT_LIBRARIES} )

  # Add all targets to the build-tree export set
  export(TARGETS RooUnfold FILE "${PROJECT_BINARY_DIR}/RooUnfoldTargets.cmake")

  # Export the package for use from the build-tree
  # (this registers the build-tree with a global CMake-registry)
  export(PACKAGE RooUnfold)

  set(CONF_INCLUDE_DIRS "${CMAKE_CURRENT_SOURCE_DIR}" "${PROJECT_BINARY_DIR}")
  set(CONF_LIBRARY_DIRS "${PROJECT_BINARY_DIR}")
  set(CONF_LIBRARIES    RooUnfold)
  configure_file(RooUnfoldConfig.cmake.in
    "${PROJECT_BINARY_DIR}/RooUnfoldConfig.cmake" @ONLY)

  if(NOT SKBUILD)
  # Install the RooUnfoldConfig.cmake and RooUnfoldConfigVersion.cmake
#  install(FILES
#    "${PROJECT_BINARY_DIR}/${CMAKE_FILES_DIRECTORY}/RooUnfoldConfig.cmake"
#    "${PROJECT_BINARY_DIR}/RooUnfoldConfigVersion.cmake"
#    DESTINATION "${PROJECT_SOURCE_DIR}" COMPONENT dev)
  install(FILES
    "${PROJECT_BINARY_DIR}/RooUnfoldConfig.cmake"
    DESTINATION "${PROJECT_SOURCE_DIR}" COMPONENT dev)
  endif()

  include(CTest)
  enable_testing()
  
  foreach(ExecSource ${RooUnfoldExecSources})
    get_filename_component(ExecName ${ExecSource} NAME_WE)
    add_executable( ${ExecName} ${ExecSource} )
    target_link_libraries ( ${ExecName} RooUnfold ${ROOT_LIBRARIES} gcov)
  endforeach()
  add_subdirectory(test)
endif()



foreach(UnitTest ${RooUnfoldUnitTests})
    get_filename_component(Test ${UnitTest} NAME_WE)
    add_executable(${Test} ${UnitTest})
    target_link_libraries(${Test} PUBLIC tests gcov)
endforeach()

add_test(Bayes test_bayes)
#add_test(SVD test_svd)
#add_test(Invert test_invert)
#add_test(BinByBin test_bbb)

file(GLOB test_methods "test/test_methods.py")
  add_test(
    NAME test_methods
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/build
    COMMAND python3 ${test_methods}
    )

file(GLOB test_fakes "test/test_fakes.py")
add_test(
  NAME test_fakes
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/build
  COMMAND python3 ${test_fakes}
  )

file(GLOB test_bin_correlation "test/test_bin_correlation.py")
add_test(
  NAME test_bin_correlation
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/build
  COMMAND python3 ${test_bin_correlation}
  )

file(GLOB test_uncertainty "test/test_uncertainty.py")
add_test(
  NAME test_uncertainty
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/build
  COMMAND python3 ${test_uncertainty}
  )

file(GLOB test_overflow "test/test_overflow.py")
add_test(
  NAME test_overflow
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/build
  COMMAND python3 ${test_overflow}
)

file(GLOB test_2D "test/test_2D.py")
add_test(
  NAME test_2D
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/build
  COMMAND python3 ${test_2D}
)

file(GLOB test_3D "test/test_3D.py")
add_test(
  NAME test_3D
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/build
  COMMAND python3 ${test_3D}
)

# first we can indicate the documentation build as an option and set it to ON by default
option(BUILD_DOC "Build documentation" ON)

# check if Doxygen is installed
find_package(Doxygen)
if (DOXYGEN_FOUND)
    # set input and output files
    set(DOXYGEN_IN ${CMAKE_CURRENT_SOURCE_DIR}/Doxyfile.cmake)
    set(DOXYGEN_OUT ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile)

    # request to configure the file
    configure_file(${DOXYGEN_IN} ${DOXYGEN_OUT} @ONLY)
    message("Doxygen build started")

    # note the option ALL which allows to build the docs together with the application
    add_custom_target( doxygen ALL
        COMMAND ${DOXYGEN_EXECUTABLE} ${DOXYGEN_OUT}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
        COMMENT "Generating API documentation with Doxygen"
        VERBATIM )
else (DOXYGEN_FOUND)
  message("Doxygen need to be installed to generate the doxygen documentation")
endif (DOXYGEN_FOUND)

if (SKBUILD)
    # install the compiled python module in src/RooUnfold to be picked up by the python wheels setup
    install(TARGETS RooUnfold LIBRARY DESTINATION src/RooUnfold)
    install(FILES ${RooUnfoldHeaders}
      ${CMAKE_CURRENT_BINARY_DIR}/G__RooUnfold.cxx
      ${CMAKE_CURRENT_BINARY_DIR}/libRooUnfold.rootmap
      ${CMAKE_CURRENT_BINARY_DIR}/libRooUnfold_rdict.pcm DESTINATION
      src/RooUnfold)

    # Set the RPATH
    if (APPLE)
      set(rpath "@loader_path")
    else()
      set(rpath "$ORIGIN")
    endif()
    set_target_properties(RooUnfold PROPERTIES INSTALL_RPATH ${rpath})
endif()
