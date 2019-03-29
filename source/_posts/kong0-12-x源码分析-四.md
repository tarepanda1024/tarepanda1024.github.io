---
title: kong0.12.x源码分析(四)---插件机制
date: 2019-03-15 12:39:50
tags: Kong
---

### Kong插件的目录结构
一个基本的Kong插件目录结构如下：
```lua
simple-plugin
├── handler.lua --通过实现```init_worker()```、```access()```等方法来实现自己的功能
└── schema.lua --存储插件配置的数据结构
```
```handler.lua```和请求的生命周期相对应
```lua
local BasePlugin = require "kong.plugins.base_plugin"
local CustomHandler = BasePlugin:extend()
function CustomHandler:new()
  CustomHandler.super.new(self, "my-custom-plugin")
end
function CustomHandler:init_worker()
  CustomHandler.super.init_worker(self)
end
function CustomHandler:certificate(config)
  CustomHandler.super.certificate(self)
end
function CustomHandler:rewrite(config)
  CustomHandler.super.rewrite(self)
end
function CustomHandler:access(config)
  CustomHandler.super.access(self)
end
function CustomHandler:header_filter(config)
  CustomHandler.super.header_filter(self)
end
function CustomHandler:body_filter(config)
  CustomHandler.super.body_filter(self)
end
function CustomHandler:log(config)
  CustomHandler.super.log(self)
end
CustomHandler.PRIORITY = 10 --控制插件优先级
return CustomHandler
```

### 插件的加载执行流程
> 整个插件的加载执行流程主要集中在```kong/init.lua```和```kong/core/plugins_iterator```这两个文件中

#### 插件的加载
整个加载过程在```init.lua```文件的```load_plugins(kong_conf, dao)```方法中。首先校验数据库正在使用的插件，配置文件中是否启用了；然后利用lua动态脚本的特性，将启用的插件元数据保存起来；最后根据优先级对插件进行排序，存储到全局的缓存中
```lua
  for plugin in pairs(kong_conf.plugins) do
    local ok, handler = utils.load_module_if_exists("kong.plugins." .. plugin .. ".handler")
    if not ok then
      return nil, plugin .. " plugin is enabled but not installed;\n" .. handler
    end
    local ok, schema = utils.load_module_if_exists("kong.plugins." .. plugin .. ".schema")
    if not ok then
      return nil, "no configuration schema found for plugin: " .. plugin
    end
    sorted_plugins[#sorted_plugins+1] = {
      name = plugin,
      handler = handler(),
      schema = schema
    }
  end
```
```lua
function _M.load_module_if_exists(module_name)
  local status, res = pcall(require, module_name)
  ---省略加载结果校验
end
```

#### 插件的执行
kong插件的执行比较简单，主要在nginx请求的各个阶段循环调用插件的相应方法即可
```lua
  for plugin, plugin_conf in plugins_iterator(singletons.loaded_plugins, true) do
    plugin.handler:rewrite(plugin_conf)
  end
```
![nginx指令顺寻](/images/2019031501.png)
需要注意的点是，插件的实例在```plugins_iterator```中加载的。插件有全局插件，也有API插件和Consumer插件，而API和ConsumerId是需要在access阶段（access阶段才根据host、method、uri等信息获取到api）才能获取到，因此access阶段之前只能执行全局的插件。

### 小结
总的来说，Kong的插件还是比较简单和优雅的，基于nginx提供的执行阶段稍作封装就实现了一套灵活的插件系统。

