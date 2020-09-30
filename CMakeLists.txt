
# http://www.th-thielemann.de/development/cmake/cmake_qmake_to_cmake.html
# https://cmake.org/cmake/help/latest/guide/tutorial/index.html
# https://cmake.org/cmake/help/latest/manual/cmake-qt.7.html
# https://doc.qt.io/qt-5/cmake-manual.html
# https://github.com/goldendict/goldendict/blob/master/goldendict.pro
# https://github.com/TeXworks/texworks/blob/master/CMakeLists.txt

## UI
# https://stackoverflow.com/questions/16245147/unable-to-include-a-ui-form-header-of-qt5-in-cmake
# https://stackoverflow.com/questions/19761767/qt-5-cmake-fails-with-undefined-reference-to-vtable-on-hello-world-with-inc-sr

## #ifdef #endif
# https://stackoverflow.com/questions/9160335/os-specific-instructions-in-cmake-how-to
# https://gitlab.kitware.com/cmake/community/-/wikis/doc/tutorials/How-To-Write-Platform-Checks
# https://cmake.org/cmake/help/latest/command/add_definitions.html


cmake_minimum_required(VERSION 3.18)

# set the project name
project(GoldenDict)

if( NOT CMAKE_BUILD_TYPE )
    SET(CMAKE_BUILD_TYPE "Release")
endif()

# https://github.com/goldendict/goldendict/blob/master/goldendict.pro
# DEFINES += PROGRAM_DATA_DIR=\\\"$$PREFIX/share/goldendict/\\\"
set(DATA_DIR \"${CMAKE_INSTALL_PREFIX}/share/goldendict/\")
message(STATUS "DATA_DIR : ${DATA_DIR}")

set(VERSION  "1.5.0-RC2+git")

# https://blog.csdn.net/iceboy314159/article/details/104729013/
set(BRANCH_NAME "")
set(COMMIT_HASH "")
set(COMMIT_DATE "")
set(BUILD_DATE "")

find_package(Git QUIET)

if(GIT_FOUND)
    execute_process(
        COMMAND ${GIT_EXECUTABLE} symbolic-ref --short -q HEAD
        OUTPUT_VARIABLE BRANCH_NAME
        OUTPUT_STRIP_TRAILING_WHITESPACE
        ERROR_QUIET
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        )
    execute_process(
        # COMMAND ${GIT_EXECUTABLE} log -1 --pretty=format:%h-%cd --date=short
        COMMAND ${GIT_EXECUTABLE} log -1 --pretty=format:%h
        OUTPUT_VARIABLE COMMIT_HASH
        OUTPUT_STRIP_TRAILING_WHITESPACE
        ERROR_QUIET
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        )
    execute_process(
        COMMAND ${GIT_EXECUTABLE} log -1 --pretty=format:%cd --date=short
        OUTPUT_VARIABLE COMMIT_DATE
        OUTPUT_STRIP_TRAILING_WHITESPACE
        ERROR_QUIET
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        )
endif()

# get build date
execute_process(
    COMMAND date +%F-%H:%M:%S
    OUTPUT_VARIABLE BUILD_DATE
    OUTPUT_STRIP_TRAILING_WHITESPACE
    ERROR_QUIET
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    )

set(VERSION_TMP ${BRANCH_NAME}-${COMMIT_HASH}-${COMMIT_DATE}-${BUILD_DATE})
message(STATUS "VERSION_TMP : ${VERSION_TMP}")

# write VERSION_TMP to file version.txt
execute_process(
    COMMAND echo ${VERSION_TMP}
    OUTPUT_FILE version.txt
    OUTPUT_STRIP_TRAILING_WHITESPACE
    ERROR_QUIET
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    )

## -DPROGRAM_VERSION is a little bit tricky
# refer to https://github.com/Passw/goldendict/blob/master/goldendict.pro
#
# PROGRAM_VERSION=\\\"$$VERSION\\\"
#
# set(VERSION \"${COMMIT_HASH}\")
#
set(VERSION \"${VERSION_TMP}\")
add_definitions(-DHAVE_X11 -DPROGRAM_VERSION=${VERSION} -DPROGRAM_DATA_DIR=${DATA_DIR} -DMAKE_EXTRA_TIFF_HANDLER -DMAKE_FFMPEG_PLAYER -DMAKE_QTMULTIMEDIA_PLAYER -DMAKE_ZIM_SUPPORT -DMAKE_CHINESE_CONVERSION_SUPPORT)

set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_AUTOUIC ON) # !!!NOTE!!!: corresponding *.cc file must be put in add_executable()

set(CMAKE_EXE_LINKER_FLAGS "-lz -lbz2 -llzo2 -leb -lX11 -lXtst -ltiff")

find_package(Qt5 COMPONENTS Core Gui Xml Network Svg PrintSupport X11Extras Help Multimedia Sql LinguistTools  REQUIRED)

## Find Qt5WebKit and Qt5WebkitWidgets
#
# !!!----- NOTE -------!!!
# CMAKE_MODULE_PATH does not work well
# but CMAKE_PREFIX_PATH works well
#
# https://blog.csdn.net/10km/article/details/51769633
set(CMAKE_MODULE_PATH $ENV{CMAKE_MODULE_PATH})
message(STATUS "CMAKE_MODULE_PATH : ${CMAKE_MODULE_PATH}")
set(CMAKE_PREFIX_PATH $ENV{HOME}/INSTALL/qtwebkit/5.212/)
message(STATUS "CMAKE_PREFIX_PATH : ${CMAKE_PREFIX_PATH}")

find_package(Qt5WebKit REQUIRED)
find_package(Qt5WebKitWidgets REQUIRED)
find_package(ZLIB REQUIRED) # https://stackoverflow.com/questions/6173915/need-help-trying-to-make-cmake-find-third-party-libraries
find_package(X11 REQUIRED)
find_package(PkgConfig REQUIRED)

## pkg-config
# https://zhuanlan.zhihu.com/p/111155574
#
# !!!NOTE!!! below URL's method is correct
#
# https://blog.csdn.net/caspiansea/article/details/79890404
# pkg-config --list-all | grep opencc
pkg_check_modules(PKGS REQUIRED IMPORTED_TARGET vorbisfile vorbis ogg hunspell ao libavutil libavformat libavcodec libswresample liblzma libzstd opencc )

message(STATUS "pkg cflags: ${PKGS_CFLAGS}")
message(STATUS "pkg ldflags: ${PKGS_LDFLAGS}")
message(STATUS "pkg include dirs: ${PKGS_INCLUDE_DIRS}")
message(STATUS "pkg library dirs: ${PKGS_LIBRARY_DIRS}")

link_directories(${PKGS_LIBRARY_DIRS})

include_directories(.
    ./qtsingleapplication/src
    /home/u/INSTALL/qtwebkit/5.212/include/QtWebKit
    /home/u/INSTALL/qtwebkit/5.212/include/QtWebKitWidgets
    ${ZLIB_INCLUDE_DIRS}
    ${X11_INCLUDE_DIR}
    ${PKGS_INCLUDE_DIRS}
    )


## How to update *.ts (for Qt Linguist) files in CMake Qt project to *.qm files ?
#
# https://doc.qt.io/qt-5/qtlinguist-cmake-qt5-add-translation.html
# https://doc.qt.io/qt-5/qtlinguist-cmake-qt5-create-translation.html
#
# https://github.com/TeXworks/texworks/tree/master/trans
#
# https://gist.github.com/giraldeau/546ba5512a74dfe9d8ea0862d66db412
#
set(TS_FILES
    locale/ru_RU.ts
    locale/zh_CN.ts
    locale/cs_CZ.ts
    locale/de_DE.ts
    locale/el_GR.ts
    locale/bg_BG.ts
    locale/ar_SA.ts
    locale/lt_LT.ts
    locale/uk_UA.ts
    locale/vi_VN.ts
    locale/it_IT.ts
    locale/pl_PL.ts
    locale/ja_JP.ts
    locale/zh_TW.ts
    locale/sq_AL.ts
    locale/pt_BR.ts
    locale/es_AR.ts
    locale/es_BO.ts
    locale/es_ES.ts
    locale/sk_SK.ts
    locale/tr_TR.ts
    locale/qu_WI.ts
    locale/tg_TJ.ts
    locale/ay_WI.ts
    locale/be_BY.ts
    locale/be_BY@latin.ts
    locale/fr_FR.ts
    locale/ko_KR.ts
    locale/nl_NL.ts
    locale/sr_SR.ts
    locale/sv_SE.ts
    locale/tk_TM.ts
    locale/fa_IR.ts
    locale/mk_MK.ts
    locale/eo_EO.ts
    locale/fi_FI.ts
    locale/jb_JB.ts
    locale/ie_001.ts
    )

set_source_files_properties(${TS_FILES} PROPERTIES OUTPUT_LOCATION "locale")
#
# qt5_add_translation(qmFiles ${TS_FILES})
#
# Difference to qt5_create_translation()
#
# Refer to https://gist.github.com/giraldeau/546ba5512a74dfe9d8ea0862d66db412
#
qt5_create_translation(QM_FILES ${CMAKE_SOURCE_DIR} ${TS_FILES})

#configure_file(resources.qrc flags.qrc ${CMAKE_BINARY_DIR} COPYONLY)
configure_file(resources.qrc ${CMAKE_BINARY_DIR} COPYONLY)
configure_file(flags.qrc ${CMAKE_BINARY_DIR} COPYONLY)


# add the executable
add_executable(GoldenDict
    folding.cc
    main.cc
    dictionary.cc
    config.cc
    sources.cc
    mainwindow.cc
    utf8.cc
    file.cc
    bgl_babylon.cc
    bgl.cc
    initializing.cc
    article_netmgr.cc
    dictzip.c
    btreeidx.cc
    stardict.cc
    chunkedstorage.cc
    xdxf2html.cc
    iconv.cc
    lsa.cc
    htmlescape.cc
    dsl.cc
    dsl_details.cc
    filetype.cc
    fsencoding.cc
    groups.cc
    groups_widgets.cc
    instances.cc
    article_maker.cc
    scanpopup.cc
    scanflag.cc
    articleview.cc
    audioplayerfactory.cc
    multimediaaudioplayer.cc
    externalaudioplayer.cc
    externalviewer.cc
    wordfinder.cc
    groupcombobox.cc
    keyboardstate.cc
    mouseover.cc
    preferences.cc
    mutex.cc
    mediawiki.cc
    sounddir.cc
    hunspell.cc
    dictdfiles.cc
    audiolink.cc
    wstring.cc
    wstring_qt.cc
    processwrapper.cc
    hotkeywrapper.cc
    hotkeyedit.cc
    langcoder.cc
    editdictionaries.cc
    loaddictionaries.cc
    transliteration.cc
    romaji.cc
    belarusiantranslit.cc
    russiantranslit.cc
    german.cc
    website.cc
    orderandprops.cc
    language.cc
    dictionarybar.cc
    broken_xrecord.cc
    history.cc
    atomic_rename.cc
    articlewebview.cc
    zipfile.cc
    indexedzip.cc
    termination.cc
    greektranslit.cc
    webmultimediadownload.cc
    forvo.cc
    country.cc
    about.cc
    programs.cc
    parsecmdline.cc
    maintabwidget.cc
    mainstatusbar.cc
    gdappstyle.cc
    ufile.cc
    xdxf.cc
    sdict.cc
    decompress.cc
    aard.cc
    mruqmenu.cc
    dictinfo.cc
    zipsounds.cc
    stylescombobox.cc
    extlineedit.cc
    translatebox.cc
    historypanewidget.cc
    wordlist.cc
    mdictparser.cc
    mdx.cc
    voiceengines.cc
    ffmpegaudio.cc
    articleinspector.cc
    delegate.cc
    zim.cc
    gddebug.cc
    gestures.cc
    tiff.cc
    dictheadwords.cc
    fulltextsearch.cc
    ftshelpers.cc
    dictserver.cc
    helpwindow.cc
    slob.cc
    ripemd.cc
    gls.cc
    splitfile.cc
    favoritespanewidget.cc
    treeview.cc

    # *.ts => *.qm
    ${QM_FILES}

    # *.qrc AUTORCC
    resources.qrc
    flags.qrc

    # for MOC
    # ./build/moc_ffmpegaudioplayer.cpp
    # ./build/moc_audioplayerinterface.cpp
    ffmpegaudioplayer.hh
    audioplayerinterface.hh

    # for MOC
    # /usr/bin/ld: CMakeFiles/GoldenDict.dir/mainwindow.cc.o: in function `DictsPaneWidget::DictsPaneWidget(QWidget*)':
    # mainwindow.cc:(.text._ZN15DictsPaneWidgetC2EP7QWidget[_ZN15DictsPaneWidgetC5EP7QWidget]+0x36): undefined reference to `vtable for DictsPaneWidget'
    dictspanewidget.hh

    # for greaterThan(QT_MAJOR_VERSION, 4) {
    wildcard.cc

    # for !(no_epwing_support)
    epwing.cc
    epwing_book.cc
    epwing_charmap.cc

    # for chinese_conversion_support
    chinese.cc
    chineseconversion.cc

    qtsingleapplication/src/qtlocalpeer.cpp
    #   qtsingleapplication/src/qtlockedfile_unix.cpp
    qtsingleapplication/src/qtsingleapplication.cpp
    qtsingleapplication/src/qtsinglecoreapplication.cpp

    )

target_link_libraries(GoldenDict
    Qt5::Core
    Qt5::Gui
    Qt5::Xml
    Qt5::Network
    Qt5::Svg
    Qt5::PrintSupport
    Qt5::X11Extras
    Qt5::Help
    Qt5::Multimedia
    Qt5::Sql
    # Refer to TeXworks
    #
    # !!!NOTE!!!
    #
    # LinguistTools only contains some Tools used to do some jobs related to i18n
    # It is not a libray, so we can not link it as executable file
    #
    # !!!NOTE!!!
    #
    # Does not link to Qt5::LinguistTools
    # We just use LinguistTools to generate *.qm files from *.ts files
    # add Qt5 module  LinguistTools in find_package() is needed,
    # since both qt5_add_translation() and qt5_create_translation() commands are in module LinguistTools
    #
    # https://github.com/TeXworks/texworks/blob/master/CMakeLists.txt#L236-L237
    #
    #   Qt5::LinguistTools

    Qt5WebKit
    Qt5WebKitWidgets

    ${ZLIB_LIBRARIES}
    ${X11_LIBRARIES}

    ${PKGS_LDFLAGS}
    )


## INSTALLATION
#
message(STATUS "CMAKE_INSTALL_PREFIX ${CMAKE_INSTALL_PREFIX}")

install(TARGETS GoldenDict
    CONFIGURATIONS Release
    RUNTIME DESTINATION ${CMAKE_INSTALL_PREFIX}/bin
    PERMISSIONS OWNER_EXECUTE
    )

# https://gist.github.com/giraldeau/546ba5512a74dfe9d8ea0862d66db412
#
# CMAKE_BINARY_DIR
#
message(STATUS "QM_FILES : ${QM_FILES}")

message(STATUS "CMAKE_BINARY_DIR : ${CMAKE_BINARY_DIR}")

install(DIRECTORY ${CMAKE_BINARY_DIR}/locale
    DESTINATION ${CMAKE_INSTALL_PREFIX}/share/goldendict
    )

install(FILES redist/icons/goldendict.png
    DESTINATION ${CMAKE_INSTALL_PREFIX}/share/pixmaps
    )

install(FILES redist/goldendict.desktop
    DESTINATION ${CMAKE_INSTALL_PREFIX}/share/applications
    PERMISSIONS OWNER_EXECUTE
    )

install(FILES redist/goldendict.appdata.xml
    DESTINATION ${CMAKE_INSTALL_PREFIX}/share/metainfo
    )

install(DIRECTORY help DESTINATION ${CMAKE_INSTALL_PREFIX}/share/goldendict
    FILES_MATCHING PATTERN "*.qch"
    )

