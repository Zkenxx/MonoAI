🚀 **MonoAI: Pro Edition**  
**终极单文件云端 AI 客户端**

MonoAI 是一个将“极简主义”和“专业级体验”融合到极致的 AI 客户端。整个应用没有任何构建工具、无需 Node.js 环境、没有杂乱的文件目录——**只有唯一的一个 .html 文件**。

但在极简的表象下，它依托 Local-First（本地优先）架构设计与 Supabase Edge Functions（边缘无服务器函数），实现了比肩大型桌面端 AI 软件（如 OpenWebUI / Chatbox）的硬核交互体验。

### ✨ 核心特性
- 📦 **绝对的单文件**：零安装，零环境配置，只需双击 index.html 即可在任何设备的浏览器中运行。
- 🛡️ **云端防杀后台**：请求完全交由云端边缘节点接管。发送消息后，即使你立刻关闭浏览器断网，云端依然会替你死死咬住 AI 数据流并存入数据库，下次打开一字不落！
- ⚡ **工业级会话切片**：完美支持针对历史对话的** “编辑提问”与“重新生成” **。精确截断废弃的时间线分支，保证 AI 上下文绝不受到脏数据污染。
- 🌊 **极致平滑的 UI**：前端自带本地 UUID 生成与毫秒级时间戳管理。搭配 Supabase Realtime，多端同步时气泡零闪烁、零错位、零重复。
- 🎨 **OpenWebUI 沉浸式视觉**：原生适配深度暗黑模式（Dark Theme），极致圆角的悬浮输入胶囊，无界流式排版，专注内容本身。
- 🔒 **零硬编码极度安全**：代码中不包含任何私人密钥。所有 API Key 和数据库密码均在首次启动时通过网页 UI 动态输入，真正做到代码与私有资产物理级隔离。

### 🛠️ 快速部署指南 (3步搞定，完全免费)
即使你完全不懂代码，只需跟着以下 3 步，即可在 5 分钟内拥有属于你自己的云端同步 AI。

**第一步：准备 Supabase 数据库 (数据云端托管)**
1. 注册并登录 Supabase 控制台。
2. 点击 New Project 创建一个新项目。
3. 进入项目后，点击左侧菜单的 **SQL Editor**，新建一个 Query，复制并运行以下 SQL 代码来创建数据表：

```sql
-- 1. 创建会话表
create table sessions (
    id uuid default gen_random_uuid() primary key,
    title text default '新对话',
    created_at timestamp with time zone default timezone('utc'::text, now()) not null,
    updated_at timestamp with time zone default timezone('utc'::text, now()) not null
);

-- 2. 创建消息表
create table messages (
    id uuid primary key,
    session_id uuid references sessions(id) on delete cascade,
    role text not null,
    content text,
    status text default 'completed',
    created_at timestamp with time zone not null
);

-- 3. 开启数据表的“实时广播”功能 (核心！)
alter publication supabase_realtime add table messages;
```

**第二步：部署云端边缘函数 (防断网生成核心)**  
无需任何本地命令行操作，直接在网页完成！

1. 在 Supabase 左侧菜单找到 **Edge Functions**。
2. 点击 **Create a new Edge Function**。
3. 将函数命名为 **chat**（名称必须是这个）。
4. 在网页出现的代码编辑器中，清空默认代码，将 MonoAI 后端 index.ts 的完整代码复制粘贴进去（代码见下文）。
5. 点击编辑器右上角的 **Deploy** 保存并部署即可。

**后端 Edge Function 代码（index.ts）**：

```ts
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
  'Access-Control-Allow-Methods': 'POST, OPTIONS',
}

Deno.serve(async (req) => {
  if (req.method === 'OPTIONS') return new Response('ok', { headers: corsHeaders })

  try {
    const { sessionId, aiMessageId, llmConfig, supabaseConfig } = await req.json()
    const supabase = createClient(supabaseConfig.url, supabaseConfig.key)

    const processStream = async () => {
      try {
        const { data: history } = await supabase.from('messages')
          .select('role, content').eq('session_id', sessionId).order('created_at', { ascending: true })
        
        let messages = (history || []).filter(m => m.content?.trim() !== '')
        
        if (llmConfig.systemPrompt?.trim()) {
            messages = [{ role: 'system', content: llmConfig.systemPrompt }, ...messages]
        }

        const response = await fetch(`${llmConfig.baseUrl || 'https://api.openai.com/v1'}/chat/completions`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${llmConfig.apiKey}` },
          body: JSON.stringify({ 
            model: llmConfig.model || 'gpt-3.5-turbo', 
            messages, stream: true, temperature: 0.7 
          })
        })

        if (!response.ok) throw new Error(`API Error: ${response.status}`)

        const reader = response.body?.getReader()
        const decoder = new TextDecoder()
        let fullContent = "", buffer = "", lastUpdate = Date.now(), checkStop = Date.now()

        while (reader) {
          if (Date.now() - checkStop > 1500) {
              const { data: check } = await supabase.from('messages').select('status').eq('id', aiMessageId).single()
              if (check?.status === 'stopped') return 
              checkStop = Date.now()
          }

          const { done, value } = await reader.read()
          if (done) break
          
          buffer += decoder.decode(value, { stream: true })
          const lines = buffer.split('\n'); buffer = lines.pop() || ''
          
          for (const line of lines) {
            if (!line.trim() || line.includes('[DONE]')) continue
            if (line.startsWith('data: ')) {
              try {
                const delta = JSON.parse(line.slice(6)).choices[0].delta?.content
                if (delta) {
                  fullContent += delta
                  if (Date.now() - lastUpdate > 300) {
                    await supabase.from('messages').update({ content: fullContent }).eq('id', aiMessageId)
                    lastUpdate = Date.now()
                  }
                }
              } catch (e) {}
            }
          }
        }
        await supabase.from('messages').update({ content: fullContent, status: 'completed' }).eq('id', aiMessageId)
      } catch (err: any) {
        await supabase.from('messages').update({ content: `**Error:** ${err.message}`, status: 'error' }).eq('id', aiMessageId)
      }
    }

    if (typeof EdgeRuntime !== 'undefined') EdgeRuntime.waitUntil(processStream())
    else processStream()

    return new Response(JSON.stringify({ success: true }), { headers: corsHeaders, status: 200 })
  } catch (error: any) {
    return new Response(JSON.stringify({ error: error.message }), { headers: corsHeaders, status: 400 })
  }
})
```

**第三步：运行前端 (index.html)**
1. 获取你的 Supabase 凭证：在 Supabase 左侧菜单点击 **Project Settings** (齿轮图标) → **API**。找到你的 **Project URL** 和 **Project API Keys (anon / public)**。
2. 将项目提供的 index.html 文件下载到你的电脑、手机或是发送到微信文件传输助手。
3. 双击在浏览器中打开它！
4. 页面首次打开会显示极简的初始化界面。输入你的 Supabase URL、Anon Key 以及你常用的大模型 API Key，点击保存即可开始畅聊。

> 💡 **提示**：配置信息会安全地储存在你当前浏览器的 LocalStorage 中，你的密钥绝对不会离开你的设备。

### 🎨 进阶使用技巧
- **切换大模型**：MonoAI 支持任何兼容 OpenAI 格式的 API。想用 Claude 或 DeepSeek？只需在 设置 中将 API Base URL 指向对应的中转地址，并将 Model 修改为对应模型名称即可。
- **时空重塑 (编辑/重生成)**：鼠标悬停在气泡上，点击“编辑”或“重新生成”，AI 将自动截断废弃的历史分支，并重新为你生成一段全新的未来。
- **人设注入 (System Prompt)**：在设置中填写“系统提示词”（例如：你是一名暴躁的天才程序员），它会在底层静默生效，每次新建对话时自动赋予 AI 灵魂。

### 🙋‍♂️ FAQ (常见问题)
**Q：为什么我发送消息后，AI 一直在闪烁打字机动画却不出字？**  
A：请检查你的大模型 API Key 和 Base URL 是否正确。如果不正确，Edge Function 连不上 OpenAI 时，气泡下方会弹出红色的 Error 报错提示。

**Q：刷新页面后，所有的历史对话怎么不见了？**  
A：请确保第一步执行 SQL 建表时，代码完全一致。如果是后来手动创建的表，必须保证 sessions 表和 messages 表都有 created_at 字段，否则前端无法按时间顺序提取历史记录。

**Q：多端同步会有延迟吗？**  
A：延迟极低！利用 Supabase 的 WebSocket 长连接，云端每生成出几个字，网页端就会在毫秒级接收并渲染，视觉上的打字体验与本地直接请求几乎完全相同。
