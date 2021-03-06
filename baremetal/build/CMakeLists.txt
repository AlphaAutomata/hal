# from 3.3 onwards, detecting whether a string exists in a list has a new syntax using if()
cmake_minimum_required(VERSION 3.3)

######################
# Derive Directories #
######################

# get bare-metal HAL root directory
get_filename_component(HAL_BAREMETAL_DIR ${CMAKE_CURRENT_SOURCE_DIR} DIRECTORY)
# get HAL root directory
get_filename_component(HAL_ROOT_DIR ${HAL_BAREMETAL_DIR} DIRECTORY)
# get repository parent directory
get_filename_component(REPO_PARENT_DIR ${HAL_ROOT_DIR} DIRECTORY)

# get source directories
set(HAL_BAREMETAL_API_DIR ${HAL_BAREMETAL_DIR}/api)

set(HAL_BAREMETAL_IMPL_DIR ${HAL_BAREMETAL_DIR}/impl)

set(HAL_BAREMETAL_INC_DIR ${HAL_BAREMETAL_IMPL_DIR}/include)
set(HAL_BAREMETAL_SRC_DIR ${HAL_BAREMETAL_IMPL_DIR}/src)

# get CMSIS dependency directories
set(CMSIS_ROOT_DIR       ${REPO_PARENT_DIR}/AlphaCMSIS)
set(CMSIS_Core_A_INC_DIR ${CMSIS_ROOT_DIR}/CMSIS/Core_A/Include)
set(CMSIS_Core_A_SRC_DIR ${CMSIS_ROOT_DIR}/CMSIS/Core_A/Source)

# get BSP include and lib paths
if(NOT XSDK_VERSION STREQUAL "NONE")
	set(BSP_ROOT_DIR ${XSDK_WORKSPACE}/${XSDK_BSP})
	
	set(BSP_CPU0_DIR ${BSP_ROOT_DIR}/ps7_cortexa9_0)
	set(BSP_CPU1_DIR ${BSP_ROOT_DIR}/ps7_cortexa9_1)
	
	set(BSP_CPU0_INC_DIR ${BSP_CPU0_DIR}/include)
	set(BSP_CPU0_LIB_DIR ${BSP_CPU0_DIR}/lib)
	
	set(BSP_CPU1_INC_DIR ${BSP_CPU1_DIR}/include)
	set(BSP_CPU1_LIB_DIR ${BSP_CPU1_DIR}/lib)
endif()

#########################
# Gather Target Sources #
#########################

# all HAL dependency files
set(
	HAL_API_FILES
		${HAL_BAREMETAL_API_DIR}/hal_platform.h
		${HAL_BAREMETAL_API_DIR}/hal_core.h
		${HAL_BAREMETAL_API_DIR}/hal_systick.h
		${HAL_BAREMETAL_API_DIR}/hal_timer_gp.h
		${HAL_BAREMETAL_API_DIR}/hal_timer_priv.h
		${HAL_BAREMETAL_API_DIR}/hal_timer_types.h
)
set(
	HAL_INC_FILES
)
set(
	HAL_SRC_FILES
		${HAL_BAREMETAL_SRC_DIR}/hal_core.c
		${HAL_BAREMETAL_SRC_DIR}/hal_systick.c
		${HAL_BAREMETAL_SRC_DIR}/hal_timer_gp.c
)
# include assembly only when cross compiling
if(CMAKE_CROSSCOMPILING)
	message(STATUS "Configuring cross-compile build system.")
	# get the assembly sources
	set(HAL_ASM_FILES ${HAL_BAREMETAL_SRC_DIR}/arm_context.s)
else()
	message(STATUS "Configuring for compilation targeting the host system.")
	# set blank assembly sources
	set(HAL_ASM_FILES "")
endif()

# all CMSIS dependency files
set(
	CMSIS_Core_A_INC_FILES
		${CMSIS_Core_A_INC_DIR}/cmsis_compiler.h
		${CMSIS_Core_A_INC_DIR}/core_ca.h
		${CMSIS_Core_A_INC_DIR}/irq_ctrl.h
)
set(
	CMSIS_CORE_A_SRC_FILES
)

# Xilinx BSP archive
if(NOT XSDK_VERSION STREQUAL NONE)
	add_library(libxil STATIC IMPORTED)
	set_target_properties(libxil PROPERTIES IMPORTED_LOCATION ${BSP_CPU0_LIB_DIR}/libxil.a)
endif()

####################
# Configure Target #
####################

# add the platform constant to HAL files only
set_source_files_properties(
	${HAL_API_FILES}
	${HAL_INC_FILES}
	${HAL_SRC_FILES}
	PROPERTIES
		COMPILE_DEFINITIONS
			HAL_PLATFORM=HAL_PLATFORM_${ARTOS_TARGET_PLATFORM}
)
# compile bare-metal HAL as a static library
add_library(
	hal_baremetal
	STATIC
		${HAL_API_FILES}
		${HAL_INC_FILES}
		${HAL_SRC_FILES}
		${HAL_ASM_FILES}
		${CMSIS_Core_A_INC_FILES}
		${CMSIS_CORE_A_SRC_FILES}
)
# set include directories
target_include_directories(
	hal_baremetal
	PUBLIC
		${HAL_BAREMETAL_API_DIR}
		${CMSIS_Core_A_INC_DIR}
	PRIVATE
		${HAL_BAREMETAL_INC_DIR}
)
# use C11 for standard atomics
set_target_properties(
	hal_baremetal
	PROPERTIES
		C_STANDARD   11
		CXX_STANDARD 11
		C_STANDARD_REQUIRED   TRUE
		CXX_STANDARD_REQUIRED TRUE
)
# link to Xilinx BSP
if(NOT XSDK_VERSION STREQUAL "NONE")
	target_include_directories(
		hal_baremetal
		PRIVATE
			${BSP_CPU0_INC_DIR}
	)
	target_link_libraries(hal_baremetal libxil)
endif()

# configure optional test application target
if(HAL_BAREMETAL_TEST)
	set(HAL_BAREMETAL_TEST_DIR ${HAL_BAREMETAL_DIR}/test)
	
	add_executable(
		hal_baremetal_test
			${HAL_BAREMETAL_TEST_DIR}/main.c
	)
	target_link_libraries(hal_baremetal_test hal_baremetal)
endif()
