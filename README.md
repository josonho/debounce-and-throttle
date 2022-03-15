下面以3个方式实现防抖节流
### 通过vue的data实现防抖节流
利用vue的data属性可以被实例全局调用，并且不被销毁的特点实现防抖节流
```html
<!DOCTYPE html>
<html lang="en">
<body>
  <div id="app">
    <h3>vue 通过data属性实现防抖节流</h3>
    <input placeholder="输入以测试防抖" @input="onDebounce">
    <input placeholder="输入以测试节流" @input="onThrottle">
  </div>
</body>
</html>

<script src="https://cdn.jsdelivr.net/npm/vue@2.6.14/dist/vue.min.js"></script>
<script>
  new Vue({
    el: '#app',
    methods: {
      onDebounce() {
        clearTimeout(this.debounceTimer)
        this.debounceTimer = setTimeout(() => {
          console.log('防抖...')
        }, 500)
      },
      
      onThrottle() {
        if (this.throttleCanRun === undefined) {
          this.throttleCanRun = true
        }
        if (!this.throttleCanRun) return
        this.throttleCanRun = false
        setTimeout(() => {
          console.log('节流...')
          this.throttleCanRun = true
        }, 500) 
      },
    }
  })
</script>
```
虽然在vue中data里面的属性也可以达到闭包的效果(变量不被销毁)，但是如果项目中需要将防抖节流抽离封装，data的属性就不能实现。
那么，有没有什么方案可以达到data变量不被销毁效果呢？这让我们想起闭包，闭包可以让变量私有化，并且变量不被销毁。
在使用闭包实现防抖节流之前，我们先看一个最初闭包实现
```javascript
var closure = (function () {
	// 函数A所在的作用域
    var a = 0
    return function () { // 函数A
    	// 在这里调用a，即函数与函数所在的作用域发生引用捆绑
        console.log(a++)
    }
})()
/* 输入看到的是a不断累加，说明a不被销毁。
并没有重置（因为a是在立即执行函数执行时声明），
立即执行函数执行时创建一个闭包，
*/
closure() // 0
closure() // 1
closure() // 2
```

### 通过闭包实现防抖节流
利用闭包中变量不被销毁，私有性实现防抖节流
```html
<!DOCTYPE html>
<html lang="en">
<body>
  <div id="app">
    <h3>vue 通过闭包实现防抖节流</h3>
    <input placeholder="输入以测试防抖" @input="onDebounce">
    <input placeholder="输入以测试节流" @input="onThrottle">
  </div>
</body>
</html>

<script src="https://cdn.jsdelivr.net/npm/vue@2.6.14/dist/vue.min.js"></script>
<script>
  /* 闭包只能声明一次，
  如果放在methods将会随事件触发而声明，
  所以需将闭包提取出来，
  通过立即执行函数创建闭包 */
  const debounce = (() => {
    // 利用闭包使变量debounceTimer私有化，不被销毁
    let debounceTimer = null 
    return function () {
      clearTimeout(debounceTimer)
      debounceTimer = setTimeout(() => {
        console.log('防抖...')
      }, 500)
    }
  })()

  const throttle = (() => {
    // 利用闭包使变量throttleCanRun私有化，不被销毁
    let throttleCanRun = true 
    return function () {
      if (!throttleCanRun) return
      throttleCanRun = false
      setTimeout(() => {
        console.log('节流...')
        throttleCanRun = true
      }, 500) 
    }
  })()

  new Vue({
    el: '#app',
    methods: {
      onDebounce() {
        debounce()
      },

      onThrottle() {
        throttle()
      },
    }
  })
</script>
```
### 闭包方式封装防抖节流
防抖节流优化封装，可在项目中抽离到为工具函数
```html
<!DOCTYPE html>
<html lang="en">
<body>
  <div id="app">
    <h3>vue 闭包实现防抖节流封装</h3>
    <p>tips: 请打开控制台观察打印变化</p>
    <input placeholder="输入以测试防抖" @input="onDebounce">
    <input placeholder="输入以测试节流" @input="onThrottle">
    <input placeholder="输入以测试时间差节流" @input="onTimeThrottle">
  </div>
</body>
</html>

<script src="https://cdn.jsdelivr.net/npm/vue@2.6.14/dist/vue.min.js"></script>
<script>
  const debounce = (() => {
    let debounceTimer = null
    return function (fn, delay) {
      clearTimeout(debounceTimer)
      debounceTimer = setTimeout(fn, delay)
    }
  })()
  
  // 计时器实现节流
  const throttle = (() => {
    // 创建闭包时声明throttleCanRun为true
    let throttleCanRun = true 
    return function (fn, delay) {
      // 只有第一次执行，或每隔delay毫秒，throttleCanRun才为true，才能往下执行
      if (!throttleCanRun) return
      /* 每次执行到这里都将throttleCanRun置为false，
      而throttleCanRun置为true需等到delay毫秒后才触发 
      达到每隔delay毫秒后，throttleCanRun变为true一次，
      达到节流的目的 */
      throttleCanRun = false
      setTimeout(() => {
        fn()
        throttleCanRun = true
      }, delay) 
    }
  })()
  

  // 时间差实现节流
  const timeThrottle = (() => {
    let start
    return function (fn, delay) {
      // 第一次触发事件时，初始化start
      if (!start) {
        /* 第一次触发事件时先执行fn，
        因为这里不执行fn的话，
        当用户触发事件维持时间少于delay毫秒时，
        fn将不被触发*/
        fn() 
        start = new Date()
      }
      // 当事件反复触发，这里判断时间差是否大于delay
      if (new Date() - start > delay) {
        fn()
        // 重置start为最新时间
        start = new Date()
      }
    }
  })()

  new Vue({
    el: '#app',
    methods: {
      onDebounce() {
        // 调用时需注意this指向，可使用箭头函数，或者使用self = this
        debounce(() => {
          console.log('防抖...')
        }, 500)
      },
      onThrottle() {
        throttle(() => {
          console.log('节流...')
        }, 300)
      },
      onTimeThrottle() {
        timeThrottle(() => {
          console.log('时间差节流...')
        }, 300)
      }
    }
  })
</script>
```

\>>> [github page 在线演示](https://josonho.github.io/debounce-and-throttle)
