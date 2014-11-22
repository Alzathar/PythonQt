#-----------------------------------------------------------------------------
# NOTE: The CMake files have been contributed to PythonQt and have not been tested with the current
# PythonQt version. They have not yet been updated to support Python 3.
#
# If you are not a CMake expert, you should better use the provided qmake profiles.
#-----------------------------------------------------------------------------

cmake_minimum_required(VERSION 3.0)

#-----------------------------------------------------------------------------
project(PythonQt)

SET(PythonQt_VERSION_MAJOR 3)
SET(PythonQt_VERSION_MINOR 0)
SET(PythonQt_VERSION_PATCH 0)
SET(PythonQt_VERSION "${PythonQt_VERSION_MAJOR}.${PythonQt_VERSION_MINOR}.${PythonQt_VERSION_PATCH}")

#-----------------------------------------------------------------------------

# By default PythonQt is compiled in Release mode
IF(NOT CMAKE_BUILD_TYPE)
  MESSAGE(STATUS "Setting build type to 'Release' as none was specified.")
  SET(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
  # Set the possible values of build type for cmake-gui
  SET_PROPERTY(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
ENDIF()

#-----------------------------------------------------------------------------
# Python libraries

find_package(PythonLibs REQUIRED)
include_directories("${PYTHON_INCLUDE_DIR}")
add_definitions(-DPYTHONQT_USE_RELEASE_PYTHON_FALLBACK)

#-----------------------------------------------------------------------------
# Build options

set(PythonQt_QT_MAJOR_VERSION "" CACHE STRING "Choose which version of Qt (4 or 5) you wand to wrap")

if (PythonQt_QT_MAJOR_VERSION)
  if (PythonQt_QT_MAJOR_VERSION VERSION_GREATER "4")
    find_package(Qt5 REQUIRED COMPONENTS Core)
    if (Qt5_FOUND)
      set(CMAKE_PREFIX_PATH ${CMAKE_PREFIX_PATH} "${Qt5_DIR}/..")
      set(PythonQt_QT_VERSION ${Qt5_VERSION} CACHE STRING "Version of Qt detected")
    endif()
  else()
    find_package(Qt4 REQUIRED)
    if (QT4_FOUND)
      set(PythonQt_QT_VERSION ${QT_VERSION_MAJOR}.${QT_VERSION_MINOR}.${QT_VERSION_PATCH} CACHE STRING "Version of Qt detected")
    endif()
  endif()
endif()

if (PythonQt_QT_VERSION)

  option(BUILD_CONSOLE_WIDGET "Include the PythonQtScriptingConsole widget (Require to wrap QtGui and QtWidgets)" OFF)

  option(PythonQt_Wrap_QtAll "Make all Qt components available in python" OFF)

  set(qtlibs Core Gui Widgets Network OpenGL PrintSupport Sql Svg UiTools WebKit WebKitWidgets Xml)
  if (PythonQt_QT_VERSION VERSION_LESS "5")
    set(qtlibs_lower)
    foreach(qtlib ${qtlibs})
      string(TOLOWER ${qtlib} qtlib_lower)
      list(APPEND qtlibs_lower ${qtlib_lower})
    endforeach()
    set(qtlibs ${qtlibs_lower})
  else()
    # Add Qt5 CMake path in CMAKE_PREFIX_PATH
    set(CMAKE_PREFIX_PATH ${CMAKE_PREFIX_PATH} ${Qt5_CMAKE_PATH})
    if (PythonQt_QT_VERSION VERSION_GREATER "4")
      set(qtlibs
        ${qtlibs}
        Multimedia
        MultimediaWidgets
        XmlPatterns
      )
    endif()
  endif()

  foreach(qtlib ${qtlibs})
    if ((PythonQt_QT_VERSION VERSION_GREATER "4") AND (${qtlib} STREQUAL "Core"))
      # Because the macro qt5_wrap_cpp is available in Qt5Core
      OPTION(PythonQt_Wrap_Qt${qtlib} "Make all of Qt${qtlib} available in python" TRUE)
    else ()
      OPTION(PythonQt_Wrap_Qt${qtlib} "Make all of Qt${qtlib} available in python" FALSE)
    endif ()
  endforeach()

  # Force option if it applies
  if(PythonQt_Wrap_QtAll)
    foreach(qtlib ${qtlibs})
      if(NOT ${PythonQt_Wrap_Qt${qtlib}})
        set(PythonQt_Wrap_Qt${qtlib} ON CACHE BOOL "Make all of Qt${qtlib} available in python" FORCE)
        message(STATUS "Enabling [PythonQt_Wrap_Qt${qtlib}] because of [PythonQt_Wrap_QtAll] evaluates to True")
      endif()
    endforeach()
  endif()

  option(PythonQt_DEBUG "Enable/Disable PythonQt debug output" OFF)
  if(PythonQt_DEBUG)
    add_definitions(-DPYTHONQT_DEBUG)
  else()
    remove_definitions(-DPYTHONQT_DEBUG)
  endif()

#-----------------------------------------------------------------------------
# Setup Qt

  if (PythonQt_QT_VERSION VERSION_GREATER "4")
  
    set(QT_LIBRARIES)
    foreach(qtlib ${qtlibs})
      if (${PythonQt_Wrap_Qt${qtlib}})
        find_package(Qt5${qtlib})
        include_directories(${Qt5${qtlib}_INCLUDE_DIRS})
        add_definitions(${Qt5${qtlib}_DEFINITIONS})
        list(APPEND QT_LIBRARIES ${Qt5${qtlib}_LIBRARIES})
        foreach(dep ${_Qt5${qtlib}_MODULE_DEPENDENCIES})
          if (NOT ${PythonQt_Wrap_Qt${dep}})
            set(PythonQt_Wrap_Qt${dep} ON CACHE BOOL "Make all of Qt${dep} available in python" FORCE)
            message(STATUS "Module Qt${dep} has been activated (Required to wrap the module ${qtlib}).")
          endif()
        endforeach()
      endif()
    endforeach()
  
    if (BUILD_TESTING)
      find_package(Qt5Test REQUIRED)
    endif()
  
  else()
    
    set(minimum_required_qt_version "4.6.2")
  
    set(found_qt_version ${QT_VERSION_MAJOR}.${QT_VERSION_MINOR}.${QT_VERSION_PATCH})

    if(${found_qt_version} VERSION_LESS ${minimum_required_qt_version})
      message(FATAL_ERROR "error: PythonQt requires Qt >= ${minimum_required_qt_version} -- you cannot use Qt ${found_qt_version}.")
    endif()

    # Enable required qt module
    set(qt4libs ${qtlibs})
    list(REMOVE_ITEM qt4libs core gui widgets)
    foreach(qtlib ${qt4libs})
      string(TOUPPER ${qtlib} qtlib_uppercase)
      if (NOT ${QT_QT${qtlib_uppercase}_FOUND})
        message(FATAL_ERROR "QT_QT${qtlib_uppercase} *not* FOUND - Try to disable PythonQt_Wrap_Qt${qtlib}")
      endif()
      set(QT_USE_QT${qtlib_uppercase} ${PythonQt_Wrap_Qt${qtlib}})
    endforeach()
  
    # Enable QtTest in Qt4 is the option BUILD_TESTING was activated
    set(QT_USE_QTTEST ${BUILD_TESTING})
  
    include(${QT_USE_FILE})

  endif()

#-----------------------------------------------------------------------------
# Print a summary of the wrapped module

  if (PythonQt_Wrap_QtUiTools OR PythonQt_Wrap_Qtuitools)
    message(STATUS "\nWARNING: The wrapping of the UiTools library can causes \"different visibility settings\"\n         warnings on Mac OS (see [QTBUG-37962] [QTBUG-3995])")
  endif()

  message(STATUS "\nQt modules wrapped" )
  message(STATUS "==================")
  foreach(qtlib ${qtlibs})
    if (${PythonQt_Wrap_Qt${qtlib}})
      string(TOLOWER ${qtlib} qtlib_lower)
      if ("${qtlib_lower}" STREQUAL "widgets")
        message(STATUS " - ${qtlib} ... Yes (included in PythonQt.QtGui namespace)")
      elseif ("${qtlib_lower}" STREQUAL "printsupport")
        message(STATUS " - ${qtlib} ... Yes (included in PythonQt.QtGui namespace)")
      elseif ("${qtlib_lower}" STREQUAL "webkitwidgets")
        message(STATUS " - ${qtlib} ... Yes (included in PythonQt.QtWebKit namespace)")
      elseif ("${qtlib_lower}" STREQUAL "multimediawidgets")
        message(STATUS " - ${qtlib} ... Yes (included in PythonQt.QtMultimedia namespace)")
      else()
        message(STATUS " - ${qtlib} ... Yes")
      endif()
    else()
      message(STATUS " - ${qtlib} ... No")
    endif()
  endforeach()
  
  message(STATUS "\nPythonQt extras" )
  message(STATUS "===============")
  if (BUILD_CONSOLE_WIDGET)
    if (PythonQt_Wrap_QtWidgets OR PythonQt_Wrap_Qtwidgets)
      message(STATUS " - Scripting console widget included ... Yes")
    else()
      message(STATUS " - Scripting console widget included ... ERROR\n\n")
      message(FATAL_ERROR "You attempted to include the sources for the PythonQtScriptingConsole widget but QtGui and QtWidgets are not wrapped.")
    endif()
  else()
    message(STATUS " - Scripting console widget included ... No")
  endif()
  
  message(STATUS "\n")

#-----------------------------------------------------------------------------
# The variable "generated_cpp_suffix" allows to conditionnally compile the generated wrappers 
# associated with the Qt version being used.
  set(generated_cpp_suffix "")
  if (PythonQt_QT_VERSION VERSION_GREATER "4")
    set(generated_cpp_suffix "_${Qt5Core_VERSION_MAJOR}${Qt5Core_VERSION_MINOR}")
  else()
    set(generated_cpp_suffix "_${QT_VERSION_MAJOR}${QT_VERSION_MINOR}")
  endif()

  if("${generated_cpp_suffix}" STREQUAL "_54")
    set(generated_cpp_suffix "_53")
  endif()
  if(("${generated_cpp_suffix}" STREQUAL "_51") OR ("${generated_cpp_suffix}" STREQUAL "_52"))
    set(generated_cpp_suffix "_50")
  endif()
  if("${generated_cpp_suffix}" STREQUAL "_46")
    set(generated_cpp_suffix "_47") # Also use 4.7 wrappers for 4.6.x version
  endif()

#-----------------------------------------------------------------------------
# Sources

  set(sources
    src/PythonQtStdDecorators.cpp
    src/PythonQt.cpp
    src/PythonQtClassInfo.cpp
    src/PythonQtImporter.cpp
    src/PythonQtObjectPtr.cpp
    src/PythonQtStdIn.cpp
    src/PythonQtStdOut.cpp
    src/PythonQtSignal.cpp
    src/PythonQtSlot.cpp
    src/PythonQtMisc.cpp
    src/PythonQtMethodInfo.cpp
    src/PythonQtConversion.cpp
    src/PythonQtSignalReceiver.cpp 
    src/PythonQtInstanceWrapper.cpp
    src/PythonQtQFileImporter.cpp
    src/PythonQtClassWrapper.cpp
    src/PythonQtBoolResult.cpp
    generated_cpp${generated_cpp_suffix}/com_trolltech_qt_core_builtin/com_trolltech_qt_core_builtin0.cpp
    generated_cpp${generated_cpp_suffix}/com_trolltech_qt_core_builtin/com_trolltech_qt_core_builtin_init.cpp
    generated_cpp${generated_cpp_suffix}/PythonQt_QtBindings.cpp
  )
  
  if (PythonQt_Wrap_QtGui OR PythonQt_Wrap_Qtgui)
    set(sources
      ${sources}
      generated_cpp${generated_cpp_suffix}/com_trolltech_qt_gui_builtin/com_trolltech_qt_gui_builtin0.cpp
      generated_cpp${generated_cpp_suffix}/com_trolltech_qt_gui_builtin/com_trolltech_qt_gui_builtin_init.cpp
    )
  endif()
  

  if (BUILD_CONSOLE_WIDGET)
    set(sources
      ${sources}
      src/gui/PythonQtScriptingConsole.cpp
    )
  endif()

#-----------------------------------------------------------------------------
# List headers.  This is list is used for the install command.

  set(headers
    src/PythonQt.h
    src/PythonQtStdDecorators.h
    src/PythonQtClassInfo.h
    src/PythonQtImporter.h
    src/PythonQtObjectPtr.h
    src/PythonQtSignal.h
    src/PythonQtSlot.h
    src/PythonQtStdIn.h
    src/PythonQtStdOut.h
    src/PythonQtMisc.h
    src/PythonQtMethodInfo.h
    src/PythonQtImportFileInterface.h
    src/PythonQtConversion.h
    src/PythonQtSignalReceiver.h
    src/PythonQtInstanceWrapper.h
    src/PythonQtClassWrapper.h
    src/PythonQtCppWrapperFactory.h
    src/PythonQtQFileImporter.h
    src/PythonQtQFileImporter.h
    src/PythonQtVariants.h
    src/PythonQtSystem.h
    src/PythonQtUtils.h
    src/PythonQtBoolResult.h
    generated_cpp${generated_cpp_suffix}/PythonQt_QtBindings.h
  )

#-----------------------------------------------------------------------------
# Headers that should run through moc

  set(moc_sources
      src/PythonQt.h
      src/PythonQtSignalReceiver.h
      src/PythonQtStdDecorators.h
      generated_cpp${generated_cpp_suffix}/com_trolltech_qt_core_builtin/com_trolltech_qt_core_builtin0.h
  )
  
  if (PythonQt_Wrap_QtGui OR PythonQt_Wrap_Qtgui)
    set(moc_sources
      ${moc_sources}
      generated_cpp${generated_cpp_suffix}/com_trolltech_qt_gui_builtin/com_trolltech_qt_gui_builtin0.h
    )
  endif()
  
  if (BUILD_CONSOLE_WIDGET)
    set(moc_sources
      ${moc_sources}
      src/gui/PythonQtScriptingConsole.h
    )
  endif()

#-----------------------------------------------------------------------------
# Add extra sources
  foreach(qtlib ${qtlibs})
    if (${PythonQt_Wrap_Qt${qtlib}})
  
      ADD_DEFINITIONS(-DPYTHONQT_WRAP_Qt${qtlib})
      string(TOLOWER ${qtlib} qtlib_lower)
    
      # NOTE: Gui is currently including Widgets and PrintSupport files. 
      # NOTE: Multimedia is currently including MultimediaWidgets files. 
      # NOTE: WebKit is currently including WebKitWidgets files.
      if ((NOT "${qtlib_lower}" STREQUAL "widgets")
          AND (NOT "${qtlib_lower}" STREQUAL "printsupport")
          AND (NOT "${qtlib_lower}" STREQUAL "multimediawidgets")
          AND (NOT "${qtlib_lower}" STREQUAL "webkitwidgets"))
       
        set(file_prefix generated_cpp${generated_cpp_suffix}/com_trolltech_qt_${qtlib_lower}/com_trolltech_qt_${qtlib_lower})

        foreach(index RANGE 0 12)
      
          # Source files
          if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/${file_prefix}${index}.cpp)
            list(APPEND sources ${file_prefix}${index}.cpp)
          endif()
      
          # Headers that should run through moc
          if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/${file_prefix}${index}.h)
            list(APPEND moc_sources ${file_prefix}${index}.h)
          endif()
      
        endforeach()

        list(APPEND sources ${file_prefix}_init.cpp)
      
      endif()

    endif()
  endforeach()

#-----------------------------------------------------------------------------
# Do wrapping
  if (PythonQt_QT_VERSION VERSION_GREATER "4")
    qt5_wrap_cpp(gen_moc_sources ${moc_sources})
  else()
    qt4_wrap_cpp(gen_moc_sources ${moc_sources})
  endif()

#-----------------------------------------------------------------------------
# Build the library

  include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/src
    )

  add_library(PythonQt SHARED
              ${sources}
              ${gen_moc_sources}
              ${gen_ui_sources}
              ${gen_qrc_sources}
    )
  set_target_properties(PythonQt PROPERTIES DEFINE_SYMBOL PYTHONQT_EXPORTS)

  set_target_properties(PythonQt  PROPERTIES
    MACOSX_RPATH "${CMAKE_INSTALL_PREFIX}/lib"
    )


  target_link_libraries(PythonQt
                ${PYTHON_LIBRARY}
                ${QT_LIBRARIES}
    )

#-----------------------------------------------------------------------------
# Install library (on windows, put the dll in 'bin' and the archive in 'lib')

  install(TARGETS PythonQt
          RUNTIME DESTINATION bin
          LIBRARY DESTINATION lib
          ARCHIVE DESTINATION lib)
  install(FILES ${headers} DESTINATION include/PythonQt)

#-----------------------------------------------------------------------------
# Testing

  option(BUILD_TESTING "Build the testing tree." OFF)
  include(CTest)

  if(BUILD_TESTING)
    create_test_sourcelist(test_sources PythonQtCppTests.cpp
      tests/PythonQtTestMain.cpp
      )

    list(APPEND test_sources
      tests/PythonQtTests.cpp
      tests/PythonQtTests.h
      )

    if (PythonQt_QT_VERSION VERSION_GREATER "4")
      QT5_WRAP_CPP(test_sources tests/PythonQtTests.h)
    else()
      QT4_WRAP_CPP(test_sources tests/PythonQtTests.h)
    endif()    

    set_property(SOURCE tests/PythonQtTestMain.cpp PROPERTY COMPILE_DEFINITIONS "main=tests_PythonQtTestMain")
  
    add_executable(PythonQtCppTests ${test_sources})
    target_link_libraries(PythonQtCppTests PythonQt ${Qt5Test_LIBRARIES})
  
    add_test(
      NAME tests_PythonQtTestMain
      COMMAND PythonQtCppTests tests/PythonQtTestMain
      )
  endif()


endif() 