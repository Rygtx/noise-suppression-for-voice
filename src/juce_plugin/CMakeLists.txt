cmake_minimum_required(VERSION 3.6)
project(rnnoise_juice_plugin LANGUAGES CXX)

set(PROJECT_NAME "rnnoise_juce_plugin")

set(CMAKE_CXX_STANDARD 14)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

set_property(GLOBAL PROPERTY USE_FOLDERS YES)
option(JUCE_ENABLE_MODULE_SOURCE_GROUPS "Show all module sources in IDE projects" ON)

# Use open-source reverse-engineered VST2 api headers
juce_set_vst2_sdk_path(${CMAKE_SOURCE_DIR}/external/FST/fst/)

set(JUCE_DEPENDENCIES
        juce::juce_audio_processors)

include_directories(${CMAKE_CURRENT_SOURCE_DIR})

# We don't want warnings from external projects, this is the simplest way to disable them.
file(GLOB_RECURSE EXTERNAL_SOURCES
        "${CMAKE_SOURCE_DIR}/external/*.c"
        "${CMAKE_SOURCE_DIR}/external/*.cpp"
        "${CMAKE_SOURCE_DIR}/external/*.h")
if(MSVC)
    set_source_files_properties(${EXTERNAL_SOURCES} PROPERTIES COMPILE_FLAGS "/W0")
else()
    set_source_files_properties(${EXTERNAL_SOURCES} PROPERTIES COMPILE_FLAGS "-w")
endif()

function(compile_plugins formats suffix default_channels)
    set(PLUGIN_NAME "${PROJECT_NAME}${suffix}")

    juce_add_plugin(${PLUGIN_NAME}
            VERSION 1.00
            PLUGIN_NAME "RNNoise suppression for voice"
            COMPANY_NAME werman
            BUNDLE_ID my.werman.noise_suppression_for_voice
            COPY_PLUGIN_AFTER_BUILD TRUE
            PLUGIN_MANUFACTURER_CODE Werm  # This has to be one uppercase, rest lower for AU formats
            PLUGIN_CODE Rnn1               # A unique four-character plugin id with at least one upper-case character
            FORMATS "${formats}"
            PRODUCT_NAME "rnnoise${suffix}"
            LV2URI "https://github.com/werman/noise-suppression-for-voice"
            VST_COPY_DIR "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/vst"
            VST3_COPY_DIR "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}"
            AU_COPY_DIR "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}"
            LV2_COPY_DIR "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}")

    set(JUCE_IMPL_SRC
            RnNoisePluginEditor.h
            RnNoiseAudioProcessor.h
            RnNoisePluginEditor.cpp
            RnNoiseAudioProcessor.cpp)
    target_sources(${PLUGIN_NAME} PRIVATE ${JUCE_IMPL_SRC})

    set_target_properties(${PLUGIN_NAME} PROPERTIES FOLDER "")

    target_compile_definitions(${PLUGIN_NAME} PRIVATE -DNUM_CHANNELS=${default_channels})
    target_compile_definitions(${PLUGIN_NAME}
            PUBLIC
            JUCE_WEB_BROWSER=0
            JUCE_USE_CURL=0
            JUCE_VST3_CAN_REPLACE_VST2=0)

    target_link_libraries(${PLUGIN_NAME}
            PRIVATE
            ${JUCE_DEPENDENCIES}
            RnNoisePluginCommon
            PUBLIC
            juce::juce_recommended_config_flags
# TODO: Enable LTO only for releases.
#            juce::juce_recommended_lto_flags
            juce::juce_recommended_warning_flags)

    set(COMPILE_OPTIONS "$<$<CONFIG:RELEASE>:-O3;>")
    target_compile_options(${PLUGIN_NAME} PRIVATE ${COMPILE_OPTIONS})

    if (MINGW)
        target_link_libraries(${PLUGIN_NAME} ${MINGW_ADDITIONAL_LINKING_FLAGS})
    endif ()
endfunction()

set(FORMATS "")

# These plugins require separate library for different channel count

if (BUILD_LV2_PLUGIN)
    list(APPEND FORMATS LV2)
endif ()

if (BUILD_VST_PLUGIN)
    list(APPEND FORMATS VST)
endif ()

if (FORMATS)
    compile_plugins("${FORMATS}" "_mono" 1)
    compile_plugins("${FORMATS}" "_stereo" 2)
endif ()

set(FORMATS "")

# These plugins are _probably_ could dynamically handle different channel count

if (BUILD_VST3_PLUGIN)
    list(APPEND FORMATS VST3)
endif ()

if (BUILD_AU_PLUGIN)
    list(APPEND FORMATS AU)
endif ()

if (BUILD_AUV3_PLUGIN)
    list(APPEND FORMATS AUv3)
endif ()

if (FORMATS)
    compile_plugins("${FORMATS}" "" 1)
endif ()