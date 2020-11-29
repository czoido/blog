---
title: "Adding dependencies to your Node.js native addons using Conan"
date: 2020-11-08T13:08:06+01:00
draft: false
tags: ["conan", "c++", "javascript", "node", "gyp"]
---

We have already covered how to write a simple C++ hello world native addon in a [previous post]({{< ref
"creating-c++-node-addons-using-napi" >}}). Now, we are going to add dependencies to that simple
example, manage them with [Conan](https://docs.conan.io/en/latest/) and run the code in Node.js.

If you are not familiar with Conan, please check the [getting
started](https://docs.conan.io/en/latest/getting_started.html) in the documentation.

As we explained in the previous post the [native addons](https://nodejs.org/api/addons.html) are
usually built generating a project file for your build-system using node-gyp. The input file for
[node-gyp](https://github.com/nodejs/node-gyp) is
[binding.py](https://github.com/nodejs/node-gyp#the-bindinggyp-file). This file contains the sources
that will be compiled as well as all the compilation options for the project. It will also contain
the information about which third-party libraries we want to link with and where are those libraries
located.

For example, if we want to add a dependency such as [yaml-cpp](https://conan.io/center/yaml-cpp)
library we should add a line like this to binding.gyp:

```json
"dependencies": ["</path/to/library-info.gyp:yaml-cpp"],
```

If you are familiar with CMake, this is something similar to a call to `target_link_libraries`.

#### Using a custom Conan generator to manage the dependencies paths

The dependencies field is a list of gyp files, each one of those containing information such as the
headers and libraries folder location, compilation flags and so on. As this is exactly the
information that a [Conan generator](https://docs.conan.io/en/latest/reference/generators.html) is
designed to provide, we will create a custom Conan generator to generate all the gyp files that will
contain the information to link the third party libraries in our native C++ addon. Explaining how a
custom Conan generator is created is out of the scope of this post, but you can read about it [in
Conan docs](https://docs.conan.io/en/latest/reference/generators/custom.html#custom-generator) and
[this
blogpost](https://blog.conan.io/2019/07/24/C++-build-systems-new-integrations-in-Conan-package-manager.html).
We will use [this minimal generator](https://github.com/czoido/conan-gyp-generator) for node-gyp.

To use it, install the generator:

```bash
git clone https://github.com/czoido/conan-gyp-generator
cd conan-gyp-generator
conan config install gyp-generator.py -tf generators
cd ..
```

#### Building the native addon

Now that the custom generator is installed You can clone the sources for this example from [this
GitHub repo](https://github.com/czoido/conan-node-module).

```bash
git clone https://github.com/czoido/conan-node-module
cd conan-node-module
```

As you can see there's already a *binding.gyp* file with this contents:

```bash
{
    "targets": [{
        "target_name": "conan_node_module",
        "sources": ["main.cpp"],
        "dependencies": ["<(module_root_dir)/conan_build/conanbuildinfo.gyp:yaml-cpp"],
        "conditions": [[
            "OS=='mac'", {
                "xcode_settings": {
                    "GCC_ENABLE_CPP_EXCEPTIONS": "YES"
                }
            }
        ]]
    }]
}
```

There are several things to point out in this file:

- `<(module_root_dir)` will be expanded to the folder where we cloned the repository when invoking
  node-gyp.
- *conanbuildinfo.gyp:yaml-cpp* is the output file of the Conan generator and *:yaml-cpp* is the way
  of telling node-gyp to add the information under `"target_name": "yaml-cpp",` from
  *conanbuildinfo.gyp*.
- Also, note that you can add some optional parameters conditioned to things like the operating
  system. Here, our project file enables cpp exceptions using `GCC_ENABLE_CPP_EXCEPTIONS`.

You can also have a look at *main.cpp* containing the sources that use the yaml-cpp library. This file
defines a function that will take a list of numbers as an argument and return its size using yaml-cpp
library.

```cpp
#include <string>
#include <iostream>

#include "yaml-cpp/yaml.h"
#include <node.h>

namespace demo
{
  using v8::FunctionCallbackInfo;
  using v8::Isolate;
  using v8::Local;
  using v8::Object;
  using v8::String;
  using v8::Value;

  void ParseYAML(const FunctionCallbackInfo<Value> &args)
  {
    Isolate *isolate = args.GetIsolate();
    String::Utf8Value str(isolate, args[0]);
    YAML::Node numbers_list = YAML::Load(std::string(*str));
    args.GetReturnValue().Set(String::NewFromUtf8(isolate, std::to_string(numbers_list.size()).c_str()).ToLocalChecked());
  }

  void Init(Local<Object> exports)
  {
    NODE_SET_METHOD(exports, "parse_yaml", ParseYAML);
  }

  NODE_MODULE(NODE_GYP_MODULE_NAME, Init)

}
```

Also, have a look at the *conanfile.txt* in the main folder, containing the information of the
dependencies, generators and options we want to use for this project:

```ini
[requires]
yaml-cpp/0.6.3

[generators]
node_gyp
virtualrunenv

[options]
yaml-cpp:shared=True
```

Note that we have under the generators section `node_gyp` and also `virtualrunenv`. The first will
create the *conanbuildinfo.gyp* with the location of the libraries and the second will help us define
some environment variables so that our application can find the shared libraries later.

Then, use the Conan generator to create the files with information about the dependencies of the project:

```bash
mkdir conan_build && cd conan_build
conan install .. --build=missing && cd ..
```

This will generate the *conanbuildinfo.gyp* file:

```json
{
    "targets": [{
        "target_name": "yaml-cpp",
        "type": "<(library)",
        "direct_dependent_settings": {
            "include_dirs": [
                "/Users/carlos/.conan/data/yaml-cpp/0.6.3/_/_/package/ba3872b419f0b5f5948b43bc6817447b031fa997/include",                    
            ],
            "libraries": [
                "-lyaml-cpp", 
                "-L/Users/carlos/.conan/data/yaml-cpp/0.6.3/_/_/package/ba3872b419f0b5f5948b43bc6817447b031fa997/lib",
                "-Wl,-rpath,<(output_dir)/build/Release/"
            ]
        }
    }]
}
```

Now we can compile our native C++ addon. Calling npm install should do a call to `node-gyp rebuild`.
You can also compile the addon calling direcly to node-gyp: ``node-gyp configure && node-gyp build´

```bash
npm install
source conan_build/activate_run.sh # activate virtualrunenv to set DYLD_LIBRARY_PATH so that it finds dependencies .so
```

The output of the build should be the conan_node_module.node file that we can import in our node
JavaScript applications. Check the code of index.js, a simple application that will use our native addon to count the number of elements of a list:

```javascript
var my_module = require("bindings")("conan_node_module");
console.log(my_module.parse_yaml("[1,3,4]"))
```

Run the application and check the output:

```bash
node index.js
```