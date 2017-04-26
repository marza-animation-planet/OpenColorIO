import os
import re
import sys
import glob
import shutil
import tarfile
import excons
import subprocess
from excons.tools import boost
from excons.tools import python



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
ocio_use_boost = (excons.GetArgument("ocio-use-boost", 0, int) != 0)
ocio_version = (1, 0, 9)

ocio_config = {"OCIO_NAMESPACE"     : ocio_namespace,
               "OCIO_VERSION_MAJOR" : str(ocio_version[0]),
               "OCIO_VERSION_MINOR" : str(ocio_version[1]),
               "OCIO_VERSION_PATCH" : str(ocio_version[2]),
               "OCIO_VERSION"       : ".".join(map(str, ocio_version)),
               "SOVERSION"          : str(ocio_version[0]),
               "OCIO_USE_BOOST_PTR" : str(1 if ocio_use_boost else 0)}

tests_config = {"OCIO_TEST_AREA"           : binDir,
                "CMAKE_CURRENT_BINARY_DIR" : binDir}


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
else:
   cppflags += " /wd4101"
   cppflags += " /wd4996"
   cppflags += " /wd4251"
   defs.append("_CRT_SECURE_NO_WARNINGS")
if sys.platform == "darwin":
   # OSSpinLockUnlock used in Mutex.h is deprecated starting MacOS 10.12
   cppflags += " -Wno-deprecated-declarations"
libincdirs = ["export", "src/core", "ext/oiio/src/include"]
libcustoms = []
if ocio_sse2:
   defs.append("USE_SSE")
   if sys.platform != "win32":
      cflags += " -msse2"
   else:
      #cflags += " /arch:SSE2"
      pass
if ocio_hideinlines:
   if sys.platform != "win32":
      cflags += " -fvisibility-inlines-hidden"
if ocio_use_boost:
   libcustoms.append(boost.Require())



# TinyXML setup
def TinyXmlDefines(static):
   if excons.GetArgument("tinyxml-use-stl", 1, int) != 0:
      return ["TIXML_USE_STL"]
   else:
      return []

rv = excons.ExternalLibRequire("tinyxml", definesFunc=TinyXmlDefines)
if rv["require"] is None:
   excons.PrintOnce("OCIO: Build TinyXML from sources ...")
   excons.Call("TinyXML", imp=["RequireTinyXml"])
   def TinyXmlRequire(env):
      RequireTinyXml(env)
else:
   TinyXmlRequire = rv["require"]

libcustoms.append(TinyXmlRequire)

# YAML-cpp setup
def YamlCppDefaultName(static):
   return ("libyaml-cpp" if (sys.platform == "win32" and static) else "yaml-cpp")

def YamlCppDefines(static):
   if sys.platform == "win32":
      return ([] if static else ["YAML_CPP_DLL"])
   else:
      return []

def YamlCppEnv(env, static):
   boost.Require(env)

rv = excons.ExternalLibRequire("yamlcpp", libnameFunc=YamlCppDefaultName, definesFunc=YamlCppDefines, extraEnvFunc=YamlCppEnv)
if not rv["require"]:
   excons.PrintOnce("OCIO: Build YAML-CPP from sources ...")
   excons.Call("yaml-cpp", imp=["RequireYamlCpp"])
   def YamlCppRequire(env):
      RequireYamlCpp(env)
else:
   YamlCppRequire = rv["require"]

libcustoms.append(YamlCppRequire)

# LCMS setup (required only by ociobakelut)
build_lcms2 = (len(COMMAND_LINE_TARGETS) == 0)
for t in COMMAND_LINE_TARGETS:
   if t in ("ocio", "ocio-tools", "ociobakelut"):
      build_lcms2 = True

def Lcms2Defines(static):
   return (["CMS_DLL"] if not static else [])

rv = excons.ExternalLibRequire("lcms2", definesFunc=Lcms2Defines)
if not rv["require"]:
   if build_lcms2:
      excons.PrintOnce("OCIO: Build lcms2 from sources ...")
      excons.Call("Little-CMS", imp=["RequireLCMS2"])
      def Lcms2Require(env):
         RequireLCMS2(env)
   else:
      def Lcms2Require(env):
         pass
else:
   Lcms2Require = rv["require"]

# TODO
# - Nuke project


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
            m = e.search(line)
            if m is not None:
               opt = m.group(1)
               if opt in opts:
                  # LLVM backend build not yet supported
                  line = line.replace(m.group(0), opts[opt])
               else:
                  excons.WarnOnce("Unsupported config option '%s'" % opt)
            dst.write(line)
   shutil.copystat(str(source[0]), str(target[0]))
   return None

def GenerateConfig(target, source, env):
   return GenerateFile(target, source, env, ocio_config)

def GenerateTester(target, source, env):
   return GenerateFile(target, source, env, tests_config)

def RunTests(target, source, env):
   if sys.platform == "win32":
      subprocess.call(os.path.splitext(str(target[0]))[0] + ".bat")
   else:
      subprocess.call(str(target[0]) + ".sh")
   return None

def GeneratePyDoc(target, source, env):
   p = subprocess.Popen(["python", str(source[0]), str(target[0])])
   p.communicate()
   return None

env["BUILDERS"]["GenerateConfig"] = Builder(action=Action(GenerateConfig, "Generating $TARGET ...", suffix=".h", src_suffix=".h.in"))
env["BUILDERS"]["GenerateTester"] = Builder(action=Action(GenerateTester, "Generating $TARGET ...", suffix="", src_suffix=".in"))
env["BUILDERS"]["GeneratePyDoc"] = Builder(action=Action(GeneratePyDoc, "Generating $TARGET ..."))


# OpenColorABI.h is generated directly in the output, avoid possible conflicts
if os.path.isfile("export/OpenColorIO/OpenColorABI.h"):
   os.remove("export/OpenColorIO/OpenColorABI.h")

if CheckConfigStatus("lib_config.status", ocio_config):
   WriteConfigStatus("lib_config.status", ocio_config)

if CheckConfigStatus("test_config.status", tests_config):
   WriteConfigStatus("test_config.status", tests_config)


abih = env.GenerateConfig(incDir + "/OpenColorABI.h", ["export/OpenColorIO/OpenColorABI.h.in", "lib_config.status"])
if sys.platform != "win32":
   tester = env.GenerateTester(binDir + "/ocio_core_tests.sh", ["src/core_tests/ocio_core_tests.sh.in", "test_config.status"])
else:
   tester = env.GenerateTester(binDir + "/ocio_core_tests.bat", ["src/core_tests/ocio_core_tests.bat.in", "test_config.status"])


InstallHeaders = env.Install(incDir, excons.glob("export/OpenColorIO/*.h"))

pydoc = env.GeneratePyDoc("src/pyglue/PyDoc.h", "src/pyglue/createPyDocH.py")


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
      env.Append(CPPDEFINES=["OpenColorIO_STATIC"])
   if ocio_sse2:
      env.Append(CPPDEFINES=["USE_SSE"])
   env.Append(CPPPATH=[excons.OutputBaseDirectory() + "/include"])
   env.Append(LIBPATH=[excons.OutputBaseDirectory() + "/lib"])
   excons.Link(env, OCIOPath(static), static=static, force=True, silent=True)
   if static:
      RequireYamlCpp(env)
      RequireTinyXml(env)
   if ocio_use_boost:
      boost.Require()(env)



projs = [
   {  "name": OCIOName(True),
      "type": "staticlib",
      "alias": "ocio-static",
      "symvis": "default",
      "defs": defs + ["OpenColorIO_STATIC"],
      "cflags": cflags,
      "cppflags": cppflags,
      "incdirs": libincdirs,
      "srcs": excons.glob("src/core/*.cpp") +
              excons.glob("src/core/md5/*.cpp") +
              excons.glob("src/core/pystring/*.cpp"),
      "custom": libcustoms
   },
   {  "name": OCIOName(False),
      "type": "sharedlib",
      "alias": "ocio-shared",
      "version": ".".join(map(str, ocio_version)),
      "soname": "lib%s.so.%d" % (ocio_libname, ocio_version[0]),
      "install_name": "lib%s.%d.dylib" % (ocio_libname, ocio_version[0]),
      "defs": defs + ["OpenColorIO_EXPORTS"],
      "cflags": cflags,
      "cppflags": cppflags,
      "incdirs": libincdirs,
      "srcs": excons.glob("src/core/*.cpp") +
              excons.glob("src/core/md5/*.cpp") +
              excons.glob("src/core/pystring/*.cpp"),
      "custom": libcustoms
   },
   # Python binding
   {  "name": "PyOpenColorIO",
      "type": "dynamicmodule",
      "alias": "ocio-python",
      "ext": python.ModuleExtension(),
      "prefix": python.ModulePrefix() + "/" + python.Version(),
      "incdirs": ["export", "src/pyglue"],
      "defs": ["PYOCIO_NAME=PyOpenColorIO"],
      "cflags": cflags,
      "cppflags": cppflags,
      "srcs": excons.glob("src/pyglue/*.cpp"),
      "custom": [lambda x: RequireOCIO(x, static=True), python.SoftRequire],
   },
   # Command line tools
   {  "name": "ociobakelut",
      "type": "program",
      "defs": defs,
      "cflags": cflags,
      "cppflags": cppflags,
      "incdirs": ["src/apps/share"],
      "srcs": excons.glob("src/apps/ociobakelut/*.cpp") + ["src/apps/share/strutil.cpp", "src/apps/share/argparse.cpp"],
      "custom": [lambda x: RequireOCIO(x, static=True), Lcms2Require],
   },
   {  "name": "ociocheck",
      "type": "program",
      "defs": defs,
      "cflags": cflags,
      "cppflags": cppflags,
      "incdirs": ["src/apps/share"],
      "srcs": excons.glob("src/apps/ociocheck/*.cpp") + ["src/apps/share/strutil.cpp", "src/apps/share/argparse.cpp"],
      "custom": [lambda x: RequireOCIO(x, static=True)],
   },
   # Unit tests
   {  "name": "ocio_core_tests",
      "type": "program",
      "alias": "ocio-tests",
      "defs": defs + ["OCIO_UNIT_TEST", "OCIO_SOURCE_DIR=\"%s\"" % os.path.abspath(".").replace("\\", "/")],
      "cflags": cflags,
      "cppflags": cppflags,
      "srcs": excons.glob("src/core/*.cpp") +
              excons.glob("src/core/md5/*.cpp") +
              excons.glob("src/core/pystring/*.cpp"),
      "custom": [lambda x: RequireOCIO(x, static=True)],
      "post": [Action(RunTests, "Running Tests ...")]
   }
]

excons.AddHelpOptions(opencolorio="""OPENCOLORIO OPTIONS
  ocio-name                    : Library name                       ["OpenColorIO"]
  ocio-suffix=<str>            : Library suffix                     [""]
                                 (Ignored if ocio-name flag is set)
  ocio-namespace=<str>         : Library namespace                  ["OpenColorIO"]
  ocio-use-sse2=0|1            : Use SSE2 instructions              [1]
  ocio-use-boost=0|1           : Use boost shared_ptr               [0]
  ocio-hide-inlines=0|1        : Hide inline functions              [1]""")

excons.AddHelpTargets({"ocio-tools": "ociobakelut, ociocheck",
                       "ocio-static": "OCIO static library",
                       "ocio-shared": "OCIO shared library",
                       "ocio-libs": "OCIO static and shared libraries",
                       "ocio": "ocio-shared, ocio-static, ocio-python, ociobakelut, ociocheck"})
if OCIOName(True) == OCIOName(False):
   excons.AddHelpTargets({OCIOName(True): "OCIO static and shared libraries"})

targets = excons.DeclareTargets(env, projs)

env.Depends(targets["ocio-shared"], InstallHeaders)
env.Depends(targets["ocio-static"], InstallHeaders)
env.Alias("ocio-tests", tester)
env.Alias("ocio-libs", targets["ocio-shared"])
env.Alias("ocio-libs", targets["ocio-static"])
env.Alias("ocio", targets["ocio-shared"])
env.Alias("ocio", targets["ocio-static"])
env.Alias("ocio", targets["ocio-python"])
env.Alias("ocio", targets["ociobakelut"])
env.Alias("ocio", targets["ociocheck"])
env.Alias("ocio-tools", targets["ociobakelut"])
env.Alias("ocio-tools", targets["ociocheck"])

Export("OCIOName OCIOPath RequireOCIO")

