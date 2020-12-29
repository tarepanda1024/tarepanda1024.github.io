---
title: kong0.12.x源码分析(二)----kong启动流程分析
date: 2018-05-01 09:29:17
tags: ['Lua','网关','Kong']
categories: 网关
---
今天我们看一下kong的启动流程，kong启动时调用kong start命令，kong的启动脚本内容如下：
```bash
#!/usr/bin/env resty

require "luarocks.loader"

package.path = "./?.lua;./?/init.lua;" .. package.path

require("kong.cmd.init")(arg)

```

在这个脚本里，通过require命令，调用kong/cmd/init.lua脚本，接下来我们看一下init脚本里放了些什么
```lua
local pl_app = require "pl.lapp"
local log = require "kong.cmd.utils.log"

local options = [[
 --v              verbose
 --vv             debug
]]

local cmds_arr = {}
local cmds = {
  start = true,
  stop = true,
  quit = true,
  restart = true,
  reload = true,
  health = true,
  check = true,
  prepare = true,
  migrations = true,
  version = true,
  roar = true
}

for k in pairs(cmds) do
  cmds_arr[#cmds_arr+1] = k
end

table.sort(cmds_arr)

local help = string.format([[
Usage: kong COMMAND [OPTIONS]

The available commands are:
 %s

Options:
%s
]], table.concat(cmds_arr, "\n "), options)

return function(args)
  local cmd_name = table.remove(args, 1)
  if not cmd_name then
    pl_app(help)
    pl_app.quit()
  elseif not cmds[cmd_name] then
    pl_app(help)
    pl_app.quit("No such command: " .. cmd_name)
  end

  local cmd = require("kong.cmd." .. cmd_name)
  local cmd_lapp = cmd.lapp
  local cmd_exec = cmd.execute

  if cmd_lapp then
    cmd_lapp = cmd_lapp .. options -- append universal options
    args = pl_app(cmd_lapp)
  end

  -- check sub-commands
  if cmd.sub_commands then
    local sub_cmd = table.remove(args, 1)
    if not sub_cmd then
      pl_app.quit()
    elseif not cmd.sub_commands[sub_cmd] then
      pl_app.quit("No such command for " .. cmd_name .. ": " .. sub_cmd)
    else
      args.command = sub_cmd
    end
  end

  xpcall(function() cmd_exec(args) end, function(err)
    if not (args.v or args.vv) then
      err = err:match "^.-:.-:.(.*)$"
      io.stderr:write("Error: " .. err .. "\n")
      io.stderr:write("\n  Run with --v (verbose) or --vv (debug) for more details\n")
    else
      local trace = debug.traceback(err, 2)
      io.stderr:write("Error: \n")
      io.stderr:write(trace .. "\n")
    end

    pl_app.quit(nil, true)
  end)
end

```
init脚本我把一些日志去掉了，看的更清晰一些。pl.lapp是一个处理命令行输入的库，[pl.lapp](https://stevedonovan.github.io/Penlight/api/libraries/pl.lapp.html)。这个脚本的主要逻辑就是，先对输入的命令参数做校验，符合要求的话，讲会调用kong.cmd.${cmdname}脚本，那么我们启动时调用的就会是kong.cmd.start脚本。那么我们再看一下start脚本里面写的什么。
```lua
local prefix_handler = require "kong.cmd.utils.prefix_handler"
local nginx_signals = require "kong.cmd.utils.nginx_signals"
local conf_loader = require "kong.conf_loader"
local DAOFactory = require "kong.dao.factory"
local kill = require "kong.cmd.utils.kill"
local log = require "kong.cmd.utils.log"

local function execute(args)
  local conf = assert(conf_loader(args.conf, {
    prefix = args.prefix
  }))   --加载配置文件

  assert(not kill.is_running(conf.nginx_pid),
         "Kong is already running in " .. conf.prefix) --检查nginx是否已经在运行

  local err
  local dao = assert(DAOFactory.new(conf))  --通过数据库配置创建数据库
  xpcall(function()
    assert(prefix_handler.prepare_prefix(conf, args.nginx_conf))

    if args.run_migrations then
      assert(dao:run_migrations())    --检查是否是数据库升级命令
    end

    local ok, err = dao:are_migrations_uptodate()  --检查数据库版本是否升级到最新
    if err then
      -- error correctly formatted by the DAO
      error(err)
    end

    if not ok then
      -- we cannot start, throw a very descriptive error to instruct the user
      error("the current database schema does not match\n"                  ..
            "this version of Kong.\n\nPlease run `kong migrations up` "     ..
            "first to update/initialise the database schema.\nBe aware "    ..
            "that Kong migrations should only run from a single node, and " ..
            "that nodes\nrunning migrations concurrently will conflict "    ..
            "with each other and might corrupt\nyour database schema!")
    end

    ok, err = dao:check_schema_consensus() -- 这个主要针对 cassandra 做检查，psql不做检查
    if err then
      -- error correctly formatted by the DAO
      error(err)
    end

    if not ok then
      error("Cassandra has not reached cluster consensus yet")
    end

    assert(nginx_signals.start(conf))  --通过指定的配置启动nginx

    log("Kong started")
  end, function(e)
    err = e -- cannot throw from this function
  end)

  if err then
    log.verbose("could not start Kong, stopping services")
    pcall(nginx_signals.stop(conf))
    log.verbose("stopped services")
    error(err) -- report to main error handler
  end
end

local lapp = [[
Usage: kong start [OPTIONS]

Start Kong (Nginx and other configured services) in the configured
prefix directory.

Options:
 -c,--conf        (optional string)   configuration file
 -p,--prefix      (optional string)   override prefix directory
 --nginx-conf     (optional string)   custom Nginx configuration template
 --run-migrations (optional boolean)  optionally run migrations on the DB
]]

return {
  lapp = lapp,
  execute = execute
}

```
我们看到这块脚本主要的功能就是先加载配置，然后检查数据库是否更新到最新版本（这个插件更新和相关更新数据库的操作都需要执行run migrations命令，使数据库结构同步到最新），而后检查执行nginx启动命令
```lua
-- nginx_signals.lua
function _M.start(kong_conf)
  local nginx_bin, err = _M.find_nginx_bin()
  if not nginx_bin then
    return nil, err
  end

  if kill.is_running(kong_conf.nginx_pid) then
    return nil, "nginx is already running in " .. kong_conf.prefix
  end

  local cmd = fmt("%s -p %s -c %s", nginx_bin, kong_conf.prefix, "nginx.conf")

  log.debug("starting nginx: %s", cmd)

  local ok, _, _, stderr = pl_utils.executeex(cmd)
  if not ok then
    return nil, stderr
  end

  log.debug("nginx started")

  return true
end

function _M.stop(kong_conf)
  return send_signal(kong_conf, "TERM")
end

function _M.quit(kong_conf, graceful)
  return send_signal(kong_conf, "QUIT")
end

function _M.reload(kong_conf)
  if not kill.is_running(kong_conf.nginx_pid) then
    return nil, "nginx not running in prefix: " .. kong_conf.prefix
  end

  local nginx_bin, err = _M.find_nginx_bin()
  if not nginx_bin then
    return nil, err
  end

  local cmd = fmt("%s -p %s -c %s -s %s",
                  nginx_bin, kong_conf.prefix, "nginx.conf", "reload")

  log.debug("reloading nginx: %s", cmd)

  local ok, _, _, stderr = pl_utils.executeex(cmd)
  if not ok then
    return nil, stderr
  end

  return true
end
```
nginx_signals脚本主要就是找到nginx的执行文件，执行start stop等命令
分析到这里我们可能就有点迷惑了，难道这样就完了，没见到有什么特别的地方呀。别急，kong的主要逻辑都写在ngix.conf文件中呢。上篇文章我们分析了kong会将templates里的文件编译成最终的nginx.conf，那么接下来我们看一下nginx.conf文件变成什么样子了（已经去除了一些代码）。
```
worker_processes auto;
daemon on;

worker_rlimit_nofile 500000;

events {
    worker_connections 16384;
    multi_accept on;
}

http {
    charset UTF-8;

    # used by traffic limit
    lua_shared_dict limit_conn_store 100m;

    # used by redis cluster
    lua_shared_dict redis_cluster_slot_locks 1m;

    lua_socket_log_errors off;

    init_by_lua_block {
        kong = require 'kong'
        kong.init()
     
    }

    init_worker_by_lua_block {
        kong.init_worker()
    }

  
    upstream kong_upstream {
        server 0.0.0.1;
        balancer_by_lua_block {
            kong.balancer()
        }
        keepalive 60;
    }

   
    server {
        server_name kong;
        listen 0.0.0.0:9000;
        error_page 400 404 408 411 412 413 414 417 /kong_error_handler;
        error_page 500 502 503 504 /kong_error_handler;
      
        client_body_buffer_size 60m;

        real_ip_header     X-Real-IP;
        real_ip_recursive  off;

        location / {
          
            default_type text/html;
          
            rewrite_by_lua_block {
                kong.rewrite()
            }

            access_by_lua_block {
              
                kong.access()
            }

            header_filter_by_lua_block {
                kong.header_filter()
            }

            body_filter_by_lua_block {
                kong.body_filter()
            }

            log_by_lua_block {
                kong.log()
            }

        }

        location = /kong_error_handler {
            internal;
            content_by_lua_block {
                kong.handle_error()
            }
        }
    }

    server {
        server_name kong_admin;
        listen 127.0.0.1:9001;

        client_max_body_size 10m;
        client_body_buffer_size 10m;

        listen 127.0.0.1:8444 ssl;
  
        location / {
            default_type application/json;
            content_by_lua_block {
                kong.serve_admin_api()
            }
        }

        location /nginx_status {
            internal;
            access_log off;
            stub_status;
        }

        location /robots.txt {
            return 200 'User-agent: *\nDisallow: /';
        }
    }
}
```
通过分析我们看到，Kong主要是在nginx的生命周期的每个阶段插入了自己的代码，从而起到增到增强nginx的作用。那么nginx的每个请求的生命周期和kong对每个请求处理的生命周期应该是一致的。事实也正是这样的，kong的启动相关的代码主要在kong/init.lua目录中的init方法和init_worker方法
```lua
function Kong.init()
  local pl_path = require "pl.path"
  local conf_loader = require "kong.conf_loader"

  -- retrieve kong_config
  local conf_path = pl_path.join(ngx.config.prefix(), ".kong_env")
  local config = assert(conf_loader(conf_path))

  local dao = assert(DAOFactory.new(config)) -- instantiate long-lived DAO
  local ok, err_t = dao:init()
  if not ok then
    error(tostring(err_t))
  end

  assert(dao:are_migrations_uptodate())

  -- populate singletons
  singletons.ip = ip.init(config)
  singletons.dns = dns(config)
  singletons.loaded_plugins = assert(load_plugins(config, dao))
  singletons.dao = dao
  singletons.configuration = config

  assert(core.build_router(dao, "init"))
end
```
kong的init方法里做的事情很简单，主要是加载配置，检查数据库结构有没有更新到最新，然后加载所有的插件，最后建立路由。
```lua
function Kong.init_worker()

  -- init DAO


  local ok, err = singletons.dao:init_worker()
  if not ok then
    ngx_log(ngx_CRIT, "could not init DB: ", err)
    return
  end


  -- init inter-worker events


  local worker_events = require "resty.worker.events"


  local ok, err = worker_events.configure {
    shm = "kong_process_events", -- defined by "lua_shared_dict"
    timeout = 5,            -- life time of event data in shm
    interval = 1,           -- poll interval (seconds)

    wait_interval = 0.010,  -- wait before retry fetching event data
    wait_max = 0.5,         -- max wait time before discarding event
  }
  if not ok then
    ngx_log(ngx_CRIT, "could not start inter-worker events: ", err)
    return
  end


  -- init cluster_events


  local dao_factory   = singletons.dao
  local configuration = singletons.configuration


  local cluster_events, err = kong_cluster_events.new {
    dao                     = dao_factory,
    poll_interval           = configuration.db_update_frequency,
    poll_offset             = configuration.db_update_propagation,
  }
  if not cluster_events then
    ngx_log(ngx_CRIT, "could not create cluster_events: ", err)
    return
  end


  -- init cache


  local cache, err = kong_cache.new {
    cluster_events    = cluster_events,
    worker_events     = worker_events,
    propagation_delay = configuration.db_update_propagation,
    ttl               = configuration.db_cache_ttl,
    neg_ttl           = configuration.db_cache_ttl,
    resty_lock_opts   = {
      exptime = 10,
      timeout = 5,
    },
  }
  if not cache then
    ngx_log(ngx_CRIT, "could not create kong cache: ", err)
    return
  end

  local ok, err = cache:get("router:version", { ttl = 0 }, function()
    return "init"
  end)
  if not ok then
    ngx_log(ngx_CRIT, "could not set router version in cache: ", err)
    return
  end


  singletons.cache          = cache
  singletons.worker_events  = worker_events
  singletons.cluster_events = cluster_events


  singletons.dao:set_events_handler(worker_events)


  core.init_worker.before()


  -- run plugins init_worker context


  for _, plugin in ipairs(singletons.loaded_plugins) do
    plugin.handler:init_worker()
  end
end
```
因为nginx是单线程的，所以很多事情只能放到定时器里来做，而nginx的定时器只可以放到init_worker阶段里做。init_worker做的主要是初始化数据库相关，然后最重要的是初始化集群管理相关逻辑（因为是采用的轮询数据库的方式，只能放到这里来做），最后调用每个插件的init_worker方法。
然后kong/init.lua还有一些其他方法，和nginx的生命周期相对应，这个我们后面再分析。


