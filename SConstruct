import os
import re
import sys
import glob
import shutil
import tarfile
import excons
import subprocess
import tarfile
from excons.tools import boost
from excons.tools import python
from excons.tools import gl
from excons.tools import glut
import SCons.Script # pylint: disable=import-error


excons.InitGlobals()

if sys.platform.startswith("linux"):
   dts = excons.GetArgument("devtoolset", "")
   if not dts or int(dts) < 6:
      print("GCC 6 or above required")
      sys.exit(0)
else:
   # on other platform, require c++11 standard
   SCons.Script.ARGUMENTS["use-c++11"] = "1"

env = excons.MakeBaseEnv()

incDir = "%s/include/OpenColorIO" % excons.OutputBaseDirectory().replace("\\", "/")
binDir = "%s/bin" % excons.OutputBaseDirectory().replace("\\", "/")

# OpenColorIO config
ocio_libname = excons.GetArgument("ocio-name", None)
if ocio_libname is None:
   ocio_libname = "OpenColorIO"
   ocio_libsuffix = excons.GetArgument("ocio-suffix", None)
   if ocio_libsuffix:
      ocio_libname += ocio_libsuffix
ocio_sse2 = (excons.GetArgument("ocio-use-sse2", 1, int) != 0)
ocio_hideinlines = (excons.GetArgument("ocio-hide-inlines", 1, int) != 0)
ocio_namespace = excons.GetArgument("ocio-namespace", "OpenColorIO")
ocio_extra_builtins = (excons.GetArgument("ocio-extra-builtints", 1, int) != 0)
ocio_version = (2, 0, 1)

ocio_config = {"OCIO_NAMESPACE"                   : ocio_namespace,
               "OpenColorIO_VERSION_MAJOR"        : str(ocio_version[0]),
               "OpenColorIO_VERSION_MINOR"        : str(ocio_version[1]),
               "OpenColorIO_VERSION_PATCH"        : str(ocio_version[2]),
               "OpenColorIO_VERSION"              : ".".join(map(str, ocio_version)),
               "OpenColorIO_VERSION_RELEASE_TYPE" : "",
               "SOVERSION"                        : "%s.%s" % (ocio_version[0], ocio_version[1])}


defs = []
cflags = ""
cppflags = ""

if sys.platform != "win32":
   cppflags += " -Wno-unused-parameter"
   cppflags += " -Wno-unused-variable"
   cppflags += " -Wno-missing-field-initializers"
   if sys.platform != "darwin":
      cppflags += " -Wno-unused-but-set-variable"
   else:
      cppflags += " -Wno-unused-private-field"
      cppflags += " -Wno-unused-function"
      cppflags += " -Wno-deprecated-register"
else:
   cppflags += " /wd4101"
   cppflags += " /wd4996"
   cppflags += " /wd4251"
   defs.append("_CRT_SECURE_NO_WARNINGS")
if sys.platform == "darwin":
   # OSSpinLockUnlock used in Mutex.h is deprecated starting MacOS 10.12
   cppflags += " -Wno-deprecated-declarations"
libincdirs = ["include", "ext/sampleicc/src/include", "src", "src/OpenColorIO", "."]
libcustoms = []
libdefs = defs[:]
if ocio_sse2:
   libdefs.append("USE_SSE")
   if sys.platform != "win32":
      cflags += " -msse2"
if ocio_extra_builtins:
   libdefs.append("ADD_EXTRA_BUILTINS")
if ocio_hideinlines and sys.platform != "win32":
   cflags += " -fvisibility-inlines-hidden"

# zlib setup
zlibspecs = {}

def ZlibName(static):
    return ("z" if sys.platform != "win32" else ("zlib" if static else "zdll"))

def ZlibDefines(static):
    return ([] if (static or sys.platform != "win32") else ["ZLIB_DLL"])

rv = excons.ExternalLibRequire("zlib", libnameFunc=ZlibName, definesFunc=ZlibDefines)
if not rv["require"]:
    excons.PrintOnce("OCIO: Build zlib from sources ...")
    excons.Call("zlib", targets=["zlib"], imp=["ZlibName", "ZlibPath", "RequireZlib"])
    def ZlibRequire(env):
        return RequireZlib(env, static=True) # pylint: disable=undefined-variable
    zlibspecs = {"with-zlib": os.path.dirname(os.path.dirname(ZlibPath(static=True))), # pylint: disable=undefined-variable
                 "zlib-name": ZlibName(static=True),
                 "zlib-static": 1}
else:
    ZlibRequire = rv["require"]

# Expat setup
def ExpatDefaultName(static):
   return ("libexpat" if (sys.platform == "win32" and static) else "expat")

def ExpatCppDefines(static):
   if static:
      return ["XML_STATIC"]
   else:
      return []

rv = excons.ExternalLibRequire("expat", libnameFunc=ExpatDefaultName, definesFunc=ExpatCppDefines)
if not rv["require"]:
   excons.PrintOnce("OCIO: Build expat from sources ...")
   excons.Call("libexpat", targets=["expat"], imp=["RequireExpat"])
   def ExpatRequire(env):
      RequireExpat(env, static=True) # pylint: disable=undefined-variable
else:
   ExpatRequire = rv["require"]

libcustoms.append(ExpatRequire)

# Half setup
def HalfDefaultName(static):
   return ("libHalf" if (sys.platform == "win32" and static) else "Half")

def HalfCppDefines(static):
   if not static and sys.platform == "win32":
      return ["OPENEXR_DLL"]
   else:
      return []

rv = excons.ExternalLibRequire("half", libnameFunc=HalfDefaultName, definesFunc=HalfCppDefines)
if not rv["require"]:
   excons.PrintOnce("OCIO: Build Half from sources ...")
   excons.Call("openexr", targets=["Half-static"], overrides=zlibspecs, imp=["RequireHalf"])
   def HalfRequire(env):
      RequireHalf(env, static=True) # pylint: disable=undefined-variable
else:
   HalfRequire = rv["require"]

libcustoms.append(HalfRequire)

# YAML-cpp setup
def YamlCppDefaultName(static):
   return ("libyaml-cpp" if (sys.platform == "win32" and static) else "yaml-cpp")

def YamlCppDefines(static):
   if sys.platform == "win32":
      return ([] if static else ["YAML_CPP_DLL"])
   else:
      return []

def YamlCppEnv(env, static):
   boost.Require()(env)

rv = excons.ExternalLibRequire("yamlcpp", libnameFunc=YamlCppDefaultName, definesFunc=YamlCppDefines, extraEnvFunc=YamlCppEnv)
if not rv["require"]:
   excons.PrintOnce("OCIO: Build YAML-CPP from sources ...")
   excons.Call("yaml-cpp", targets=["yamlcpp"], imp=["RequireYamlCpp"])
   def YamlCppRequire(env):
      RequireYamlCpp(env) # pylint: disable=undefined-variable
else:
   YamlCppRequire = rv["require"]

libcustoms.append(YamlCppRequire)

# LCMS setup (required only by ociobakelut)
def Lcms2Defines(static):
   return (["CMS_DLL"] if not static else [])

rv = excons.ExternalLibRequire("lcms2", definesFunc=Lcms2Defines)
if not rv["require"]:
   excons.PrintOnce("OCIO: Build lcms2 from sources ...")
   excons.Call("Little-CMS", targets=["lcms2"], overrides=zlibspecs, imp=["RequireLCMS2"])
   def Lcms2Require(env):
      RequireLCMS2(env) # pylint: disable=undefined-variable
else:
   Lcms2Require = rv["require"]

# GLEW setup
def GlewName(static):
   suffix = "_s" if static else ""
   return ("libglew" if (sys.platform == "win32" and static) else "glew") + suffix

def GlewCppDefines(static):
   if static:
      return ["GLEW_STATIC"]
   else:
      return []

rv = excons.ExternalLibRequire("glew", libnameFunc=GlewName, definesFunc=GlewCppDefines)
if not rv["require"]:
   excons.PrintOnce("OCIO: Build GLEW from sources ...")
   excons.Call("glew", targets=["GLEW"], imp=["RequireGlew"])
   def GlewRequire(env):
      RequireGlew(env, static=True) # pylint: disable=undefined-variable
else:
   GlewRequire = rv["require"]


# TODO
# - Nuke project


# pybind
if not os.path.isfile("pybind11/include/pybind11/pybind11.h"):
   cmd = "git submodule update --init pybind11"
   p = subprocess.Popen(cmd, shell=True)
   p.communicate()

# pystring
if not os.path.isfile("pystring/pystring.h"):
   cmd = "git submodule update --init pystring"
   p = subprocess.Popen(cmd, shell=True)
   p.communicate()



def CheckConfigStatus(path, opts):
   if not os.path.isfile(path):
      return True
   else:
      with open(path, "r") as f:
         for line in f.readlines():
            spl = line.strip().split(" ")
            if spl[0] in opts:
               if str(opts[spl[0]]) != " ".join(spl[1:]):
                  return True
      return False

def WriteConfigStatus(path, opts):
   with open(path, "w") as f:
      for k, v in opts.iteritems():
         f.write("%s %s\n" % (k, v))
      f.write("\n")

def GenerateFile(target, source, env, opts):
   with open(str(source[0]), "r") as src:
      with open(str(target[0]), "w") as dst:
         e = re.compile("@([^@]+)@")
         for line in src.readlines():
            remain = line
            line = ""
            m = e.search(remain)
            while m is not None:
               opt = m.group(1)
               if opt in opts:
                  line += remain[:m.start(0)] + str(opts[opt])
               else:
                  excons.WarnOnce("Unsupported config option '%s'" % opt)
                  line += remain[:m.end(0)]
               remain = remain[m.end(0):]
               m = e.search(remain)
            line += remain
            dst.write(line)
   shutil.copystat(str(source[0]), str(target[0]))
   return None

def GenerateConfig(target, source, env):
   return GenerateFile(target, source, env, ocio_config)


env["BUILDERS"]["GenerateConfig"] = SCons.Script.Builder(action=SCons.Script.Action(GenerateConfig, "Generating $TARGET ...", suffix=".h", src_suffix=".h.in"))


# OpenColorABI.h is generated directly in the output, avoid possible conflicts
if os.path.isfile("include/OpenColorIO/OpenColorABI.h"):
   os.remove("include/OpenColorIO/OpenColorABI.h")

if CheckConfigStatus("lib_config.status", ocio_config):
   WriteConfigStatus("lib_config.status", ocio_config)


abih = env.GenerateConfig(incDir + "/OpenColorABI.h", ["include/OpenColorIO/OpenColorABI.h.in", "lib_config.status"])

InstallHeaders = env.Install(incDir, excons.glob("include/OpenColorIO/*.h"))


def OCIOName(static=True):
   if sys.platform == "win32" and static:
      return "lib" + ocio_libname
   else:
      return ocio_libname

def OCIOPath(static=True):
   name = OCIOName(static)
   if sys.platform == "win32":
      libname = name + ".lib"
   else:
      libname = "lib" + name + (".a" if static else excons.SharedLibraryLinkExt())
   return excons.OutputBaseDirectory() + "/lib/" + libname

def RequireOCIO(env, static=True):
   if static:
      env.Append(CPPDEFINES=["OpenColorIO_SKIP_IMPORTS"])
   env.Append(CPPPATH=[excons.OutputBaseDirectory() + "/include"])
   env.Append(LIBPATH=[excons.OutputBaseDirectory() + "/lib"])
   excons.Link(env, OCIOPath(static), static=static, force=True, silent=True)
   if sys.platform == "darwin":
      env.Append(LINKFLAGS=" -framework Carbon -framework IOKit")
   elif sys.platform == "win32":
      env.Append(LIBS=["user32", "gdi32"])
   if static:
      YamlCppRequire(env)
      ExpatRequire(env)
      HalfRequire(env)

BASEsrc = filter(lambda x: not os.path.basename(x).startswith("SystemMonitor_"), excons.glob("src/OpenColorIO/*.cpp"))

BUILTINSsrc = excons.glob("src/OpenColorIO/transforms/builtins/*.cpp")
if not ocio_extra_builtins:
   BUILTINSsrc = filter(lambda x: not os.path.basename(x).endswith("Cameras.cpp") and os.path.basename(x) != "Displays.cpp", BUILTINSsrc)

PYSTRINGsrc = ["pystring/pystring.cpp"]

projs = [
   {  "name": OCIOName(True),
      "type": "staticlib",
      "alias": "ocio-static",
      "defs": libdefs + ["OpenColorIO_SKIP_IMPORTS"],
      "cflags": cflags,
      "cppflags": cppflags,
      "incdirs": libincdirs,
      "srcs": BASEsrc + BUILTINSsrc + PYSTRINGsrc +
              excons.glob("src/OpenColorIO/apphelpers/*.cpp") +
              excons.CollectFiles("src/OpenColorIO/fileformats", patterns=["*.cpp"], recursive=True) +
              excons.glob("src/OpenColorIO/md5/*.cpp") +
              excons.CollectFiles("src/OpenColorIO/ops", patterns=["*.cpp"], recursive=True) +
              excons.glob("src/OpenColorIO/transforms/*.cpp"),
      "custom": libcustoms
   },
   {  "name": OCIOName(False),
      "type": "sharedlib",
      "alias": "ocio-shared",
      "symvis": "hidden",
      "version": ".".join(map(str, ocio_version)),
      "soname": "lib%s.so.%d" % (ocio_libname, ocio_version[0]),
      "install_name": "lib%s.%d.dylib" % (ocio_libname, ocio_version[0]),
      "defs": libdefs + ["OpenColorIO_EXPORTS"],
      "cflags": cflags,
      "cppflags": cppflags,
      "linkflags": ("" if sys.platform != "darwin" else "-framework Carbon -framework IOKit"),
      "incdirs": libincdirs,
      "srcs": BASEsrc + BUILTINSsrc + PYSTRINGsrc +
              excons.glob("src/OpenColorIO/apphelpers/*.cpp") +
              excons.CollectFiles("src/OpenColorIO/fileformats", patterns=["*.cpp"], recursive=True) +
              excons.glob("src/OpenColorIO/md5/*.cpp") +
              excons.CollectFiles("src/OpenColorIO/ops", patterns=["*.cpp"], recursive=True) +
              excons.glob("src/OpenColorIO/transforms/*.cpp"),
      "custom": libcustoms
   },
   # Python binding
   {  "name": "PyOpenColorIO",
      "type": "dynamicmodule",
      "alias": "ocio-python",
      "ext": python.ModuleExtension(),
      "vismap": ("src/bindings/python/PyOpenColorIO%s.map" % ("_osx" if sys.platform == "darwin" else "") if sys.platform != "win32" else None),
      "prefix": python.ModulePrefix() + "/" + python.Version(),
      "incdirs": ["src/bindings/python", "pybind11/include", "src", "share/docs"], # empty docstrings
      "defs": ["PYOCIO_NAME=PyOpenColorIO"],
      "cflags": cflags,
      "cppflags": cppflags,
      "srcs": excons.CollectFiles("src/bindings/python", patterns=["*.cpp"]),
      "custom": [lambda x: RequireOCIO(x, static=True), python.SoftRequire],
   },
   # Command line tools
   {  "name": "ociobakelut",
      "type": "program",
      "defs": defs,
      "cflags": cflags,
      "cppflags": cppflags,
      "incdirs": ["src"],
      "srcs": excons.glob("src/apps/ociobakelut/*.cpp") +
              ["src/apputils/strutil.cpp", "src/apputils/argparse.cpp"],
      "custom": [lambda x: RequireOCIO(x, static=True), Lcms2Require],
   },
   {  "name": "ociocheck",
      "type": "program",
      "defs": defs,
      "cflags": cflags,
      "cppflags": cppflags,
      "incdirs": ["src"],
      "srcs": excons.glob("src/apps/ociocheck/*.cpp") +
              ["src/apputils/strutil.cpp", "src/apputils/argparse.cpp"],
      "custom": [lambda x: RequireOCIO(x, static=True)],
   },
   {  "name": "ociochecklut",
      "type": "program",
      "defs": defs + ["OCIO_GL_ENABLED", "OCIO_GPU_ENABLED"],
      "cflags": cflags,
      "cppflags": cppflags,
      "incdirs": ["src", "src/libutils/oglapphelpers"],
      "srcs": excons.glob("src/apps/ociochecklut/*.cpp") +
              excons.glob("src/libutils/oglapphelpers/*.cpp") +
              ["src/apputils/strutil.cpp", "src/apputils/argparse.cpp"],
      "custom": [lambda x: RequireOCIO(x, static=True),
                 glut.Require, GlewRequire, gl.Require],
   },
   {  "name": "ociomakeclf",
      "type": "program",
      "defs": defs,
      "cflags": cflags,
      "cppflags": cppflags,
      "incdirs": ["src"],
      "srcs": excons.glob("src/apps/ociomakeclf/*.cpp") +
              ["src/apputils/strutil.cpp", "src/apputils/argparse.cpp"],
      "custom": [lambda x: RequireOCIO(x, static=True)],
   },
   {  "name": "ociowrite",
      "type": "program",
      "defs": defs,
      "cflags": cflags,
      "cppflags": cppflags,
      "incdirs": ["src"],
      "srcs": excons.glob("src/apps/ociowrite/*.cpp") +
              ["src/apputils/strutil.cpp", "src/apputils/argparse.cpp"],
      "custom": [lambda x: RequireOCIO(x, static=True)],
   }
]

excons.AddHelpOptions(opencolorio="""OPENCOLORIO OPTIONS
  ocio-name                       : Library name                        ["OpenColorIO"]
  ocio-suffix=<str>               : Library suffix                      [""]
                                    (Ignored if ocio-name flag is set)
  ocio-namespace=<str>            : Library namespace                   ["OpenColorIO"]
  ocio-use-sse2=0|1               : Use SSE2 instructions               [1]
  ocio-hide-inlines=0|1           : Hide inline functions               [1]
  ocio-extra-builtins=0|1         : Extra builtins transforms           [1]""")

excons.AddHelpTargets({"ocio-tools": "ociobakelut, ociocheck",
                       "ocio-static": "OCIO static library",
                       "ocio-shared": "OCIO shared library",
                       "ocio-libs": "OCIO static and shared libraries",
                       "ocio": "ocio-shared, ocio-static, ocio-python, ociobakelut, ociocheck, ociochecklut, ociomakeclt, ociowrite"})
if OCIOName(True) == OCIOName(False):
   excons.AddHelpTargets({OCIOName(True): "OCIO static and shared libraries"})

targets = excons.DeclareTargets(env, projs)

env.Depends(targets["ocio-shared"], InstallHeaders)
env.Depends(targets["ocio-static"], InstallHeaders)
env.Alias("ocio-libs", targets["ocio-shared"])
env.Alias("ocio-libs", targets["ocio-static"])
env.Alias("ocio", targets["ocio-shared"])
env.Alias("ocio", targets["ocio-static"])
env.Alias("ocio", targets["ocio-python"])
env.Alias("ocio", targets["ociobakelut"])
env.Alias("ocio", targets["ociocheck"])
env.Alias("ocio", targets["ociochecklut"])
env.Alias("ocio", targets["ociomakeclf"])
env.Alias("ocio", targets["ociowrite"])
env.Alias("ocio-tools", targets["ociobakelut"])
env.Alias("ocio-tools", targets["ociocheck"])
env.Alias("ocio-tools", targets["ociochecklut"])
env.Alias("ocio-tools", targets["ociomakeclf"])
env.Alias("ocio-tools", targets["ociowrite"])

SCons.Script.Export("OCIOName OCIOPath RequireOCIO")

