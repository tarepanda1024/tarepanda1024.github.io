---
title: nuxt使用element ui2.x踩坑记录
date: 2017-12-01 17:09:20
tags:
categories: 前端
---
### 1. 第一步我们使用nuxt官方提供的starter模板创建项目
```bash
vue init nuxt-community/starter-template <project-name>
```
### 2. 安装相关依赖
```bash
cd <project-name>
npm install
```
### 3. 安装element-ui (v2.0.7)
```bash
npm i element-ui -S
npm install babel-plugin-component -D
```

### 4. 我们需要配置一下nuxt.config.js
```javascript
module.exports = {
  /*
  ** Headers of the page
  */
  head: {
    title: 'web',
    meta: [
      { charset: 'utf-8' },
      { name: 'viewport', content: 'width=device-width, initial-scale=1' },
      { hid: 'description', name: 'description', content: 'Nuxt.js project' }
    ],
    link: [
      { rel: 'icon', type: 'image/x-icon', href: '/favicon.ico' }
    ]
  },
  /*
  ** Customize the progress bar color
  */
  loading: { color: '#3B8070' },
  /*
  ** Build configuration
  */
  plugins: [
    { src: '~plugins/element-ui', ssr: true }
  ],
  css: [
    'element-ui/lib/theme-chalk/index.css'
  ],
  build: {
    vendor: [
      'element-ui'
    ],
    babel: {
      "plugins": [["component", [
        {
          "libraryName": "element-ui",
          "styleLibraryName": "theme-chalk"
        },
        'transform-async-to-generator',
        'transform-runtime'
      ]]],
      comments: true
    },
  }
}

```

### 5. 在plugins目录中新建一个element-ui.js文件,文件内容为:
```javascript
import Vue from 'vue'
import ElementUI from 'element-ui'
Vue.use(ElementUI)
```

 ###  6. ok,这样我们的环境就配好了，在pages目录中新建一个login.vue
 
 ```javascript
<template>
<div>
  <el-button>默认按钮</el-button>
  <el-button type="primary">主要按钮</el-button>
  <el-button type="success">成功按钮</el-button>
  <el-button type="info">信息按钮</el-button>
  <el-button type="warning">警告按钮</el-button>
  <el-button type="danger">危险按钮</el-button>
</div>
</template>

<script>
export default {
  data() {
    return {};
  },
  methods: {}
};
</script>

<style >

</style>

 ```
 
 ### 7.执行```npm run dev ``` 打开``` http://localhost:3000/login```，
 这样就看到一排element-ui样式的按钮了
 