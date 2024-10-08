cmake_minimum_required(VERSION 3.10)

set(common_properties "")
set(common_libraries "")
include_directories(./include)

########## Boost (required) ####################
find_package(Boost 1.54 REQUIRED COMPONENTS program_options regex)

########## GLUT GUI Library ####################
find_package(GLUT)
if(GLUT_FOUND)
    ########## Pangolin GUI Library ####################
    find_package(Pangolin)
    if(Pangolin_FOUND)
        include_directories(${Pangolin_INCLUDE_DIRS})
        include_directories(${TOON_INCLUDE_PATHS})
        add_library(slambench-ui-pangolin  src/SLAMBenchUI_Pangolin.cpp  include/SLAMBenchUI_Pangolin.h)
        target_link_libraries(slambench-ui-pangolin ${Pangolin_LIBRARIES})

		if(CMAKE_COMPILER_IS_GNUCC AND CMAKE_CXX_COMPILER_VERSION VERSION_GREATER 8.0)
		  target_compile_options(slambench-ui-pangolin     PRIVATE "-Wno-error=catch-value=")
		endif()
    endif(Pangolin_FOUND)
endif(GLUT_FOUND)

########## PCL Library ####################
find_package(PCL)
if (PCL_FOUND)
    include_directories(${PCL_INCLUDE_DIRS})
    link_directories(${PCL_LIBRARY_DIRS})
    add_definitions(${PCL_DEFINITIONS})
    set(common_libraries "${common_libraries}" "${PCL_LIBRARIES}")
endif(PCL_FOUND)

########## Main Library ####################
set(sensors_files
src/io/sensor/AccelerometerSensor.cpp
src/io/sensor/CameraSensor.cpp
src/io/sensor/CameraSensorFinder.cpp
src/io/sensor/DepthSensor.cpp
src/io/sensor/EventCameraSensor.cpp
src/io/sensor/GroundTruthSensor.cpp
src/io/sensor/GyroSensor.cpp
src/io/sensor/IMUSensor.cpp
src/io/sensor/OdomSensor.cpp
src/io/sensor/PointCloudSensor.cpp
src/io/sensor/SensorCollection.cpp
src/io/sensor/Sensor.cpp
src/io/sensor/SensorDatabase.cpp
)

add_library(slambench-c-wrapper 
	src/library_wrapper.cpp 
)
set_target_properties(slambench-c-wrapper  PROPERTIES COMPILE_FLAGS "-fPIC")

set(MetricFiles
    src/ColumnWriter.cpp
    src/metrics/Phase.cpp
    src/metrics/ATEMetric.cpp
    src/metrics/RPEMetric.cpp
    src/metrics/PowerMetric.cpp
    src/metrics/DurationMetric.cpp
    src/metrics/MemoryMetric.cpp
    src/metrics/memory_utils/CUDAMonitor.cpp
    src/metrics/Metric.cpp
    src/metrics/MetricManager.cpp
    src/metrics/DepthEstimationMetric.cpp

    src/outputs/Output.cpp
    src/outputs/OutputManager.cpp
    src/outputs/OutputManagerWriter.cpp
    src/outputs/TrajectoryAlignmentMethod.cpp
    src/outputs/TrajectoryInterface.cpp
    
    src/values/Value.cpp
    src/values/ValueDispatch.cpp
    src/values/ValueInterface.cpp
    src/values/ValuePrinter.cpp
)

##########       XU3  MONITORING       ####################
#  Only work on the Odroid XU3 so far
###########################################################
set(common_properties "${common_properties}  -DXU3_MONITORING")
list(APPEND MetricFiles src/metrics/power_utils/XU3Monitor.cpp)
	
########## PAPI  MONITORING (optional) ####################
# You need to install papi-devel and lm_sensors-devel
###########################################################
find_package(PAPI)
if (PAPI_FOUND) 
   	include_directories(${PAPI_INCLUDE_DIRS})
	set(common_properties "${common_properties}  -DPAPI_MONITORING")
	set(common_libraries   "${common_libraries}" ${PAPI_LIBRARIES})
	list(APPEND MetricFiles src/metrics/power_utils/PAPIMonitor.cpp)
endif(PAPI_FOUND)

add_library(slambench-libraries SHARED)

target_sources(slambench-libraries PRIVATE ${MetricFiles})
set_target_properties(slambench-libraries PROPERTIES COMPILE_FLAGS "${common_properties}")

target_sources(slambench-libraries PRIVATE
	${sensors_files}
	src/io/core/Core.cpp
	src/io/FrameBuffer.cpp
	src/io/FrameBufferSource.cpp
	src/io/FrameFormat.cpp
	src/io/SLAMFile.cpp
	src/io/SLAMFrame.cpp
	src/io/PixelFormat.cpp
	src/io/InputInterface.cpp
	src/io/InputInterfaceManager.cpp
	src/io/FrameSource.cpp

	src/io/deserialisation/Deserialiser.cpp
	src/io/deserialisation/SLAMFileDeserialiser.cpp
	src/io/deserialisation/SLAMFileHeaderDeserialiser.cpp
	src/io/deserialisation/SLAMFrameDeserialiser.cpp
	src/io/deserialisation/SensorCollectionDeserialiser.cpp
	
	src/io/format/DataFormatter.cpp
	src/io/format/PointCloud.cpp
	src/io/serialisation/Serialiser.cpp
	src/io/serialisation/SLAMFileHeaderSerialiser.cpp
	src/io/serialisation/SLAMFileSerialiser.cpp
	src/io/serialisation/SLAMFrameSerialiser.cpp
	src/lodepng.cpp
)

target_sources(slambench-libraries PRIVATE
	src/SLAMBenchConfiguration.cpp
	src/sb_malloc.cpp
	src/ParameterComponent.cpp
	src/ParameterManager.cpp
	src/ResultWriter.cpp
)

target_include_directories(slambench-libraries PRIVATE ${Boost_INCLUDE_DIRS})
target_link_libraries(slambench-libraries PRIVATE pthread  ${Boost_LIBRARIES})

SET(EXTRA_LIBS_FOR_MEMORY "")

find_package(OpenCL)
if(OPENCL_FOUND)
    	SET(EXTRA_LIBS_FOR_MEMORY "${EXTRA_LIBS_FOR_MEMORY};${OPENCL_LIBRARIES}")
endif(OPENCL_FOUND)

find_package(CUDA)
if(CUDA_FOUND)
  STRING(REPLACE "_static.a" ".so" CUDA_LIBRARIES "${CUDA_LIBRARIES}")
  SET(EXTRA_LIBS_FOR_MEMORY "${EXTRA_LIBS_FOR_MEMORY};${CUDA_LIBRARIES}")
endif(CUDA_FOUND)

########## OpenNI 2.0 and OpenNI 1.5 (optional) ####################
find_package(OpenNI)
if(OpenNI_FOUND)
	message(STATUS "SLAMBENCH will use OpenNI 1.5, include path = ${OPENNI_INCLUDE_DIR}")
	include_directories(${OPENNI_INCLUDE_DIR})
	set(common_properties "${common_properties} -DDO_OPENNI15 -Wno-error=strict-aliasing ")
	set(common_libraries "${common_libraries}" "${OPENNI_LIBRARY}")

	add_library(slambench-io-oni15
        src/io/openni15/ONI15Frame.cpp
        src/io/openni15/ONI15FrameStream.cpp
        src/io/openni15/ONI15InputInterface.cpp
    )
	add_definitions(-DDO_OPENNI15=1)
	target_link_libraries(slambench-io-oni15 ${OPENNI_LIBRARIES})
    target_link_libraries(slambench-libraries PRIVATE slambench-io-oni15)
	SET_TARGET_PROPERTIES(slambench-io-oni15  PROPERTIES COMPILE_FLAGS "-fPIC -Wno-error=unused-but-set-variable -Wno-error=return-type  -Wno-error=strict-aliasing")
else(OpenNI_FOUND)
	message(STATUS "SLAMBENCH WILL NOT USE OPENNI 1.2 !!")
endif(OpenNI_FOUND)

find_package(OpenNI2)
if(OpenNI2_FOUND)
	message(STATUS "SLAMBENCH will use OpenNI 2.X, include path = ${OPENNI2_INCLUDE_PATHS}")
	include_directories(${OPENNI2_INCLUDE_PATHS})
	set(common_properties "${common_properties} -DDO_OPENNI20")
	set(common_libraries "${common_libraries}" "${OPENNI2_LIBRARIES}")

	add_library(slambench-io-oni2
        src/io/openni/ONI2Frame.cpp
        src/io/openni/ONI2FrameStream.cpp
        src/io/openni/ONI2InputInterface.cpp
    )
	add_definitions(-DDO_OPENNI20=1)
    target_link_libraries(slambench-io-oni2 ${OPENNI2_LIBRARIES})
	set_target_properties(slambench-io-oni2  PROPERTIES COMPILE_FLAGS "-fPIC -Wno-error=return-type -Wno-unused-variable -Wno-error=strict-aliasing")
    target_link_libraries(slambench-libraries PRIVATE slambench-io-oni2)
else(OpenNI2_FOUND)
	message(STATUS "SLAMBENCH WILL NOT USE OPENNI 2.X !!")
endif(OpenNI2_FOUND)

find_package(realsense2)
if(realsense2_FOUND)
	message(STATUS "Found realsense:" ${realsense2_INCLUDE_DIR})
	include_directories(${realsense2_INCLUDE_DIR})
	add_library(slambench-io-realsense
			src/io/realsense/RealSense2Frame.cpp
			src/io/realsense/RealSense2FrameStream.cpp
			src/io/realsense/RealSense2InputInterface.cpp)
	add_definitions(-DDO_REALSENSE=1)
	set_target_properties(slambench-io-realsense  PROPERTIES COMPILE_FLAGS "-fPIC -Wno-unused-variable -Wno-error=return-type  -Wno-error=strict-aliasing")
	target_link_libraries(slambench-io-realsense ${realsense2_LIBRARY})
	target_link_libraries(slambench-libraries PRIVATE slambench-io-realsense)
else()
	message(STATUS "No realsense")
endif()

target_link_libraries(slambench-libraries PRIVATE "-Wl,--no-as-needed;${EXTRA_LIBS_FOR_MEMORY};-Wl,--as-needed" ${common_libraries} dl)
set_target_properties(slambench-libraries PROPERTIES COMPILE_FLAGS "-fPIC ${common_properties}")

