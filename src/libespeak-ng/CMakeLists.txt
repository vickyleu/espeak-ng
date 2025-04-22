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
  # 添加后置构建命令，在 espeak-ng 库生成后执行
  add_custom_command(TARGET espeak-ng POST_BUILD
          COMMAND ${CMAKE_COMMAND} -E echo "开始合并库..."
          COMMAND ${CMAKE_COMMAND} -E echo "espeak-ng 库位置: $<TARGET_FILE:espeak-ng>"
          COMMAND ${CMAKE_COMMAND} -E echo "ucd 库位置: ${UCD_LIB_PATH}/libucd.a"

          # 创建一个shell脚本来处理合并
          COMMAND ${CMAKE_COMMAND} -E echo "#!/bin/bash" > ${CMAKE_CURRENT_BINARY_DIR}/merge_lib.sh
          COMMAND ${CMAKE_COMMAND} -E echo "set -e" >> ${CMAKE_CURRENT_BINARY_DIR}/merge_lib.sh
          COMMAND ${CMAKE_COMMAND} -E echo "set -x" >> ${CMAKE_CURRENT_BINARY_DIR}/merge_lib.sh
          COMMAND ${CMAKE_COMMAND} -E echo "WORKDIR=${CMAKE_CURRENT_BINARY_DIR}/temp_obj" >> ${CMAKE_CURRENT_BINARY_DIR}/merge_lib.sh
          COMMAND ${CMAKE_COMMAND} -E echo "rm -rf \$WORKDIR" >> ${CMAKE_CURRENT_BINARY_DIR}/merge_lib.sh
          COMMAND ${CMAKE_COMMAND} -E echo "mkdir -p \$WORKDIR" >> ${CMAKE_CURRENT_BINARY_DIR}/merge_lib.sh
          COMMAND ${CMAKE_COMMAND} -E echo "cd \$WORKDIR" >> ${CMAKE_CURRENT_BINARY_DIR}/merge_lib.sh
          COMMAND ${CMAKE_COMMAND} -E echo "${CMAKE_AR} x $<TARGET_FILE:espeak-ng>" >> ${CMAKE_CURRENT_BINARY_DIR}/merge_lib.sh
          COMMAND ${CMAKE_COMMAND} -E echo "${CMAKE_AR} x ${UCD_LIB_PATH}/libucd.a" >> ${CMAKE_CURRENT_BINARY_DIR}/merge_lib.sh
          COMMAND ${CMAKE_COMMAND} -E echo "FILES=\$(ls *.o)" >> ${CMAKE_CURRENT_BINARY_DIR}/merge_lib.sh
          COMMAND ${CMAKE_COMMAND} -E echo "if [ -z \"\$FILES\" ]; then echo \"No object files found\"; exit 1; fi" >> ${CMAKE_CURRENT_BINARY_DIR}/merge_lib.sh
          COMMAND ${CMAKE_COMMAND} -E echo "${CMAKE_AR} cr $<TARGET_FILE:espeak-ng>.new \$FILES" >> ${CMAKE_CURRENT_BINARY_DIR}/merge_lib.sh
          COMMAND ${CMAKE_COMMAND} -E echo "${CMAKE_RANLIB} $<TARGET_FILE:espeak-ng>.new" >> ${CMAKE_CURRENT_BINARY_DIR}/merge_lib.sh
          COMMAND ${CMAKE_COMMAND} -E echo "mv $<TARGET_FILE:espeak-ng>.new $<TARGET_FILE:espeak-ng>" >> ${CMAKE_CURRENT_BINARY_DIR}/merge_lib.sh
          COMMAND ${CMAKE_COMMAND} -E echo "rm -rf \$WORKDIR" >> ${CMAKE_CURRENT_BINARY_DIR}/merge_lib.sh

          # 执行脚本
          COMMAND chmod +x ${CMAKE_CURRENT_BINARY_DIR}/merge_lib.sh
          COMMAND ${CMAKE_CURRENT_BINARY_DIR}/merge_lib.sh

          COMMENT "将 libucd.a 合并到 libespeak-ng.a"
          VERBATIM
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