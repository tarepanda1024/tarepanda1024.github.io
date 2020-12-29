---
title: kong0.12.x源码分析(三)----kong事件通知机制
categories: 网关
date: 2018-07-31 20:30:01
tags: ['Lua','网关','Kong']
---


最近一直在搞网关插件，没来的及写博客。不过也有好处，就是对代码的熟悉程度更高了一点。
今天我们来看一下kong的事件通知机制。
kong的事件通知基于[lua-resty-worker-events](https://github.com/Kong/lua-resty-worker-events) 这个库，这个库也是Kong开发的。
Kong的事件通知主要涉及两个方面，一个方面是单个节点内的事件通知，比如api信息更新，插件更新等；另一个方面是集群内的信息同步

我们先看单个节点内的事件通知：
单个节点内的事件主要采用发布订阅的方式，大家首先都到事件中心注册感兴趣的事件，然后当这些事件发生时，事件中心再进行回调处理。那么lua-resty-worker-events这个库就扮演着事件中心的角色。

事件注册的相关代码主要在 kong/core/router.lua文件里：
```lua
      worker_events.register(function(data)
        if not data.schema then
          log(ngx.ERR, "[events] missing schema in crud subscriber")
          return
        end

        if not data.entity then
          log(ngx.ERR, "[events] missing entity in crud subscriber")
          return
        end


        local cache_key = dao[data.schema.table]:entity_cache_key(data.entity)
        if cache_key then
          cache:invalidate(cache_key)
        end


        if data.old_entity then
          cache_key = dao[data.schema.table]:entity_cache_key(data.old_entity)
          if cache_key then
            cache:invalidate(cache_key)
          end
        end

        if not data.operation then
          log(ngx.ERR, "[events] missing operation in crud subscriber")
          return
        end

        local entity_channel           = data.schema.table
        local entity_operation_channel = fmt("%s:%s", data.schema.table,
                                             data.operation)

        -- crud:apis
        local _, err = worker_events.post_local("crud", entity_channel, data)
        if err then
          log(ngx.ERR, "[events] could not broadcast crud event: ", err)
          return
        end

        -- crud:apis:create
        _, err = worker_events.post_local("crud", entity_operation_channel, data)
        if err then
          log(ngx.ERR, "[events] could not broadcast crud event: ", err)
          return
        end
      end, "dao:crud")


```
从上面的代码我们可以看到，当数据库发生更新操作时，主要执行清除缓存的操作（缓存清除了，下次请求到来就会从数据库中重新拉取数据）
然后剩余的一些事件注册也在这个文件里，我们看一下主要也都是关于数据库变更的事件通知，做的主要操作也是清除缓存。另外还有一些事件会通知到集群的其它节点，比如：```cluster_events:broadcast("balancer:targets", key)```。这个集群事件我们随后再分析。
接下来我们看一下事件的生产者：
从上面的事件订阅那里我们可以看到事件生产者主要应该在dao层这里，我们打开　kong/dao/dao.lua文件，在　update、insert和delete方法可以很明显的看到类似的语句。说明我们的判断是正确的。
```lua
local _, err = self.events.post_local("dao:crud", "delete", {
        schema = self.schema,
        operation = "delete",
        entity = row,
      })
```
总的来说单个node内的集群主要就是采用发布订阅的方式来进行事件的更新。
那么集群范围内又是如何更新节点的呢？
我们看一下kong/cluster_events.lua文件，可以很明显的看到有一个subscribe方法，也有一个broadcast方法，难道也是采用的发布订阅模式？其实的确是这样的:
```lua
function _M:broadcast(channel, data, nbf)
  if type(channel) ~= "string" then
    return nil, "channel must be a string"
  end

  if type(data) ~= "string" then
    return nil, "data must be a string"
  end

  if nbf and type(nbf) ~= "number" then
    return nil, "nbf must be a number"
  end

  -- insert event row

  --log(DEBUG, "broadcasting on channel: '", channel, "' data: ", data,
  --           " with nbf: ", nbf and nbf or "none")

  local ok, err = self.strategy:insert(self.cluster_name, self.node_id, channel, ngx_now(), data, nbf)
  if not ok then
    return nil, err
  end

  return true
end
```

可以看到发布事件是主要就是往数据库中插入一条数据
然后我们看一下订阅相关的代码
```lua
function _M:subscribe(channel, cb, start_polling)
  if type(channel) ~= "string" then
    return error("channel must be a string")
  end

  if type(cb) ~= "function" then
    return error("callback must be a function")
  end

  if not self.callbacks[channel] then
    self.callbacks[channel] = { cb }

    insert(self.channels, channel)

  else
    insert(self.callbacks[channel], cb)
  end

  if start_polling == nil then
    start_polling = true
  end

  if not self.polling and start_polling then
    -- start recurring polling timer

    local ok, err = timer_at(self.poll_interval, poll_handler, self)
    if not ok then
      return nil, "failed to start polling timer: " .. err
    end

    self.polling = true
  end

  return true
end
local function poll(self)
  -- get events since last poll

  local min_at, err = self.shm:get(CURRENT_AT_KEY)
  if err then
    return nil, "failed to retrieve 'at' in shm: " .. err
  end

  if not min_at then
    return nil, "no 'at' in shm"
  end

  -- apply grace period

  min_at = min_at - self.poll_offset - 0.001

  local max_at = ngx_now()

  log(DEBUG, "polling events from: ", min_at, " to: ", max_at)

  for rows, err, page in self.strategy:select_interval(self.channels, self.cluster_name, min_at, max_at) do
    if err then
      return nil, "failed to retrieve events from DB: " .. err
    end

    if page == 1 then
      local ok, err = self.shm:safe_set(CURRENT_AT_KEY, max_at)
      if not ok then
        return nil, "failed to set 'at' in shm: " .. err
      end
    end

    for i = 1, #rows do
      local row = rows[i]

      if row.node_id ~= self.node_id then
        local ran, err = self.events_shm:get(row.id)
        if err then
          return nil, "failed to probe if event ran: " .. err
        end

        if not ran then
          log(DEBUG, "new event (channel: '", row.channel, "') data: '",
                     row.data, "' nbf: '", row.nbf or "none", "'")

          local exptime = self.poll_interval + self.poll_offset

          -- mark as ran before running in case of long-running callbacks
          local ok, err = self.events_shm:set(row.id, true, exptime)
          if not ok then
            return nil, "failed to mark event as ran: " .. err
          end

          local cbs = self.callbacks[row.channel]
          if cbs then
            for j = 1, #cbs do
              if not row.nbf then
                -- unique callback run without delay
                local ok, err = pcall(cbs[j], row.data)
                if not ok and not ngx_debug then
                  log(ERR, "callback threw an error: ", err)
                end

              else
                -- unique callback run after some delay
                local delay = max(row.nbf - ngx_now(), 0)

                log(DEBUG, "delaying nbf event by ", delay, "s")

                local ok, err = timer_at(delay, nbf_cb_handler, cbs[j], row)
                if not ok then
                  log(ERR, "failed to schedule nbf event timer: ", err)
                end
              end
            end
          end
        end
      end
    end
  end

  return true
end
```
可以看到订阅这里主要就是维持了一个定时器，不断根据间隔去数据库中poll数据，发现有数据更新了，执行一下数据更新回调（kong源代码中没有cluster_name字段，这个是我为了公司业务新添加的字段不影响阅读）

kong的事件更新机制主要还是发布订阅模型。





