project(JNI)
cmake_minimum_required (VERSION 3.6.0)

# options
option(BUILD_TEST "Build and run tests" OFF)
option(BUILD_SHARED_LIBS "Build shared libraries" ON)
option(ENABLE_LOGGING "Enable logging with google-glog library" ON)
option(BOOST_USE_SIGNALS2 "Boost use signals2 instead of signals" ON)
option(LEVELDB_USE_SNAPPY "Leveldb use snappy" ON)
option(OPENCC_BUILD_SHARED_LIBS "Build shared libraries for opencc" ON)

# includes
include_directories(
  librime/src
  librime/thirdparty/include
  marisa-trie/include
  leveldb/include
  yaml-cpp/include
  libiconv/include
  leveldb/include
  snappy
  ${CMAKE_BINARY_DIR}/include
  )

# flags
set(CMAKE_BUILD_TYPE release)
add_definitions(-w)
#set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -s")
if(UNIX)
  add_definitions(-fPIC)
elseif(WIN32)
  set(STATIC_FLAGS "-static-libstdc++ -static-libgcc -static")
  set(CMAKE_SHARED_LIBRARY_PREFIX "")
  set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${STATIC_FLAGS}")
endif()

if (NOT OPENCC_BUILD_SHARED_LIBS)
  add_definitions(-DOpencc_BUILT_AS_STATIC)
endif()

# snappy
set(SNAPPY_SOURCES
  snappy/snappy.cc
  snappy/snappy-sinksource.cc
  )
add_library(snappy-static STATIC ${SNAPPY_SOURCES})
target_include_directories(snappy-static BEFORE PRIVATE ${CMAKE_BINARY_DIR}/include/snappy)

MESSAGE("Configuring snappy ...")
include(CheckIncludeFile)
check_include_file("byteswap.h" HAVE_BYTESWAP_H)
check_include_file("stddef.h" HAVE_STDDEF_H)
check_include_file("stdint.h" HAVE_STDINT_H)
check_include_file("sys/endian.h" HAVE_SYS_ENDIAN_H)
check_include_file("sys/mman.h" HAVE_SYS_MMAN_H)
check_include_file("sys/resource.h" HAVE_SYS_RESOURCE_H)
check_include_file("sys/time.h" HAVE_SYS_TIME_H)
check_include_file("sys/uio.h" HAVE_SYS_UIO_H)
check_include_file("unistd.h" HAVE_UNISTD_H)
check_include_file("windows.h" HAVE_WINDOWS_H)

include(CheckLibraryExists)
check_library_exists(z zlibVersion "" HAVE_LIBZ)
check_library_exists(lzo2 lzo1x_1_15_compress "" HAVE_LIBLZO2)

include(CheckCXXSourceCompiles)
check_cxx_source_compiles(
    "int main(void) { return __builtin_expect(0, 1); }" HAVE_BUILTIN_EXPECT)

check_cxx_source_compiles(
    "int main(void) { return __builtin_ctzll(0); }" HAVE_BUILTIN_CTZ)

include(CheckSymbolExists)
check_symbol_exists("mmap" "sys/mman.h" HAVE_FUNC_MMAP)
check_symbol_exists("sysconf" "unistd.h" HAVE_FUNC_SYSCONF)

configure_file(
  "${CMAKE_CURRENT_LIST_DIR}/snappy/cmake/config.h.in"
  "${CMAKE_BINARY_DIR}/include/snappy/config.h"
)

set(HAVE_STDINT_H_01  ${HAVE_STDINT_H})
set(HAVE_STDDEF_H_01  ${HAVE_STDDEF_H})
set(HAVE_SYS_UIO_H_01 ${HAVE_SYS_UIO_H})
if(NOT HAVE_STDINT_H_01)
  set(HAVE_STDINT_H_01 0)
endif(NOT HAVE_STDINT_H_01)
if(NOT HAVE_STDDEF_H_01)
  set(HAVE_STDDEF_H_01 0)
endif(NOT HAVE_STDDEF_H_01)
if(NOT HAVE_SYS_UIO_H_01)
  set(HAVE_SYS_UIO_H_01 0)
endif(NOT HAVE_SYS_UIO_H_01)

set(SNAPPY_MAJOR 1)
set(SNAPPY_MINOR 1)
set(SNAPPY_PATCHLEVEL 7)
configure_file(
  "${CMAKE_CURRENT_LIST_DIR}/snappy/snappy-stubs-public.h.in"
  "${CMAKE_BINARY_DIR}/include/snappy-stubs-public.h"
  )

# marisa
file(GLOB_RECURSE MARISA_SOURCES marisa-trie/lib/marisa/*.cc)
add_library(marisa-static STATIC ${MARISA_SOURCES})
target_include_directories(marisa-static BEFORE PRIVATE
  marisa-trie/lib
  )

# marisa tools
set(MARISA_CMDOPT marisa-trie/tools/cmdopt.cc)
foreach(tool benchmark build common-prefix-search dump lookup predictive-search reverse-lookup)
set(tool marisa-${tool})
add_executable(${tool} ${MARISA_CMDOPT} marisa-trie/tools/${tool}.cc)
target_link_libraries(${tool} marisa-static ${STATIC_FLAGS})
endforeach(tool)

# iconv
set(ICONV_SOURCES
  libiconv/libcharset/lib/localcharset.c
  libiconv/lib/iconv.c
  libiconv/lib/relocatable.c
  )
add_library(iconv-static STATIC ${ICONV_SOURCES})
target_compile_definitions(iconv-static PRIVATE
  LIBDIR="c" BUILDING_LIBICONV IN_LIBRARY
  )
target_include_directories(iconv-static BEFORE PRIVATE
  libiconv/lib
  libiconv/libcharset/include
  )

# boost
file(GLOB BOOST_SOURCES
  boost/libs/filesystem/src/*.cpp
  boost/libs/regex/src/*.cpp
  boost/libs/system/src/error_code.cpp
  boost/libs/locale/src/encoding/codepage.cpp
  )
if(BOOST_USE_SIGNALS2)
  add_definitions(-DBOOST_SIGNALS2)
else(BOOST_USE_SIGNALS2)
  add_definitions(-DBOOST_SIGNALS_NO_DEPRECATION_WARNING)
  file(GLOB BOOST_SIGNAL_SOURCES boost/libs/signals/src/*.cpp)
endif()
add_library(boost-static STATIC ${BOOST_SOURCES} ${BOOST_SIGNAL_SOURCES})
target_compile_definitions(boost-static PRIVATE BOOST_LOCALE_WITH_ICONV)
target_link_libraries(boost-static iconv-static)

file(GLOB_RECURSE BOOST_HEADERS boost/libs/*.ipp boost/libs/*.h boost/libs/*.hpp)
foreach(head ${BOOST_HEADERS})
  string(REGEX REPLACE "^.*boost/libs" "boost/libs" head ${head})
  string(REGEX REPLACE "^.*include/boost/" "" headout ${head})
  configure_file(
    ${CMAKE_CURRENT_LIST_DIR}/${head}
    ${CMAKE_BINARY_DIR}/include/boost/${headout} COPYONLY)
endforeach(head)

# leveldb
set(LEVELDB_SOURCES
  leveldb/db/builder.cc
  leveldb/db/c.cc
  leveldb/db/db_impl.cc
  leveldb/db/db_iter.cc
  leveldb/db/dbformat.cc
  leveldb/db/dumpfile.cc
  leveldb/db/filename.cc
  leveldb/db/log_reader.cc
  leveldb/db/log_writer.cc
  leveldb/db/memtable.cc
  leveldb/db/repair.cc
  leveldb/db/table_cache.cc
  leveldb/db/version_edit.cc
  leveldb/db/version_set.cc
  leveldb/db/write_batch.cc
  leveldb/table/block.cc
  leveldb/table/block_builder.cc
  leveldb/table/filter_block.cc
  leveldb/table/format.cc
  leveldb/table/iterator.cc
  leveldb/table/merger.cc
  leveldb/table/table.cc
  leveldb/table/table_builder.cc
  leveldb/table/two_level_iterator.cc
  leveldb/util/arena.cc
  leveldb/util/bloom.cc
  leveldb/util/cache.cc
  leveldb/util/coding.cc
  leveldb/util/comparator.cc
  leveldb/util/crc32c.cc
  leveldb/util/env.cc
  leveldb/util/filter_policy.cc
  leveldb/util/hash.cc
  leveldb/util/histogram.cc
  leveldb/util/logging.cc
  leveldb/util/options.cc
  leveldb/util/status.cc
  leveldb/util/env_posix.cc
  leveldb/port/port_posix.cc
  leveldb/port/port_posix_sse.cc
  )
set(LEVELDB_FLAGS _REENTRANT NDEBUG LEVELDB_PLATFORM_POSIX LEVELDB_ATOMIC_PRESENT)
if(ANDROID)
list(APPEND LEVELDB_FLAGS OS_ANDROID)
elseif(UNIX)
list(APPEND LEVELDB_FLAGS OS_LINUX)
elseif(WIN32)
list(APPEND LEVELDB_FLAGS CYGWIN)
endif()
add_library(leveldb-static STATIC ${LEVELDB_SOURCES})
target_include_directories(leveldb-static BEFORE PRIVATE leveldb)
target_compile_definitions(leveldb-static PRIVATE ${LEVELDB_FLAGS})
target_compile_options(leveldb-static PRIVATE -fno-builtin-memcmp)
if(LEVELDB_USE_SNAPPY)
target_compile_definitions(leveldb-static PRIVATE SNAPPY)
target_link_libraries(leveldb-static snappy-static)
endif()
if(WIN32)
target_link_libraries(leveldb-static snappy-static shlwapi)
endif()

# yaml-cpp
#add_subdirectory(yaml-cpp)
file(GLOB YAML-CPP_SOURCES yaml-cpp/src/*.cpp)
add_library(yaml-cpp-static STATIC ${YAML-CPP_SOURCES})
add_definitions(-DYAML_CPP_NO_CONTRIB)

# miniglog
#add_subdirectory(miniglog)
set(GLOG_SOURCES miniglog/miniglog/logging.cc)
add_library(miniglog-static STATIC ${GLOG_SOURCES})
configure_file(
  "${CMAKE_CURRENT_LIST_DIR}/miniglog/miniglog/logging.h"
  "${CMAKE_BINARY_DIR}/include/glog/logging.h"
  COPYONLY)

# opencc
#add_subdirectory(OpenCC)
set(OPENCC_SOURCES
  OpenCC/src/BinaryDict.cpp
  OpenCC/src/Config.cpp
  OpenCC/src/Conversion.cpp
  OpenCC/src/ConversionChain.cpp
  OpenCC/src/Converter.cpp
  OpenCC/src/DartsDict.cpp
  OpenCC/src/Dict.cpp
  OpenCC/src/DictConverter.cpp
  OpenCC/src/DictEntry.cpp
  OpenCC/src/DictGroup.cpp
  OpenCC/src/MaxMatchSegmentation.cpp
  OpenCC/src/PhraseExtract.cpp
  OpenCC/src/SimpleConverter.cpp
  OpenCC/src/Segmentation.cpp
  OpenCC/src/TextDict.cpp
  OpenCC/src/UTF8StringSlice.cpp
  OpenCC/src/UTF8Util.cpp
  )

if(OPENCC_BUILD_SHARED_LIBS)
  add_library(opencc ${OPENCC_SOURCES})
else(OPENCC_BUILD_SHARED_LIBS)
  add_library(opencc STATIC ${OPENCC_SOURCES})
endif()

target_include_directories(opencc BEFORE PRIVATE
  OpenCC/deps/darts-clone
  OpenCC/deps/rapidjson-0.11
  OpenCC/deps/tclap-1.2.1
  )
target_compile_definitions(opencc PRIVATE VERSION="1.0.5")
if(ANDROID)
elseif(UNIX)
target_compile_definitions(opencc PRIVATE PKGDATADIR="/usr/share/opencc")
endif()

set(
  LIBOPENCC_HEADERS
  BinaryDict.hpp
  Common.hpp
  Config.hpp
  Conversion.hpp
  ConversionChain.hpp
  Converter.hpp
  DartsDict.hpp
  Dict.hpp
  DictConverter.hpp
  DictEntry.hpp
  DictGroup.hpp
  Exception.hpp
  Export.hpp
  Lexicon.hpp
  MaxMatchSegmentation.hpp
  Optional.hpp
  PhraseExtract.hpp
  Segmentation.hpp
  Segments.hpp
  SerializableDict.hpp
  SimpleConverter.hpp
  TextDict.hpp
  UTF8StringSlice.hpp
  UTF8Util.hpp
  opencc.h
)
foreach(head ${LIBOPENCC_HEADERS})
  configure_file(
    "${CMAKE_CURRENT_LIST_DIR}/OpenCC/src/${head}"
    "${CMAKE_BINARY_DIR}/include/opencc/${head}"
    COPYONLY)
endforeach(head)

# opencc tools
add_executable(opencc_tool OpenCC/src/tools/CommandLine.cpp)
target_link_libraries(opencc_tool opencc ${STATIC_FLAGS})
target_include_directories(opencc_tool BEFORE PRIVATE
  OpenCC/src
  OpenCC/deps/tclap-1.2.1
  )
## opencc_dict
add_executable(opencc_dict OpenCC/src/tools/DictConverter.cpp)
target_link_libraries(opencc_dict opencc ${STATIC_FLAGS})
target_include_directories(opencc_dict BEFORE PRIVATE
  OpenCC/src
  OpenCC/deps/tclap-1.2.1
  )
## opencc_phrase_extract
add_executable(opencc_phrase_extract OpenCC/src/tools/PhraseExtract.cpp)
target_link_libraries(opencc_phrase_extract opencc ${STATIC_FLAGS})
target_include_directories(opencc_phrase_extract BEFORE PRIVATE
  OpenCC/src
  OpenCC/deps/tclap-1.2.1
  )

# rime
#add_subdirectory(librime)
find_package (Threads)
file(GLOB_RECURSE RIME_SOURCES librime/src/*.cc)
add_library(rime SHARED ${RIME_SOURCES})
target_link_libraries(rime
  opencc
  yaml-cpp-static
  marisa-static
  miniglog-static
  leveldb-static
  boost-static
  ${CMAKE_THREAD_LIBS_INIT}
  )
target_compile_definitions(rime PRIVATE
  RIME_BUILD_SHARED_LIBS
  RIME_VERSION="1.4.0"
  RIME_ENABLE_LOGGING
  RIME_EXPORTS
  RIME_EXTRA_MODULES=""
  )
if(ANDROID)
  target_link_libraries(rime log atomic)
elseif(WIN32)
  target_link_libraries(rime wsock32)
endif()

# rime tools
foreach(tool api_console console deployer dict_manager patch)
set(tool rime_${tool})
add_executable(${tool} "librime/tools/${tool}.cc")
target_link_libraries(${tool} rime ${CMAKE_THREAD_LIBS_INIT})
endforeach(tool)

# rime_jni
if(UNIX)
file(GLOB RIME_JNI_SOURCES librime_jni/*.cc)
add_library(rime_jni SHARED ${RIME_JNI_SOURCES})
target_link_libraries(rime_jni rime opencc)
string(TIMESTAMP TODAY "%Y%m%d")
execute_process(
  COMMAND git --git-dir ${CMAKE_CURRENT_LIST_DIR}/../../../../.git describe --tags
  OUTPUT_VARIABLE TRIME_VERSION
  )
string(STRIP ${TRIME_VERSION} TRIME_VERSION)
set(TRIME_VERSION "${TRIME_VERSION}-${TODAY}")
execute_process(
  COMMAND git --git-dir ${CMAKE_CURRENT_LIST_DIR}/OpenCC/.git describe --tags
  OUTPUT_VARIABLE OPENCC_VERSION
  )
string(STRIP ${OPENCC_VERSION} OPENCC_VERSION)
execute_process(
  COMMAND git --git-dir ${CMAKE_CURRENT_LIST_DIR}/librime/.git describe --tags
  OUTPUT_VARIABLE LIBRIME_VERSION
  )
string(STRIP ${LIBRIME_VERSION} LIBRIME_VERSION)
set(LIBRIME_VERSION "${LIBRIME_VERSION}-${ANDROID_ABI}")
MESSAGE(${LIBRIME_VERSION})
target_compile_definitions(rime_jni PRIVATE
  CLASSNAME="com/osfans/trime/Rime"
  TRIME_VERSION="${TRIME_VERSION}"
  OPENCC_VERSION="${OPENCC_VERSION}"
  LIBRIME_VERSION="${LIBRIME_VERSION}"
  )

if(ANDROID)
  target_link_libraries(rime_jni log)
else(ANDROID)
  target_include_directories(rime_jni BEFORE PRIVATE
    /usr/lib/jvm/default/include
    /usr/lib/jvm/default/include/linux
  )
endif(ANDROID)
endif(UNIX)
