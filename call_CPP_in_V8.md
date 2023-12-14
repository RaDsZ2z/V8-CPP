将V8源码编译成VS项目后

此代码直接运行在hello-world.cc中
```cpp
// Copyright 2015 the V8 project authors. All rights reserved.
// Use of this source code is governed by a BSD-style license that can be
// found in the LICENSE file.

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "include/libplatform/libplatform.h"
#include "include/v8-context.h"
#include "include/v8-initialization.h"
#include "include/v8-isolate.h"
#include "include/v8-local-handle.h"
#include "include/v8-primitive.h"
#include "include/v8-script.h"
// Copyright 2012 the V8 project authors. All rights reserved.
// Use of this source code is governed by a BSD-style license that can be
// found in the LICENSE file.

#include <errno.h>
#include <stdlib.h>
#include <string.h>
#include <sys/stat.h>

#include <algorithm>
#include <fstream>
#include <iomanip>
#include <iterator>
#include <string>
#include <tuple>
#include <type_traits>
#include <unordered_map>
#include <utility>
#include <vector>

#ifdef ENABLE_VTUNE_JIT_INTERFACE
#include "src/third_party/vtune/v8-vtune.h"
#endif

#include "include/libplatform/libplatform.h"
#include "include/libplatform/v8-tracing.h"
#include "include/v8-function.h"
#include "include/v8-initialization.h"
#include "include/v8-inspector.h"
#include "include/v8-isolate.h"
#include "include/v8-json.h"
#include "include/v8-locker.h"
#include "include/v8-profiler.h"
#include "include/v8-wasm.h"
#include "src/api/api-inl.h"
#include "src/base/cpu.h"
#include "src/base/logging.h"
#include "src/base/platform/memory.h"
#include "src/base/platform/platform.h"
#include "src/base/platform/time.h"
#include "src/base/platform/wrappers.h"
#include "src/base/sanitizer/msan.h"
#include "src/base/sys-info.h"
#include "src/base/utils/random-number-generator.h"
#include "src/compiler-dispatcher/optimizing-compile-dispatcher.h"
#include "src/d8/d8-console.h"
#include "src/d8/d8-platforms.h"
#include "src/d8/d8.h"
#include "src/debug/debug-interface.h"
#include "src/deoptimizer/deoptimizer.h"
#include "src/diagnostics/basic-block-profiler.h"
#include "src/execution/microtask-queue.h"
#include "src/execution/v8threads.h"
#include "src/execution/vm-state-inl.h"
#include "src/flags/flags.h"
#include "src/handles/maybe-handles.h"
#include "src/heap/parked-scope-inl.h"
#include "src/init/v8.h"
#include "src/interpreter/interpreter.h"
#include "src/logging/counters.h"
#include "src/logging/log-file.h"
#include "src/objects/managed-inl.h"
#include "src/objects/objects-inl.h"
#include "src/objects/objects.h"
#include "src/parsing/parse-info.h"
#include "src/parsing/parsing.h"
#include "src/parsing/scanner-character-streams.h"
#include "src/profiler/profile-generator.h"
#include "src/sandbox/testing.h"
#include "src/snapshot/snapshot.h"
#include "src/tasks/cancelable-task.h"
#include "src/utils/ostreams.h"
#include "src/utils/utils.h"

#ifdef V8_OS_DARWIN
#include <mach/mach.h>
#include <mach/task_policy.h>
#endif

#ifdef V8_ENABLE_MAGLEV
#include "src/maglev/maglev-concurrent-dispatcher.h"
#endif  // V8_ENABLE_MAGLEV

#if V8_OS_POSIX
#include <signal.h>
#endif  // V8_OS_POSIX

#ifdef V8_FUZZILLI
#include "src/d8/cov.h"
#endif  // V8_FUZZILLI

#ifdef V8_USE_PERFETTO
#include "perfetto/tracing/track_event.h"
#include "perfetto/tracing/track_event_legacy.h"
#endif  // V8_USE_PERFETTO

#ifdef V8_INTL_SUPPORT
#include "unicode/locid.h"
#endif  // V8_INTL_SUPPORT

#ifdef V8_OS_LINUX
#include <sys/mman.h>  // For MultiMappedAllocator.
#endif

#if !defined(_WIN32) && !defined(_WIN64)
#include <unistd.h>
#else
#include <windows.h>
#endif  // !defined(_WIN32) && !defined(_WIN64)

#if V8_ENABLE_WEBASSEMBLY
#include "src/trap-handler/trap-handler.h"
#endif  // V8_ENABLE_WEBASSEMBLY

#ifndef DCHECK
#define DCHECK(condition) assert(condition)
#endif

#ifndef CHECK
#define CHECK(condition) assert(condition)
#endif
using namespace v8;
//此函数执行对应的v8代码并输出返回结果
//执行函数时，无论返回值是整型或字符串，都是以字符串接收的
//后续或许可以改进？
void ExecuteScript(v8::Isolate* isolate, const char* script_source) {
  // 创建一个句柄作用域，以便在退出函数时释放局部句柄。
  v8::HandleScope handle_scope(isolate);

  // 创建一个包含脚本的字符串。
  v8::Local<v8::String> source =
      v8::String::NewFromUtf8(isolate, script_source,
                              v8::NewStringType::kNormal)
          .ToLocalChecked();

  // 编译这个脚本。
  v8::Local<v8::Script> script =
      v8::Script::Compile(isolate->GetCurrentContext(), source)
          .ToLocalChecked();

  // 运行这个脚本，结果保存在 result 中。
  v8::Local<v8::Value> result =
      script->Run(isolate->GetCurrentContext()).ToLocalChecked();

  // 将结果转换为字符串并打印。
  v8::String::Utf8Value utf8(isolate, result);
  printf("%s\n", *utf8);
}
void Hello(const v8::FunctionCallbackInfo<v8::Value>& args) {
  v8::Isolate* isolate = args.GetIsolate();
  // 设置返回值
  args.GetReturnValue().Set(v8::String::NewFromUtf8(isolate,
                                                    "Hello World!",
                                                    v8::NewStringType::kNormal)
                                .ToLocalChecked());
}
void Add(const v8::FunctionCallbackInfo<v8::Value>& args) {
  v8::Isolate* isolate = args.GetIsolate();
  v8::HandleScope handleScope(isolate);  // 保证所有局部 V8 对象在函数退出时释放

  if (args.Length() < 2) {
    isolate->ThrowException(
        v8::String::NewFromUtf8(isolate, "The function requires 2 arguments",
                                v8::NewStringType::kNormal)
            .ToLocalChecked());
    return;
  }

  if (args[0]->IsString() && args[1]->IsString()) {
    v8::String::Utf8Value str1(isolate, args[0]);
    v8::String::Utf8Value str2(isolate, args[1]);
    std::string result = std::string(*str1) + std::string(*str2);
    args.GetReturnValue().Set(
        v8::String::NewFromUtf8(isolate, result.c_str(),
                                v8::NewStringType::kNormal)
            .ToLocalChecked());
  } else if (args[0]->IsNumber() && args[1]->IsNumber()) {
    double num1 = args[0]->NumberValue(isolate->GetCurrentContext()).FromJust();
    double num2 = args[1]->NumberValue(isolate->GetCurrentContext()).FromJust();
    double sum = num1 + num2;
    args.GetReturnValue().Set(v8::Number::New(isolate, sum));
  } else {
    isolate->ThrowException(v8::String::NewFromUtf8(isolate,
                                                    "Invalid argument types",
                                                    v8::NewStringType::kNormal)
                                .ToLocalChecked());
  }
}
// C++ 类
class Point {
 public:
  int x, y;

  Point(int x, int y) : x(x), y(y) {}

  void Print() { std::cout << "Point(" << x << ", " << y << ")" << std::endl; }
};

// 胶水代码
void Point_Constructor(const v8::FunctionCallbackInfo<v8::Value>& args) {
  v8::Isolate* isolate = args.GetIsolate();

  if (args.IsConstructCall()) {
    // 调用 Point 构造函数逻辑

    // 检查参数个数和类型
    if (args.Length() == 2 && args[0]->IsInt32() && args[1]->IsInt32()) {
      int x = args[0]->Int32Value(isolate->GetCurrentContext()).ToChecked();
      int y = args[1]->Int32Value(isolate->GetCurrentContext()).ToChecked();

      // 创建一个新的Point实例
      Point* obj = new Point(x, y);

      // 包装C++对象作为V8 对象
      v8::Local<v8::Object> objWrap = args.This();
      objWrap->SetInternalField(0, v8::External::New(isolate, obj));

      // 设定对象的析构逻辑
      objWrap->SetAlignedPointerInInternalField(0, obj);
    } else {
      isolate->ThrowException(
          v8::String::NewFromUtf8(isolate, "Invalid arguments",
                                  v8::NewStringType::kNormal)
              .ToLocalChecked());
      return;
    }
  } else {
    isolate->ThrowException(v8::String::NewFromUtf8(isolate,
                                                    "Must be called with new",
                                                    v8::NewStringType::kNormal)
                                .ToLocalChecked());
  }
}

void Point_Print(const v8::FunctionCallbackInfo<v8::Value>& args) {
  //v8::Isolate* isolate = args.GetIsolate();

  // 取出本地包装的C++ 对象
  v8::Local<v8::Object> self = args.Holder();
  Point* point =
      static_cast<Point*>(self->GetAlignedPointerFromInternalField(0));

  // 调用C++ 对象方法
  point->Print();
}

v8::Local<v8::FunctionTemplate> CreatePointTemplate(v8::Isolate* isolate) {
  v8::EscapableHandleScope handle_scope(isolate);

  v8::Local<v8::FunctionTemplate> tpl =
      v8::FunctionTemplate::New(isolate, Point_Constructor);
  tpl->SetClassName(
      v8::String::NewFromUtf8(isolate, "Point", v8::NewStringType::kNormal)
          .ToLocalChecked());
  tpl->InstanceTemplate()->SetInternalFieldCount(1);  // 指定内部字段（用以保存C++对象）

  // 添加原型方法
  v8::Local<v8::Signature> signature = v8::Signature::New(isolate, tpl);
  v8::Local<v8::FunctionTemplate> printTpl = v8::FunctionTemplate::New(
      isolate, Point_Print, v8::Local<v8::Value>(), signature);
  tpl->PrototypeTemplate()->Set(isolate, "print", printTpl);

  return handle_scope.Escape(tpl);
}






void test(const char* argv0) {
  // 初始化V8引擎
  v8::V8::InitializeICUDefaultLocation(argv0);
  v8::V8::InitializeExternalStartupData(argv0);
  std::unique_ptr<v8::Platform> platform = v8::platform::NewDefaultPlatform();
  v8::V8::InitializePlatform(platform.get());
  v8::V8::Initialize();

  // 创建一个新的隔离环境
  v8::Isolate::CreateParams create_params;
  create_params.array_buffer_allocator =
      v8::ArrayBuffer::Allocator::NewDefaultAllocator();
  v8::Isolate* isolate = v8::Isolate::New(create_params);

  
  {
    v8::Isolate::Scope isolate_scope(isolate);  // 进入这个隔离环境的作用域
    v8::HandleScope handle_scope(isolate);  // 用于管理JavaScript对象的内存

    //------------
    //这里的三行是为了暴露C++的类加的
    v8::Local<v8::ObjectTemplate> global = v8::ObjectTemplate::New(isolate);
    v8::Local<v8::FunctionTemplate> ptTpl = CreatePointTemplate(isolate);
    global->Set(isolate, "Point", ptTpl);
    //------------

    v8::Local<v8::Context> context = v8::Context::New(isolate, nullptr, global);
    //为了暴露C++类把下面这一行换成了上面这一行
    //v8::Local<v8::Context> context = v8::Context::New(isolate);
    v8::Context::Scope context_scope(context);  // 进入这个上下文的作用域
    //1. js调用C++函数 (没有参数)
    {
      // 将我的函数模板与函数的实际实现关联
      v8::Local<v8::FunctionTemplate> func_template = v8::FunctionTemplate::New(isolate, Hello);

      // 将函数模板转换为函数对象
      v8::Local<v8::Function> func = func_template->GetFunction(context).ToLocalChecked();

      // 将函数对象绑定到全局对象的一个属性
      // 并在js中给函数取名为hello
      context->Global()->Set(context,v8::String::NewFromUtf8(isolate, "hello",v8::NewStringType::kNormal).ToLocalChecked(),func).Check();

      // 执行V8代码并输出返回值
      ExecuteScript(isolate, "hello();");//这个ExecuteScript函数是自己定义的
    }
    //2. js调用C++函数 (参数为两个整数或两个字符串)
    {
      // 将我的函数模板与函数的实际实现关联
      v8::Local<v8::FunctionTemplate> func_template =
          v8::FunctionTemplate::New(isolate, Add);

      // 将函数模板转换为函数对象
      v8::Local<v8::Function> func =
          func_template->GetFunction(context).ToLocalChecked();

      // 将函数对象绑定到全局对象的一个属性
      context->Global()
          ->Set(context,
                v8::String::NewFromUtf8(isolate, "Add",
                                        v8::NewStringType::kNormal)
                    .ToLocalChecked(),
                func)
          .Check();

      // 准备执行一段包含调用myFunction的JS代码
      ExecuteScript(isolate, "Add(11,22);");  // 这个函数是自己定义的
      ExecuteScript(isolate, "Add('11','22');");  // 这个函数是自己定义的
    }
    //3. C++变量暴露给js (字符串)
    {
      std::string str = "this is a string from C++";
      // 转换为 V8 字符串
      v8::Local<v8::String> v8_str =
          v8::String::NewFromUtf8(isolate, str.c_str(),
                                  v8::NewStringType::kNormal)
              .ToLocalChecked();
      // 在全局对象上设置属性
      // 可以选择用一个正确的属性名称替代 "a_str"
      context->Global()
          ->Set(context,
                v8::String::NewFromUtf8(isolate, "a_str",
                                        v8::NewStringType::kNormal)
                    .ToLocalChecked(),
                v8_str)
          .Check();

      ExecuteScript(isolate, "a_str;");
    }
    //4. C++变量暴露给js (数字)
    {
      int num = 114514;

      // 转换为 V8 数字
      v8::Local<v8::Number> v8_num = v8::Number::New(isolate, num);
      // 在全局对象上设置属性
      // 可以选择用一个正确的属性名称替代 "a_num"
      context->Global()
          ->Set(context,
                v8::String::NewFromUtf8(isolate, "a_num",
                                        v8::NewStringType::kNormal)
                    .ToLocalChecked(),
                v8_num)
          .Check();
      ExecuteScript(isolate, "a_num;");
    }
    //5. C++变量暴露给js (类)
    {
      // 在此处执行JavaScript代码
      v8::Local<v8::String> source = v8::String::NewFromUtf8Literal(
          isolate, "let p = new Point(1,2);p.print();");

      // Compile the source code.
      v8::Local<v8::Script> script =
          v8::Script::Compile(context, source).ToLocalChecked();

      // Run the script to get the result.
      v8::Local<v8::Value> result = script->Run(context).ToLocalChecked();

      // Convert the result to an UTF8 string and print it.
      v8::String::Utf8Value utf8(isolate, result);
      // printf("%s\n", *utf8);
    }
  }

  // 清理隔离环境
  isolate->Dispose();
  v8::V8::Dispose();
  v8::V8::DisposePlatform();
  delete create_params.array_buffer_allocator;

  // isolate->Dispose();
  // v8::V8::Dispose();
  // v8::V8::DisposePlatform();
  // delete create_params.array_buffer_allocator;
}
int main(int argc, char* argv[]) {
  test(argv[0]);
}

#undef CHECK
#undef DCHECK

```
