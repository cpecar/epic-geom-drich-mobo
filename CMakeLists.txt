cmake_minimum_required(VERSION 3.12 FATAL_ERROR)

# CMP0074: find_package() uses <PackageName>_ROOT variables
cmake_policy(SET CMP0074 NEW)

PROJECT(epic
  LANGUAGES CXX
  DESCRIPTION "DD4hep Geometry Description of the EPIC Experiment"
  )

# Set version based on git
include(cmake/git_version.cmake)
set_git_version(CMAKE_PROJECT_VERSION)

# C++ standard
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD 17 CACHE STRING "Set the C++ standard to be used")
if(NOT CMAKE_CXX_STANDARD MATCHES "17|20|23")
  message(FATAL_ERROR "Unsupported C++ standard: ${CMAKE_CXX_STANDARD}")
endif()
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Build type
if (NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE RelWithDebInfo CACHE STRING "Build type configuration" FORCE)
  message(STATUS "Setting default build type: ${CMAKE_BUILD_TYPE}")
endif()

# Error on all warnings
if(NOT CMAKE_BUILD_TYPE STREQUAL "Release")
  add_compile_options(-Wall -Wextra -Werror -pedantic)
endif()

# Export compile commands as json for run-clang-tidy
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Install to the top directory by default
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    set(CMAKE_INSTALL_PREFIX ${CMAKE_SOURCE_DIR}/prefix CACHE PATH "Install in $src/prefix/ directory by default" FORCE)
endif()

# Install into GNU standard directories
include(GNUInstallDirs)

# Dependencies
find_package(DD4hep 1.27 REQUIRED COMPONENTS DDCore DDRec)
find_package(fmt REQUIRED)

#-----------------------------------------------------------------------------------
set(a_lib_name ${PROJECT_NAME})

set(DD4HEP_SET_RPATH TRUE)
dd4hep_configure_output(INSTALL ${CMAKE_INSTALL_PREFIX})
dd4hep_set_compiler_flags()

file(GLOB sources CONFIGURE_DEPENDS src/*.cpp)
dd4hep_add_plugin(${a_lib_name}
  SOURCES ${sources}
  USES ROOT::Core ROOT::Gdml
  )
target_link_libraries(${a_lib_name}
  PUBLIC DD4hep::DDCore DD4hep::DDRec fmt::fmt
  )

#-----------------------------------------------------------------------------------
# Parse jinja templates: once by default, and once for all yml files
set(TEMPLATE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/templates)
set(TEMPLATE_XML ${PROJECT_NAME}.xml.jinja2)

file(GLOB CONFIG_YMLS ${CMAKE_CURRENT_SOURCE_DIR}/configurations/*.yml)
foreach(config_yml ${CONFIG_YMLS})
    get_filename_component(config ${config_yml} NAME_WE)
    add_custom_target(${config}.xml ALL)
    add_custom_command(TARGET ${config}.xml
        COMMAND
            ${CMAKE_CURRENT_SOURCE_DIR}/bin/make_detector_configuration
                --dir ${TEMPLATE_DIR}
                --template ${TEMPLATE_XML}
                --config ${config_yml}
                --output ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}_${config}.xml
        COMMENT "Creating configuration ${config} for ${PROJECT_NAME}"
        )
    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}_${config}.xml
        DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/
        )
endforeach()

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}_full.xml
    DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/
    RENAME ${PROJECT_NAME}.xml
    )

#-----------------------------------------------------------------------------------
# Install the detector description files.
install(DIRECTORY compact/
    DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/compact
    FILES_MATCHING PATTERN "*.xml" PATTERN "*.gdml"
    )

#-----------------------------------------------------------------------------------
# Install the detector calibration files.
install(DIRECTORY calibrations/
    DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/calibrations
    )

#-----------------------------------------------------------------------------------
# Configure and install compact/version.xml
#
configure_file(templates/version.xml.in ${CMAKE_CURRENT_BINARY_DIR}/version.xml @ONLY)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/version.xml
    DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/compact
)

#-----------------------------------------------------------------------------------
# Configure and install detector setup script
#
configure_file(templates/thisepic.sh.in ${CMAKE_CURRENT_BINARY_DIR}/thisepic.sh @ONLY)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/thisepic.sh
    DESTINATION ${CMAKE_INSTALL_BINDIR}
)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/thisepic.sh
    DESTINATION ${CMAKE_INSTALL_PREFIX}
    RENAME setup.sh
)

# install programs
install(PROGRAMS bin/g4MaterialScan_to_csv
  DESTINATION ${CMAKE_INSTALL_BINDIR}
)
