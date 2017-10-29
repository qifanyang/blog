## why
    一直做java后端开发,这次换工作出乎意料在做后台管理系统,以前很少做网页相关,做不一样的事情还是挺不错  
期间再次学习了下javascript.网页使用公司前端同学自己写的UI框架,名叫KISS-UI,有bug真还是个大麻烦,KISS-UI,以后离开公司估计再不会使用这个UI框架了,处于好奇比较了下市面上的VUE/Angular/React    

## Vue
    打开Vue教程,第一眼就看见声明式渲染,看了下demo
~~~
<div id="app">
  {{ message }}
</div>

var app = new Vue({
  el: '#app',
  data: {
    message: 'Hello Vue!'
  }
})

//输出  Hello Vue!
~~~
    这种写法太熟悉了,Kiss-ui也是这么写,通过html element声明一个占位节点,然后绑定一个js对象,动态创建  
html并插入占位节点,当数据改变时更新节点html内容  

## virtual DOM
    对于表格数据改变,重新构建表格开销是很大的,更新DOM也很麻烦.将数据全部保存在js对象中,当数据变化时  
对比js数据变化,重新渲染变化的内容,不再是直接操作DOM元素,而是js对象,所以叫虚拟DOM. React最早引入虚拟  DOM  

## Angular
    新的Angular需要使用TypeScript,增加了入门学习难度,对于构建大型去学习还是值得  

## React
    可以使用ReactJs做web,也可以使用React Native开发app.引入了JSX语法.  

## 对比
    感觉Vue相对容易入门,中文文档方便学习  

## 参考
https://www.zhihu.com/question/29504639  
https://cn.vuejs.org/v2/guide/  
https://facebook.github.io/react/docs/installation.html  

