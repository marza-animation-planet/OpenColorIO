import os
import re
import sys
import glob
import tarfile
import excons
from excons.tools import boost
from excons.tools import python



env = excons.MakeBaseEnv()


# OpenColorIO config
ocio_libname = excons.GetArgument("ocio-lib-name", None)
if ocio_libname is None:
   ocio_libname = "OpenColorIO"
   ocio_libsuffix = excons.GetArgument("ocio-lib-suffix", None)
   if ocio_libsuffix:
      ocio_libname += ocio_libsuffix
ocio_static_libsuffix = excons.GetArgument("ocio-static-lib-suffix", "_s")
ocio_sse2 = (excons.GetArgument("ocio-use-sse2", "1", int) != 0)
ocio_hideinlines = (excons.GetArgument("ocio-hide-inlines", "1", int) != 0)
ocio_namespace = excons.GetArgument("ocio-namespace", "OpenColorIO")
ocio_use_boost = (excons.GetArgument("ocio-use-boost", "0", int) != 0)
ocio_version = (1, 0, 9)

ocio_config = {"OCIO_NAMESPACE"     : ocio_namespace,
               "OCIO_VERSION_MAJOR" : str(ocio_version[0]),
               "OCIO_VERSION_MINOR" : str(ocio_version[1]),
               "OCIO_VERSION_PATCH" : str(ocio_version[2]),
               "OCIO_VERSION"       : ".".join(map(str, ocio_version)),
               "SOVERSION"          : str(ocio_version[0]),
               "OCIO_USE_BOOST_PTR" : str(1 if ocio_use_boost else 0)}

libdefs = []
libcflags = ""
libcppflags = ""
if sys.platform != "win32":
   libcppflags += " -Wno-unused-parameter"
   libcppflags += " -Wno-unused-variable"
   libcppflags += " -Wno-missing-field-initializers"
   if sys.platform != "darwin":
      libcppflags += " -Wno-unused-but-set-variable"
   else:
      libcppflags += " -Wno-unused-private-field"
      libcppflags += " -Wno-unused-function"
if sys.platform == "darwin":
   # OSSpinLockUnlock used in Mutex.h is deprecated starting MacOS 10.12
   libcppflags += " -Wno-deprecated-declarations"
libincdirs = ["export", "src/core", "ext/oiio/src/include"]
libcustoms = []
if ocio_sse2:
   libdefs.append("USE_SSE")
   if sys.platform != "win32":
      libcflags += " -msse2"
   else:
      #libcflags += " /arch:SSE2"
      pass
if ocio_hideinlines:
   if sys.platform != "win32":
      libcflags += " -fvisibility-inlines-hidden"
if ocio_use_boost:
   libcustoms.append(boost.Require())


projs = []


# TinyXML setup
tinyxml_libname = excons.GetArgument("tinyxml-lib-name", "tinyxml")
tinyxml_static = (excons.GetArgument("tinayxml-static", "1", int) != 0)
tinyxml_inc, tinyxml_lib = excons.GetDirs("tinyxml", silent=True)
if tinyxml_inc is None and tinyxml_lib is None:
   if not os.path.isdir("ext/tinyxml"):
      print("=== Extracting tinyxml_2_6_1.tar.gz...")
      f = tarfile.open("ext/tinyxml_2_6_1.tar.gz")
      f.extractall("ext")
      f.close()
   tinyxml_static = True
   libdefs.append("TIXML_USE_STL")
   projs.append({ "name": "tinyxml",
                  "type": "staticlib",
                  "defs": ["TIXML_USE_STL"],
                  "cflags": libcflags,
                  "cppflags": libcppflags,
                  "srcs": glob.glob("ext/tinyxml/tiny*.cpp"),
                  "custom": libcustoms})
   tinyxml_inc = "ext/tinyxml"
   tinyxml_lib = excons.OutputBaseDirectory() + "/lib"
else:
   # TINYXML_USE_STL?
   pass
tinyxml_incdirs = ([] if tinyxml_inc is None else [tinyxml_inc])
tinyxml_libdirs = ([] if tinyxml_lib is None else [tinyxml_lib])
tinyxml_staticlibs = ([tinyxml_libname] if tinyxml_static else [])
tinyxml_libs = ([] if tinyxml_static else [tinyxml_libname])

# YAMLcpp setup
yamlcpp_libname = excons.GetArgument("yamlcpp-lib-name", "yaml-cpp")
yamlcpp_static = (excons.GetArgument("yamlcpp-static", "1", int) != 0)
yamlcpp_inc, yamlcpp_lib = excons.GetDirs("yamlcpp", silent=True)
if yamlcpp_inc is None and yamlcpp_lib is None:
   if not os.path.isdir("ext/yaml-cpp"):
      print("=== Extracting yaml-cpp-0.3.0.tar.gz...")
      f = tarfile.open("ext/yaml-cpp-0.3.0.tar.gz")
      f.extractall("ext")
      f.close()
   yamlcpp_static = True
   projs.append({ "name": "yaml-cpp",
                  "type": "staticlib",
                  "defs": ["YAML_CPP_NO_CONTRIB"],
                  "cflags": libcflags,
                  "cppflags": libcppflags,
                  "incdirs": ["ext/yaml-cpp/include"],
                  "srcs": glob.glob("ext/yaml-cpp/src/[a-zA-Z]*.cpp"),
                  "custom": libcustoms})
   yamlcpp_inc = "ext/yaml-cpp/include"
   yamlcpp_lib = excons.OutputBaseDirectory() + "/lib"
   libdefs.append("OLDYAML")
else:
   # For YAML-CPP older than 0.5.0, OLDYAML has to be defined
   pass
yamlcpp_incdirs = ([] if yamlcpp_inc is None else [yamlcpp_inc])
yamlcpp_libdirs = ([] if yamlcpp_lib is None else [yamlcpp_lib])
yamlcpp_staticlibs = ([yamlcpp_libname] if yamlcpp_static else [])
yamlcpp_libs = ([] if yamlcpp_static else [yamlcpp_libname])

# LCMS setup
lcms_libname = excons.GetArgument("lcms-lib-name", "lcms2")
lcms_static = (excons.GetArgument("lcms-static", "1", int) != 0)
lcms_inc, lcms_lib = excons.GetDirs("lcms", silent=True)
if lcms_inc is None and lcms_lib is None:
   if not os.path.isdir("ext/lcms2-2.1"):
      print("=== Extracting lcms2-2.1.tar.gz...")
      f = tarfile.open("ext/lcms2-2.1.tar.gz")
      f.extractall("ext")
      f.close()
   lcms_static = True
   lcms_cppflags = libcppflags
   if sys.platform != "win32":
      lcms_cppflags += " -Wno-strict-aliasing"
   projs.append({ "name": "lcms2",
                  "type": "staticlib",
                  "cflags": libcflags,
                  "cppflags": lcms_cppflags,
                  "incdirs": ["ext/lcms2-2.1/include"],
                  "srcs": glob.glob("ext/lcms2-2.1/src/*.c"),
                  "custom": libcustoms})
   lcms_inc = "ext/lcms2-2.1/include"
   lcms_lib = excons.OutputBaseDirectory() + "/lib"
else:
   # For lcms2
   pass
lcms_incdirs = ([] if lcms_inc is None else [lcms_inc])
lcms_libdirs = ([] if lcms_lib is None else [lcms_lib])
lcms_staticlibs = ([lcms_libname] if lcms_static else [])
lcms_libs = ([] if lcms_static else [lcms_libname])
lcms_cppflags = ""
if sys.platform != "win32":
   lcms_cppflags += " -Wno-strict-aliasing"

# TODO
# - Nuke project

def GenerateConfig(target, source, env):
   with open(str(source[0]), "r") as src:
      with open(str(target[0]), "w") as dst:
         e = re.compile("@([^@]+)@")
         for line in src.readlines():
            m = e.search(line)
            if m is not None:
               opt = m.group(1)
               if opt in ocio_config:
                  # LLVM backend build not yet supported
                  line = line.replace(m.group(0), ocio_config[opt])
               else:
                  excons.WarnOnce("Unsupported config option '%s'" % opt)
            dst.write(line)
   return None

env["BUILDERS"]["GenerateConfig"] = Builder(action=Action(GenerateConfig, "Generating $TARGET ...", suffix=".h", src_suffix=".h.in"))

abih = env.GenerateConfig("export/OpenColorIO/OpenColorABI.h.in")

includeBasedir = "%s/include/OpenColorIO" % excons.OutputBaseDirectory()
InstallHeaders  = env.Install(includeBasedir, glob.glob("export/OpenColorIO/*.h"))
InstallHeaders += env.Install(includeBasedir, abih)

env.Command("src/pyglue/PyDoc.h", "src/pyglue/createPyDocH.py", "python $SOURCE $TARGET")

projs.extend([
   {  "name": ocio_libname + ocio_static_libsuffix,
      "type": "staticlib",
      "alias": "staticlib",
      "defs": libdefs + ["OpenColorIO_STATIC"],
      "cflags": libcflags,
      "cppflags": libcppflags,
      "incdirs": libincdirs + tinyxml_incdirs + yamlcpp_incdirs,
      "srcs": glob.glob("src/core/*.cpp") +
              glob.glob("src/core/md5/*.cpp") +
              glob.glob("src/core/pystring/*.cpp"),
      "libdirs": tinyxml_libdirs + yamlcpp_libdirs,
      "staticlibs": tinyxml_staticlibs + yamlcpp_staticlibs,
      "libs": tinyxml_libs + yamlcpp_libs,
      "custom": libcustoms
   },
   {  "name": ocio_libname,
      "type": "sharedlib",
      "alias": "sharedlib",
      "version": ".".join(map(str, ocio_version)),
      "soname": "lib%s.so.%d" % (ocio_libname, ocio_version[0]),
      "install_name": "lib%s.%d.dylib" % (ocio_libname, ocio_version[0]),
      "defs": libdefs + ["OpenColorIO_EXPORTS"],
      "cflags": libcflags,
      "cppflags": libcppflags,
      "incdirs": libincdirs + tinyxml_incdirs + yamlcpp_incdirs,
      "srcs": glob.glob("src/core/*.cpp") +
              glob.glob("src/core/md5/*.cpp") +
              glob.glob("src/core/pystring/*.cpp"),
      "libdirs": tinyxml_libdirs + yamlcpp_libdirs,
      "staticlibs": tinyxml_staticlibs + yamlcpp_staticlibs,
      "libs": tinyxml_libs + yamlcpp_libs,
      "custom": libcustoms
   },
   # Python binding
   {  "name": "PyOpenColorIO",
      "type": "dynamicmodule",
      "alias": "python",
      "ext": python.ModuleExtension(),
      "prefix": python.ModulePrefix() + "/" + python.Version(),
      "incdirs": libincdirs + ["src/pyglue"],
      "defs": ["OpenColorIO_STATIC", "PYOCIO_NAME=PyOpenColorIO"],
      "cflags": libcflags,
      "cppflags": libcppflags,
      "srcs": glob.glob("src/pyglue/*.cpp"),
      "libdirs": tinyxml_libdirs + yamlcpp_libdirs,
      "staticlibs": [ocio_libname + ocio_static_libsuffix] + tinyxml_staticlibs + yamlcpp_staticlibs,
      "libs": tinyxml_libs + yamlcpp_libs,
      "custom": libcustoms + [python.SoftRequire],
      "deps": [ocio_libname + ocio_static_libsuffix]
   },
   # Command line tools
   {  "name": "ociobakelut",
      "type": "program",
      "defs": ["OpenColorIO_STATIC"],
      "cflags": libcflags,
      "cppflags": libcppflags,
      "incdirs": libincdirs + ["src/apps/share"] + lcms_incdirs,
      "srcs": glob.glob("src/apps/ociobakelut/*.cpp") + ["src/apps/share/strutil.cpp", "src/apps/share/argparse.cpp"],
      "libdirs": tinyxml_libdirs + yamlcpp_libdirs + lcms_libdirs,
      "staticlibs": [ocio_libname + ocio_static_libsuffix] + tinyxml_staticlibs + yamlcpp_staticlibs + lcms_staticlibs,
      "libs": tinyxml_libs + yamlcpp_libs + lcms_libs,
      "custom": libcustoms
   },
   {  "name": "ociocheck",
      "type": "program",
      "defs": ["OpenColorIO_STATIC"],
      "cflags": libcflags,
      "cppflags": libcppflags,
      "incdirs": libincdirs + ["src/apps/share"],
      "srcs": glob.glob("src/apps/ociocheck/*.cpp") + ["src/apps/share/strutil.cpp", "src/apps/share/argparse.cpp"],
      "libdirs": tinyxml_libdirs + yamlcpp_libdirs,
      "staticlibs": [ocio_libname + ocio_static_libsuffix] + tinyxml_staticlibs + yamlcpp_staticlibs,
      "libs": tinyxml_libs + yamlcpp_libs,
      "custom": libcustoms
   }
])



ocio_build_opts = """OPENCOLORIO OPTIONS
   lib-name                     : Library name                       ["OpenColorIO"]
   lib-suffix=<str>             : Library suffix                     [""]
                                  (Ignored if lib-name flag is set)
   static-lib-suffix=<str>      : Static library addition suffix     ["_s"]
   namespace=<str>              : Library namespace                  ["OpenColorIO"]
   use-sse2=0|1                 : Use SSE2 instructions              [1]
   use-boost=0|1                : Use boost shared_ptr               [0]
   hide-inlines=0|1             : Hide inline functions              [1]"""

tinyxml_build_opts = """TINYXML OPTIONS
   with-tinyxml=<str>           : TinyXML prefix                     [""]
   with-tinyxml-inc=<str>       : TinyXML headers directory          [<prefix>/include]
   with-tinyxml-lib=<str>       : TinyXML libraries directory        [<prefix>/lib]
   tinyxml-lib-name=<str>       : TinyXML library name               ["tinyxml"]
   tinyxml-static=0|1           : Use static TinyXML static library  [1]"""

yamlcpp_build_opts = """YAMLCPP OPTIONS
   with-yamlcpp=<str>           : YAMLcpp prefix                     [""]
   with-yamlcpp-inc=<str>       : YAMLcpp headers directory          [<prefix>/include]
   with-yamlcpp-lib=<str>       : YAMLcpp libraries directory        [<prefix>/lib]
   yamlcpp-lib-name=<str>       : YAMLcpp library name               ["yaml-cpp"]
   yamlcpp-static=0|1           : Use static YAMLcpp static library  [1]"""

lcms_build_opts = """LCMS2 OPTIONS
   with-lcms=<str>           : LCMS prefix                     [""]
   with-lcms-inc=<str>       : LCMS headers directory          [<prefix>/include]
   with-lcms-lib=<str>       : LCMS libraries directory        [<prefix>/lib]
   lcms-lib-name=<str>       : LCMS library name               ["lcms2"]
   lcms-static=0|1           : Use static LCMS static library  [1]"""

excons.AddHelpOptions(opencolorio=ocio_build_opts,
                      tinyxml=tinyxml_build_opts,
                      yamlcpp=yamlcpp_build_opts,
                      lcms=lcms_build_opts)

# excons.AddHelpTargets({"libs": "All libraries",
#                        "libs-static": "All static libraries",
#                        "libs-shared": "All shared libraries",
#                        "ilmbase": "All IlmBase libraries",
#                        "ilmbase-static": "All IlmBase static libraries",
#                        "ilmbase-shared": "All IlmBase shared librarues",
#                        "bins": "All command line tools",
#                        "python": "All python bindings",
#                        "tests": "All tests"})

targets = excons.DeclareTargets(env, projs)

env.Depends(targets["staticlib"], InstallHeaders)
env.Depends(targets["sharedlib"], InstallHeaders)

env.Alias("tools", targets["ociobakelut"])
env.Alias("tools", targets["ociocheck"])
