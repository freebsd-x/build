# ── FreeBSD-X — full platform build (drives out/, the live-seat run build) ─────
# The compositor's source groups, shaders, and Wayland-protocol generation are
# shared with the standalone `xserver` build via core/xserver/build_common/
# meson.build (subdir'd below) — a single source of truth, so adding a compositor
# source / shader / protocol is a one-file edit there. Only the external
# dependency lists and target definitions differ per build and live in each file;
# keep those roughly aligned with core/xserver/meson.build.
project('freebsdx', ['c', 'cpp'],
  version         : '1.0.0',
  default_options : ['warning_level=3', 'cpp_std=c++23', 'c_std=c11'],
)

# ── External dependencies ──────────────────────────────────────────────────────
libdrm_dep    = dependency('libdrm',          required : true)
gbm_dep       = dependency('gbm',             required : true)
egl_dep       = dependency('egl',             required : true)
glesv2_dep    = dependency('glesv2',          required : true)
libseat_dep   = dependency('libseat',         required : true)
libinput_dep  = dependency('libinput',        required : true)
libudev_dep   = dependency('libudev',         required : true)
vulkan_dep    = dependency('vulkan',          required : true)
wayland_server_dep    = dependency('wayland-server',    required : true)
wayland_protocols_dep = dependency('wayland-protocols', version  : '>=1.15', required : true)
freetype_dep  = dependency('freetype2',       required : true)
harfbuzz_dep  = dependency('harfbuzz',        required : true)
fontconfig_dep= dependency('fontconfig',      required : true)
libpng_dep    = dependency('libpng16',        required : true)
thread_dep    = dependency('threads')
xkbcommon_dep = dependency('xkbcommon',       required : true)
poppler_cpp_dep = dependency('poppler-cpp',   required : true)
curl_dep      = dependency('libcurl',         required : true)

cpp = meson.get_compiler('cpp')

# Silence pervasive, intentional-idiom warnings so -Werror can gate on real
# issues instead. These are extension/style diagnostics the codebase relies on
# by design (fixed-size wire structs use trailing zero-length pad members and
# partial designated/aggregate initializers throughout): not bugs, and rewriting
# hundreds of sites would be churn with no benefit. Real dead-code / nodiscard /
# unused warnings are left enabled. Applied only when the compiler accepts them.
add_project_arguments(
  cpp.get_supported_arguments([
    '-Wno-zero-length-array',
    '-Wno-missing-designated-field-initializers',
    '-Wno-c99-extensions',
    '-Wno-missing-field-initializers',
    # wlroots C headers use `namespace` as a struct field; wayland_bridge.cc
    # #defines it away to include them from C++ — an intentional, isolated hack.
    '-Wno-keyword-macro',
  ]),
  language : 'cpp',
)

libutil_dep    = cpp.find_library('util',    required : true)
libpam_dep     = cpp.find_library('pam',     required : true)
libcrypto_dep  = cpp.find_library('crypto',  required : true)
libarchive_dep = cpp.find_library('archive', required : true)
libexecinfo_dep = cpp.find_library('execinfo', required : true)  # backtrace() for crash_handler

# FFmpeg — video/audio decode for VideoPlayer, Camera, MusicPlayer
libavformat_dep   = dependency('libavformat',   required : true)
libavcodec_dep    = dependency('libavcodec',    required : true)
libavutil_dep     = dependency('libavutil',     required : true)
libswscale_dep    = dependency('libswscale',    required : true)
libswresample_dep = dependency('libswresample', required : true)
ffmpeg_deps = [
  libavformat_dep, libavcodec_dep, libavutil_dep,
  libswscale_dep, libswresample_dep,
]

# gtest / gmock — optional; consumed by unit tests throughout the tree
gtest_dep = dependency('gtest', main : true, required : false)
gmock_dep = dependency('gmock',             required : false)

# ── Shared compositor fragment (single source of truth) ───────────────────────
# Defines compositor_core_srcs, xbus_service_srcs, compositor_shader_files, and
# wayland_protocol_srcs — shared verbatim with the standalone core/xserver build.
# Adding a compositor source / shader / protocol is a one-file edit there. The
# root build appends its extras (xkit UI shaders, main.cc) below.
subdir('core/xserver/build_common')

# ── GLSL shader compilation ───────────────────────────────────────────────────
glslc    = find_program('glslc', required : true)
glsl_gen = generator(glslc,
  output    : '@PLAINNAME@.spv',
  arguments : ['@INPUT@', '-o', '@OUTPUT@'],
)

# Compositor + xrender shaders come from the shared fragment; the root build also
# needs the xkit UI shaders (the standalone xserver target doesn't). Kept as a
# list of two GeneratedLists (they don't support `+`); executable() flattens it.
compiled_shaders = [
  glsl_gen.process(compositor_shader_files),
  glsl_gen.process([
    'sdk/xkit/shaders/ui_solid.vert',
    'sdk/xkit/shaders/ui_solid.frag',
    'sdk/xkit/shaders/ui_image.vert',
    'sdk/xkit/shaders/ui_image.frag',
    'sdk/xkit/shaders/ui_text.vert',
    'sdk/xkit/shaders/ui_text.frag',
  ]),
]

# ── Unified include directories ───────────────────────────────────────────────
inc_root = include_directories(
  'include',
  'core/xserver/include',
  'sdk/xrender/include',
  'sdk/xkit/include',
  'core/wayland-bridge/include',
)

# ── SDK subcomponents ─────────────────────────────────────────────────────────
# Each subdir declares a static library and a *_dep variable for use below.
#   sdk/xrender → libxrender, xrender_dep
#   sdk/xkit    → libxkit,   xkit_dep   (depends on xrender_dep)
subdir('sdk/xrender')
subdir('sdk/xkit')

# ── SDK shared library (libfreebsdx-ui) ───────────────────────────────────────
# Bundles libxrender + libxkit into a single shared library that all apps link.
libfreebsdx_ui = shared_library('freebsdx-ui',
  # crash_handler comes in via link_whole libxkit; libexecinfo below satisfies
  # its backtrace() at the final .so link.
  link_whole   : [libxrender, libxkit],
  # ABI versioning: full version tracks the project; soversion is the major only.
  # Emits libfreebsdx-ui.so.1 (SONAME) -> .so.1.0.0. Bump soversion on any
  # ABI-breaking change so old app binaries fail to load loudly (see docs/ABI.md).
  version      : meson.project_version(),
  soversion    : '1',
  dependencies : [
    freetype_dep, harfbuzz_dep, fontconfig_dep, libpng_dep, vulkan_dep,
    gbm_dep, egl_dep, glesv2_dep, thread_dep, libexecinfo_dep,
  ] + ffmpeg_deps,
  install : true,
)

freebsdx_ui_dep = declare_dependency(
  link_with           : libfreebsdx_ui,
  include_directories : inc_root,
  dependencies        : [
    freetype_dep, harfbuzz_dep, fontconfig_dep, libpng_dep, vulkan_dep,
    gbm_dep, egl_dep, glesv2_dep, thread_dep,
  ] + ffmpeg_deps,
)

# ── Installable app SDK: headers + pkg-config (A2) ────────────────────────────
# Turns the in-tree library into something an out-of-tree .xapp can build against
# with `dependency('freebsdx-ui')`. Public surface = the app-facing header trees
# under one root, <includedir>/freebsdx: shared/, xkit/, xrender/, plus the small
# xserver transport subset that xkit's public headers pull in (fd + socket +
# message pool). Server-internal headers are deliberately NOT installed.
sdk_incdir = get_option('includedir') / 'freebsdx'
install_headers('include/shared/protocol.h', 'include/shared/logger.h',
                subdir : 'freebsdx/shared')
install_subdir('sdk/xkit/include/xkit',       install_dir : sdk_incdir)
install_subdir('sdk/xrender/include/xrender',  install_dir : sdk_incdir)
install_headers('core/xserver/include/xserver/unique_fd.h',
                'core/xserver/include/xserver/ipc_socket.h',
                'core/xserver/include/xserver/xbus_message_pool.h',
                subdir : 'freebsdx/xserver')

import('pkgconfig').generate(
  libfreebsdx_ui,
  name        : 'freebsdx-ui',
  filebase    : 'freebsdx-ui',
  description : 'FreeBSD-X application SDK (xkit UI toolkit + xrender engine)',
  version     : meson.project_version(),
  subdirs     : ['freebsdx'],
  # System deps an app inherits transitively through the public headers.
  requires    : ['freetype2', 'harfbuzz', 'fontconfig', 'vulkan', 'libpng16'],
)

compositor_deps = [
  libdrm_dep, gbm_dep, egl_dep, glesv2_dep,
  libseat_dep, libinput_dep, libudev_dep,
  vulkan_dep, wayland_server_dep,
  freetype_dep, harfbuzz_dep, fontconfig_dep,
  thread_dep, libpng_dep, xkbcommon_dep,
  libpam_dep, libcrypto_dep, libarchive_dep, libutil_dep, libexecinfo_dep,
]

# ── Compositor executable ──────────────────────────────────────────────────────
executable('freebsdx',
  compositor_core_srcs + xbus_service_srcs + files('core/xserver/src/main.cc'),
  wayland_protocol_srcs,
  compiled_shaders,
  include_directories : inc_root,
  dependencies        : compositor_deps,
  install             : true,
)

# Tiny setuid-root PAM helper: lets the unprivileged compositor verify a
# password without running the whole display server as root. install_mode makes
# `sudo meson install` set it setuid root (owner root, mode 4755) automatically —
# exactly how the base system ships su/login — so no manual chmod is needed.
executable('xauthcheck',
  ['core/xserver/src/xauthcheck.cc'],
  include_directories : inc_root,
  dependencies        : [libpam_dep],
  install             : true,
  install_mode        : ['rwsr-xr-x', 'root', 'root'],
)

# xsessiond: unified seat + session daemon (phase 1 = authentication). Runs as
# root (started by rc.d / init), so the unprivileged compositor can verify a
# password over a socket instead of a per-check setuid exec. libpam for auth;
# libseat for the opt-in phase-2b libseat-client mode (XSESSIOND_LIBSEAT). No
# compositor deps. See core/xserver/docs/xsessiond-design.md.
xsessiond_exe = executable('xsessiond',
  ['core/xserver/src/xsessiond.cc'],
  include_directories : inc_root,
  dependencies        : [libpam_dep, libseat_dep],
  install             : true,
)

# Smoke test: spawn the real xsessiond on a scratch socket and drive the real
# client (protocol, single-process concurrency, auth/session gating, device
# SCM_RIGHTS brokering). Plain exit-code test (no gtest). Positive PAM is the
# live-seat tier (scripts/xsessiond-verify.sh), not this. Daemon path = argv[1].
test('xsessiond smoke',
  executable('xsessiond_smoke_test',
    'core/xserver/tests/smoke/xsessiond_daemon_test.cc',
    'core/xserver/src/xsessiond_client.cc',
    'core/xserver/src/xsessiond_device.cc',
    include_directories : inc_root,
    dependencies        : [libseat_dep],
  ),
  args    : [xsessiond_exe],
  suite   : 'smoke',
  timeout : 30,
)

# ── Service daemons ────────────────────────────────────────────────────────────
# Simple single-service daemons share ipc_socket.cc.
foreach svc : [
  ['xnotify',  'core/services/xnotify/src/main.cc',  'core/xserver/src/notification_service.cc'],
  ['xportal',  'core/services/xportal/src/main.cc',  'core/xserver/src/portal_service.cc'],
  ['xstorage', 'core/services/xstorage/src/main.cc', 'core/xserver/src/storage_service.cc'],
  ['xconfig',  'core/services/xconfig/src/main.cc',  'core/xserver/src/config_service.cc'],
  ['xsystem',  'core/services/xsystem/src/main.cc',  'core/xserver/src/system_service.cc'],
  ['xupdate',  'core/services/xupdate/src/main.cc',  'core/xserver/src/update_service.cc'],
]
  executable(svc[0],
    [svc[1], svc[2],
     'core/xserver/src/ipc_socket.cc',
     'core/xserver/src/xbus_message_pool.cc',
     'core/xserver/src/broadcast_stub.cc',
     'core/xserver/src/crash_handler.cc'],
    include_directories : inc_root,
    dependencies        : [thread_dep, libexecinfo_dep],
    install             : true,
  )
endforeach

# xaudio needs privacy_service.cc (mic-mute enforcement checks PrivacyService::micBlocked())
executable('xaudio',
  ['core/services/xaudio/src/main.cc',
   'core/xserver/src/audio_service.cc',
   'core/xserver/src/privacy_service.cc',
   'core/xserver/src/config_service.cc',
   'core/xserver/src/ipc_socket.cc',
   'core/xserver/src/xbus_message_pool.cc',
   'core/xserver/src/broadcast_stub.cc',
   'core/xserver/src/crash_handler.cc'],
  include_directories : inc_root,
  dependencies        : [thread_dep, libexecinfo_dep],
  install             : true,
)

# xnetwork needs two service files
executable('xnetwork',
  ['core/services/xnetwork/src/main.cc',
   'core/xserver/src/network_service.cc',
   'core/xserver/src/config_service.cc',
   'core/xserver/src/ipc_socket.cc',
   'core/xserver/src/xbus_message_pool.cc',
   'core/xserver/src/broadcast_stub.cc',
   'core/xserver/src/crash_handler.cc'],
  include_directories : inc_root,
  dependencies        : [thread_dep, libexecinfo_dep],
  install             : true,
)

# ── Desktop shell ──────────────────────────────────────────────────────────────
executable('xshell',
  ['core/xshell/src/main.cc', 'core/xshell/src/shell_panels.cc', 'core/xshell/src/app_discovery.cc'],
  include_directories : [inc_root, include_directories('core/xshell/include')],
  dependencies        : [freebsdx_ui_dep, thread_dep],
  install             : true,
)

# ── Applications — standard (freebsdx_ui_dep only) ───────────────────────────
foreach app : [
  ['demoapp',     'apps/DemoApp.xapp/src/main.cc'],
  ['settings_app','apps/Settings.xapp/src/main.cc'],
  ['installer',   'apps/Installer.xapp/src/main.cc'],
  ['calculator',  'apps/Calculator.xapp/src/main.cc'],
  ['appmanager',  'apps/AppManager.xapp/src/main.cc'],
  ['files',       'apps/Files.xapp/src/main.cc'],
  ['browser',     'apps/Browser.xapp/src/main.cc'],
  ['imageviewer', 'apps/ImageViewer.xapp/src/main.cc'],
  ['calendar',    'apps/Calendar.xapp/src/main.cc'],
  ['texteditor',  'apps/TextEditor.xapp/src/main.cc'],
  ['terminal',    'apps/Terminal.xapp/src/main.cc'],
  ['contacts',    'apps/Contacts.xapp/src/main.cc'],
  ['notes',       'apps/Notes.xapp/src/main.cc'],
  ['archive',     'apps/Archive.xapp/src/main.cc'],
  ['colorpicker', 'apps/ColorPicker.xapp/src/main.cc'],
  ['weather',     'apps/Weather.xapp/src/main.cc'],
  ['diskmanager', 'apps/DiskManager.xapp/src/main.cc'],
  ['fontmanager', 'apps/FontManager.xapp/src/main.cc'],
  ['taskmanager', 'apps/TaskManager.xapp/src/main.cc'],
]
  executable(app[0], app[1],
    include_directories : inc_root,
    dependencies        : [freebsdx_ui_dep],
    install             : true,
  )
endforeach

# ── Applications — with extra dependencies ────────────────────────────────────
executable('gpudemo',
  'apps/GpuDemo.xapp/src/main.cc',
  include_directories : inc_root,
  dependencies        : [freebsdx_ui_dep, egl_dep, glesv2_dep, libdrm_dep, gbm_dep],
  install             : true,
)

executable('xlogin',
  'core/xlogin/src/main.cc',
  include_directories : inc_root,
  dependencies        : [freebsdx_ui_dep, libpam_dep],
  install             : true,
)

# Session components (spawned by compositor services, not launched from the
# Launchpad): the lock screen (LockService) and the screen saver
# (ScreenSaverService). Auth is service-side, so these need only the UI dep.
executable('lockscreen',
  'core/lockscreen/src/main.cc',
  include_directories : inc_root,
  dependencies        : [freebsdx_ui_dep],
  install             : true,
)
executable('screensaver',
  'core/screensaver/src/main.cc',
  include_directories : inc_root,
  dependencies        : [freebsdx_ui_dep],
  install             : true,
)

foreach app : [
  ['videoplayer', 'apps/VideoPlayer.xapp/src/main.cc'],
  ['camera',      'apps/Camera.xapp/src/main.cc'],
  ['musicplayer', 'apps/MusicPlayer.xapp/src/main.cc'],
]
  executable(app[0], app[1],
    include_directories : inc_root,
    dependencies        : [freebsdx_ui_dep] + ffmpeg_deps,
    install             : true,
  )
endforeach

executable('pdfviewer',
  'apps/PDFViewer.xapp/src/main.cc',
  include_directories : inc_root,
  dependencies        : [freebsdx_ui_dep, poppler_cpp_dep],
  install             : true,
)

# Maps needs libcurl (OSM tiles + Nominatim geocoding + GPS) and libpng (tiles).
executable('maps',
  'apps/Maps.xapp/src/main.cc',
  include_directories : inc_root,
  dependencies        : [freebsdx_ui_dep, curl_dep, libpng_dep],
  install             : true,
)

# Mail needs libcurl (IMAP + SMTP transport).
executable('mail',
  'apps/Mail.xapp/src/main.cc',
  include_directories : inc_root,
  dependencies        : [freebsdx_ui_dep, curl_dep],
  install             : true,
)

# ── Tools & utilities ─────────────────────────────────────────────────────────
executable('xpkg',
  ['tools/xpkg.cc', 'core/xshell/src/app_discovery.cc'],
  include_directories : [inc_root, include_directories('core/xshell/include')],
  dependencies        : [freebsdx_ui_dep],
  install             : true,
)

foreach tool : [
  ['freebsdx-diag', ['tools/diag.cc',
                     'core/xserver/src/seat_manager.cc',
                     'core/xserver/src/drm_device.cc',
                     'core/xserver/src/cursor_glyphs.cc']],
  ['freebsdx-audit',['tools/system_audit.cc',
                     'core/xserver/src/seat_manager.cc',
                     'core/xserver/src/drm_device.cc',
                     'core/xserver/src/cursor_glyphs.cc']],
]
  executable(tool[0], tool[1],
    include_directories : inc_root,
    dependencies        : [freebsdx_ui_dep, libseat_dep, libdrm_dep, gbm_dep, vulkan_dep],
    install             : true,
  )
endforeach

executable('freebsdx-verify',
  ['tools/auto_verify.cc',
   'core/xshell/src/app_discovery.cc',
   'core/xserver/src/seat_manager.cc',
   'core/xserver/src/drm_device.cc',
   'core/xserver/src/cursor_glyphs.cc'],
  include_directories : [inc_root, include_directories('core/xshell/include')],
  dependencies        : [freebsdx_ui_dep, libseat_dep, libdrm_dep, gbm_dep, vulkan_dep],
  install             : true,
)

# Crash spool collector/dedup — pure C++ (std::filesystem), no UI deps.
executable('xcrash-collect', 'tools/crash_collect.cc',
  include_directories : inc_root,
  install             : true,
)

foreach tool : [
  ['freebsdx-diag-ui',  'tools/diag_ui.cc'],
  ['freebsdx-health',   'tools/runtime_health_check.cc'],
  ['freebsdx-monoaudio-smoke', 'tools/monoaudio_smoke.cc'],
  ['freebsdx-truetone-smoke',  'tools/truetone_smoke.cc'],
  ['freebsdx-optstorage-smoke','tools/optstorage_smoke.cc'],
  ['freebsdx-stagemanager-smoke','tools/stagemanager_smoke.cc'],
  ['freebsdx-trusteddevices-smoke','tools/trusteddevices_smoke.cc'],
  ['freebsdx-loginitems-smoke','tools/loginitems_smoke.cc'],
  ['freebsdx-defaultapps-smoke','tools/defaultapps_smoke.cc'],
  ['freebsdx-keyboardshortcuts-smoke','tools/keyboardshortcuts_smoke.cc'],
  ['freebsdx-headless-input-smoke','tools/headless_input_smoke.cc'],
  ['freebsdx-headless-gui-smoke','tools/headless_gui_smoke.cc'],
  ['freebsdx-headless-widget-smoke','tools/headless_widget_smoke.cc'],
  ['freebsdx-headless-capture','tools/headless_capture.cc'],
]
  executable(tool[0], tool[1],
    include_directories : inc_root,
    dependencies        : [freebsdx_ui_dep],
    install             : true,
  )
endforeach

executable('freebsdx-cursor-preview',
  ['tools/cursor_preview.cc',
   'core/xserver/src/cursor_glyphs.cc'],
  include_directories : inc_root,
  install             : false,
)

executable('icon_gen',
  'tools/icon_gen.cc',
  include_directories : inc_root,
  dependencies        : [libpng_dep],
  install             : false,
)

# Preview tool: rasterize icon bundles (SVG/PNG) through xrender and tile them
# into one PNG. Companion to icon_gen.
executable('icon_montage',
  'tools/icon_montage.cc',
  include_directories : [inc_root, include_directories('sdk/xrender/include')],
  dependencies        : [freebsdx_ui_dep, libpng_dep],
  install             : false,
)

executable('xserver_visual_test',
  ['core/xserver/src/visual_debugger.cc',
   'core/xserver/src/drm_device.cc',
   'core/xserver/src/cursor_glyphs.cc',
   'core/xserver/src/seat_manager.cc'],
  include_directories : inc_root,
  dependencies        : [libdrm_dep, libseat_dep],
  install             : false,
)

# ── xserver unit tests ────────────────────────────────────────────────────────
# These live in core/xserver/meson.build for the standalone build but must be
# declared here too so they are picked up by the root `meson test --suite unit`.
if gtest_dep.found()

  # WindowStack — pure logic: hit-testing, geometry, z-order, workspace switching
  test('xserver window_stack',
    executable('xserver_window_stack_test',
      'core/xserver/tests/unit/window_stack_test.cc',
      include_directories : inc_root,
      dependencies        : [gtest_dep, vulkan_dep],
    ),
    suite : 'unit',
  )

  # Protocol structs — sizes, field layout, RenderMessage geometry round-trip
  test('xserver protocol',
    executable('xserver_protocol_test',
      'core/xserver/tests/unit/protocol_test.cc',
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  # IpcSocket transport — SEQPACKET boundary preservation, oversized-datagram
  # truncation tolerance, and SCM_RIGHTS fd passing
  test('xserver ipc transport',
    executable('xserver_ipc_transport_test',
      ['core/xserver/tests/unit/ipc_transport_test.cc',
       'core/xserver/src/ipc_socket.cc',
       'core/xserver/src/xbus_message_pool.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep, thread_dep],
    ),
    suite : 'unit',
  )

  # i18n Translator — message-catalog parse/lookup/locale logic
  test('xkit translator',
    executable('xkit_translator_test',
      ['sdk/xkit/tests/unit/translator_test.cc',
       'sdk/xkit/src/translator.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  # Screen-reader phrasing — pure node→utterance logic
  test('xkit a11y_phrasing',
    executable('xkit_a11y_phrasing_test',
      ['sdk/xkit/tests/unit/a11y_phrasing_test.cc',
       'sdk/xkit/src/a11y_phrasing.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  # Network connection parsing — pure sockstat parser
  test('xserver net_conn',
    executable('xserver_net_conn_test',
      ['core/xserver/tests/unit/net_conn_test.cc',
       'core/xserver/src/net_conn.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  # NMEA 0183 GPS sentence parsing — pure logic
  test('xserver nmea_parse',
    executable('xserver_nmea_parse_test',
      ['core/xserver/tests/unit/nmea_parse_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  # DNS + Proxy config — pure parse/serialize logic
  test('xserver net_config',
    executable('xserver_net_config_test',
      ['core/xserver/tests/unit/net_config_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  # WoL magic-packet + saved-Wi-Fi parsing — pure logic
  test('xserver net_extra',
    executable('xserver_net_extra_test',
      ['core/xserver/tests/unit/net_extra_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  # Process sort/page + syslog line parse — pure logic
  test('xserver proc_syslog',
    executable('xserver_proc_syslog_test',
      ['core/xserver/tests/unit/proc_syslog_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  # pkg upgrade parser — pure logic
  test('xserver pkg_parse',
    executable('xserver_pkg_parse_test',
      ['core/xserver/tests/unit/pkg_parse_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  test('xserver usb_list',
    executable('xserver_usb_list_test',
      ['core/xserver/tests/unit/usb_list_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  test('xserver smart_list',
    executable('xserver_smart_list_test',
      ['core/xserver/tests/unit/smart_list_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  test('xserver sysadmin_parse',
    executable('xserver_sysadmin_parse_test',
      ['core/xserver/tests/unit/sysadmin_parse_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  test('xserver kmod_list',
    executable('xserver_kmod_list_test',
      ['core/xserver/tests/unit/kmod_list_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  test('xserver battery_health',
    executable('xserver_battery_health_test',
      ['core/xserver/tests/unit/battery_health_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  test('xserver audio_sinks',
    executable('xserver_audio_sinks_test',
      ['core/xserver/tests/unit/audio_sinks_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  test('xserver gpu_info',
    executable('xserver_gpu_info_test',
      ['core/xserver/tests/unit/gpu_info_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  test('xserver frame_perf',
    executable('xserver_frame_perf_test',
      ['core/xserver/tests/unit/frame_perf_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  test('xserver battery_drain',
    executable('xserver_battery_drain_test',
      ['core/xserver/tests/unit/battery_drain_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  test('xserver keymap',
    executable('xserver_keymap_test',
      ['core/xserver/tests/unit/keymap_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  test('xserver adblock',
    executable('xserver_adblock_test',
      ['core/xserver/tests/unit/adblock_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  test('xserver location',
    executable('xserver_location_test',
      ['core/xserver/tests/unit/location_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  test('xserver energy',
    executable('xserver_energy_test',
      ['core/xserver/tests/unit/energy_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  test('xserver color_profile',
    executable('xserver_color_profile_test',
      ['core/xserver/tests/unit/color_profile_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  test('xserver dns_resolver',
    executable('xserver_dns_resolver_test',
      ['core/xserver/tests/unit/dns_resolver_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  test('xserver priv_ops',
    executable('xserver_priv_ops_test',
      ['core/xserver/tests/unit/priv_ops_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  test('xserver firewall',
    executable('xserver_firewall_test',
      ['core/xserver/tests/unit/firewall_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  test('xserver backlight',
    executable('xserver_backlight_test',
      ['core/xserver/tests/unit/backlight_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  test('xserver display_arrange',
    executable('xserver_display_arrange_test',
      ['core/xserver/tests/unit/display_arrange_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  # Do-Not-Disturb expiry math — pure logic
  test('xserver dnd_time',
    executable('xserver_dnd_time_test',
      ['core/xserver/tests/unit/dnd_time_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  # System-inspection helpers — pure osVersion + volume paging
  test('xserver sys_inspect',
    executable('xserver_sys_inspect_test',
      ['core/xserver/tests/unit/sys_inspect_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  # Login-session helpers — pure classify/id/paging logic
  test('xserver session_info',
    executable('xserver_session_info_test',
      ['core/xserver/tests/unit/session_info_test.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  # Window tiling geometry — pure master/stack layout math
  test('xserver window_layout',
    executable('xserver_window_layout_test',
      'core/xserver/tests/unit/window_layout_test.cc',
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  # PermissionStore — per-app capability grant logic + persistence
  test('xserver permission_store',
    executable('xserver_permission_store_test',
      ['core/xserver/tests/unit/permission_store_test.cc',
       'core/xserver/src/permission_store.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep, thread_dep],
    ),
    suite : 'unit',
  )

  # AuthThrottle — brute-force back-off / lockout decision logic
  test('xserver auth_throttle',
    executable('xserver_auth_throttle_test',
      ['core/xserver/tests/unit/auth_throttle_test.cc',
       'core/xserver/src/auth_throttle.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep, thread_dep],
    ),
    suite : 'unit',
  )

  # SecurityEvents service — audit log record/list/paginate/persist over XBus
  test('xserver security_events',
    executable('xserver_security_events_test',
      ['core/xserver/tests/unit/security_events_test.cc',
       'core/xserver/src/security_events_service.cc',
       'core/xserver/src/ipc_socket.cc',
       'core/xserver/src/xbus_message_pool.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep, thread_dep],
    ),
    suite : 'unit',
  )

  # CrashReporter service — spool scan/list/paginate/delete over XBus
  test('xserver crash_reporter',
    executable('xserver_crash_reporter_test',
      ['core/xserver/tests/unit/crash_reporter_test.cc',
       'core/xserver/src/crash_reporter_service.cc',
       'core/xserver/src/ipc_socket.cc',
       'core/xserver/src/xbus_message_pool.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep, thread_dep],
    ),
    suite : 'unit',
  )

  # XBusMessagePool — lock-free slab: acquire/release, exhaustion, RAII, concurrency
  test('xserver message_pool',
    executable('xserver_message_pool_test',
      'core/xserver/tests/unit/message_pool_test.cc',
      'core/xserver/src/xbus_message_pool.cc',
      include_directories : inc_root,
      dependencies        : [gtest_dep, thread_dep],
    ),
    suite : 'unit',
  )

  # shell_util.h — shellQuote and isValidXxx predicates (header-only, no deps)
  test('xserver shell_util',
    executable('xserver_shell_util_test',
      'core/xserver/tests/unit/shell_util_test.cc',
      include_directories : inc_root,
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  # xshell shell_helpers.h — parseTimerInput, fmtCountdown, parseCalInput,
  # fmtHeatmap, fmtNetSpark, dndCountdownStr (header-only, no deps)
  test('xshell helpers',
    executable('xshell_helpers_test',
      'core/xshell/tests/helpers_test.cc',
      include_directories : [inc_root,
        include_directories('core/xshell/src'),
        include_directories('core/xshell/include')],
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  # shell_query — quickMathEval recursive-descent evaluator + searchFiles
  test('xshell query',
    executable('xshell_query_test',
      'core/xshell/tests/query_test.cc',
      include_directories : [inc_root,
        include_directories('core/xshell/src'),
        include_directories('core/xshell/include')],
      dependencies        : [gtest_dep],
    ),
    suite : 'unit',
  )

  # panel builders — construct all 149 buildXPanel(ctx) with a null XBus.
  # Links the real shell_panels.cc against test-only stub definitions of the
  # file-scope state main.cc normally provides (panels_test_stubs.cc).
  test('xshell panels smoke',
    executable('xshell_panels_smoke_test',
      ['core/xshell/tests/panels_smoke_test.cc',
       'core/xshell/tests/panels_test_stubs.cc',
       'core/xshell/src/shell_panels.cc',
       'core/xshell/src/app_discovery.cc'],
      include_directories : [inc_root,
        include_directories('core/xshell/src'),
        include_directories('core/xshell/include')],
      dependencies        : [gtest_dep, freebsdx_ui_dep, thread_dep],
    ),
    suite : 'unit',
  )

  # XBusClient framing — HandleIncoming must drain every coalesced XBusMessage
  test('xkit xbus framing',
    executable('xkit_xbus_framing_test',
      ['sdk/xkit/tests/xbus_framing_test.cc',
       'sdk/xkit/src/xbus_client.cc',
       'core/xserver/src/ipc_socket.cc',
       'core/xserver/src/xbus_message_pool.cc'],
      include_directories : inc_root,
      dependencies        : [gtest_dep, thread_dep],
    ),
    suite : 'unit',
  )

  # MimeService — mimeFromExtension table, empty-registry handler queries
  test('xserver mime_service',
    executable('xserver_mime_service_test',
      'core/xserver/tests/unit/mime_service_test.cc',
      'core/xserver/src/mime_service.cc',
      'core/xserver/src/config_service.cc',
      'core/xserver/src/ipc_socket.cc',
      'core/xserver/src/xbus_message_pool.cc',
      include_directories : inc_root,
      dependencies        : [gtest_dep, thread_dep],
    ),
    suite : 'unit',
  )

  # MotionEngine — spring physics convergence and window animation
  test('xserver motion_engine',
    executable('xserver_motion_engine_test',
      'core/xserver/tests/unit/motion_engine_test.cc',
      'core/xserver/src/motion_engine.cc',
      include_directories : inc_root,
      dependencies        : [gtest_dep, vulkan_dep],
    ),
    suite : 'unit',
  )

  if gmock_dep.found()

    # SeatManager — VT-switch failure and successful dispatch (mock-based)
    test('xserver seat_manager',
      executable('xserver_seat_manager_test',
        'core/xserver/tests/unit/seat_manager_test.cc',
        include_directories : [inc_root,
          include_directories('core/xserver/tests')],
        dependencies        : [gtest_dep, gmock_dep],
      ),
      suite : 'unit',
    )

    # DisplayServer — DRM modeset failure and success (mock-based)
    test('xserver display_server',
      executable('xserver_display_server_test',
        'core/xserver/tests/unit/display_server_test.cc',
        include_directories : [inc_root,
          include_directories('core/xserver/tests')],
        dependencies        : [gtest_dep, gmock_dep],
      ),
      suite : 'unit',
    )

  endif

endif

# ── Integration & end-to-end tests ────────────────────────────────────────────
# Unit tests for sdk/xrender and sdk/xkit are declared in their own meson.build
# files (included via subdir() above) and automatically registered with meson test.
if gtest_dep.found()

  # IPC pipeline smoke test — socketpair-based, no hardware required.
  # Covers: round-trip, pipelining, pool-based receive, SCM_RIGHTS fd passing,
  # empty payload, and concurrent sends from multiple threads.
  test('xserver ipc_pipeline',
    executable('xserver_ipc_pipeline_test',
      'core/xserver/tests/smoke/ipc_pipeline_test.cc',
      'core/xserver/src/ipc_socket.cc',
      'core/xserver/src/xbus_message_pool.cc',
      include_directories : inc_root,
      dependencies        : [gtest_dep, thread_dep],
    ),
    suite   : 'smoke',
    timeout : 30,
  )

  # Service round-trip smoke test — drives real service handleMessage() calls over
  # a socketpair and verifies reply framing (full frame, echoed transaction_id,
  # opcode/service_name, decodable payload) + persistence round-trips. No compositor.
  test('xserver service_roundtrip',
    executable('xserver_service_roundtrip_test',
      'core/xserver/tests/smoke/service_roundtrip_test.cc',
      'core/xserver/src/ipc_socket.cc',
      'core/xserver/src/xbus_message_pool.cc',
      'core/xserver/src/net_conn.cc',
      'core/xserver/src/netconn_service.cc',
      'core/xserver/src/proxy_service.cc',
      'core/xserver/src/wol_service.cc',
      'core/xserver/src/sysinfo_service.cc',
      'core/xserver/src/mounted_vol_service.cc',
      'core/xserver/src/process_service.cc',
      'core/xserver/src/syslog_service.cc',
      'core/xserver/src/datasaver_service.cc',
      'core/xserver/src/dnd_service.cc',
      'core/xserver/src/micmute_service.cc',
      'core/xserver/src/rotation_lock_service.cc',
      'core/xserver/src/wakelock_service.cc',
      'core/xserver/src/accessibility_service.cc',
      'core/xserver/src/scrolldir_service.cc',
      'core/xserver/src/micmute_service.cc',
      'core/xserver/src/gps_service.cc',
      'core/xserver/src/timezone_service.cc',
      'core/xserver/src/disk_usage_service.cc',
      'core/xserver/src/thermal_service.cc',
      'core/xserver/src/mem_pressure_service.cc',
      'core/xserver/src/cursor_size_service.cc',
      'core/xserver/src/text_size_service.cc',
      'core/xserver/src/screen_timeout_service.cc',
      'core/xserver/src/pointer_service.cc',
      'core/xserver/src/color_filter_service.cc',
      'core/xserver/src/nightlight_service.cc',
      'core/xserver/src/touchpad_service.cc',
      'core/xserver/src/display_scale_service.cc',
      'core/xserver/src/system_sounds_service.cc',
      'core/xserver/src/privacy_service.cc',
      'core/xserver/src/focus_mode_service.cc',
      'core/xserver/src/auto_brightness_service.cc',
      'core/xserver/src/refresh_rate_service.cc',
      'core/xserver/src/dock_service.cc',
      'core/xserver/src/gesture_service.cc',
      'core/xserver/src/a11y_settings_service.cc',
      'core/xserver/src/update_channel_service.cc',
      'core/xserver/src/spatial_audio_service.cc',
      'core/xserver/src/stylus_service.cc',
      'core/xserver/src/camera_service.cc',
      'core/xserver/src/config_service.cc',
      'core/xserver/src/usb_service.cc',
      'core/xserver/src/smart_service.cc',
      'core/xserver/src/kmod_service.cc',
      'core/xserver/src/sysadmin_services.cc',
      'core/xserver/src/battery_health_service.cc',
      'core/xserver/src/audio_output_service.cc',
      'core/xserver/src/gpu_monitor_service.cc',
      'core/xserver/src/frame_perf_service.cc',
      'core/xserver/src/battery_drain_service.cc',
      'core/xserver/src/keymap_service.cc',
      'core/xserver/src/adblock_service.cc',
      'core/xserver/src/location_service.cc',
      'core/xserver/src/kbd_backlight_service.cc',
      'core/xserver/src/energy_service.cc',
      'core/xserver/src/color_profile_service.cc',
      'core/xserver/src/dns_config_service.cc',
      'core/xserver/src/firewall_service.cc',
      'core/xserver/src/backlight_service.cc',
      'core/xserver/src/display_arrange_service.cc',
      'core/xserver/src/broadcast_stub.cc',
      'core/xserver/src/priv_broker_stub.cc',
      include_directories : inc_root,
      dependencies        : [gtest_dep, thread_dep],
    ),
    suite   : 'smoke',
    timeout : 30,
  )

  # Offscreen render runtime test — Vulkan compositor renders to an in-memory
  # target (no display/swapchain); pixels are read back and verified.
  # Skipped automatically when Vulkan ICD is unavailable.
  offscreen_compositor_srcs = files(
    'core/xserver/src/vulkan_compositor.cc',
    'core/xserver/src/window_decorator.cc',
    'core/xserver/src/gpu_profiler.cc',
    'core/xserver/src/frame_perf_service.cc', # vulkan_compositor.cc publishes frame timings here
    'core/xserver/src/ipc_socket.cc',
    'core/xserver/src/xbus_message_pool.cc',
    # xrender pipeline (compositor side — typography_backend.cc not needed here)
    'sdk/xrender/src/entity.cc',
    'sdk/xrender/src/memory_allocator.cc',
    'sdk/xrender/src/texture.cc',
    'sdk/xrender/src/vulkan_context.cc',
    'sdk/xrender/src/display_list.cc',
    'sdk/xrender/src/font_atlas.cc',
    'sdk/xrender/src/text_shaper.cc',
    'sdk/xrender/src/font_manager.cc',
    'sdk/xrender/src/image_decoder.cc',
    'sdk/xrender/src/svg_engine.cc',
  )

  test('xserver offscreen_render',
    executable('xserver_offscreen_render_test',
      'core/xserver/tests/runtime/offscreen_render_test.cc',
      offscreen_compositor_srcs,
      include_directories : inc_root,
      dependencies        : [
        gtest_dep,
        vulkan_dep,
        libdrm_dep,
        freetype_dep,
        harfbuzz_dep,
        fontconfig_dep,
        libpng_dep,
        thread_dep,
      ],
    ),
    suite   : 'runtime',
    timeout : 60,
    workdir : meson.build_root(),
  )

  executable('test_ui',
    'tests/e2e/test_ui.cc',
    include_directories : inc_root,
    dependencies        : [freebsdx_ui_dep],
  )
  executable('gui_auditor',
    'tests/e2e/gui_auditor.cc',
    include_directories : inc_root,
    dependencies        : [freebsdx_ui_dep, libpng_dep],
  )
  executable('shell_a11y_test',
    'tests/e2e/shell_a11y_test.cc',
    include_directories : inc_root,
    dependencies        : [freebsdx_ui_dep],
  )
endif
