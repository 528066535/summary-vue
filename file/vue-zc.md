## Object 的变化侦测

实际上，Array的变化侦测与Object不同。

### 什么是变化侦测

变化侦测就是侦测一个数据的变化，只有知道了数据的变化，才能继续做其他事情。JS有两个办法可以侦测到对象的变化，一个是Object.defineProperty和
ES6的Proxy。vue2.x 版本使用的是前者。Vue3.x会用ES6的Proxy方法去重写这部分代码。

### Object.defineProperty

直接上代码：

    function defineReactive(data, key, val) {
        Object.defineProperty(data, key, {
            enumerable: true,
            configurable: true,
            get: function() {
                return val
            },
            set: function(newVal) {
                if(val === newVal) {
                    return
                }
                val = newVal
            }
        })
    }

这里把 Object.defineProperty 再次封装，其中get方法可以获取到data中的某个数据的值，set方法，可以知道数据的变化，参数只需要三个。

### 依赖收集和触发

我们知道，get可以获取到某个数据的值，而在get的过程中，我们会把这些数据收藏起来，这个过程可以称作依赖的收集，而当调用set方法的时候，我们需要
通知其他方法，数据发生了变化，这时候会触发在get中收集的依赖。

#### 依赖收集和触发的代码逻辑

那这些依赖，具体是怎么收集起来的呢？假设某个依赖是被存到了window.target上，并且假设window.target是一个函数，那么我们可以这么写。

上代码：

    function defineReactive(data, key, val) {
        **let dep = [] //新增**
        Object.defineProperty(data, key, {
            enumerable: true,
            configurable: true,
            get: function() {
                **dep.push(window.target)//新增**
                return val
            },
            set: function(newVal) {
                if(val === newVal) {
                    return
                }
                **
                //新增
                for(let i = 0; i < dep.length; i++) {
                    dep[i](newVal, val)
                }
                **
                val = newVal
            }
        })
    }

#### 依赖被收集在哪

#### 依赖是谁

### 什么是Watch

### 侦测所有的数据

### Object中无法侦测的部分

