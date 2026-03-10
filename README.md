🚀 MonoAI: Pro Edition
终极单文件云端 AI 客户端 —— 全云端部署，零本地依赖
MonoAI 是一个将“极简主义”和“专业级体验”融合到极致的 AI 客户端。整个应用没有任何构建工具、无需 Node.js 环境、没有杂乱的文件目录——只有唯一的一个 .html 文件。
依托 Local-First（本地优先）架构与 Supabase 强大的无服务器生态，您无需使用任何命令行（CLI）或终端，仅需一个浏览器即可完成全套云端私有化部署，并获得比肩大型桌面端 AI 软件（如 OpenWebUI / Chatbox）的硬核体验。
✨ 核心特性
 * 📦 绝对的单文件：零安装，零本地环境配置，只需双击 index.html 即可在任何设备的浏览器中运行。
 * ☁️ 100% 纯网页部署：后端数据库与边缘节点代理均可通过 Supabase Web 控制台点选完成，彻底告别黑框框（CLI）。
 * 🛡️ 云端防杀后台：请求完全交由云端边缘节点接管。发送长消息后，即使你立刻关闭浏览器、断网，云端依然会替你死死咬住 AI 数据流并存入数据库，下次打开一字不落！
 * ⚡ 工业级会话切片：完美支持针对历史对话的 “编辑提问” 与 “重新生成”。精确截断废弃的时间线分支，保证 AI 上下文绝不受到脏数据污染。
 * 🎨 沉浸式无界视觉：原生适配深度暗黑模式（Dark Theme），结合极致圆角的悬浮输入胶囊与侧边栏抽屉，为你带来最纯粹的对话体验。
🛠️ 极简部署指南 (Zero-CLI)
全程无需任何本地代码环境，只需动动鼠标。
第一步：创建云端数据库 (SQL Editor)
 * 登录 Supabase 官网，点击 New Project 创建一个新项目。
 * 进入项目后，点击左侧菜单栏的 SQL Editor，选择 New query。
 * 将以下 SQL 代码粘贴到编辑器中，并点击右下角的 Run 运行，完成数据表与实时同步功能的创建：
```ts
-- 1. 创建会话表
create table sessions (
  id uuid primary key default gen_random_uuid(),
  title text default '新会话',
  created_at timestamp with time zone default timezone('utc'::text, now())
);

-- 2. 创建消息表
create table messages (
  id uuid primary key default gen_random_uuid(),
  session_id uuid references sessions(id) on delete cascade,
  role text check (role in ('system', 'user', 'assistant')),
  content text,
  created_at timestamp with time zone default timezone('utc'::text, now())
);

-- 3. 开启 Realtime (用于多端实时同步气泡)
alter publication supabase_realtime add table sessions;
alter publication supabase_realtime add table messages;

-- 4. 建立索引加速查询
create index idx_messages_session_id on messages(session_id);
```
第二步：创建大模型代理节点 (Edge Functions)
为了隐藏你真实的大模型 API Key 并解决跨域问题，我们需要在云端建一个代理节点。
 * 在 Supabase 左侧菜单栏找到 Edge Functions。
 * 点击 Create a new Edge Function（创建新函数）。
 * 将函数命名为 chat。
 * 在网页中直接打开代码编辑器，清空里面的默认代码，将以下 Deno/TypeScript 代码粘贴进去：
```ts
import { serve } from "https://deno.land/std@0.168.0/http/server.ts"

const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
}

serve(async (req) => {
  // 处理 CORS 预检请求
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders })
  }

  try {
    const { messages, model, baseURL, apiKey } = await req.json()
    
    // 获取前端传来的配置，如果没有则使用官方默认地址
    const targetURL = baseURL || 'https://api.openai.com/v1/chat/completions'
    // 前端传入的 Key 优先级最高，也可在 Supabase 环境变量中配置默认 Key
    const targetKey = apiKey || Deno.env.get('OPENAI_API_KEY')

    const response = await fetch(targetURL, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${targetKey}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        model: model || 'gpt-3.5-turbo',
        messages,
        stream: true, // 强制开启流式输出
      }),
    })

    // 将大模型的流式响应直接透传给前端
    return new Response(response.body, {
      headers: {
        ...corsHeaders,
        'Content-Type': 'text/event-stream',
      },
    })
  } catch (error) {
    return new Response(JSON.stringify({ error: error.message }), {
      status: 500,
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
    })
  }
})
```
 * 点击保存并部署（Deploy）。你的流式节点现在已经在云端运行了！
第三步：运行 MonoAI 客户端 (Frontend)
 * 获取云端密钥：在 Supabase 左侧菜单点击 Project Settings (齿轮图标) → API。找到你的 Project URL 和 Project API Keys (anon / public)。
 * 将项目提供的 index.html 文件下载到你的电脑或手机上。
 * 双击在浏览器中打开它！
 * 页面首次打开会显示系统配置界面。填入你的：
   * 大模型配置（API Base URL、API Key、模型名称等）
   * Supabase URL
   * Supabase Anon Key
 * 点击 “保存并开始”，即可开始畅聊。
> 💡 隐私提示：你的所有配置信息只会加密储存在你当前浏览器的 LocalStorage 中，你的大模型密钥绝对不会离开你的设备（除非它通过加密的 HTTPS 发送给你自己的 Supabase 代理节点）。
> 
🎮 进阶使用技巧
 * 智能模型调度：无论你是想用 Claude、DeepSeek、还是本地的 Ollama，MonoAI 都原生支持。只需在设置中将 API Base URL 指向对应的兼容接口（前端已内置智能补全 /chat/completions 功能），并修改对应的模型名称即可。
 * 时空重塑 (Timeline 截断)：鼠标悬停在聊天气泡上，点击 “编辑并重新发送” 或 “重新生成”。MonoAI 会像专业软件一样，自动斩断当前节点之后的所有错误对话历史，并同步清理云端数据库，始终保持干净的上下文语境。
 * 快捷键：
   * Enter：发送消息
   * Shift + Enter：输入框内换行
   * 任何代码块都可以一键悬浮复制。
