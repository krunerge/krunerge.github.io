---
title: Ceph coverage
date: 2021-11-25 10:11:11
tags: ceph coverage
---

# 1.目标
最近想对Ceph代码进行代码覆盖了查看，也就是coverage，发现网上资料不多，这里对步骤进行记录，这里介绍一下相关知识，如果要对c，c++程序进程code coverage，则需要在编译的时候增加-ftest-coverage -fprofile-arcs两个参数，同时在连接的时候增加链接库gcov，这样会在生成的二进制中生成一些记录代码覆盖率的信息。

# 2.启动coverage
## 2.1编译开启支持
首先查看代码src/CMakeLists.txt
```
if(${ENABLE_COVERAGE})
  find_program(HAVE_GCOV gcov)
  if(NOT HAVE_GCOV)
    message(FATAL_ERROR "Coverage Enabled but gcov Not Found")
  endif(NOT HAVE_GCOV)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fprofile-arcs -ftest-coverage -O0")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fprofile-arcs -ftest-coverage")
  list(APPEND EXTRALIBS gcov)
endif(${ENABLE_COVERAGE})
```
可以看到在ENABLE_COVERAGE情况下，编译增加了-fprofile-arcs -ftest-coverage，同时EXTRALIBS增加了gcov，好记下来我们开启coverage编译，
```shell
./do_cmake.sh -DENABLE_COVERAGE=ON
```

## 2.2 部署环境
编译完之后可以使用vstart.sh快速部署一个ceph集群，这个时候会遇到问题
```
Populating config ...
Traceback (most recent call last):
  File "/data/ceph/grov/ceph/build/bin/ceph", line 140, in <module>
    import rados
ImportError: /data/ceph/build/lib/cython_modules/lib.2/rados.so: undefined symbol: __gcov_merge_add
```
ceph命令脚本相关的rados.so这个python c库出现符号表找不到，这是因为在编译链接的时候没有加入lgcov，虽然在CMakelists中gcov加入了EXTRALIBS，但是rados.so这个比较复杂，
有兴趣的朋友可以研究一下rados.so库的编译规则，ceph自定义了一个cmake函数，在cmake/modules/Distutils.cmake文件中
```
function(distutils_add_cython_module name src)
  get_property(compiler_launcher GLOBAL PROPERTY RULE_LAUNCH_COMPILE)
  get_property(link_launcher GLOBAL PROPERTY RULE_LAUNCH_LINK)
  # When using ccache, CMAKE_C_COMPILER is ccache executable absolute path
  # and the actual C compiler is CMAKE_C_COMPILER_ARG1.
  # However with a naive
  # set(PY_CC ${compiler_launcher} ${CMAKE_C_COMPILER} ${CMAKE_C_COMPILER_ARG1})
  # distutils tries to execve something like "/usr/bin/cmake gcc" and fails.
  # Removing the leading whitespace from CMAKE_C_COMPILER_ARG1 helps to avoid
  # the failure.
  string(STRIP "${CMAKE_C_COMPILER_ARG1}" c_compiler_arg1)
  string(STRIP "${CMAKE_CXX_COMPILER_ARG1}" cxx_compiler_arg1)
  # Note: no quotes, otherwise distutils will execute "/usr/bin/ccache gcc"
  # CMake's implicit conversion between strings and lists is wonderful, isn't it?
  string(REPLACE " " ";" cflags ${CMAKE_C_FLAGS})
  list(APPEND cflags -iquote${CMAKE_SOURCE_DIR}/src/include -w)
  # This little bit of magic wipes out __Pyx_check_single_interpreter()
  # Note: this is reproduced in distutils_install_cython_module
  list(APPEND cflags -D'void0=dead_function\(void\)')
  list(APPEND cflags -D'__Pyx_check_single_interpreter\(ARG\)=ARG \#\# 0')
  set(PY_CC ${compiler_launcher} ${CMAKE_C_COMPILER} ${c_compiler_arg1} ${cflags})
  set(PY_CXX ${compiler_launcher} ${CMAKE_CXX_COMPILER} ${cxx_compiler_arg1})
  set(PY_LDSHARED ${link_launcher} ${CMAKE_C_COMPILER} ${c_compiler_arg1} "-shared")
  add_custom_target(${name} ALL
    COMMAND
    env
    CC="${PY_CC}"
    CXX="${PY_CXX}"
    LDSHARED="${PY_LDSHARED}" 
    OPT=\"-DNDEBUG -g -fwrapv -O2 -w\"
    LDFLAGS=-L${CMAKE_LIBRARY_OUTPUT_DIRECTORY}
    CYTHON_BUILD_DIR=${CMAKE_CURRENT_BINARY_DIR}
    CEPH_LIBDIR=${CMAKE_LIBRARY_OUTPUT_DIRECTORY}
    ${PYTHON${PYTHON_VERSION}_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/setup.py
    build --verbose --build-base ${CYTHON_MODULE_DIR}
    --build-platlib ${CYTHON_MODULE_DIR}/lib.${PYTHON${PYTHON_VERSION}_VERSION_MAJOR}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    DEPENDS ${src})
endfunction(distutils_add_cython_module)
```
add_custom_target定义了一条编译命令，最后是调用rados目录下面的setup.py进行build，类似如下命令
```
python rados/setup.py build --verbose --build-base xxx
```
这样会编译出rados.so，同时我们看到这里用env定义了环境变量，那为什么没有把gcov在链接的时候加入呢，这个要看一下setup.py这个python代码
```
def get_python_flags():
    ...

    for ldflag in filter_unsupported_flags(subprocess.check_output(
            [python_config, "--ldflags"]).strip().decode('utf-8').split()):
        if ldflag.startswith('-l'):
            ldflags['l'].append(ldflag.replace('-l', ''))
        if ldflag.startswith('-L'):
            ldflags['L'].append(ldflag.replace('-L', ''))
        else:
            ldflags['extras'].append(ldflag)
			
    ...

    return {
        'cflags': cflags,
        'ldflags': ldflags
    }
```
这里可以看到ldflags就是获取的链接时的库，他是从Python2.7-config --ldflags里面获取的，没有加入gcov，嗯，是个小问题，改一下

## 2.3 链接增加gcov
- 在src/CMakeLists.txt文件
```
if(${ENABLE_COVERAGE})
  find_program(HAVE_GCOV gcov)
  if(NOT HAVE_GCOV)
    message(FATAL_ERROR "Coverage Enabled but gcov Not Found")
  endif(NOT HAVE_GCOV)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fprofile-arcs -ftest-coverage -O0")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fprofile-arcs -ftest-coverage")
  list(APPEND EXTRALIBS gcov)
+ set(PY_LIBS gcov)
endif(${ENABLE_COVERAGE})
```

- 在cmake/modules/Distutils.cmake文件
```
function(distutils_add_cython_module name src)
  ...
  add_custom_target(${name} ALL
    COMMAND
    env
    CC="${PY_CC}"
    CXX="${PY_CXX}"
    LDSHARED="${PY_LDSHARED}" 
+   LIBS="${PY_LIBS}"
    OPT=\"-DNDEBUG -g -fwrapv -O2 -w\"
    LDFLAGS=-L${CMAKE_LIBRARY_OUTPUT_DIRECTORY}
    CYTHON_BUILD_DIR=${CMAKE_CURRENT_BINARY_DIR}
    CEPH_LIBDIR=${CMAKE_LIBRARY_OUTPUT_DIRECTORY}
    ${PYTHON${PYTHON_VERSION}_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/setup.py
    build --verbose --build-base ${CYTHON_MODULE_DIR}
    --build-platlib ${CYTHON_MODULE_DIR}/lib.${PYTHON${PYTHON_VERSION}_VERSION_MAJOR}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    DEPENDS ${src})
endfunction(distutils_add_cython_module)
```

- 在src/pybind/rados/setup.py文件
```
def get_python_flags():
    ...

    for ldflag in filter_unsupported_flags(subprocess.check_output(
            [python_config, "--ldflags"]).strip().decode('utf-8').split()):
        if ldflag.startswith('-l'):
            ldflags['l'].append(ldflag.replace('-l', ''))
        if ldflag.startswith('-L'):
            ldflags['L'].append(ldflag.replace('-L', ''))
        else:
            ldflags['extras'].append(ldflag)

+   if os.environ.get('LIBS', None):
+      for lib in os.environ.get('LIBS').split(" "):
+          ldflags['l'].append(lib)

    return {
        'cflags': cflags,
        'ldflags': ldflags
    }
```
这样就能编译通过，同时，pybind下面rbd，rgw，cephfs的setup.py也做同样修改

# 3.收集coverage信息
## 3.1 跑集群
在ceph集群搭建起来之后，可以创建一个pool，创建rbd卷，用rbd bench或者rados bench简单的跑一个测试
```
rbd  bench  test/test --io-type write --io-pattern rand   --io-total 1G 
```

## 3.2 收集所有信息
```
lcov --base-directory src --directory src/ --capture --output-file ceph.info
```
src是包含有gcda文件的大目录
这里收集到的信息会包含有ceph调用其他第三方库的统计信息，比如boost，这些需要去掉要不就不是ceph的code coverage

## 3.3 过滤
```
lcov  --remove  ceph.info '/opt/*' '/usr/*' '*/build/*'  '*/rocksdb/*' -o result.info
```
通过如上命令过滤出ceph的coverage

## 3.3 生成html
```
genhtml result.info -o /data/ceph_result
```
生成html可以在页面展示，这里可以使用nginx把/data/ceph_result作为root目录进行展示

## 3.4 nginx配置
```
... 
server {
        listen       81;
        listen       [::]:81;
        server_name  _;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        error_page 404 /404.html;
        location = /404.html {
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }

        location / {
        autoindex on;
        root /data/ceph_result;
        }
...		
```

# 4. 网页展示
