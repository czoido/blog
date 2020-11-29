---
title: "Creating C++ Node Addons using N-API"
date: 2020-11-06T12:31:18+01:00
draft: false
tags: ["c++", "javascript", "node", "gyp"]
---

With Node.js addons we can load our C++ libraries as if they were ordinary Node.js modules. The
recommended way to implement these addons is using the [N-API](https://nodejs.org/api/n-api.html).
The N-API is delivered as a part of the Node.js distribution. since v8 and became stable in v10. 

One "Hello World" addon could look something like this:

```cpp
// hello.cpp
#include <node.h>

namespace demo {

    using v8::FunctionCallbackInfo;
    using v8::Isolate;
    using v8::Local;
    using v8::Object;
    using v8::String;
    using v8::Value;

    void HelloWorld(const FunctionCallbackInfo<Value>& args) {
        Isolate* isolate = args.GetIsolate();
        args.GetReturnValue().Set(String::NewFromUtf8(isolate, "hello world").ToLocalChecked());
    }

    void Initialize(Local<Object> exports) {
        NODE_SET_METHOD(exports, "hello_world", HelloWorld);
    }

    NODE_MODULE(NODE_GYP_MODULE_NAME, Initialize)

}  // namespace demo
```

After building this code, we will get a *hello_world.node* file that is a dynamic library that wraps
our `HelloWorld` function. Using `require()` in our JavaScript code we will import this C++ function
to our Node.js application like this:

```javascript
require('./path/to/hello_world.node');
```

Instead of loading this module directly we will use a helper module called
[node-bindings](https://www.npmjs.com/package/bindings). This module checks all the possible
locations that a native addon would be built at, and returns the first one that loads successfully.
For example, if we built the debug version our native module will be located in the
`'./build/Debug/addon'` path, but if we compiled the debug version it will be in
`'./build/Release/addon'`. So, we will require our native module like this:

```javascript
var addon = require('bindings')('addon_name')
```

### Building the node modules

To compile our module we will use [node-gyp](https://github.com/nodejs/node-gyp), a tool written
specifically to compile Node.js addons. This tool wraps
[gyp-next](https://github.com/nodejs/gyp-next), a tool created by the Chromium team that starting
from a script will generate project files for other build systems such as XCode projects, Visual
Studio projects, Ninja build files, and Makefiles.

The input file for node-gyp is called *binding.gyp* and looks like this:

```json
{
  "targets": [
    {
      "target_name": "hello_world",
      "sources": [ "hello.cpp" ]
    }
  ]
}
```

This file is telling node-gyp to compile generate a Makefile to compile *hello.cpp* and create a library that will be called *hello_world.node*. To create this Makefile do:

```bash
node-gyp configure
```

Now you could go to the build folder and run directly the project files generated to compile the node addon or do it using node-gyp:

```bash
node-gyp build
```

Using this module in our Node.js application should be as simple as:

```javascript
// hello.js
const hello_module = require('bindings')('hello_world');
console.log(hello_module.hello_world());
```

Test this running `node hello.js`

You can find all the sources for this example in [this github
repository](https://github.com/czoido/napi-hello-world).