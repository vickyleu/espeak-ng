set(ESPEAK_CONFIG_DIR ${CMAKE_CURRENT_BINARY_DIR}/include)
set(UCD_DIR ${CMAKE_CURRENT_BINARY_DIR}/../ucd-tools)
set(ESPEAK_CONFIG_H ${ESPEAK_CONFIG_DIR}/config.h)
configure_file(config.h.in ${ESPEAK_CONFIG_H})

add_library(espeak-ng-config INTERFACE)
target_include_directories(espeak-ng-config INTERFACE ${ESPEAK_CONFIG_DIR})


add_library(espeak-ng
  common.c
  mnemonics.c
  error.c
  ieee80.c

  compiledata.c
  compiledict.c

  dictionary.c
  encoding.c
  intonation.c
  langopts.c
  numbers.c
  phoneme.c
  phonemelist.c
  readclause.c
  setlengths.c
  soundicon.c
  spect.c
  ssml.c
  synthdata.c
  synthesize.c
  tr_languages.c
  translate.c
  translateword.c
  voices.c
  wavegen.c
  speech.c

  espeak_api.c
)

target_include_directories(espeak-ng BEFORE PRIVATE $<TARGET_PROPERTY:espeak-include,INTERFACE_INCLUDE_DIRECTORIES>)

if (NOT MSVC)
  target_compile_options(espeak-ng PRIVATE
    "-fPIC"
    "-fvisibility=hidden"
    "-fno-exceptions"
    "-fwrapv"

    "-pedantic"

    "-Wunused-parameter"
    "-Wunused"
    "-Wuninitialized"
    "-Wreturn-type"
    "-Wmissing-prototypes"
    "-Wint-conversion"
    "-Wimplicit"
    "-Wmisleading-indentation"
  )
endif()
target_compile_definitions(espeak-ng PRIVATE "LIBESPEAK_NG_EXPORT=1")
if (NOT BUILD_SHARED_LIBS)
  target_compile_definitions(espeak-ng INTERFACE "LIBESPEAK_NG_EXPORT=1")
endif()

if (USE_ASYNC)
  target_sources(espeak-ng PRIVATE
    event.c
    fifo.c
    espeak_command.c
  )
endif(USE_ASYNC)

if (USE_MBROLA)
  target_sources(espeak-ng PRIVATE
    mbrowrap.c
    compilembrola.c
    synth_mbrola.c
  )
endif(USE_MBROLA)

if (USE_KLATT)
  target_sources(espeak-ng PRIVATE klatt.c)
endif(USE_KLATT)

if (USE_SPEECHPLAYER)
  target_sources(espeak-ng PRIVATE sPlayer.c)
  target_link_libraries(espeak-ng PRIVATE speechPlayer)
endif(USE_SPEECHPLAYER)

if (HAVE_LIBSONIC AND USE_LIBSONIC)
  target_link_libraries(espeak-ng PRIVATE ${SONIC_LIB})
  target_include_directories(espeak-ng PRIVATE ${SONIC_INC})
endif()

if (HAVE_LIBPCAUDIO AND USE_LIBPCAUDIO)
  target_link_libraries(espeak-ng PRIVATE ${PCAUDIO_LIB})
  target_include_directories(espeak-ng PRIVATE ${PCAUDIO_INC})
endif()

target_link_libraries(espeak-ng PRIVATE espeak-ng-config )
# 展开 ucd 的路径
get_filename_component( UCD_LIB_PATH ${UCD_DIR} ABSOLUTE)
MESSAGE("UCD_LIB_PATH===${UCD_LIB_PATH}")

if (NOT BUILD_SHARED_LIBS)
  # 不要直接检查文件是否存在，因为它可能是在构建过程中创建的
  # 假设 ucd 是一个有效的目标或者会在后续过程中生成

  # 首先，确保 espeak-ng 先正确链接 ucd 目标
  target_link_libraries(espeak-ng PRIVATE ucd)

  # 然后，我们可以在适当的位置添加合并库的步骤
  # 假设我们知道 ucd 库会在哪里生成

  # 创建一个变量来存储预期的 ucd 库位置
  # 这可能需要根据您的构建系统进行调整
  set(UCD_LIB_FILE "${UCD_LIB_PATH}/libucd.a")
  set(ESPEAK_LIB_FILE "$<TARGET_FILE:espeak-ng>")
  set(MERGED_LIB "${CMAKE_CURRENT_BINARY_DIR}/libespeak-ng-full.a")

  # 添加一个自定义目标，该目标依赖于 espeak-ng 和 ucd
  add_custom_target(merge_libraries ALL
          DEPENDS espeak-ng ucd
          COMMAND ${CMAKE_COMMAND} -E echo "开始合并库..."
          COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_CURRENT_BINARY_DIR}/temp_obj
          COMMAND ${CMAKE_COMMAND} -E echo "解压 espeak-ng 库..."
          COMMAND bash -c "cd ${CMAKE_CURRENT_BINARY_DIR}/temp_obj && ${CMAKE_AR} x ${ESPEAK_LIB_FILE}"
          COMMAND ${CMAKE_COMMAND} -E echo "解压 ucd 库..."
          COMMAND bash -c "cd ${CMAKE_CURRENT_BINARY_DIR}/temp_obj && ${CMAKE_AR} x ${UCD_LIB_FILE}"
          COMMAND ${CMAKE_COMMAND} -E echo "创建合并库..."
          COMMAND bash -c "cd ${CMAKE_CURRENT_BINARY_DIR}/temp_obj && ${CMAKE_AR} crs ${MERGED_LIB} *.o"
          COMMAND ${CMAKE_RANLIB} ${MERGED_LIB}
          COMMAND ${CMAKE_COMMAND} -E echo "替换原始库..."
          COMMAND ${CMAKE_COMMAND} -E copy ${MERGED_LIB} ${ESPEAK_LIB_FILE}
          COMMAND ${CMAKE_COMMAND} -E remove_directory ${CMAKE_CURRENT_BINARY_DIR}/temp_obj
          COMMENT "正在合并 espeak-ng 和 ucd 库"
  )

else()
  target_link_libraries(espeak-ng PRIVATE ucd)
endif()

if (NOT MSVC)
  target_link_libraries(espeak-ng PRIVATE m)
endif()
target_link_libraries(espeak-ng PUBLIC espeak-include)

if (MINGW)
  target_link_options(espeak-ng PRIVATE "-static-libstdc++" "-static")
endif()



install(TARGETS espeak-ng)