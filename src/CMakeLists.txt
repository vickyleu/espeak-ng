add_library(espeak-include INTERFACE)
add_subdirectory(ucd-tools)
target_include_directories(espeak-include INTERFACE include include/compat
        ucd-tools/src/include  # 关键路径
)
if(USE_SPEECHPLAYER)
  add_subdirectory(speechPlayer)
endif()
add_subdirectory(libespeak-ng)

if(BUILD_ESPEAK_NG_EXE)
  add_executable(espeak-ng-bin espeak-ng.c)
  target_link_libraries(
    espeak-ng-bin PRIVATE espeak-ng espeak-ng-config
  )
  if (MINGW)
    target_link_options(espeak-ng-bin PRIVATE "-static-libstdc++" "-static")
  endif()
  if (MSVC)
    target_sources(espeak-ng-bin PRIVATE compat/getopt.c)
  endif()
  if (NOT WIN32)
    add_custom_target(
      speak-ng ALL
      COMMAND ${CMAKE_COMMAND} -E create_symlink espeak-ng ${CMAKE_CURRENT_BINARY_DIR}/speak-ng
      COMMAND ${CMAKE_COMMAND} -E create_symlink espeak-ng ${CMAKE_CURRENT_BINARY_DIR}/espeak
      COMMAND ${CMAKE_COMMAND} -E create_symlink espeak-ng ${CMAKE_CURRENT_BINARY_DIR}/speak
      COMMENT "Link espeak-ng to compat names"
      DEPENDS espeak-ng-bin
    )
  endif()
  install(TARGETS espeak-ng-bin)
endif()

# 无论是否构建可执行文件，始终安装头文件
install(DIRECTORY include/espeak include/espeak-ng TYPE INCLUDE)

if (ESPEAK_COMPAT AND NOT WIN32 AND BUILD_ESPEAK_NG_EXE)
  install(FILES ${CMAKE_CURRENT_BINARY_DIR}/speak-ng DESTINATION bin)
  install(FILES ${CMAKE_CURRENT_BINARY_DIR}/espeak DESTINATION bin)
  install(FILES ${CMAKE_CURRENT_BINARY_DIR}/speak DESTINATION bin)
endif()