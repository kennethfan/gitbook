---
description: lua脚本批量添加redis key前缀
---

# lua脚本批量添加redis key前缀

脚本

```lua
local prefix = ARGV[1]

redis.call('SELECT', '6')
local keys = redis.call('KEYS', '*')

local lastKey = ''
for i, key in ipairs(keys) do
	if not string.find(key, prefix, 1, true) then
		local newKey = prefix .. key
		lastKey = newKey
		-- 检查 newKey 是否已经存在
		if 0 == redis.call('EXISTS', newKey) then
			redis.call('RENAME', key, newKey)
		end
	end
end


return lastKey
```

redis-cli命令

```bash
redis-cli -h <host> -p <port> -a <password> eval "$(cat rename.lua)" 0 <prefix>
```
