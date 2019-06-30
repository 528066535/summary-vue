## Array 的变化侦测

实际上，Array的变化侦测与Object不同。

### 追踪变化

object可以通过Object.defineProperty的setter来追踪到变化，当数据发生变化时，会出发setter。

而在数组中，我们可以使用push等方法来修改数组，所以，当用户调用push等方法的时候，可以收集到数组的变化。

那么我们如何拦截push等操作数据的方法呢？聪明的程序猿用自定义的push方法覆盖原生的push方法，这样，在每次调用push方法的时候，都能拦截到数据的变化了。

### 拦截器

其实拦截器中的方法名称是和Array.prototype方法都是一模一样的，除了一些会改变数组的方法，我们加了一层处理。而这些方法包含7个，push、pop、
shift、unshift、splice、sort、reverse。

上代码：

    const arrayProto = Array.prototype
    export const arrayMethods = Object.create(arrayProto);

    ;['push','pop','shift','unshift','splice','sort','reverse'].forEach(function(method) {
        const original = arrayProto[method]
        Object.defineProperty(arrayMethods, method, {
            value: function mutator(...args) {
                return original.apply(this, args)
            },
            enumerable: false,
            writable: true,
            configurable: true
        })
    })

上面代码创建了一个新的对象，包含数组所有的方法。然后又重写了这个新对象中的某些方法，这些方法的特点是会修改数组的元素。而在调用这些方法的时候
都会进入mutator方法，这时候，我们就能拦截到会让数组元素变化时候的方法了，然后再做去一些其他的事情，比如发送变化的通知。

### 拦截器覆盖Array原型

### _proto_不兼容的处理

### 收集依赖

### 依赖数组

### 收集依赖

### Observer实例

### 向依赖发送通知

### 侦测数组中的元素变化

### 侦测新增元素的变化

### 无法侦测的情况

### 总结