点击lua可查看脚本代码
-- 灵根/天赋自动读取器（纯读取版，含数组调试输出）
-- 进程：WuXiuXianTu.exe
-- 功能：在 GetCharacterDataByUID 返回时，自动读取角色等级、五行灵根和天赋列表

local PROCESS_NAME = "WuXiuXianTu.exe"
local MODULE_NAME = "GameAssembly.dll"
local FUNC_RVA = 0x45B7D0                -- GetCharacterDataByUID
local LEVEL_OFFSET = 0x40                -- 等级
local UID_OFFSET = 0x10                  -- UID字符串指针偏移
local TALENT_LIST_OFFSET = 0xA0          -- talentSkillList
local LINGGEN_OFFSET = 0x58              -- 五行灵根指针偏移

-- 获取进程ID
local p = io.popen('tasklist /FI "IMAGENAME eq ' .. PROCESS_NAME .. '" /FO CSV /NH')
local result = p:read("*a")
p:close()
local pid = result:match('"(%d+)"')
if not pid then print("未找到进程 " .. PROCESS_NAME); return end
pid = tonumber(pid)

debugProcess(pid)
print("已附加到进程 PID=" .. pid)

local moduleBase = getAddress(MODULE_NAME)
if not moduleBase or moduleBase == 0 then print("未找到模块 " .. MODULE_NAME); return end
print(string.format("模块基址: 0x%X", moduleBase))

local funcAddr = moduleBase + FUNC_RVA
print(string.format("GetCharacterDataByUID 地址: 0x%X", funcAddr))

local tempReturnBreakpoint = nil
local lastCharacterPtr = 0   -- 用于去重

-- 安全读取 IL2CPP 字符串
local function readIl2CppString(strPtr)
  if not strPtr or strPtr == 0 then return "null" end
  local len = readInteger(strPtr + 0x10)
  if not len or len <= 0 then return "" end
  local ok, result = pcall(readString, strPtr + 0x18, len, true)  -- true表示宽字符串
  if not ok then
    ok, result = pcall(readString, strPtr + 0x18, len * 2, true)
    if not ok then result = "" end
  end
  return result
end

-- 断点回调
local function onBreakpoint()
  local rip = RIP
  if rip == funcAddr then
    -- 函数入口断点：设置返回断点
    local returnAddr = readPointer(RSP)
    if returnAddr and returnAddr ~= 0 then
      if tempReturnBreakpoint then debug_removeBreakpoint(tempReturnBreakpoint) end
      debug_setBreakpoint(returnAddr)
      tempReturnBreakpoint = returnAddr
    end
  elseif tempReturnBreakpoint and rip == tempReturnBreakpoint then
    -- 函数返回断点：RAX = 角色对象指针
    local characterPtr = RAX
    if characterPtr and characterPtr ~= 0 then
      -- 去重：同一角色对象只处理一次
      if characterPtr == lastCharacterPtr then
        if tempReturnBreakpoint then debug_removeBreakpoint(tempReturnBreakpoint) end
        tempReturnBreakpoint = nil
        return 1
      end
      lastCharacterPtr = characterPtr

      local level = readInteger(characterPtr + LEVEL_OFFSET)
      local uidPtr = readPointer(characterPtr + UID_OFFSET)
      local uid = readIl2CppString(uidPtr)
      print(string.format("角色对象: 0x%X, UID=%s, 等级=%d", characterPtr, uid, level))

      -- ========== 读取五行灵根 ==========
      local lingGenPtr = readPointer(characterPtr + LINGGEN_OFFSET)
      if lingGenPtr and lingGenPtr ~= 0 then
        local gold = readInteger(lingGenPtr + 0x20)   -- 金
        local wood = readInteger(lingGenPtr + 0x24)   -- 木
        local water = readInteger(lingGenPtr + 0x28)  -- 水
        local fire = readInteger(lingGenPtr + 0x2C)   -- 火
        local earth = readInteger(lingGenPtr + 0x30)  -- 土
        print(string.format("基础五行灵根: 金=%d, 木=%d, 水=%d, 火=%d, 土=%d (灵根结构=0x%X)",
          gold, wood, water, fire, earth, lingGenPtr))
      else
        print("五行灵根: 未找到灵根结构")
      end
      -- ====================================

      -- 读取天赋列表
      local talentListPtr = readPointer(characterPtr + TALENT_LIST_OFFSET)
      if talentListPtr and talentListPtr ~= 0 then
        local arrayPtr = readPointer(talentListPtr + 0x10)
        local size = readInteger(talentListPtr + 0x18)
        print(string.format("天赋技能列表 (共 %d 个):", size))

        if arrayPtr and size > 0 then
          -- 打印数组对象信息，方便验证偏移
          local arrayMaxLength = readInteger(arrayPtr + 0x18)
          print(string.format("  [调试] 数组对象: 0x%X, 数组长度字段(+0x18)=%d", arrayPtr, arrayMaxLength))

          for i = 0, size - 1 do
            local skillIdPtr = readPointer(arrayPtr + 0x20 + i * 8)
            if skillIdPtr and skillIdPtr ~= 0 then
              local skillId = readIl2CppString(skillIdPtr)
              print(string.format("  [%d] %s (ptr=0x%X)", i, skillId, skillIdPtr))
            else
              print(string.format("  [%d] null", i))
            end
          end
        else
          print("  列表为空")
        end
      else
        print("天赋列表指针为空")
      end
    else
      print("返回对象为空")
    end

    -- 清理临时返回断点
    if tempReturnBreakpoint then
      debug_removeBreakpoint(tempReturnBreakpoint)
      tempReturnBreakpoint = nil
    end
  end
  return 1
end

debug_setBreakpoint(funcAddr)
debugger_onBreakpoint = onBreakpoint

print("脚本已启动，请切换角色。")
print("停止脚本：debug_removeBreakpoint(0x" .. string.format("%X", funcAddr) .. ")")
