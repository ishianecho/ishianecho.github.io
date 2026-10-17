---
title: "Neovim 原生 API 入门"
summary: "面向 Vim 老用户介绍 Neovim 的 Lua 配置体系与 vim.* 标准库：vim.opt 与 vim.keymap、vim.fs、vim.ui、vim.system、vim.snippet、vim.lsp.config，以及 0.13 的官方插件管理 vim.pack"
date: 2026-10-10
categories: [Tools & Utilities, Editor, Neovim]
tags: [tools, neovim]
---

## 概述

- 参考文档：
  - [Neovim User Documentation](https://neovim.io/doc/user/)
  - [Neovim 0.13 News](https://neovim.io/doc/user/news-0.13.html)
  - [lua-guide](https://neovim.io/doc/user/lua-guide.html)
- Neovim 是 Vim 的分支，内置 Lua 解释器、LSP 客户端、Treesitter 与终端，配置从`vimrc`迁到`init.lua`
- 与 Vim 的关系：**模式与按键语法基本不变**，差异集中在配置语言、默认行为与内建子系统，官方清单在`:h vim-differences`
- 结构：

  ```mermaid
  flowchart LR
    A["init.lua"] --> B["vim.* 标准库"]
    B --> C["vim.lsp(内置 LSP 客户端)"]
    B --> D["vim.treesitter"]
    B --> E["vim.pack(0.13 插件管理)"]
    F["任意 GUI / 终端"] -->|"RPC"| G["Neovim 核心"]
  ```

- 本文假设读者已会 Vim 基本操作，编辑入门见[Vim 入门](/2024/09/07/vim/)，各语言 LSP 的安装见[Neovim LSP 服务器](/2026/08/15/nvim-lsp-server/)

## 快速开始

- 配置入口是`~/.config/nvim/init.lua`，最小可用示例：

  ```lua
  vim.opt.number = true                  -- 行号
  vim.opt.termguicolors = true           -- 24 位真彩色
  vim.g.mapleader = " "                  -- leader 设为空格

  vim.keymap.set("n", "<leader>w", ":w<CR>", { desc = "保存" })

  vim.pack.add({                          -- 0.13 官方插件安装
    { src = "https://github.com/nvim-treesitter/nvim-treesitter" },
    { src = "https://github.com/echasnovski/mini.nvim" },
  })
  ```

- 查手册：`:h lua-guide`入门，之后按模块`:h vim.api`、`:h vim.pack`、`:h vim.ui`
- 试运行 Lua：`:= 表达式`在命令行直接求值，`:lua 语句`执行单句

## Lua 桥接 Vim

- 五个入口对象：
  - `vim.opt`：选项对象，支持链式与列表式赋值，`vim.opt.listchars = { tab = "→ " }`，只要字符串时用`vim.o`
  - `vim.g`/`vim.b`/`vim.w`：全局、缓冲区、窗口变量，对应 VimL 的`g:`、`b:`、`w:`
  - `vim.fn`：调用 VimL 函数，`vim.fn.getcwd()`
  - `vim.cmd`：执行 Ex 命令，`vim.cmd.colorscheme("habamax")`
  - `vim.api.*`：底层 C API，`nvim_create_autocmd`等都在这里，`:h api`
- 键位用`vim.keymap.set`，比`nnoremap`多了`desc`描述与`buffer`局部选项，**which-key 类插件靠`desc`生成提示**
- 异步原语：`vim.defer_fn(fn, ms)`延时执行，`vim.schedule(fn)`在事件循环安全点执行，`vim.uv`是 libuv 的完整绑定
- autocmd 与用户命令：

  ```lua
  vim.api.nvim_create_autocmd("TextYankPost", {
    callback = function() vim.hl.on_yank() end,   -- 0.11 起 vim.hl 取代 vim.highlight
  })

  vim.api.nvim_create_user_command("Trim", function()
    vim.cmd("%s/\\s\\+$//e")                       -- 删行尾空白
  end, {})
  ```

## `vim.fs` 与 `vim.ui`

- `vim.fs`文件系统工具：
  - `vim.fs.find("Cargo.toml", { upward = true })`从当前文件向上找
  - `vim.fs.root(0, { ".git" })`定位项目根，`0`表示当前缓冲区
  - `vim.fs.normalize(path)`跨平台整理路径分隔符
- `vim.ui`交互接口：
  - `vim.ui.input()`与`vim.ui.select()`是异步回调式，插件可整体接管其 UI，这就是 dressing 类插件"只换皮不改逻辑"的原因
  - `vim.ui.open(目标)`用系统默认程序打开 URL 或路径(0.12 起)，等价`xdg-open`但跨平台

## `vim.system` 与 `vim.snippet`

- `vim.system()`(0.10 起)跑子进程，取代手写`io.popen`：

  ```lua
  local out = vim.system({ "rg", "--version" }):wait()     -- 同步等待
  vim.system({ "make" }, { text = true }, function(res)    -- 异步回调
    vim.notify("make 退出码 " .. res.code)
  end)
  ```

- `vim.snippet`(0.10 起)原生 LSP 风格 snippet 引擎：
  - `vim.snippet.expand("fn $1($2) $0")`直接展开，支持 Tabstop、占位符与选择分支
  - 配合内置 LSP 起步不需要第三方 snippet 插件

## LSP 配置

- 0.11 起的写法：`vim.lsp.config`定义，`vim.lsp.enable`启用，取代往`setup{}`里塞整个配置：

  ```lua
  vim.lsp.config("clangd", {
    cmd = { "clangd", "--background-index" },
    root_markers = { "compile_commands.json" },   -- 0.13 起用 root_markers 声明根标记
  })
  vim.lsp.enable("clangd")
  ```

- `LspAttach`事件里做按缓冲区键位：

  ```lua
  vim.api.nvim_create_autocmd("LspAttach", {
    callback = function(args)
      vim.keymap.set("n", "gd", vim.lsp.buf.definition, { buffer = args.buf })
    end,
  })
  ```

- 内置 LSP 客户端不含补全 UI，社区主流是`blink.cmp`与`nvim-cmp`，服务器侧安装见[Neovim LSP 服务器](/2026/08/15/nvim-lsp-server/)

## `vim.pack` 与插件生态

- 0.13 的`vim.pack`是官方(实验性)插件管理：
  - `vim.pack.add({...})`克隆到`stdpath("data")`下并立即加载，条目是`src`(仓库地址，支持完整 URL)与可选的`version`
  - `vim.pack.update()`交互式更新，可传名字列表只更新部分
  - 只有安装与加载，锁文件与复杂延迟加载仍需`lazy.nvim`等第三方方案
- Lua 插件的约定结构：`lua/`目录下的模块加`setup()`入口，用法固定为：

  ```lua
  require("gitsigns").setup({})
  ```

## 从 Vim 迁移

- `init.vim`仍被支持，旧配置可直接沿用，但新代码建议全部写 Lua
- 默认行为差异：Neovim 默认启用鼠标、更好的撤销持久化与`syntax`等，逐项看`:h vim-defaults`
- 必须换的习惯：
  - `set`换`vim.opt`，`let g:`换`vim.g`
  - `nnoremap`换`vim.keymap.set`
  - `vim-plug`换`vim.pack`或`lazy.nvim`
- 多套配置隔离：`NVIM_APPNAME=work nvim`让 Neovim 读`~/.config/work/`下的整套配置(0.9 起)

## 相关阅读

- [Vim 入门](/2024/09/07/vim/)：模式体系与编辑命令
- [Neovim LSP 服务器](/2026/08/15/nvim-lsp-server/)：各语言 LSP 的安装与配置
