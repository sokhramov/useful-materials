#####################################################################################################################################################
#                                       ==============================================================================                              #
#                                                               ##### How to use CMakeLists #####                                                   #
#                                       ==============================================================================                              #
#                                                                                                                                                   #
#                                                                                                                                                   #
#   1.  Requirements: CMake 3.17    C11     CXX17                                                                                                   #
#   2.  Create MakeFile project using STM32CubeMX with option "Copy only necessary libraries"                                                       #
#   3.  Copy this file into project's directory                                                                                                     #
#   4.  Add user source files (line 172) and included directories (line 126) into this file                                                         #
#   5.  Execute this file and set next variables:   USE_CXX                     (OFF/ON     default OFF)                                            #
#                                                   CMAKE_BUILD_TYPE            (Release/RelWithDebInfo/MinSizeRel/Debug  default Debug)            #
#                                                   CMAKE_PREFIX                (default arm-none-eabi)                                             #
#                                                                                                                                                   #
#                                        !!! Do not execute CMakeLists in project directory !!!                                                     #
#                                                                                                                                                   #
#   6.  Use cmake --install . to copy executable files in output directory                                                                          #
#                                                                                                                                                   #
#####################################################################################################################################################


cmake_minimum_required(VERSION 3.17)

set(CMAKE_SYSTEM_NAME Generic)

set(CMAKE_SYSTEM_VERSION 1)

set(CMAKE_BUILD_TYPE "Debug")
set(USE_CXX OFF)
set(CMAKE_PREFIX arm-none-eabi)

set(CMAKE_C_COMPILER ${CMAKE_PREFIX}-gcc)
set(CMAKE_ASM_COMPILER ${CMAKE_PREFIX}-gcc)
set(CMAKE_AR ${CMAKE_PREFIX}-ar)
set(CMAKE_OBJCOPY ${CMAKE_PREFIX}-objcopy)
set(CMAKE_OBJDUMP ${CMAKE_PREFIX}-objdump)
set(SIZE ${CMAKE_PREFIX}-size)
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

# project settings, use folder name (name in CubeMX) as project name

string(REGEX MATCH "[^\/]+$" BUFF_FOR_PROJECT_NAME ${CMAKE_CURRENT_SOURCE_DIR})

if(USE_CXX)
    project(${BUFF_FOR_PROJECT_NAME} C CXX ASM)
    set(CMAKE_CXX_STANDARD 14)
    set(CMAKE_CXX_STANDARD_REQUIRED ON)
    set(CMAKE_CXX_COMPILER arm-none-eabi-g++)
else()
    project(${BUFF_FOR_PROJECT_NAME} C ASM)
endif()

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)


message("Variables:
        CMAKE_BUILD_TYPE = ${CMAKE_BUILD_TYPE}
        USE_CXX = ${USE_CXX}
        CMAKE_PREFIX = ${CMAKE_PREFIX}"
        )

# read Makefile for parsing
file(READ ${CMAKE_SOURCE_DIR}/Makefile MAKEFILE_BUFF)

 
# parsing for cortex kernel, fpu and vpf names
string(REGEX MATCH "-mcpu=[^\n]*" MCPU ${MAKEFILE_BUFF})
string(REGEX MATCH "[^=]*$" MCPU ${MCPU})
string(REGEX MATCH "[^-]m*$" MCORE ${MCPU})

string(REGEX MATCH "-mfpu=[^\n]*" MFPU ${MAKEFILE_BUFF})
string(REGEX MATCH "[^=]*$" MFPU ${MFPU})

string(REGEX MATCH "-mfloat-abi=[^\n]*" MFLOAT_ABI ${MAKEFILE_BUFF})
string(REGEX MATCH "[^=]*$" MFLOAT_ABI ${MFLOAT_ABI})


add_compile_options(-mfloat-abi=${MFLOAT_ABI})

if ("${MFLOAT_ABI}" STREQUAL "hard")
# hardware floating point
    add_compile_definitions(ARM_MATH_CM${MCORE}; ARM_MATH_MATRIX_CHECK; ARM_MATH_ROUNDING)
    add_compile_options(-mcpu=${MCPU} -mthumb -mfpu=${MFPU})
    add_link_options(-mcpu=${MCPU} -mthumb -mfloat-abi=${MFLOAT_ABI} -mfpu=${MFPU})

elseif ("${MFLOAT_ABI}" STREQUAL "soft")
# software floating point    
    add_compile_options(-mcpu=${MCPU} -mthumb -mthumb-interwork)
    add_compile_options(-ffunction-sections -fdata-sections -fno-common -fmessage-length=0)
endif()


 

if ("${CMAKE_BUILD_TYPE}" STREQUAL "Release")
    message(STATUS "Maximum optimization for speed")
    add_compile_options(-Ofast)

elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "RelWithDebInfo")
    message(STATUS "Maximum optimization for speed, debug info included")
    add_compile_options(-Ofast -g)

elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "MinSizeRel")
    message(STATUS "Maximum optimization for size")
    add_compile_options(-Os)

elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "Debug")
    message(STATUS "Minimal optimization, debug info included")
    add_compile_options(-Og -g)

endif ()

 

# parsing for include dir

string(REGEX MATCHALL "-I[^\ \n]*" INCLUDE_DIRS_RAW ${MAKEFILE_BUFF})

foreach(INCLUDE_DIR ${INCLUDE_DIRS_RAW})
    string(REPLACE "-I" "${CMAKE_SOURCE_DIR}/" INCLUDE_DIR ${INCLUDE_DIR})
    string(REPLACE ";" "\n" INCLUDE_DIR ${INCLUDE_DIR})
    list(APPEND INCLUDE_DIRS ${INCLUDE_DIR})
endforeach()

 

include_directories(
    ${INCLUDE_DIRS}
    # You can add your's dir with headers
    )

 

# parsing for definitions

string(REGEX MATCHALL "-D[^\ \n]*" DEFINITIONS_PARSE ${MAKEFILE_BUFF})

add_definitions(${DEFINITIONS_PARSE})

 

# parsing for source dir

file(STRINGS ${CMAKE_SOURCE_DIR}/Makefile MAKEFILE_BUFF)

foreach(A ${MAKEFILE_BUFF})
    if (A MATCHES "C_SOURCES =[^\n]*")
        string(REGEX MATCH "  .+$" A ${A})
        string(REGEX MATCHALL "[^ ]+" A ${A})
        set(SOURCE_FILES_BUF ${A})
    endif()
endforeach()

 

# find startup file
file(GLOB LINKER "startup_*.s")

#message(${LINKER})

file(GLOB SOURCES
    ${SOURCE_FILES_BUF}
    ${LINKER}
    # You can add your's source file
)

 

# find lincker script

file(GLOB LINKER_SCRIPT "STM32*.ld")


add_link_options(-Wl,-gc-sections,--print-memory-usage,-Map=${PROJECT_BINARY_DIR}/${PROJECT_NAME}.map)
add_link_options(-mcpu=${MCPU} -mthumb -mthumb-interwork)
add_link_options(-T ${LINKER_SCRIPT})


# this options for C++
if(USE_CXX)
    add_link_options(-specs=nosys.specs -lstdc++)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-register")
endif()


add_executable(${PROJECT_NAME}.elf ${SOURCES} ${LINKER_SCRIPT})


set(HEX_FILE ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.hex)
set(BIN_FILE ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.bin)

 

add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
        COMMAND ${CMAKE_OBJCOPY} -Oihex $<TARGET_FILE:${PROJECT_NAME}.elf> ${HEX_FILE}
        COMMAND ${CMAKE_OBJCOPY} -Obinary $<TARGET_FILE:${PROJECT_NAME}.elf> ${BIN_FILE}
        COMMENT "Building: 
                ${HEX_FILE}
                ${BIN_FILE}"
                )


install(TARGETS ${PROJECT_NAME}.elf
                DESTINATION ${CMAKE_CURRENT_SOURCE_DIR}/output
                )
install(FILES ${HEX_FILE}   ${BIN_FILE}
                DESTINATION ${CMAKE_CURRENT_SOURCE_DIR}/output
                )   
