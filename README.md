Native Abstractions for Node.js
===============================

**A header file filled with macro and utility goodness for making addon development for Node.js easier across versions 0.8, 0.10 and 0.11, and eventually 0.12.**

***Current version: 0.1.0*** *(See [nan.h](https://github.com/rvagg/nan/blob/master/nan.h) for changelog)*

Thanks to the crazy changes in V8 (and some in Node core), keeping native addons compiling happily across versions, particularly 0.10 to 0.11/0.12, is a minor nightmare. The goal of this project is to store all logic necessary to develop native Node.js addons without having to inspect `NODE_MODULE_VERSION` and get yourself into a macro-tangle.

This project also contains some helper utilities that make addon development a bit more pleasant.

 * **[Usage](#usage)**
 * **[Example](#example)**
 * **[API](#api)**

<a name="usage"></a>
## Usage

Simply copy **[nan.h](https://github.com/rvagg/nan/blob/master/nan.h)** from this repository into your project and load it in your source files with `#include "nan.h"`. The version is listed at the top of the file so be sure to check back here regularly for updates.

<a name="example"></a>
## Example

See **[LevelDOWN](https://github.com/rvagg/node-leveldown/pull/48)** for a full example of **NAN** in use.

or a simpler example, see the **[async pi estimation example](https://github.com/rvagg/nan/tree/master/examples/async_pi_estimate)** in the examples directory for full code an explanation of what this Monte Carlo Pi estimation example does. Below are just some parts of the full example that illustrate the use of **NAN**.

Compare to the current 0.10 version of this example, found in the [node-addon-examples](https://github.com/rvagg/node-addon-examples/tree/master/9_async_work) repository and also a 0.11 version of the same found [here](https://github.com/kkoopa/node-addon-examples/tree/5c01f58fc993377a567812597e54a83af69686d7/9_async_work).

Note that there is no embedded version sniffing going on here and also the async work is made much simpler, see below for details on the `NanAsyncWorker` class.

```c++
// addon.cc
#include <node.h>
#include "nan.h"
// ...

using namespace v8;

void InitAll(Handle<Object> exports) {
  exports->Set(NanSymbol("calculateSync"),
    FunctionTemplate::New(CalculateSync)->GetFunction());

  exports->Set(NanSymbol("calculateAsync"),
    FunctionTemplate::New(CalculateAsync)->GetFunction());
}

NODE_MODULE(addon, InitAll)
```

```c++
// sync.h
#include <node.h>
#include "nan.h"

NAN_METHOD(CalculateSync);
```

```c++
// sync.cc
#include <node.h>
#include "nan.h"
#include "sync.h"
// ...

using namespace v8;

// Simple synchronous access to the `Estimate()` function
NAN_METHOD(CalculateSync) {
  NanScope();

  // expect a number as the first argument
  int points = args[0]->Uint32Value();
  double est = Estimate(points);

  NanReturnValue(Number::New(est));
}
```

```c++
// async.cc
#include <node.h>
#include "nan.h"
#include "async.h"

// ...

using namespace v8;

class PiWorker : public NanAsyncWorker {
 public:
  PiWorker(NanCallback *callback, int points)
    : NanAsyncWorker(callback), points(points) {}
  ~PiWorker() {}

  // Executed inside the worker-thread.
  // It is not safe to access V8, or V8 data structures
  // here, so everything we need for input and output
  // should go on `this`.
  void Execute () {
    estimate = Estimate(points);
  }

  // Executed when the async work is complete
  // this function will be run inside the main event loop
  // so it is safe to use V8 again
  void HandleOKCallback () {
    NanScope();

    Local<Value> argv[] = {
        Local<Value>::New(Null())
      , Number::New(estimate)
    };

    callback->Call(2, argv);
  };

 private:
  int points;
  double estimate;
};

// Asynchronous access to the `Estimate()` function
NAN_METHOD(CalculateAsync) {
  NanScope();

  int points = args[0]->Uint32Value();
  NanCallback *callback = new NanCallback(args[1].As<Function>());

  NanAsyncQueueWorker(new PiWorker(callback, points));
  NanReturnUndefined();
}
```

<a name="api"></a>
## API

 * <a href="#api_nan_method"><b><code>NAN_METHOD</code></b></a>
 * <a href="#api_nan_getter"><b><code>NAN_GETTER</code></b></a>
 * <a href="#api_nan_setter"><b><code>NAN_SETTER</code></b></a>
 * <a href="#api_nan_return_value"><b><code>NanReturnValue</code></b></a>
 * <a href="#api_nan_return_undefined"><b><code>NanReturnUndefined</code></b></a>
 * <a href="#api_nan_scope"><b><code>NanScope</code></b></a>
 * <a href="#api_nan_object_wrap_handle"><b><code>NanObjectWrapHandle</code></b></a>
 * <a href="#api_nan_symbol"><b><code>NanSymbol</code></b></a>
 * <a href="#api_nan_from_v8_string"><b><code>NanFromV8String</code></b></a>
 * <a href="#api_nan_boolean_option_value"><b><code>NanBooleanOptionValue</code></b></a>
 * <a href="#api_nan_uint32_option_value"><b><code>NanUInt32OptionValue</code></b></a>
 * <a href="#api_nan_throw_error"><b><code>NanThrowError</code></b>, <b><code>NanThrowTypeError</code></b>, <b><code>NanThrowRangeError</code></b>, <b><code>NanThrowError(Local<Value>)</code></b></a>
 * <a href="#api_nan_new_buffer_handle"><b><code>NanNewBufferHandle(char *, uint32_t)</code></b>, <b><code>NanNewBufferHandle(uint32_t)</code></b></a>
 * <a href="#api_nan_has_instance"><b><code>NanHasInstance</code></b></a>
 * <a href="#api_nan_persistent_to_local"><b><code>NanPersistentToLocal</code></b></a>
 * <a href="#api_nan_dispose"><b><code>NanDispose</code></b></a>
 * <a href="#api_nan_assign_persistent"><b><code>NanAssignPersistent</code></b></a>
 * <a href="#api_nan_callback"><b><code>NanCallback</code></b></a>
 * <a href="#api_nan_async_worker"><b><code>NanAsyncWorker</code></b></a>
 * <a href="#api_nan_async_queue_worker"><b><code>NanAsyncQueueWorker</code></b></a>

<a name="api_nan_method"></a>
### NAN_METHOD(methodname)

Use `NAN_METHOD` to define your V8 accessible methods:

```c++
// .h:
class Foo : public node::ObjectWrap {
  ...

  static NAN_METHOD(Bar);
  static NAN_METHOD(Baz);
}


// .cc:
NAN_METHOD(Foo::Bar) {
  ...
}

NAN_METHOD(Foo::Baz) {
  ...
}
```

The reason for this macro is because of the method signature change in 0.11:

```c++
// 0.10 and below:
v8::Handle<v8::Value> name(const v8::Arguments& args)

// 0.11 and above
void name(const v8::FunctionCallbackInfo<v8::Value>& args)
```

The introduction of `FunctionCallbackInfo` brings additional complications:

<a name="api_nan_getter"></a>
### NAN_GETTER(methodname)

Use `NAN_GETTER` to declare your V8 accessible getters. You get a `Local<String>` `property` and an appropriately typed `args` object that can act like the `args` argument to a `NAN_METHOD` call.

You can use `NanReturnUndefined()` and `NanReturnValue()` in a `NAN_GETTER`.

<a name="api_nan_setter"></a>
### NAN_SETTER(methodname)

Use `NAN_SETTER` to declare your V8 accessible setters. Same as `NAN_GETTER` but you also get a `Local<Value>` `value` object to work with.

You can use `NanReturnUndefined()` and `NanReturnValue()` in a `NAN_SETTER`.

<a name="api_nan_return_value"></a>
### NanReturnValue(v8::Handle<v8::Value>)

Use `NanReturnValue` when you want to return a value from your V8 accessible method:

```c++
NAN_METHOD(Foo::Bar) {
  ...

  NanReturnValue(v8::String::New("FooBar!"));
}
```

No `return` statement required.

<a name="api_nan_return_undefined"></a>
### NanReturnUndefined()

Use `NanReturnUndefined` when you don't want to return anything from your V8 accessible method:

```c++
NAN_METHOD(Foo::Baz) {
  ...

  NanReturnUndefined();
}
```

<a name="api_nan_scope"></a>
### NanScope()

The introduction of `isolate` references for many V8 calls in Node 0.11 makes `NanScope()` necessary, use it in place of `v8::HandleScope scope`:

```c++
NAN_METHOD(Foo::Bar) {
  NanScope();

  NanReturnValue(v8::String::New("FooBar!"));
}
```

<a name="api_nan_object_wrap_handle"></a>
### v8::Local<v8::Object> NanObjectWrapHandle(Object)

When you want to fetch the V8 object handle from a native object you've wrapped with Node's `ObjectWrap`, you should use `NanObjectWrapHandle`:

```c++
NanObjectWrapHandle(iterator)->Get(v8::String::NewSymbol("end"))
```

<a name="api_nan_symbol"></a>
### v8::String NanSymbol(char *)

This isn't strictly about compatibility, it's just an easier way to create string symbol objects (i.e. `v8::String::NewSymbol(x)`), for getting and setting object properties, or names of objects.

```c++
bool foo = false;
if (obj->Has(NanSymbol("foo")))
  foo = optionsObj->Get(NanSymbol("foo"))->BooleanValue()
```

<a name="api_nan_from_v8_string"></a>
### char* NanFromV8String(v8::Handle<v8::Value>)

When you want to convert a V8 string to a `char*` use `NanFromV8String`. Just remember that you'll end up with an object that you'll need to `delete[]` at some point:

```c++
char* name = NanFromV8String(args[0]);
```

<a name="api_nan_boolean_option_value"></a>
### bool NanBooleanOptionValue(v8::Handle<v8::Value>, v8::Handle<v8::String>[, bool])

When you have an "options" object that you need to fetch properties from, boolean options can be fetched with this pair. They check first if the object exists (`IsEmpty`), then if the object has the given property (`Has`) then they get and convert/coerce the property to a `bool`.

The optional last parameter is the *default* value, which is `false` if left off:

```c++
// `foo` is false unless the user supplies a truthy value for it
bool foo = NanBooleanOptionValue(optionsObj, NanSymbol("foo"));
// `bar` is true unless the user supplies a falsy value for it
bool bar = NanBooleanOptionValueDefTrue(optionsObj, NanSymbol("bar"), true);
```

<a name="api_nan_uint32_option_value"></a>
### uint32_t NanUInt32OptionValue(v8::Handle<v8::Value>, v8::Handle<v8::String>[, uint32_t])

Similar to `NanBooleanOptionValue`, use `NanUInt32OptionValue` to fetch an integer option from your options object. Requires all 3 arguments as a default is not optional:

```c++
uint32_t count = NanUInt32OptionValue(optionsObj, NanSymbol("count"), 1024);
```

<a name="api_nan_throw_error"></a>
### NanThrowError(message), NanThrowTypeError(message), NanThrowRangeError(message), NanThrowError(Local<Value>)

For throwing `Error`, `TypeError` and `RangeError` objects. You should `return` this call:

```c++
return NanThrowError("you must supply a callback argument");
```

Can also handle any custom object you may want to throw.

<a name="api_nan_new_buffer_handle"></a>
### v8::Local<v8::Object> NanNewBufferHandle(char *, uint32_t), v8::Local<v8::Object> NanNewBufferHandle(uint32_t)

The `Buffer` API has changed a little in Node 0.11, this helper provides consistent access to `Buffer` creation:

```c++
NanNewBufferHandle((char*)value.data(), value.size());
```

Can also be used to initialize a `Buffer` with just a `size` argument.

<a name="api_nan_has_instance"></a>
### bool NanHasInstance(Persistent<FunctionTemplate>&, Handle<Value>)

Can be used to check the type of an object to determine it is of a particular class you have already defined and have a `Persistent<FunctionTemplate>` handle for.

<a name="api_nan_persistent_to_local"></a>
### v8::Local<Type> NanPersistentToLocal(v8::Persistent<Type>&)

Aside from `FunctionCallbackInfo`, the biggest and most painful change to V8 in Node 0.11 is the many restrictions now placed on `Persistent` handles. They are difficult to assign and difficult to fetch the original value out of.

Use `NanPersistentToLocal` to convert a `Persistent` handle back to a `Local` handle.

```c++
v8::Local<v8::Object> handle = NanPersistentToLocal(persistentHandle);
```

<a name="api_nan_dispose"></a>
### void NanDispose(v8::Persistent<v8::Object> &)

Use `NanDispose` to dispose a `Persistent` handle.

```c++
NanDispose(persistentHandle);
```

<a name="api_nan_assign_persistent"></a>
### NanAssignPersistent(type, handle, object)

Use `NanAssignPersistent` to assign a non-`Persistent` handle to a `Persistent` one. You can no longer just declare a `Persistent` handle and assign directly to it later, you have to `Reset` it in Node 0.11, so this makes it easier.

In general it is now better to place anything you want to protect from V8's garbage collector as properties of a generic `v8:Object` and then assign that to a `Persistent`. This works in older versions of Node also if you use `NanAssignPersistent`:

```c++
v8::Persistent<v8::Object> persistentHandle;

...

v8::Local<v8::Object> obj = v8::Object::New();
obj->Set(NanSymbol("key"), keyHandle); // where keyHandle might be a v8::Local<v8::String>
NanAssignPersistent(v8::Object, persistentHandle, obj)
```

<a name="api_nan_callback"></a>
### NanCallback

Because of the difficulties imposed by the changes to `Persistent` handles in V8 in Node 0.11, creating `Persistent` versions of your `v8::Local<v8::Function>` handles is annoyingly tricky. `NanCallback` makes it easier by taking your `Local` handle, making it persistent until the `NanCallback` is deleted and even providing a handy `Call()` method to fetch and execute the callback `Function`.

```c++
v8::Local<v8::Function> callbackHandle = callback = args[0].As<v8::Function>();
NanCallback *callback = new NanCallback(callbackHandle);
// pass `callback` around and it's safe from GC until you:
delete callback;
```

You can execute the callback like so:

```c++
// no arguments:
callback->Call(0, NULL);

// an error argument:
v8::Local<v8::Value> argv[] = {
  v8::Exception::Error(v8::String::New("fail!"))
};
callback->Call(1, argv);

// a success argument:
v8::Local<v8::Value> argv[] = {
  v8::Local<v8::Value>::New(v8::Null()),
  v8::String::New("w00t!")
};
callback->Call(2, argv);
```

`NanCallback` also has a `Local<Function> GetCallback()` method that you can use to fetch a local handle to the underlying callback function if you need it.

<a name="api_nan_async_worker"></a>
### NanAsyncWorker

`NanAsyncWorker` is an abstract class that you can subclass to have much of the annoying async queuing and handling taken care of for you. It can even store arbitrary V8 objects for you and have them persist while the async work is in progress.

See a rough outline of the implementation:

```c++
class NanAsyncWorker {
public:
  NanAsyncWorker (NanCallback *callback);

  // Clean up persistent handles and delete the *callback
  virtual ~NanAsyncWorker ();

  // Check the `char *errmsg` property and call HandleOKCallback()
  // or HandleErrorCallback depending on whether it has been set or not
  virtual void WorkComplete ();

  // You must implement this to do some async work. If there is an
  // error then set `errmsg` to to a message and the callback will
  //be passed that string in an Error object
  virtual void Execute ();

protected:
  // Set this if there is an error, otherwise it's NULL
  const char *errmsg;

  // Save a V8 object in a Persistent handle to protect it from GC
  void SavePersistent(const char *key, v8::Local<v8::Object> &obj);

  // Fetch a stored V8 object (don't call from within `Execute()`)
  v8::Local<v8::Object> GetFromPersistent(const char *key);

  // Default implementation calls the callback function with no arguments.
  // Override this to return meaningful data
  virtual void HandleOKCallback ();

  // Default implementation calls the callback function with an Error object
  // wrapping the `errmsg` string
  virtual void HandleErrorCallback ();
};
```

<a name="api_nan_async_queue_worker"></a>
### NanAsyncQueueWorker(NanAsyncWorker *)

`NanAsyncQueueWorker` will run a `NanAsyncWorker` asynchronously via libuv. Both the *execute* and *after_work* steps are taken care of for you&mdash;most of the logic for this is embedded in `NanAsyncWorker`.

Licence &amp; copyright
-----------------------

Copyright (c) 2013 Rod Vagg

Native Abstractions for Node.js is licensed under an MIT +no-false-attribs license. All rights not explicitly granted in the MIT license are reserved. See the included LICENSE file for more details.