
[结合源码解密 egg 运行原理](https://zhuanlan.zhihu.com/p/29102746)这篇文章从`egg-bin`开始到`loader`完成的过程解析的很清楚，整理为下图：

![egg-run.png](https://github.com/linyongkangm/Blog/blob/master/public/images/egg-run.png)

但是仍然有几个问题需要看源码才能知道是`为什么`或`怎么做`的:
- [为什么要使用 Symbol.for('egg#eggPath') 来指定当前框架的路径](#Mark1)
- [loader的顺序](#Mark2)
- [Controller的延时实例化](#Mark3)
- [Service的延时实例化](#Mark4)


##为什么要使用 Symbol.for('egg#eggPath') 来指定当前框架的路径

```JavaScript
class Application extends egg.Application {
  get [EGG_PATH]() {
    // 返回 framework 路径
    return path.dirname(__dirname);
  }
}
```

因为框架继承的实现方案是基于类继承的，每一层框架都必须继承上一层框架，内部需要收集每个框架所在目录，这就可以得到`eggPaths`属性，然后才可以load其他文件：

```JavaScript
  getEggPaths() {
    // avoid require recursively
    const EggCore = require('../egg');
    const eggPaths = [];

    let proto = this.app;

    // Loop for the prototype chain
    //沿着原型链收集Symbol.for('egg#eggPath')的值
    while (proto) {
      proto = Object.getPrototypeOf(proto);
      if (proto === Object.prototype || proto === EggCore.prototype) {
        break;
      }
      const eggPath = proto[Symbol.for('egg#eggPath')];
      const realpath = fs.realpathSync(eggPath);
      if (!eggPaths.includes(realpath)) {
        eggPaths.unshift(realpath);
      }
    }
    return eggPaths;
  }
```
<div id="Mark2"></div>
## loader的顺序
在`loadPlugins`的时候就会使用`eggPaths`:
```JavaScript
const eggPluginConfigPaths = this.eggPaths.map(eggPath => path.join(eggPath, 'config/plugin.default.js'));
```
最后经过[读入，合并，匹配match，按依赖排序](https://github.com/linyongkangm/egg-core/blob/eb4b12b3f0242fed5fd9a959850d7ce9e8666c25/lib/loader/mixin/plugin.js#L57)等步骤就会生成`getLoadUnits`属性，它是按依赖排序的插件信息表。
*有非内置插件的load必须先执行loadPlugin,因为会生成`orderPlugins`属性，在`getLoadUnits`会给其他load使用。*

然后分别是:
```JavaScript
loadConfig();
loadApplicationExtend();
loadRequestExtend();
loadResponseExtend();
loadContextExtend();
loadHelperExtend();
loadCustomApp();
loadService();
loadMiddleware();
```
都是使用`getLoadUnits`：
```JavaScript
  getLoadUnits() {
    if (this.dirs) {
      return this.dirs;
    }
    const dirs = this.dirs = [];
    if (this.orderPlugins) {
      for (const plugin of this.orderPlugins) {
        dirs.push({
          path: plugin.path,
          type: 'plugin',
        });
      }
    }
    // framework or egg path
    for (const eggPath of this.eggPaths) {
      dirs.push({
        path: eggPath,
        type: 'framework',
      });
    }
    // application
    dirs.push({
      path: this.options.baseDir,
      type: 'app',
    });
    return dirs;
  }
```
这里就可以看到先推入`orderPlugins`,然后推入框架的路径,最后推入应用的路径:*按插件 => 框架 => 应用依次加载*


<div id="Mark3"></div>
## Controller的延时实例化
在loaderController的时候首先会对收集到的每个控制器中的方法进行包装：
```JavaScript
initializer: (obj, opt) => {
  if (is.class(obj)) {
    obj.prototype.pathName = opt.pathName;
    obj.prototype.fullPath = opt.path;
    return wrapClass(obj);
  }
  return obj;
}
// Controller 是控制器类，来自模块的exports
function wrapClass(Controller) {
  let proto = Controller.prototype;
  const ret = {};
  // tracing the prototype chain
  while (proto !== Object.prototype) {
    const keys = Object.getOwnPropertyNames(proto);
    for (const key of keys) {
      // getOwnPropertyNames will return constructor
      // that should be ignored
      if (key === 'constructor') {
        continue;
      }
      ret[key] = methodToMiddleware(Controller, key);
    }
    proto = Object.getPrototypeOf(proto);
  }
  return ret;
  function methodToMiddleware(Controller, key) {
    return function classControllerMiddleware(...args) {
      const controller = new Controller(this);
      return utils.callFn(controller[key], args, controller);
    };
  }
}
```
包装的结果就是控制器中的方法（除了constructor）对应一个classControllerMiddleware方法，然后会根据控制器的路由挂载到对应的app.controller中：
```JavaScript
load() {
  const items = this.parse();//所有控制器组成的数组
  const target = this.options.target;//就是this.app.controller
  for (const item of items) {
    debug('loading item %j', item);
    // item { properties: [ 'a', 'b', 'c'], exports }
    // => target.a.b.c = exports
    // 遍历路径历次挂载，注意这里挂载的是用控制器相应方法封装的classControllerMiddleware
    item.properties.reduce((target, property, index) => {
      let obj;
      if (index === item.properties.length - 1) {
        obj = item.exports;
        if (obj && !is.primitive(obj)) {
          obj[FULLPATH] = item.fullpath;
          obj[EXPORTS] = true;
        }
      } else {
        obj = target[property] || {};// 挂载
      }
      target[property] = obj;
      return obj;
    }, target);
  }
  return target;
}
```

然后在接收到请求的时候执行的就会执行classControllerMiddleware:
```JavaScript
 function classControllerMiddleware(...args) {
   //这里的Controller就是来自包装方法时产生的闭包；
  const controller = new Controller(this);//对每个请求对新建一个Controller实例
  // 这里的key也是来自包装方法时产生的闭包
  return utils.callFn(controller[key], args, controller);//这里就可以使用新建的controller调用相应的方法了
};
```

<div id="Mark4"></div>
## Service的延时实例化
同样在loaderService的时候首先收集Service，然后会挟持app.context.service的getter：
```JavaScript
Object.defineProperty(app.context, property, {
  get() {
    
    // distinguish property cache,
    // cache's lifecycle is the same with this context instance
    // e.x. ctx.service1 and ctx.service2 have different cache
    if (!this[CLASSLOADER]) {
      this[CLASSLOADER] = new Map();
    }
    const classLoader = this[CLASSLOADER];
    let instance = classLoader.get(property);//先在context中读缓存
    if (!instance) {
      instance = getInstance(target, this);//target是Sercice的键值对
      classLoader.set(property, instance);//缓存
    }
    return instance;
  },
});
```
这里的this就是app.context，在每次请求的时候Koa都会生成一个新的context，这样就没了CLASSLOADER，就会重新生成，从而在每次请求的时候都会生成一个Service实例。
