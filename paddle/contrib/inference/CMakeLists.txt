# Copyright (c) 2016 PaddlePaddle Authors. All Rights Reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

if(APPLE)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-error=pessimizing-move")
endif(APPLE)


set(inference_deps paddle_inference_api paddle_fluid_api)
if(WITH_GPU AND TENSORRT_FOUND)
    set(inference_deps ${inference_deps} paddle_inference_tensorrt_subgraph_engine)
endif()

function(inference_api_test TARGET_NAME)
    if (WITH_TESTING)
        set(options "")
        set(oneValueArgs "")
        set(multiValueArgs ARGS)
        cmake_parse_arguments(inference_test "${options}" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})

        set(PYTHON_TESTS_DIR ${PADDLE_BINARY_DIR}/python/paddle/fluid/tests)
        cc_test(${TARGET_NAME}
                SRCS ${TARGET_NAME}.cc
                DEPS "${inference_deps}"
                ARGS --dirname=${PYTHON_TESTS_DIR}/book/)
        if(inference_test_ARGS)
            set_tests_properties(${TARGET_NAME}
                    PROPERTIES DEPENDS "${inference_test_ARGS}")
        endif()
    endif(WITH_TESTING)
endfunction(inference_api_test)

cc_library(paddle_inference_api
    SRCS paddle_inference_api.cc paddle_inference_api_impl.cc
    DEPS ${FLUID_CORE_MODULES} ${GLOB_OP_LIB})
if(NOT APPLE)
  set(LINK_FLAGS "-Wl,--retain-symbols-file ${CMAKE_CURRENT_SOURCE_DIR}/paddle_inference_api.sym")
  set_target_properties(paddle_inference_api PROPERTIES LINK_FLAGS "${LINK_FLAGS}")
endif()

# Here the shared library doesn't depend on other fluid libraries, or double free will occur.
cc_library(paddle_inference_api_shared SHARED
    SRCS paddle_inference_api.cc paddle_inference_api_impl.cc)
add_dependencies(paddle_inference_api_shared ${FLUID_CORE_MODULES} ${GLOB_OP_LIB})
set_target_properties(paddle_inference_api_shared PROPERTIES OUTPUT_NAME paddle_inference_api)

if(NOT APPLE)
  set(LINK_FLAGS "-Wl,--version-script ${CMAKE_CURRENT_SOURCE_DIR}/paddle_inference_api.map")
  set_target_properties(paddle_inference_api_shared PROPERTIES LINK_FLAGS "${LINK_FLAGS}")
  FILE(WRITE ${CMAKE_CURRENT_BINARY_DIR}/check_symbol.cmake
    "execute_process(COMMAND bash -c \"${CMAKE_CURRENT_SOURCE_DIR}/check_symbol.sh"
    " ${CMAKE_CURRENT_BINARY_DIR}/libpaddle_inference_api.so\" RESULT_VARIABLE symbol_res)\n"
    "if(NOT \"\${symbol_res}\" STREQUAL \"0\")\n"
    "  message(FATAL_ERROR \"Check symbol failed.\")\n"
    "endif()\n")
  add_custom_command(
    OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/.check_symbol"
    COMMAND ${CMAKE_COMMAND} -P "${CMAKE_CURRENT_BINARY_DIR}/check_symbol.cmake"
    DEPENDS paddle_inference_api_shared)
  add_custom_target(check_symbol ALL DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/.check_symbol")
endif()

cc_test(test_paddle_inference_api
        SRCS test_paddle_inference_api.cc
        DEPS paddle_inference_api)

inference_api_test(test_paddle_inference_api_impl
                    ARGS test_word2vec test_image_classification)

if(WITH_GPU AND TENSORRT_FOUND)
cc_library(paddle_inference_tensorrt_subgraph_engine
        SRCS paddle_inference_api_tensorrt_subgraph_engine.cc
        DEPS paddle_inference_api analysis tensorrt_engine paddle_inference_api paddle_fluid_api)

inference_api_test(test_paddle_inference_api_tensorrt_subgraph_engine ARGS test_word2vec)
endif()

if (WITH_ANAKIN) # only needed in CI
    # Due to Anakin do not have official library releases and the versions of protobuf and cuda do not match Paddle's,
    # so anakin library will not be merged to our official inference library. To use anakin prediction API, one need to
    # compile the libinference_anakin_api.a and compile with anakin.so.
    nv_library(inference_anakin_api SRCS paddle_inference_api.cc paddle_inference_api_anakin_engine.cc)
    nv_library(inference_anakin_api_shared SHARED SRCS paddle_inference_api.cc paddle_inference_api_anakin_engine.cc)
    target_compile_options(inference_anakin_api BEFORE PUBLIC ${ANAKIN_COMPILE_EXTRA_FLAGS})
    target_compile_options(inference_anakin_api_shared BEFORE PUBLIC ${ANAKIN_COMPILE_EXTRA_FLAGS})
    target_link_libraries(inference_anakin_api anakin anakin_saber_common)
    target_link_libraries(inference_anakin_api_shared anakin anakin_saber_common)
    if (WITH_TESTING)
        cc_test(inference_anakin_test SRCS paddle_inference_api_anakin_engine_tester.cc
                                  ARGS --model=${ANAKIN_INSTALL_DIR}/mobilenet_v2.anakin.bin
                                  DEPS inference_anakin_api)
        target_compile_options(inference_anakin_test BEFORE PUBLIC ${ANAKIN_COMPILE_EXTRA_FLAGS})
     endif(WITH_TESTING)
endif()

if(WITH_TESTING)
    add_subdirectory(demo)
endif()
