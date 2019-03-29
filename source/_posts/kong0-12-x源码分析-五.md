---
title: kong0.12.x源码分析(五)---路由匹配策略
date: 2019-03-15 12:40:11
tags: Kong
---

> Kong的路由匹配策略基本上是Kong最核心的部分了，逻辑虽然不是很复杂，但是代码还是比较复杂的。

整个路由匹配流程分为两部分：第一部分是系统启动初始化时对路由分类，建立索引。第二部分时请求到来时，根据请求的特征去查找相关路由。具体过程如下：

### 1. 路由初始化

Kong的路由初始化是在```init()```阶段，调用kong.core.handler中的```build_router()```

```lua
local function build_router(dao, version)
  local apis, err = dao.apis:find_all() --从数据库中加载所有的API
  if err then
    return nil, "could not load APIs: " .. err
  end

  for i = 1, #apis do
    -- alias since the router expects 'headers' as a map
    if apis[i].hosts then
      apis[i].headers = { host = apis[i].hosts }
    end
  end

  table.sort(apis, function(api_a, api_b)
    return api_a.created_at < api_b.created_at -- 根据api的创建时间进行排序
  end)

  router, err = Router.new(apis) -- 创建路由
  if not router then
    return nil, "could not create router: " .. err
  end

  if version then
    router_version = version
  end

  return true
end
```
首先启动时从数据库中加载所有的API,并根据创建时间对API进行排序，然后调用```router```创建一个新的路由器。

```lua
-- kong/core/router.lua
local plain_indexes = {
    hosts             = {},
    uris              = {},
    methods           = {},
  }  -- 存储普通的host、uri和methods
 local uris_prefixes  = {} -- 存储前缀匹配的uri
 local uris_regexes   = {} -- 存储含有正则的uri
 local wildcard_hosts = {} --存储带有正则的host
 local categories = {} -- 对API进行分类

 for i = 1, #apis do
    -- 第一步：解析API
    local api_t, err = marshall_api(apis[i])
    if not api_t then
      return nil, err
    end
    -- 第二步：对API进行分组
    categorize_api_t(api_t, api_t.match_rules, categories)
    -- 第三步：创建快速索引
    index_api_t(api_t, plain_indexes, uris_prefixes, uris_regexes,
                wildcard_hosts)
 end

 -- uri按照从长到短进行排序
 table.sort(uris_prefixes, function(a, b)
    return #a > #b
 end)

 table.sort(uris_prefixes, function(uri_t_a, uri_t_b)
    return #uri_t_a.value > #uri_t_b.value
 end)


```
创建路由器的过程分为三个步骤：解析API，对API进行分组，建立快速索引
#### 1.1 解析API

> 解析API做的事情就是把数据库中的格式转换成自己的数据结构，并标记uri是否是前缀匹配，host是否带有通配符等

```lua
-- Kong主要采用host uri 和method来匹配路由，每种元素都有自己的编码
local MATCH_RULES = {
  HOST            = 0x01,
  URI             = 0x02,
  METHOD          = 0x04,
}
local function marshall_api(api)
  if not (api.headers or api.methods or api.uris) then
    return nil, "could not categorize API"
  end

 -- 解析后的api_t数据结构
  local api_t      = {
    api            = api, --原始api
    strip_uri      = api.strip_uri, -- 是否strip uri
    preserve_host  = api.preserve_host, --传给upstream是否保留host
    match_rules    = 0x00, --当前api所属的匹配策略码，亦即所属类别
    hosts          = {},
    uris           = {},
    methods        = {},
    upstream_url_t = {},
  }
```
解析hosts:

> Kong将host、uri和method各分配了一个策略码，然后对他们进行按位与从而达到分类的目的

```lua
  if api.headers then
    ---省略校验代码
    local host_values = api.headers["Host"] or api.headers["host"]

    if #host_values > 0 then
      -- 采用按位 或 确定策略
      api_t.match_rules = bor(api_t.match_rules, MATCH_RULES.HOST)

      for _, host_value in ipairs(host_values) do
        if find(host_value, "*", nil, true) then
          -- 查看是否带有通配符
          local wildcard_host_regex = host_value:gsub("%.", "\\.")
                                                :gsub("%*", ".+") .. "$"
          insert(api_t.hosts, {
            wildcard = true,
            value    = host_value,
            regex    = wildcard_host_regex,
          })

        else
          insert(api_t.hosts, {
            value = host_value,
          })
        end

        api_t.hosts[host_value] = host_value
      end
    end
  end
```
解析uri和method的逻辑和解析host的逻辑类似。如果一个api同时配置了host、uri 和method，那么类别为7。
#### 1.2. API分组
API分组比较简单，主要将上一步解析出来的API按照类别重新组织一下数据结构

```lua
local function categorize_api_t(api_t, bit_category, categories)
  local category = categories[bit_category]
  if not category then
    category          = {
      apis_by_hosts   = {},
      apis_by_uris    = {},
      apis_by_methods = {},
      all             = {},
    }
    categories[bit_category] = category
  end
  insert(category.all, api_t)

  for _, host_t in ipairs(api_t.hosts) do
    if not category.apis_by_hosts[host_t.value] then
      category.apis_by_hosts[host_t.value] = {}
    end

    insert(category.apis_by_hosts[host_t.value], api_t)
  end
  -- 省略uri 和method分组逻辑，和host相同
end
```

#### 1.3 建立索引
这里说建立索引其实是将```host```、```uri```和```method```分别拆出来放到单独的数组里面
```lua
local function index_api_t(api_t, plain_indexes, uris_prefixes, uris_regexes,
                           wildcard_hosts)
  for _, host_t in ipairs(api_t.hosts) do
    if host_t.wildcard then
      insert(wildcard_hosts, host_t)

    else
      plain_indexes.hosts[host_t.value] = true
    end
  end

  for _, uri_t in ipairs(api_t.uris) do
    if uri_t.is_prefix then
      plain_indexes.uris[uri_t.value] = true
      insert(uris_prefixes, uri_t)

    else
      insert(uris_regexes, uri_t)
    end
  end

  for method in pairs(api_t.methods) do
    plain_indexes.methods[method] = true
  end
end
```
路由的建立上面已经分析完了，主要做的事情是解析API,然后将API根据特征分类，最后再对host、uri和method建立快速索引

### 2. 路由匹配

Kong匹配API是在access阶段的
```lua
function Kong.access()
  local ctx = ngx.ctx
  core.access.before(ctx)
  -- 省略后面执行插件的逻辑
```

最终会到router的exec方法中
```lua

  function self.exec(ngx)
    local method      = ngx.req.get_method()
    local request_uri = ngx.var.request_uri
    local uri         = request_uri
    do
      local idx = find(uri, "?", 2, true)
      if idx then
        uri = sub(uri, 1, idx - 1)
      end
    end

    local req_host = ngx.var.http_host or ""

    local match_t = find_api(method, uri, req_host, ngx)
    if not match_t then
      return nil
    end

    return match_t
  end
```
再到find_api方法
首先检查缓存中是否存在
```lua
    local cache_key = fmt("%s:%s:%s", req_method, req_uri, req_host)

    do
      local match_t = cache:get(cache_key)
      if match_t then
        return match_t
      end
    end
```
如果缓存中不存在，根据刚开始建立的索引来判断当前请求所属的类别
```lua
   if plain_indexes.hosts[req_host] then
      -- 通过 按位或计算类别
      req_category = bor(req_category, MATCH_RULES.HOST)

    elseif req_host then
      for i = 1, #wildcard_hosts do
        local from, _, err = re_find(req_host, wildcard_hosts[i].regex, "ajo")
        if err then
          log(ERR, "could not match wildcard host: ", err)
          return
        end

        if from then
          hits.host    = wildcard_hosts[i].value
          -- 通过 按位或计算类别
          req_category = bor(req_category, MATCH_RULES.HOST)
          break
        end
      end
    end
   -- 省略uri和method的匹配过程，和上面的类似

```

计算出所属的类别后，根据类别值判断出其在应该查找的API集合中的索引。比如一个请求带有host uri和methoud三个参数，那么其应该从 {host、uri、method}(API类别7) {host、uri}{API类别3}依次往下寻找匹配的API。查找时先从一个精简的集合中查找，如果查不到再全量查找。
```lua
local CATEGORIES = {
  bor(MATCH_RULES.HOST,   MATCH_RULES.URI,     MATCH_RULES.METHOD),
  bor(MATCH_RULES.HOST,   MATCH_RULES.URI),
  bor(MATCH_RULES.HOST,   MATCH_RULES.METHOD),
  bor(MATCH_RULES.METHOD, MATCH_RULES.URI),
      MATCH_RULES.HOST,
      MATCH_RULES.URI,
      MATCH_RULES.METHOD,
} 
local categories_len = #CATEGORIES

local CATEGORIES_LOOKUP = {}
for i = 1, categories_len do
  CATEGORIES_LOOKUP[CATEGORIES[i]] = i
end

local category_idx = CATEGORIES_LOOKUP[req_category]
local matched_api

-- 根据类别值判断出其在应该查找的API集合中的索引。比如一个请求带有host uri和methoud三个参数，那么其应该
-- 从 {host、uri、method}(API类别7) {host、uri}{API类别3}依次往下寻找匹配的API
while category_idx <= categories_len do
  local bit_category = CATEGORIES[category_idx]
  local category     = categories[bit_category]

  if category then
    -- 从一个精简结合中查找
    -- 如果精简集合中查不到，则全量查找此类别的所有API
  end
end
```
精简集合查找过程
```lua
local reduced_candidates, category_candidates = reduce(category, bit_category, ctx)
  if reduced_candidates then
    for i = 1, #reduced_candidates do
      if match_api(reduced_candidates[i], ctx) then
        matched_api = reduced_candidates[i]
        break
      end
    end
  end
```

```lua

do
  local reducers = {
    [MATCH_RULES.HOST] = function(category, ctx)
      return category.apis_by_hosts[ctx.hits.host]
    end,

    [MATCH_RULES.URI] = function(category, ctx)
      return category.apis_by_uris[ctx.hits.uri or ctx.req_uri]
    end,

    [MATCH_RULES.METHOD] = function(category, ctx)
      return category.apis_by_methods[ctx.req_method]
    end,
  }
  reduce = function(category, bit_category, ctx)
    -- run cached reducer
    if type(reducers[bit_category]) == "function" then
      return reducers[bit_category](category, ctx), category.all
    end

    -- build and cache reducer

    local reducers_set = {}
     
    -- 采用按位与将非1、2、4的API类别转为1、2、4的精简子集
    for _, bit_match_rule in pairs(MATCH_RULES) do
      if band(bit_category, bit_match_rule) ~= 0 then
        reducers_set[#reducers_set + 1] = reducers[bit_match_rule]
      end
    end

    reducers[bit_category] = function(category, ctx)
      local min_len = 0
      local smallest_set

      -- 选择集合元素最小的作为结果集返回
      -- 比如当前类别为7,其中国 host集合中有5个API，uri有3个API,method有8个API，那么返回uri这个集合中的API作为查找集
      for i = 1, #reducers_set do
        local candidates = reducers_set[i](category, ctx)
        if candidates ~= nil and (not smallest_set or #candidates < min_len)
        then
          min_len = #candidates
          smallest_set = candidates
        end
      end

      return smallest_set
    end

    return reducers[bit_category](category, ctx), category.all
  end
end
```

得到应该查找的API集合后，就采用匹配器进行比较
```lua
for i = 1, #reduced_candidates do
  if match_api(reduced_candidates[i], ctx) then
    matched_api = reduced_candidates[i]
    break
  end
end
```
```lua
do
  local matchers = {
    [MATCH_RULES.HOST] = function(api_t, ctx)
      local host = api_t.hosts[ctx.hits.host or ctx.req_host]
      if host then
        ctx.matches.host = host

        return true
      end
    end,
    -- 省略uri和method的比较过程
  }
  match_api = function(api_t, ctx)
    -- run cached matcher
    if type(matchers[api_t.match_rules]) == "function" then
      clear_tab(ctx.matches)
      return matchers[api_t.match_rules](api_t, ctx)
    end
    local matchers_set = {}
    -- 仍然转化为对host、method和uri的比较
    for _, bit_match_rule in pairs(MATCH_RULES) do
      if band(api_t.match_rules, bit_match_rule) ~= 0 then
        matchers_set[#matchers_set + 1] = matchers[bit_match_rule]
      end
    end

    matchers[api_t.match_rules] = function(api_t, ctx)
      clear_tab(ctx.matches)

      for i = 1, #matchers_set do
        if not matchers_set[i](api_t, ctx) then
          return
        end
      end

      return true
    end

    return matchers[api_t.match_rules](api_t, ctx)
  end


```
从精简集合中如果找不到的话就全量查找此类别中的所有API
```lua
if not matched_api then
  for i = 1, #category_candidates do
    if match_api(category_candidates[i], ctx) then
      matched_api = category_candidates[i]
      break
    end
  end
end
```

综上所述，Kong的路由匹配主要采用请求的```host```、```url```和```method```三元组来对请求进行分组，从而找到应该匹配的API。第一次匹配可能比较慢，后续存入缓存后，匹配速度就很快了。这种匹配策略有一个缺点，就是无法匹配哪些不仅仅靠host、uri和method区分的API，比如OpenAPI。针对OpenAPI类型的匹配还需要进行定制。

