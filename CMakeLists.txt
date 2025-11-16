cmake_minimum_required(VERSION 3.16)

# set(CMAKE_VERBOSE_MAKEFILE ON)

# Path to toolchain file. This one has to be before 'project()' below
set(CMAKE_TOOLCHAIN_FILE ${CMAKE_SOURCE_DIR}/cmake/arm-none-eabi-gcc.cmake)
# Generate compile_commands.json for vscode
set(CMAKE_EXPORT_COMPILE_COMMANDS ON CACHE INTERNAL "")

# Setup project, output, and linker files
project(Test)
set(EXECUTABLE ${PROJECT_NAME}.elf)
set(LINKER_FILE ${CMAKE_SOURCE_DIR}/tm4c123gh6pm.ld)

# find system install of TivaWare libraries
# Optionally set this via command line: -DTIVAWARE_ROOT=/path/to/tivaware
set(TIVAWARE_ROOT "" CACHE PATH "Root directory of system-installed TivaWare")

if(NOT TIVAWARE_ROOT)
  # Try common locations, first existing wins (other paths are incorrect, update based on your systems)
  if(EXISTS "/usr/local/ti/tivaware_2_2_0_295")
    set(TIVAWARE_ROOT "/usr/local/ti/tivaware_2_2_0_295")
  elseif(EXISTS "/opt/ti/tivaware_2_2_0_295")
    set(TIVAWARE_ROOT "/opt/ti/tivaware_2_2_0_295")
  elseif(EXISTS "C:/ti/TivaWare_C_Series-2.2.0.295")
    set(TIVAWARE_ROOT "C:/ti/TivaWare_C_Series-2.2.0.295")
  else()
    message(WARNING "TivaWare not found in standard locations; please set TIVAWARE_ROOT manually.")
  endif()
endif()

# Check if TIVAWARE_ROOT include directory exists
if(TIVAWARE_ROOT AND EXISTS "${TIVAWARE_ROOT}/inc")
  message(STATUS "Using TivaWare from: ${TIVAWARE_ROOT}")
else()
  message(FATAL_ERROR "TIVAWARE_ROOT path not set or invalid. Set TIVAWARE_ROOT manually to a valid TivaWare root directory.")
endif()

enable_language(C ASM)
set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)

set(APP_SRC
    src/startup.c
    src/main.c
    src/PLL.c
    src/SysTick.c
)

add_executable(${EXECUTABLE} ${APP_SRC})

# List of include directories
target_include_directories(${EXECUTABLE} PRIVATE
    inc
    ${TIVAWARE_ROOT}
)

# List of compiler defines, prefix with -D compiler option
target_compile_definitions(${EXECUTABLE} PRIVATE
    -DPART_TM4C123GH6PM
    -DTARGET_IS_TM4C123_RB1
)

target_link_libraries(${EXECUTABLE} PUBLIC
    c
    m
    rdimon
    gcc
)

# Compiler options
target_compile_options(${EXECUTABLE} PRIVATE
    -mcpu=cortex-m4
    -mthumb
    -mfpu=fpv4-sp-d16
    -mfloat-abi=hard
    # Optimizations
    -ffunction-sections
    -fdata-sections
    -O0
    -g3
    # Compile warnings
    -Wall
    -Werror
    -Wextra
    -Wpedantic
)

# Linker options
target_link_options(${EXECUTABLE} PRIVATE
    -T${LINKER_FILE}
    -mcpu=cortex-m4
    -mthumb
    -mfpu=fpv4-sp-d16
    -mfloat-abi=hard
    -specs=rdimon.specs
    -Wl,--entry=ResetISR
    -Wl,-Map=${PROJECT_NAME}.map,--cref
    -Wl,--gc-sections
    -Wl,-print-memory-usage
)

# Print executable size as part of the post build process
add_custom_command(TARGET ${EXECUTABLE}
    POST_BUILD
    COMMAND ${CMAKE_SIZE_UTIL} ${EXECUTABLE}
)

# Create hex, bin and S-Record files after the build
set(BIN_FILE ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.bin)
set(S19_FILE ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.s19)
set(HEX_FILE ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.hex)
add_custom_command(TARGET ${EXECUTABLE} POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:${EXECUTABLE}> ${BIN_FILE}
    # COMMAND ${CMAKE_OBJCOPY} -O srec --srec-len=64 $<TARGET_FILE:${EXECUTABLE}> ${S19_FILE}
    # COMMAND ${CMAKE_OBJCOPY} -O ihex $<TARGET_FILE:${EXECUTABLE}> ${HEX_FILE}
    COMMENT "Building ${BIN_FILE}"
)

set(OPENOCD_CFG ${CMAKE_SOURCE_DIR}/openocd.cfg)
include(cmake/flash-target.cmake)
flash_target(${BIN_FILE} flash)

set(VSCODE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/.vscode")
include(cmake/vscode-debug.cmake)
vscode_debug(${PROJECT_NAME} ${EXECUTABLE})
