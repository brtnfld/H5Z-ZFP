# Define headers.
set(h5z_zfp_headers
  H5Zzfp.h H5Zzfp_lib.h H5Zzfp_plugin.h H5Zzfp_props.h H5Zzfp_version.h)

#------------------------------------------------------------------------------#
# Static library
#------------------------------------------------------------------------------#
# Define source
set(h5z_zfp_lib_source
  ${h5z_zfp_headers}
  H5Zzfp_props_private.h
  H5Zzfp.c H5Zzfp_props.c)

if (FORTRAN_INTERFACE)
  list(APPEND h5z_zfp_lib_source H5Zzfp_props_f.F90)
endif ()

add_library(h5z_zfp_static STATIC ${h5z_zfp_lib_source})
target_compile_definitions(h5z_zfp_static PRIVATE H5Z_ZFP_AS_LIB)
target_include_directories(h5z_zfp_static
  PUBLIC
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>"
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>"
    "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>"
    ${HDF5_INCLUDE_DIRS}
    "$<$<BOOL:${FORTRAN_INTERFACE}>:${HDF5_FORTRAN_INCLUDE_DIRS}>"
)
target_link_libraries(h5z_zfp_static PUBLIC zfp::zfp ${HDF5_LIBRARIES} $<$<BOOL:${FORTRAN_INTERFACE}>:${HDF5_FORTRAN_LIBRARIES}>)
set_target_properties(h5z_zfp_static PROPERTIES POSITION_INDEPENDENT_CODE 1)
set_target_properties(h5z_zfp_static PROPERTIES
  OUTPUT_NAME h5zzfp
  RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/bin
  ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/lib
  LIBRARY_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/lib)
install(TARGETS h5z_zfp_static
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR})

#------------------------------------------------------------------------------#
# Shared library
#------------------------------------------------------------------------------#
# Define source
set(h5z_zfp_plugin_source
  ${h5z_zfp_headers}
  H5Zzfp.c)

add_library(h5z_zfp_shared SHARED ${h5z_zfp_plugin_source})
target_include_directories(h5z_zfp_shared
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
    ${HDF5_INCLUDE_DIRS})
target_link_libraries(h5z_zfp_shared PUBLIC zfp::zfp ${HDF5_LIBRARIES})
set_target_properties(h5z_zfp_shared PROPERTIES
  OUTPUT_NAME h5zzfp
  RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/plugin
  ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/plugin
  LIBRARY_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/plugin
  BUILD_RPATH "$<TARGET_FILE_DIR:zfp::zfp>"
  INSTALL_RPATH "$ORIGIN"
  INSTALL_RPATH_USE_LINK_PATH ON)
install(TARGETS h5z_zfp_shared
  RUNTIME DESTINATION plugin
  ARCHIVE DESTINATION plugin
  LIBRARY DESTINATION plugin)

#------------------------------------------------------------------------------#
# Sign the plugin if H5Z_ZFP_SIGN_PLUGIN is enabled
#------------------------------------------------------------------------------#
if (H5Z_ZFP_SIGN_PLUGIN)
  find_program(H5SIGN_TOOL h5sign
    HINTS ${HDF5_ROOT}/bin ${HDF5_DIR}/../../../bin
    DOC "Path to the h5sign tool")
  if (NOT H5SIGN_TOOL)
    message(FATAL_ERROR
      "H5Z_ZFP_SIGN_PLUGIN is ON but h5sign was not found. "
      "Set H5SIGN_TOOL to the path of the h5sign executable.")
  endif ()
  if (NOT H5Z_ZFP_SIGNING_KEY)
    message(FATAL_ERROR
      "H5Z_ZFP_SIGN_PLUGIN is ON but H5Z_ZFP_SIGNING_KEY is not set. "
      "Set H5Z_ZFP_SIGNING_KEY to the path of the RSA private key PEM file.")
  endif ()
  add_custom_command(TARGET h5z_zfp_shared POST_BUILD
    COMMAND ${H5SIGN_TOOL} -p $<TARGET_FILE:h5z_zfp_shared> -k ${H5Z_ZFP_SIGNING_KEY}
    COMMENT "Signing plugin $<TARGET_FILE:h5z_zfp_shared>"
    VERBATIM)
  message(STATUS "Plugin signing enabled: key=${H5Z_ZFP_SIGNING_KEY}, tool=${H5SIGN_TOOL}")
endif ()

#------------------------------------------------------------------------------#
# Install header and module files.
#------------------------------------------------------------------------------#
install(FILES ${h5z_zfp_headers}
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

if (FORTRAN_INTERFACE)
  install(FILES ${CMAKE_BINARY_DIR}/bin/h5zzfp_props_f.mod
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
endif ()
