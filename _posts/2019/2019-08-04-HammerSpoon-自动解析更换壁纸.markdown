---
layout: article
key: e2de4e1e-ba2c-43a0-8afb-f666204151e6
title: "HammerSpoon-自动解析更换壁纸"
date: 2019-08-04 14:54:38 +0800
categories: tool
tags: tool
---

# [HammerSpoon](http://www.hammerspoon.org/) 介绍



# 自动更换壁纸

## 壁纸来源网站

`https://wallhaven.cc`

## 实现方式

通过 `curl` 命令获取并解析壁纸来源网站中的图片的 url ，因为该网站展示的图片是压缩后的，只有
点击图片后才显示高清图片，此处可以通过 sed 替换 url 中的部分字段，获取高清图片的 url。

通过随机page和随机选取页面上的图片来实现壁纸的随机。

# 源码

保存名为 `init.lua`

```lua
local obj={}
obj.__index = obj

-- Metadata
obj.name = "ScreenImage"
obj.version = "1.0"
obj.author = "surreal <shu_wenjun@qq.com>"
-- 壁纸来源网站
obj.targetUrl = "https://wallhaven.cc"

obj.localPathPrefix = os.getenv("HOME") .. "/.Trash/wallpaper/"

local function mysplit(inputstr, sep)
    if sep == nil then
            sep = "%s"
    end
    local t={}
    for str in string.gmatch(inputstr, "([^"..sep.."]+)") do
            table.insert(t, str)
    end
    return t
end

local function curl_callback(exitCode, stdOut, stdErr)
    if exitCode == 0 then
        obj.task = nil
        obj.last_pic_name = obj.pic_name
        local localpath = obj.localPathPrefix .. obj.pic_name
        print("localpath: " .. localpath)
        hs.screen.mainScreen():desktopImageURL("file://" .. localpath)
    else
        print(stdOut, stdErr)
    end
end

local function imageRequest()
    local PIC_URL_REGEX = [[ grep -o 'src="[^"]\+\.jpg"' | grep -o 'https.*\.jpg' | sed 's/https:\/\/th\./https:\/\/w\./;s/\/small\//\/full\//;s/\.cc\/lg\//\.cc\/full\//;s/\([^/]*\.jpg\)/wallhaven-\1/g' ]]
    local PAGE_EXPRESSION = [[/random?page=]]
    -- 可以使用解析的方式获得really totalpage
    local TOTAL_PAGE = 10000
    local initTargeImageUrl = 'https://wallhaven.cc'
    local PIC_URL_PREFIX = ""
    local PIC_URL_SUFFIX = ""
    local PIC_NAME_SUFFIX = ""

    local user_agent_str = "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_5) AppleWebKit/603.2.4 (KHTML, like Gecko) Version/10.1.1 Safari/603.2.4"
    
    math.randomseed(os.time())
    local randomPage = math.random(TOTAL_PAGE)
    print("page: ", randomPage)
    local targeImageUrl = initTargeImageUrl .. PAGE_EXPRESSION .. randomPage
    print("targeImageUrl: ", targeImageUrl)
    local allPhotoUrls = hs.execute([[ /usr/bin/curl ]] .. [[']] .. targeImageUrl .. [[']] .. " | " .. PIC_URL_REGEX)
    local allPhotoUrlsTable = mysplit(allPhotoUrls, "\n")
    for k,v in pairs(allPhotoUrlsTable) do
        print("allPhotoUrlsTable: ", k, v)
    end
    -- picture 下载地址
    obj.pic_url = PIC_URL_PREFIX .. allPhotoUrlsTable[math.random(#allPhotoUrlsTable)] .. PIC_URL_SUFFIX
    -- picture name ,同时不带路径分割修饰符
    obj.pic_name = hs.http.urlParts(obj.pic_url).lastPathComponent .. PIC_NAME_SUFFIX
    print("pic_url: " .. obj.pic_url)
    print("pic_name: " .. obj.pic_name)

    if obj.last_pic_name ~= hs.http.urlParts(obj.pic_name).lastPathComponent then
        if obj.task then
            obj.task:terminate()
            obj.task = nil
        end
        hs.fs.rmdir(obj.localPathPrefix)
        hs.fs.mkdir(obj.localPathPrefix)
        local localpath = obj.localPathPrefix .. obj.pic_name
        obj.task = hs.task.new("/usr/bin/curl", curl_callback, {"-A", user_agent_str, obj.pic_url, "-o", localpath})

        print("task: " .. [[ /usr/bin/curl -A "]] .. user_agent_str .. [[" "]] .. obj.pic_url .. [[" -o ]] .. localpath)
        obj.task:start()
    end
end

function obj:init()
    if obj.timer == nil then
        obj.timer = hs.timer.doEvery(60 * 5, function() imageRequest() end)
        obj.timer:setNextTrigger(5)
    else
        obj.timer:start()
    end
end

obj:init()
return obj
```

