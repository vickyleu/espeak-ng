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
  # espeak-ng 已经完成静态库 libespeak-ng.a 的生成
  # 1. 生成 MRI 脚本
  set(BUNDLE_SCRIPT "${CMAKE_CURRENT_BINARY_DIR}/bundle.scr")
  file(WRITE "${BUNDLE_SCRIPT}"
          "CREATE libespeak-ng-full.a\n"
          "ADDMOD libespeak-ng.a\n"
          "ADDMOD ${UCD_LIB_PATH}/libucd.a\n"
          "SAVE\n"
          "END\n"
  )
  # 2. 在 espeak-ng 静态库 build 完成后执行打包、ranlib 并替换
  add_custom_command(TARGET espeak-ng
          POST_BUILD
          COMMAND ${CMAKE_RANLIB} "$<TARGET_FILE:espeak-ng>"
          COMMAND ${CMAKE_AR} -M < "${BUNDLE_SCRIPT}"
          COMMAND ${CMAKE_RANLIB} "${CMAKE_CURRENT_BINARY_DIR}/libespeak-ng-full.a"
          COMMAND ${CMAKE_COMMAND} -E copy
          "${CMAKE_CURRENT_BINARY_DIR}/libespeak-ng-full.a"
          "${CMAKE_CURRENT_BINARY_DIR}/libespeak-ng.a"
          COMMENT "合并 ucd 库、重建索引并替换为完整的 libespeak-ng.a"
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