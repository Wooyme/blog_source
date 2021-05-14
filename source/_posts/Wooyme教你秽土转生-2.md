---
title: Wooyme教你秽土转生(2)
date: 2020-03-16 19:50:48
tags:
     - Hacker
---

# 前言
上一篇讲了如何秽土转生，但是有个问题，Memcached在电信下效果极差。所以在经过不少搜索之后我看中了另一个缓存数据库——**Redis**。选择Redis就会出现另一个问题，Redis的安全配置并不像Memcached那么弱智。全球虽然也有非常多的Redis服务器，但是大部分还是需要验证或是没有anonymous的写入权限。

所以这就需要我们找出那些真正可用的Redis服务器。

# ZoomEye
我在前篇也提到了**ZoomEye**，根据ZoomEye的数据显示，全球有超过50万台Memcached服务器。那么什么是ZoomEye呢？ ZoomEye是一家网络安全的搜索引擎，通过后台运行的庞大扫描程序扫描整个互联网，并记录各个IP开放的各种端口。
> https://www.zoomeye.org/

搜索Redis比较好用的关键字是
> cluster +app:redis +subdivisions:"香港"

后面两个分别是限制应用和地区，第一个`cluster`算是个小技巧。因为ZoomEye的扫描器在连接到Redis后会运行info命令，如果Redis没有权限验证，那么在返回的结果里就会有`cluster`。所以在搜索的时候加上`cluster`可以让ZoomEye只返回没有权限验证的结果。

## cli
网页搜索不好整理结果，所以有大牛做了cli的版本
> https://github.com/gelim/zoomeye
> 
> python zoomeye.py -l 记录数 --user 用户名 --password 密码 "cluster +app:redis +subdivisions:香港"

# Nmap
在拿到了无需登录即可执行命令的Redis服务器列表之后还需要判断服务器是否有匿名读写的权限。这个时候就要看Nmap了。

## Nmap Script
我写了个Nmap的脚本用于检测是否可以写
```lua
local brute = require "brute"
local creds = require "creds"
local redis = require "redis"
local shortport = require "shortport"
local stdnse = require "stdnse"

description = [[
Performs brute force passwords auditing against a Redis key-value store.
]]

---
-- @usage
-- nmap -p 6379 <ip> --script redis-brute
--
-- @output
-- PORT     STATE SERVICE
-- 6379/tcp open  unknown
-- | redis-brute:
-- |   Accounts
-- |     toledo - Valid credentials
-- |   Statistics
-- |_    Performed 5000 guesses in 3 seconds, average tps: 1666
--
--

author = "Wooyme"
license = "Same as Nmap--See https://nmap.org/book/man-legal.html"
categories = {"intrusive", "brute"}


portrule = shortport.port_or_service(6379, "redis")

local function checkRedis(host, port)

  local helper = redis.Helper:new(host, port)
  local status = helper:connect()
  if( not(status) ) then
    return false, "Failed to connect to server"
  end

  local status, response = helper:reqCmd("SET", "thisisaredistest","thisisaredistest")
  if ( not(status) ) then
    return false, "Failed to request SET command"
  end

  if ( redis.Response.Type.ERROR == response.type ) then
    if ( "-ERR operation not permitted" == response.data ) or
        ( "-NOAUTH Authentication required." == response.data) then
      return false, "Need Authentication"
    end
  end
  return true, host
end

action = function(host, port)
  local status, msg =  checkRedis(host, port)
  return msg
end
```
脚本是用redis-brute改的。

> nmap -Pn -sT -p 6379 -iL iplist.txt --script=redis-brute.nse | grep -oE "ip: \b([0-9]{1,3}\.){3}[0-9]{1,3}\b" | grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b"

使用这样的命令就可以整理出最后的可用ip列表。

扫描了一下香港400个无需权限的结果里有200个左右的匿名可写。

# 总结
其实世界还上有不需要鉴权还能匿名读写的Redis服务器我觉得也是挺夸张的了，而且不少是IDC的，无语。