cmake_minimum_required(VERSION 3.16)
project(texture3)

# ソースコード一覧
set(SOURCES
    main.cpp
    box.cpp
    trackball.cpp
    box.h
    trackball.h
)

# リソースファイル
set(RESOURCES tire.raw)

if(APPLE)
    # macOS の設定

    # アプリケーションバンドルとして設定
    add_executable(${PROJECT_NAME} MACOSX_BUNDLE ${SOURCES} ${RESOURCES})

    # tire.raw をアプリケーションバンドル (Resourcesディレクトリ) に組み込む
    set_source_files_properties(${RESOURCES} PROPERTIES MACOSX_PACKAGE_LOCATION Resources)

    # Info.plist を設定してバンドル内に配置する
    # CMakeに標準の Info.plist テンプレートを使わせるか、独自に指定します。
    # すでにディレクトリにある Info.plist を使う場合、エラー防止のため cmake の変数書式にした .in を用意します
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/Info.plist.in")
        set_target_properties(${PROJECT_NAME} PROPERTIES MACOSX_BUNDLE_INFO_PLIST "${CMAKE_CURRENT_SOURCE_DIR}/Info.plist.in")
    endif()

    # GLUT と OpenGL のフレームワークを使用
    find_library(GLUT_LIBRARY GLUT REQUIRED)
    find_library(OPENGL_LIBRARY OpenGL REQUIRED)
    target_link_libraries(${PROJECT_NAME} PRIVATE ${GLUT_LIBRARY} ${OPENGL_LIBRARY})

elseif(WIN32)
    # Windows の設定
    add_executable(${PROJECT_NAME} WIN32 ${SOURCES})

    # スタートアッププロジェクトを texture3 にする
    set_property(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}" PROPERTY VS_STARTUP_PROJECT ${PROJECT_NAME})

    # デバッグ実行時に tire.raw を読み込めるように作業ディレクトリを設定
    set_property(TARGET ${PROJECT_NAME} PROPERTY VS_DEBUGGER_WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}")
    
    # 実行ファイルの出力先に tire.raw をコピー (直接起動する用)
    add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            "${CMAKE_CURRENT_SOURCE_DIR}/${RESOURCES}"
            $<TARGET_FILE_DIR:${PROJECT_NAME}>
    )

    # FetchContent を使って freeglut ソースをダウンロードしてビルド
    include(FetchContent)
    set(FREEGLUT_BUILD_DEMOS OFF CACHE BOOL "" FORCE)
    set(FREEGLUT_BUILD_SHARED_LIBS OFF CACHE BOOL "" FORCE)
    FetchContent_Declare(
        freeglut
        GIT_REPOSITORY https://github.com/FreeGLUTProject/freeglut.git
        GIT_TAG        v3.4.0
    )
    FetchContent_MakeAvailable(freeglut)

    # main.cpp が "glut.h" に直接インクルードできるようにパスを通す
    FetchContent_GetProperties(freeglut SOURCE_DIR freeglut_SOURCE_DIR)
    target_include_directories(${PROJECT_NAME} PRIVATE "${freeglut_SOURCE_DIR}/include/GL")

    # glext.h と khrplatform.h のダウンロード
    set(GL_HEADERS_DIR "${CMAKE_CURRENT_BINARY_DIR}/gl_headers")
    if(NOT EXISTS "${GL_HEADERS_DIR}/GL/glext.h")
        file(DOWNLOAD "https://www.khronos.org/registry/OpenGL/api/GL/glext.h"
             "${GL_HEADERS_DIR}/GL/glext.h" SHOW_PROGRESS)
    endif()
    if(NOT EXISTS "${GL_HEADERS_DIR}/KHR/khrplatform.h")
        file(DOWNLOAD "https://www.khronos.org/registry/EGL/api/KHR/khrplatform.h"
             "${GL_HEADERS_DIR}/KHR/khrplatform.h" SHOW_PROGRESS)
    endif()
    target_include_directories(${PROJECT_NAME} PRIVATE "${GL_HEADERS_DIR}")

    # OpenGL と FreeGLUT をリンク
    target_link_libraries(${PROJECT_NAME} PRIVATE freeglut_static opengl32)
    target_compile_definitions(${PROJECT_NAME} PRIVATE FREEGLUT_STATIC)

elseif(UNIX)
    # Ubuntu Linux の設定
    add_executable(${PROJECT_NAME} ${SOURCES})

    # pkg-config を使用してフラグやライブラリを設定
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(GLUT REQUIRED glut)
    pkg_check_modules(GL REQUIRED gl)

    target_include_directories(${PROJECT_NAME} PRIVATE ${GLUT_INCLUDE_DIRS} ${GL_INCLUDE_DIRS})
    target_link_directories(${PROJECT_NAME} PRIVATE ${GLUT_LIBRARY_DIRS} ${GL_LIBRARY_DIRS})
    target_link_libraries(${PROJECT_NAME} PRIVATE ${GLUT_LIBRARIES} ${GL_LIBRARIES})
    target_compile_options(${PROJECT_NAME} PRIVATE ${GLUT_CFLAGS_OTHER} ${GL_CFLAGS_OTHER})

    # プログラム起動時に tire.raw を読み込めるよう、ビルド後にコピー
    add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            "${CMAKE_CURRENT_SOURCE_DIR}/${RESOURCES}"
            $<TARGET_FILE_DIR:${PROJECT_NAME}>
    )
endif()
