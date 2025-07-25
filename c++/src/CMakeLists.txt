# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.

set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${CXX17_FLAGS} ${WARN_FLAGS}")

INCLUDE(CheckCXXSourceCompiles)

CHECK_CXX_SOURCE_COMPILES("
    #include<fcntl.h>
    #include<unistd.h>
    int main(int,char*[]){
      int f = open(\"/x/y\", O_RDONLY);
      char buf[100];
      return pread(f, buf, 100, 1000) == 0;
    }"
  HAS_PREAD
)

CHECK_CXX_SOURCE_COMPILES("
    #include<time.h>
    int main(int,char*[]){
      struct tm time2020;
      return !strptime(\"2020-02-02 12:34:56\", \"%Y-%m-%d %H:%M:%S\", &time2020);
    }"
  HAS_STRPTIME
)

CHECK_CXX_SOURCE_COMPILES("
    int main(){
      int a;
      return __builtin_add_overflow(1, 2, &a);
    }"
  HAS_BUILTIN_OVERFLOW_CHECK
)

CHECK_CXX_SOURCE_COMPILES("
    #ifdef __clang__
      #pragma clang diagnostic push
      #pragma clang diagnostic ignored \"-Wdeprecated\"
      #pragma clang diagnostic pop
   #elif defined(__GNUC__)
      #pragma GCC diagnostic push
      #pragma GCC diagnostic ignored \"-Wdeprecated\"
      #pragma GCC diagnostic pop
   #elif defined(_MSC_VER)
      #pragma warning( push )
      #pragma warning( disable : 4996 )
      #pragma warning( pop )
   #else
     unknownCompiler!
   #endif
   int main(int, char *[]) {}"
  HAS_DIAGNOSTIC_PUSH
)

CHECK_CXX_SOURCE_COMPILES("
    #include<cmath>
    int main(int, char *[]) {
      return std::isnan(1.0f);
    }"
  HAS_STD_ISNAN
)

CHECK_CXX_SOURCE_COMPILES("
    #include<string>
    int main(int, char *[]) {
      double d = 5;
      std::to_string(d);
    }"
  HAS_DOUBLE_TO_STRING
)

CHECK_CXX_SOURCE_COMPILES("
    #include<cstdint>
    #include<string>
    int main(int, char *[]) {
      int64_t d = 5;
      std::to_string(d);
    }"
  HAS_INT64_TO_STRING
)

INCLUDE(CheckCXXSourceRuns)

CHECK_CXX_SOURCE_RUNS("
    #include<time.h>
    int main(int, char *[]) {
      time_t t = -14210715; // 1969-07-20 12:34:45
      struct tm *ptm = gmtime(&t);
      return !(ptm && ptm->tm_year == 69);
    }"
  HAS_PRE_1970
)

CHECK_CXX_SOURCE_RUNS("
    #include<stdlib.h>
    #include<time.h>
    int main(int, char *[]) {
      setenv(\"TZ\", \"America/Los_Angeles\", 1);
      tzset();
      struct tm time2037;
      struct tm time2038;
      strptime(\"2037-05-05 12:34:56\", \"%Y-%m-%d %H:%M:%S\", &time2037);
      strptime(\"2038-05-05 12:34:56\", \"%Y-%m-%d %H:%M:%S\", &time2038);
      return (mktime(&time2038) - mktime(&time2037)) <= 31500000;
    }"
  HAS_POST_2038
)

set(CMAKE_REQUIRED_INCLUDES ${ZLIB_INCLUDE_DIR})
set(CMAKE_REQUIRED_LIBRARIES orc_zlib)
CHECK_CXX_SOURCE_COMPILES("
    #define Z_PREFIX
    #include<zlib.h>
    z_stream strm;
    int main(int, char *[]) {
        deflateReset(&strm);
    }"
  NEEDS_Z_PREFIX
)

configure_file (
  "Adaptor.hh.in"
  "${CMAKE_CURRENT_BINARY_DIR}/Adaptor.hh"
  )

add_custom_command(OUTPUT orc_proto.pb.h orc_proto.pb.cc
   COMMAND ${PROTOBUF_EXECUTABLE}
        -I ../../orc-format_ep-prefix/src/orc-format_ep/src/main/proto/orc/proto
        --cpp_out="${CMAKE_CURRENT_BINARY_DIR}"
        ../../orc-format_ep-prefix/src/orc-format_ep/src/main/proto/orc/proto/orc_proto.proto
)

set(SOURCE_FILES
  "${CMAKE_CURRENT_BINARY_DIR}/Adaptor.hh"
  orc_proto.pb.h
  io/InputStream.cc
  io/OutputStream.cc
  io/Cache.cc
  sargs/ExpressionTree.cc
  sargs/Literal.cc
  sargs/PredicateLeaf.cc
  sargs/SargsApplier.cc
  sargs/SearchArgument.cc
  sargs/TruthValue.cc
  wrap/orc-proto-wrapper.cc
  Adaptor.cc
  BlockBuffer.cc
  BloomFilter.cc
  BpackingDefault.cc
  ByteRLE.cc
  ColumnPrinter.cc
  ColumnReader.cc
  ColumnWriter.cc
  Common.cc
  Compression.cc
  ConvertColumnReader.cc
  CpuInfoUtil.cc
  Exceptions.cc
  Geospatial.cc
  Int128.cc
  LzoDecompressor.cc
  MemoryPool.cc
  Murmur3.cc
  OrcFile.cc
  Reader.cc
  RLEv1.cc
  RLEV2Util.cc
  RleDecoderV2.cc
  RleEncoderV2.cc
  RLE.cc
  SchemaEvolution.cc
  Statistics.cc
  StripeStream.cc
  Timezone.cc
  TypeImpl.cc
  Vector.cc
  Writer.cc)

if(BUILD_LIBHDFSPP)
  set(SOURCE_FILES ${SOURCE_FILES} OrcHdfsFile.cc)
endif(BUILD_LIBHDFSPP)

if(BUILD_ENABLE_AVX512)
  set(SOURCE_FILES
    ${SOURCE_FILES}
    BpackingAvx512.cc)
endif(BUILD_ENABLE_AVX512)

add_library (orc STATIC ${SOURCE_FILES})

target_link_libraries (orc
  INTERFACE
    ${ORC_INSTALL_INTERFACE_TARGETS}
  PRIVATE
    $<BUILD_INTERFACE:orc::protobuf>
    $<BUILD_INTERFACE:orc::zlib>
    $<BUILD_INTERFACE:orc::snappy>
    $<BUILD_INTERFACE:orc::lz4>
    $<BUILD_INTERFACE:orc::zstd>
    $<BUILD_INTERFACE:orc::sparsehash>
    $<BUILD_INTERFACE:${LIBHDFSPP_LIBRARIES}>
  )

target_include_directories (orc
  INTERFACE
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
  PUBLIC
    $<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/c++/include>
    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/c++/include>
  PRIVATE
    ${CMAKE_CURRENT_BINARY_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${LIBHDFSPP_INCLUDE_DIR}
)

if (BUILD_LIBHDFSPP)
  target_compile_definitions(orc PUBLIC -DBUILD_LIBHDFSPP)
endif (BUILD_LIBHDFSPP)

if (BUILD_CPP_ENABLE_METRICS)
  message(STATUS "Enable the metrics collection")
  target_compile_definitions(orc PUBLIC ENABLE_METRICS=1)
else ()
  message(STATUS "Disable the metrics collection")
  target_compile_definitions(orc PUBLIC ENABLE_METRICS=0)
endif ()

add_dependencies(orc orc-format_ep)

install(TARGETS orc EXPORT orc_targets)
install(EXPORT orc_targets
  DESTINATION ${ORC_INSTALL_CMAKE_DIR}
  NAMESPACE "orc::"
  FILE "orcTargets.cmake")
