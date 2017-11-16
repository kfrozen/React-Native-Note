# React-Native-Note

**React Native js调用Native端module流程**

- 实质是通过require("NativeModules")来调用Native端的Module，这里其实是指向了NativeModules.js，这个js file里面有这么一句：

```
	// export this method as a global so we can call it from native
	global.__fbGenNativeModule = genModule;
```
即将genModule这个方法export给全局变量global的__fbGenNativeModule属性。

- 之后进入JSCNativeModules.cpp中，具体方法是：

```
	JSValueRef JSCNativeModules::getModule(JSContextRef context, JSStringRef jsName){
	
			......
			
			auto module = createModule(moduleName, context);
			
			......
	}
```
进入到createModule方法：

```
	folly::Optional<Object> JSCNativeModules::createModule(const std::string& name, JSContextRef context) {
	
			......
			
			m_genNativeModuleJS = global.getProperty("__fbGenNativeModule").asObject();
			
			......
			
			Value moduleInfo = m_genNativeModuleJS->callAsFunction({
   			 	Value::fromDynamic(context, result->config),
    			Value::makeNumber(context, result->index)
  			});
  			CHECK(!moduleInfo.isNull()) << "Module returned from genNativeModule is null";

  			folly::Optional<Object> module(moduleInfo.asObject().getProperty("module").asObject());

  			ReactMarker::logTaggedMarker(ReactMarker::NATIVE_MODULE_SETUP_STOP, name.c_str());

  			return module;
	
	}
```

可以看到这里C++层取出了global对象中保存的__fbGenNativeModule属性指向的内容，即NativeModules中的genModule方法，并调用，在该方法中，会进一步对module中每一个定义的方法调用genMethod方法。进一步，在genMethod方法中，会调用：
```
	 BatchedBridge.enqueueNativeCall(moduleID, methodID, args, onFail, onSuccess);
```
这个方法等同于：
```
	 MessageQueue.enqueueNativeCall(moduleID, methodID, args, onFail, onSuccess);
```
在这个方法中，会将module，method和params的信息存入_queue这个数组变量中：
```
	 this._queue[MODULE_IDS].push(moduleID);
    this._queue[METHOD_IDS].push(methodID);
    this._queue[PARAMS].push(params);
    
    ....
    
    global.nativeFlushQueueImmediate(queue); //重点在这里，这里通过全局变量global携带的nativeFlushQueueImmediate方法调用了`JSCExecutor.cpp`中的flushQueueImmediate方法
```
之后的C++层流程为：
```
	 void JSCExecutor::callNativeModules --> NativeToJsBridge.cpp :void callNativeModules(
      JSExecutor& executor, folly::dynamic&& calls, bool isEndOfBatch) -->  m_registry->callNativeMethod(call.moduleId, call.methodId, std::move(call.arguments), call.callId); --> void ModuleRegistry::callNativeMethod(unsigned int moduleId, unsigned int methodId, folly::dynamic&& params, int callId) --> ModuleRegistry : modules_[moduleId]->invoke(methodId, std::move(params), callId);
```
到这里，就回调到了java中的接口，这个invoke实际就是NativeModule中定义的接口方法：
```
	public interface NativeModule {
  		interface NativeMethod {
    		void invoke(JSInstance jsInstance, ReadableNativeArray parameters);
    		String getType();
  		}
  	}
```
这个接口的具体实现位于JavaMethodWrapper.java中。

总结一下，流程大概是：Js调用会调用到C++中flushQueueImmediate函数，参数就是Js函数的参数加上调用的模块名构成的一个JSON字符串。C++中，有一个parseMethodCalls方法，会从Js传递的JSON中，解析出moduleName,function name,参数等一些列信息，然后C++层会调用Java层的ReactCallback类，Java代码中，会根据传递来的ModuleName functionName找到对应的模块中的方法，然后通过反射执行这些方法，并把参数传递过去。这样就完成了Js对Native代码的调用。
	

