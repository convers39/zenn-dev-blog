---
title: "AstroNvimをセットアップしてみた"
emoji: "⌨️"
type: "tech"
topics:
  - "vim"
  - "neovim"
published: true
published_at: "2022-09-25 16:48"
---


# TL; DR
![](https://storage.googleapis.com/zenn-user-upload/03d7b6ed9ecd-20220925.png)
[AstroNvim](https://astronvim.github.io/)
[ Config repo ](https://github.com/convers39/astro-nvim-config)

**23/08/27追記**
自分のAstroNvimセットアップがかなり重くなったりして、最近はまた[NvChad](https://nvchad.com/)に切り替えています。似ているようなセットアップを更新中です。blazing fastで気持ち良いです。
（Astroと比べてデフォルトでついているプラグインや設定が少なめなので全く同じものを作るには多少労力がかかるかもしれません。が、これを機に実際にあまりいらないプラグインを減らしていくのも良いかなと。）

[NvChad config](https://github.com/convers39/nvchad-config)

**23/03/14追記**
全体像がわかるように[こちらの動画](https://youtu.be/stqUbv-5u2s)がおすすめです。どれかのセットアップに限らず、全部共通する内容なので、既にvim経験があって本格的にnvimの~~IDE~~PDE（Personal Development Environment）を作るには結構良いイントロダクションだと思います。関連するレポは[こちら](https://github.com/nvim-lua/kickstart.nvim)です。

**23/03/12追記**
[3.0リリース](https://github.com/AstroNvim/AstroNvim/releases/tag/v3.0.0)されています。パッケージ管理のPackerがLazyに入れ替えられているほか、この記事の内容と合わない挙動・仕様の可能性があるため、ver2からのアップデートはしばらく待っても良い（以前筆者がリクエストした[ロールバック機能](https://github.com/AstroNvim/AstroNvim/blob/main/lua/astronvim/utils/updater.lua#L118)が実装されましたのでトライするコストがだいぶ下がると思います）。

# 経緯

最近若干ハマっています。

NeoVimをIDE感覚で使ってみようと前々から思っていて、上記のAstroNvimはやく半年ぐらい使っていました。最近は別のセットアップNvChadを見つけて、試してみようと思って色々と弄ってみましたが、結局AstroNvimに戻りました。

が、それで面白くないので、NvChadを試している間に比較しながら、他の方のコンフィグを参考して、自分のものもだいぶ更新しました。

まだまだ模索中の部分は多々ありますが、現段階のものをシェアしようと思います。

## VSCodeで良くない？

筆者自身もVSCodeのファンです。VSCodeでvimエクステンションを使って、色々とキーマッピング変更したりして使っています。これからのプラグイン、設定変更なども、時々VSCodeみたいにxxxしたいとの理由があります。また、マージ時の競合解消とか、デバッグの時とか、liveshareの時とか、remote containerの時とか、やはりVSCodeは捨てられません。だったらVSCodeで良くないか？

全然良いと思いますが、自分にとってVimには言葉にできないロマンがあります（笑）。開発者がエディターに慣れるのではなく、カスタマイズしていくことによって、エディターが開発者になれるようになります。Vimを使うというのは、継続的コミットメントが求められています。**自分にしか持てない剣を鍛えていく**(IDE -> PDE aka. Personal Development Environment by [TJ DeVries](https://github.com/tjdevries)氏)、このプロセスが開発者にとって非常に価値のあることだと信じています。

## NeoVim開発上最低限に必要なもの

AstroNvim, NvChad, LunaVim, SpaceVimなどいろんなセットアップを試したことがあります。これらはすでに最低限の内容を含めてくれているので、いずれかを起点にするのは良いでしょう。0からやるのは可能ですが、若干ハードルが高いので熟練な方以外にお勧めしません。

最低限に必要なプラグインについて、大体以下となります。

- **プラグイン管理ツール**　vim-plugとか、packerとか。これがないと結構不便なので必須です。
- **lsp(language server protocol)周り** これがあるからこそVSCodeとかのように、フォーマットを直したり、リントしたりすることが可能になります。null-ls, nvim-lspconfig, masonとかがセットになっています。
- **言語解析とハイライト** これがないとテキストが全部プレーンテキストのように黒白なので必須です。treesitter一択かも。ちなみにneovim 0.8から内蔵されるらしいです。
- **入力サジェスト** 無論必須レベルですね。昔はcoc使っていたのですが、パフォーマンス的にcmpが多く採用されると思われます。cmpでカスタムsnippet、tabnineなどのソースを導入することが可能です。
- **ファイルブラウザー** 横にフォルダー構造を表示するやつ。neo-tree, nvim-treeとかが代表的（ただtelescopeを使うならエクステンションのfile_browserでもあり）。
- **ステータスバー**　現在のブランチ、言語、問題の数、vimのモードなどなどいろんな情報を表示します。個人的に必須です。lualine, felineなどがあります。
- **バッファーバー**　ステータスバーは通常下に位置しますが、バッファーを管理・表示するプラグインで上に現在開いているファイルを表示します。bufferline, tablineなどがあります。
- **コマンドパレット的なもの**　VSCodeでcmd+Pで呼び出せるものですが色々と便利ですね。telescopeでもちろんできますが、コマンドパレットを遥かに超える領域にいるので、nvimを使いこなす上で必須です。
- **セッション管理**　セッション管理のプラグインで前に開いたファイルを保存してくれます。ないと若干困るやつですね。auto-session, session managerとかがよく使われます。
- **ターミナル**　簡易にターミナルを出したり隠したりするのが良いでしょう。toggletermとか。
- **デバッガー**　使うならvimspectorかnvim-dapかです。vimspectorはvim scriptで書いたもので昔からあるやつです。後者はnvimのために作られていますが、3つのプラグインがセットになっています。今のところまだ使っていません（**更新：2.9からdap機能内蔵されている**）。

（なんか漏れたりしたらご指摘お願いします。。）

で、AstroNvimには上記のデバッガー以外全部デフォルトプラグインリストに入れています。この辺りは[公式の紹介](https://astronvim.github.io/acknowledgements#-plugins-used-in-astronvim)を見れば早いでしょう。

上記はコアとなる構成パーツなので、カスタマイズしていく中でも基本的に最優先にすべきではないかと思います。その上でさらに自分に合うスピード上げ術を探れば良いかと。

# カスタマイズ

AstroNvimをベースに工夫した内容です。いずれも、「xxxしたいからxxxをした」とのロジックになります。すべてのファイルは`.config/nvim/lua/user/`フォルダーにあります。

## コア機能

### lspのサーバーを自動インストールしたい

**注意：最新のv2以上では、mason-tool-installerがデフォルトプラグインから削除されているため、プラグインリストに追加する必要がある(公式は[こちら](https://astronvim.github.io/Configuration/v2_migration)、設定は[こちら](https://github.com/convers39/astro-nvim-config/commit/6351775da1e86397d4fc25af9226e7889d965aae#diff-fe1ff5a8e98a1f60c3b662cae823154fbd9992084f949998a628bf51479ecbc8R69))**

lspのソースには4種類に分けることができます（[リストはこちら](https://github.com/jose-elias-alvarez/null-ls.nvim/blob/main/doc/BUILTINS.md)）。Masonを使うことで管理がしやすくなります。

![](https://storage.googleapis.com/zenn-user-upload/e5bf07837c41-20220925.png)

AstroNvimのデフォルトキーはspace+l+Iになりますが、`:Mason`のコマンドを実行すると同じことなのでremapは好きのように。

`mappings.lua`ファイルではカスタムキーマップのテーブルをリターンしています。構造としては以下となります。

```lua
-- mappings.lua
local M = {n={}, i={}, v={}, t={}}
M.n['custom key map'] = {'command', desc = 'xxx'}
return M
```

n, i, v, tはそれぞれのモードと対応してます。tはターミナルモードのことです。ターミナルを開いていて、カーサーがターミナルにある状態だとターミナルモードだと判断されます。

マッピングのカスタマイズはここまでとして、Masonを使うことでインストールとかが簡単になりますが、例えば違うマシンに移るときに毎回再度インストールしなければなりません（個人用と仕事用とか）。となれば、やはり自動インストールして欲しいですね。

ここで2つのファイルを追加します。

```lua
-- plugins/mason-lspconfig.lua
return {
  automatic_installation = true,
}

-- plugins/mason-tool-installer.lua
return {
  ensure_installed = {
    -- Lsp
    "pyright",
    "lua-language-server",
    "typescript-language-server",
    "rust-analyzer",
    "vim-language-server",
    "html-lsp",
    "css-lsp",
    "json-lsp",
    "emmet-ls",

    -- Formatter
    "prettierd",
    "stylua",
    "black",
    "yamlfmt",

    -- Linter
    "mypy",
    "hadolint",
    "eslint_d",

    -- Diagnostics
    "cspell",

    -- Dap
    -- "debugpy",
  },
}
```

これでnvim自体を再インストールするときでも、自動で指定のlsをインストールしてくれます。地味に助かります。

lspと似ていますが、`:TSInstall xxx`でtreesitterのシンタクスハイライト機能を言語別でインストールすることが可能です。これも自動でやってもらうことができます。

```lua
-- plugins/treesitter.lua
return {
  ensure_installed = {
    "bash",
    "css",
    "dockerfile",
    "html",
    "javascript",
    "json",
    "lua",
    "markdown",
    "python",
    "scss",
    "toml",
    "tsx",
    "typescript",
    "vim",
    "yaml",
    "rust",
  },
}
```

### tabnine使いたい

copilotの件は残念でしたが、tabnineがまだ使えるので、vscodeではtabnineに戻しました。それで、nvimにももちろん使えます。

```lua
-- plugins/init.lua
  ["tzachar/cmp-tabnine"] = {
    requires = "hrsh7th/nvim-cmp",
    after = "nvim-cmp",
    run = "./install.sh",
    config = function()
      require("cmp_tabnine.config").setup {
        max_lines = 1000,
        max_num_results = 20,
        sort = true,
        run_on_every_keystroke = true,
        snippet_placeholder = "..",
        ignored_file_types = {
          -- lua = true
        },
        show_prediction_strength = false,
      }
      require("core.utils").add_cmp_source { name = "cmp_tabnine", priority = 1000, max_item_count = 7 }
    end,
  },
-- cmp.lua
return {
  source_priority = {
    cmp_tabnine = 1000,
    nvim_lsp = 900,
    nvim_lua = 800,
    luasnip = 700,
    buffer = 500,
    path = 250,
  }
}
```

tabnineの優先順位を調整可能ですが、一応最優先にしていますので、候補の一番上に来ます。

### snippets使いたい

自炊のものと、既存のものの選択肢があります。既存のものであれば、次のプラグインがお勧めです。

```lua
-- plugins/init.lua
  ["rafamadriz/friendly-snippets"] = { event = { nil } },
```

自炊であれば、例えば`snippets`フォルダーを作っておき、次のファイルでパスを指定します。

```lua
-- luasnip.lua
return {
  vscode_snippet_paths = { "./lua/user/snippets" },
}
```

それで自炊の場合は`package.json`ファイルで、どのsnippetをどのタイプのファイルに使うかを指定する必要があります。詳細設定は冒頭のレポに参照してください。

```json
{
  "name": "user snippets",
  "engines": {
    "vscode": "^1.11.0"
  },
  "contributes": {
    "snippets": [
      {
        "language": "typescriptreact",
        "path": "./react.json"
      },
      {
        "language": "typescriptreact",
        "path": "./html.json"
      },
      {
        "language": "javascriptreact",
        "path": "./react.json"
      },
      {
        "language": "javascriptreact",
        "path": "./html.json"
      },
      {
        "language": "html",
        "path": "./html.json"
      },
      {
        "language": "typescript",
        "path": "./javascript.json"
      },
      {
        "language": "javascript",
        "path": "./javascript.json"
      }
    ]
  }
}
```

![](https://storage.googleapis.com/zenn-user-upload/5f8e4d6f9a71-20220925.gif)

### ファイル保存するときに自動でフォーマット直したい

**注意：最新バージョンAstroNvim2.0以上にアップデートするとこちらの設定はデフォルトとなるため、追加は不要**

AstroNvimではデフォルト設定になっていますが、一応メモとして。

```lua
-- plugins/null-ls.lua
return function(config)
  -- ...
  -- NOTE: You can remove this on attach function to disable format on save
  config.on_attach = function(client)
    if client.resolved_capabilities.document_formatting then
      vim.api.nvim_create_autocmd("BufWritePre", {
        desc = "Auto format before save",
        pattern = "<buffer>",
        callback = vim.lsp.buf.formatting_sync,
      })
    end
  end
  return config
end
```

フック関数を定義してファイル保存（buffer write）する前にフォーマッターを実行することです。逆に不要な場合（huskyとかコミット時にやるとか）はコメントアウトすると良いでしょう。

### 画面スプリット、移動、リサイズをシンプルにやりたい

AstroNvimのデフォルトの画面移動では、`ctrl+h/j/k/l`となっています。このまま運用します。画面分割は同じく`ctrl`を利用して、`ctrl+s/v`にしましたが、visual blockモード、保存のキーと競合するのでやはり`option`に変えました。それで、画面リサイズは`option+h/j/k/l`に。

```lua
-- mappings.lua
return {
  n = {
    -- resize
    ["<A-l>"] = { ":vertical resize +2<CR>" },
    ["<A-h>"] = { ":vertical resize -2<CR>" },
    ["<A-j>"] = { ":resize -2<CR>" },
    ["<A-k>"] = { ":resize +2<CR>" },
    ["<A-=>"] = { "<C-w>=", desc = "Resize equal" },
    -- split
    ["<A-v>"] = { "<C-w>v", desc = "Split window vertically" },
    ["<A-s>"] = { "<C-w>s", desc = "Split window horizontally" },
  }
}
```

### ターミナルを簡単にオンオフしたい

デフォルトでは`toggleterm`を使っていて、キーはspace+t+f/v/hになっていますが、3回も押すのが正直不便です。。

![](https://storage.googleapis.com/zenn-user-upload/0efad88d95c8-20220925.png)

ここはnvchadを参考にして、メタキー（option）+ iでフロートターミナルのオンオフにしています。また、option+h/j/k/lは画面サイズ調整に振っているので、水平と垂直のターミナルのオンオフはoption+V, option+Hにしています。

```lua
-- mappings.lua
return {
  n = {
    -- terminal
    ["<A-i>"] = { "<cmd>ToggleTerm direction=float<cr>", desc = "Toggle floating terminal" },
    ["<A-H>"] = { "<cmd>ToggleTerm size=10 direction=horizontal<cr>", desc = "Toggle horizontal terminal" },
    ["<A-V>"] = { "<cmd>ToggleTerm size=80 direction=vertical<cr>", desc = "Toggle vertical terminal" },
  },  
  t = {
    ["<A-i>"] = { "<cmd>ToggleTerm direction=float<cr>", desc = "toggle floating terminal" },
    ["<A-H>"] = { "<cmd>ToggleTerm size=10 direction=horizontal<cr>", desc = "toggle horizontal terminal" },
    ["<A-V>"] = { "<cmd>ToggleTerm size=80 direction=vertical<cr>", desc = "toggle vertical terminal" },
  },
}
```

ちなみに`ctrl+h/j/k/l`で移動するのはターミナルモードにも適応可能です。

### 今のバッファー以外を全部閉じたい

バッファーをついつい開きすぎて、今のタスクは一段落したからとにかく全部ファイルを閉じたい！という場面があります。

`close all`と`close all except current`のパターンがありますが、全部閉じるのが割とシンプルで、`:%bd`で達成できます。ただ、BufOnlyといった機能を実現するには、[プラグイン](https://github.com/Asheq/close-buffers.vim)を使うか、少しハッキングが必要です。

今自分が模索したものはこちら：
```lua
    -- close buffer
    ["<leader>C"] = {
      '<cmd>sil! exe "wa|%bd|e#|bd#|normal `"<cr>"',
      desc = "Save and close other buffers",
    },
    ["<leader>bo"] = {
      '<cmd>sil! exe "%bd|e#|bd#|normal `"<cr>"',
      desc = "Close other buffers except unsaved",
    },
```

違いは説明通り、前者では全て保存して、今のバッファー以外を閉じますが、後者は保存していないものはそのままにします。中身について軽く解説しますと

```
sil! -> silent!実行、!をつけるとエラーがあってもメッセージが表示されません
exe "" -> ここはsilと合わせて、""内部のものを静かに実行してねとのこと
wa -> write all, 全部保存
%bd -> buffer delete, 全部削除
e# -> edit #, %bdで現在のバッファーも削除されてしまうので、それを回復
bd# -> buffer delete #, %bdの副作用でno nameの空のバッファーが出来ちゃうのでそれを削除
normal `"-> ノーマルモードで`"を押して最後のカーサーにいた場所に戻る
```

というところです。

### ダイナミックなキーマップを作りたい

[`bufferline`](https://github.com/akinsho/bufferline.nvim)というプラグインでは、`go_to_buffer`+バッファー番号で該当バッファーに移動することが可能。ただ、キーマップ作るときに、`go_to_buffer 1`、`go_to_buffer 2`とかいちいち書くのが面倒ですし、10個以上になると効かなくなる問題もあります。パラメーターを受けるキーマップできないのか、と思いました。

基本この場面はコマンドを使う（`:BufferLineGoToBuffer 10`）のですが、ダイナミックなキーマップは作れないことはない。例えば、vimのAPIの`input`を使えば、入力内容を引数とすることが可能です。

```lua
    ["<A-t>"] = {
      function() require("bufferline").go_to_buffer(vim.fn.input "Buf number: ", true) end,
      desc = "Go to buffer by absolute number",
      noremap = true,
      silent = true,
    },
```

ただ、`<A-arg>`のように直接マッピングにダイナミックな内容を入れるのはおそらく無理。あくまでもworkaroundとして利用できるかと思います。


## モーションと編集

### multi-cursor使いたい

VSCodeのcmd+dは非常に便利です。option押しながらクリックしてマルチカーサーするのも結構使います。

Vimにはvisual-blockモードがありますが、移動と編集とか含めると無理があります。ここでは、[こちらのプラグイン](https://github.com/mg979/vim-visual-multi)を導入します。

このプラグインのキーマップは、ナビゲーションとかに使うのと結構競合があるので、デフォルトの`<C-n>`以外はほぼremapしないと使えない状態です。同じ単語に対して`<C-n>`でVSCodeの`cmd+d`と同じ効果ですが、カーサーを増やすときの`ctrl+UP/DOWN`はMac自身のショートカットとぶつかるので、次でremapしています。

```lua
-- mappings.lua
    -- multi-cursor
    ["<A-K>"] = { "<cmd>call vm#commands#add_cursor_up(0, v:count1)<cr>" },
    ["<A-J>"] = { "<cmd>call vm#commands#add_cursor_down(0, v:count1)<cr>" },
```

フォーカス中のカーサーを、`[`, `]`で移動したり、変更したくない箇所を`q`でスキップしたり、結構便利な機能が揃っています。

一つ見落としやすいところは、このプラグイン自身には2つのモード（cursor/extend）があり、タブキーで切り替えする必要があります。

例えば、上記でカーサーを増やしましたが、そのままv->eを押しても、今ハイライト中の一つのカーサーしかendに行きません。他のカーサーも同じ動きするには、カーサー増やした後、一回tab押して、extend modeに入ってから、マルチカーサーのセレクトが効くようになります。

もしくは、s+i+bとか（select in block）で、`()`ないのテキストを素早くセレクトできます。この辺りはcib, da"などと同じロジックなので直ぐになれるかと思います。

公式のgithubには色々gifあるのでここは省略します。他にもいろんな使い方があり、非常にポテンシャルのあるプラグインなのでもっと使いこなせたいですね。

### ファイル内にある同じ単語の間に移動したい

これは本題とあまり関係がないですが、マルチカーサーからの関連発想です。

通常では、`/`で単語を検索すると、全て当てはまるものがハイライトされます。すると`n`と`N`でカーサーを移動させることができます。

意外と知られていないもう一つのやり方があります。`*`を押すことで、現在カーサーにある単語を全部ハイライトして、`n`と`N`で移動させるのも可能です。これは場合によって検索よりだいぶ効率が良いですね。


### vim-surround使いたい

VSCodeのvimモードではついているものです。というかもうネイティブで内蔵して欲しい機能ですね。。プラグインは[こちら](https://github.com/tpope/vim-surround)

公式にはgifがないのでちょっとだけペーストします。

![](https://storage.googleapis.com/zenn-user-upload/070459a033bf-20220924.gif)

### 長い単語の真ん中に効率よく届きたい

[ lightspeed ](https://github.com/ggandor/lightspeed.nvim)を導入していますが、正直sキーの作動原理イマイチ理解していない状態です。。それで[clever-f](https://github.com/rhysd/clever-f.vim)の機能はちゃんとついているのでそのつもりで使っています。

例えば、キャメルケース、スネークケースで書いている文字だと、一つの単語wordとして認識されるので、wでもeでも真ん中のキャラクターに届くのが難しい場合があります。するとfでfindしてみても、重複なキャラクターがあると何回もf-xを押さなければならない(**訂正：`;`と`,`で次・前のキャラクターに移動可能**、ただ自分の`;`をコマンドモードの`:`にマップしているので使えない)。その時の救いがclever-fですね。一回fx押せば、次はfを押すだけで次のxまで移動できます。

![](https://storage.googleapis.com/zenn-user-upload/b8039a19d123-20220924.gif)

上記の問題だけに注目して解決するword-motionというプラグインもあります（[こちら](https://github.com/chaoren/vim-wordmotion)）。違うケースの単語もw/e/bが思う通り届くようになります。ただ、lightspeedとのモチベーションが違うというか、結局複数の単語で長くなった場合数回w/eとか押さないといけない。また、yiwとかでwの動作が上書きされているので、'camelCase'の'camel'もしくは'Case'しかyankできなくなります。といったデメリットから不採用にしました。

### 数行のコードを選択して上下に移動したい

VSCodeでoptionキーを押しながらアローキーで上下移動できます。それと同じ感覚にしたいです。

もちろん、vimネイティブなやり方で、visualモードで選択して、カットしてからペーストするのもありですが、より直感的になるのは良いと思います。これはキーマップの変更だけで達成可能です。

```lua
-- mappings.lua
return {
  v = {
    ["J"] = { ":move '>+1<CR>gv-gv", desc = "Move lines of code up" },
    ["K"] = { ":move '<-2<CR>gv-gv", desc = "Move lines of code down" },
  },
}
```

![](https://storage.googleapis.com/zenn-user-upload/521b70a751b4-20220924.gif)

### ドキュメントテンプレを入れたい

いくつかパッケージ試してみましたが([neogen](https://github.com/danymat/neogen), [nvim-tree-docs](https://github.com/nvim-treesitter/nvim-tree-docs))、今一番使いやすいと思っているのは[vim-doge](https://github.com/kkoomen/vim-doge)でした。

![](https://storage.googleapis.com/zenn-user-upload/f2cfb6117d66-20221002.gif)

function expression, declaration, arrow functionいずれも正常に生成できますが、jsdoc使っているのでtypeとインターフェースは動いていないです。todoの箇所を飛ぶときは`ctrl+j/k`で設定していますが、cmpのサジェスト候補ナビゲーションとかぶっているので（タブキーも被っている）飛ぶ前に一回`ctrl+e`で閉じています。もしくは一回ノーマルモードに戻ってから飛ぶのもありです。ここはもう少しキーマップの設定を考える必要があるかもしれませんが一旦これで使っています。

## ナビゲーションと検索

![](https://storage.googleapis.com/zenn-user-upload/60ee491cfdf1-20220925.png)

主に[telescope](https://github.com/nvim-telescope/telescope.nvim)が輝くところです。このプラグインは神です。プラグインというより、何かしらのピッカー（picker）を抽象化したFWとしてもみられるかもしれません。通常は検索インプット、結果リスト、一部ではプレビューエリアがついています。内蔵のピッカーだけではなく、コミュニティーで開発されたエクステンションも豊富です。自分はまだまだ使いこなせていないのですが、効率アップにはこれがかなり重要だと意識しています。

### vscodeのようなコマンドパレットが欲しい

はい、早速ですが、コマンドパレット的に、vimのコマンドを検索したい場合はどうしよう。ていうか、コマンド多すぎてキー覚えられないけど。。

telescopeの内蔵keymapsピッカーで悩み解決！AstroVimではデフォルトspace + skになっています。キーマップだけでなく、詳細のdescriptionも検索対象です。

![](https://storage.googleapis.com/zenn-user-upload/af44769d9045-20220924.png)

自分でキーマップ設置するときに、`desc`オプションを付けることで、descriptionの内容を変更できます。

```lua
-- mappings.lua
return {
  n = {
    ['custom mapping'] = {'<cmd>Telescope keymaps<cr>', desc = 'Show keymaps'}
    -- もしくは
    ['custom mapping'] = {function() require('telescope.builtin').keymaps() end, desc = 'Show keymaps'}
  }
}
```

他のtelescopeが呼び出せるピッカーは、`:Telescope`を打って、タブキーで確認したり、`:h Telescope.builtin`とかでドキュメントを確認すると良いでしょう。

### ファイル名で検索したい

続いてもう一つ必須のbuiltinピッカーとして、ファイル名でファイルを検索することがよく使うでしょう。

デフォルトのキーはspace+f+f, find fileでよく覚えられるのでそのままにしています。

他にも関連するピッカーがあります。例えばspace+f+o(find old)でvscodeのcmd+pと似ている過去に開かれたファイルを降順で探してくれます。開いているバッファーの中から探す場合は、space+f+b(find buffer)で探すと範囲がだいぶ縮められます。この辺りは基本デフォルト設定で要求を満たせます。

### /ではなく、fuzzy検索したい

`/`の検索は良いのですが、完全一致しないといけないので、ぼんやりとした検索したい時は若干使いづらいです。telescopeの出番です。

![](https://storage.googleapis.com/zenn-user-upload/d96ca4525175-20220925.gif)


現在のバッファー内で、fuzzy検索して、エンターキー押したら該当行に飛びます。最高です。

![](https://storage.googleapis.com/zenn-user-upload/e2e49d507d9d-20220925.png)

もちろん、fuzzy検索なのでスペリング若干間違っても問題なく拾ってくれます。

### すべてのファイルでキーワード検索したい

で、プロジェクト内のファイルの間に検索したい時はどうしよう。これは、ファイル内で`/`検索することを、プロジェクトのすべてのファイルを対象に検索することです。VSCodeで`command+F`と似ています。内蔵ピッカーの`live_grep`がありますが、これを使うには、[`ripgrep`](https://github.com/BurntSushi/ripgrep#installation)をインストールする必要があります。

![](https://storage.googleapis.com/zenn-user-upload/671b38392932-20220925.gif)

これとは別にビルトインの`grep_string`ピッカーがありますが、現在カーサーの置いている単語にたいしてファイル間検索してくれます。gi, gr, gdとかとの使い分けになると思いますが、とにかく今カーサーにある'foo'という名前の全て出現場所を探したい時はこれを使って良いでしょう。

### 検索範囲制限したい

上記の検索を行うときに、ショートカットではなく、コマンドで`:Telescope live_grep`を入力しました。その理由というのは、実際使っていないからです。

例えば、tsxのファイルだけを検索したい、とあるフォルダ内だけで検索したい場合はどうしよう。特に、プロジェクトの大きくなったり、FE側とBE側が同じレポにあったりする場合、FE側のコードを絞って検索したい、コンポーネントだけ検索したい`live_grep`だけでは物足りない場面が出てきます。

ここで内蔵のピッカーでは難しいので、エクステンションの導入です。

```lua
-- plugins/init.lua
  ["nvim-telescope/telescope-live-grep-args.nvim"] = {
    after = "telescope.nvim",
    config = function() require("telescope").load_extension "live_grep_args" end,
  },
-- plugins/telescope.lua
return {
  -- ...
  extensions = {
    "live_grep_args",
  },
}
```

すると、`ripgrep`のパラメーターを入れることが可能になります。

![](https://storage.googleapis.com/zenn-user-upload/a4fa7b8ad20c-20220925.gif)


### 検索した結果に対して単語の置き換えしたい

で、検索はこれで良いとして、とある変数名とかを検索して別の名前に置き換えしたい時はどうしよう。

ここは、quick fix listを少し触れないといけない([こちら](https://qiita.com/yuku_t/items/0c1aff03949cb1b8fe6b))。検索結果に対して、tabキーを押すと、ファイルの前に`+`が出てきます。これは、このファイルをquick fix listに送るようにする、との目印です。検索結果のすべてのファイルにおいて、置き換えしたい（もちろんこの操作に限らない）場合は、デフォルトの`ctrl+q`でquick fix listに追加します。tabキーで選択したファイルのみ送るたい場合は、`option+q`となります。この辺りマッピングは、telescopeのセットアップファイルで編集可能です。

それで、quick fist listに送られたファイルに対して、次のコマンドで置き換えを行います。

```
cdo %s/<target>/<replace>/gc | :w
```

`%s`はvimネーティブなコマンドなので割愛します。`| :w`で変更内容を全部保存します。`cdo`について詳細は[こちら](https://qiita.com/tommy6073/items/b4000435b661523ca744)。自分はいつも`/gc`の`c`つけていますが、これは置き換えをする前に一回確認することです。不要な場合は`/g`のみで良いでしょう。

![](https://storage.googleapis.com/zenn-user-upload/aa607957a645-20220925.gif)


### TODOの項目だけを検索したい

よくファイル内で、`TODO`、`NOTE`などのコメント置いたりします。永遠にTODOされてしまう可能性を軽減するために、TODOの箇所だけを探したいですね。必要なのはこちらの[ プラグイン ](https://github.com/folke/todo-comments.nvim)です。

telescopeと統合できるので、`:Telescope todo-comments`でピッカーを起動できます。公式サイトにスクリーンショットがあるのでここは省略します。

### ファイルに問題ありの箇所を確認かつナビゲートしたい

上記の`todo-comments`と関係ありますが、`trouble`を借ります（[こちら](https://github.com/folke/trouble.nvim)）。

linterなどで発見した問題をすべてリストに送ります。デフォルトのキーマップで、mでdocumentモードとworkspaceモードの切り替えができます。また、リンター問題を修復するときに、こちらのキーマップで素早くナビゲートできます：

```lua
    -- trouble
    ["<leader>xx"] = { "<cmd>TroubleToggle document_diagnostics<cr>", noremap = true, silent = true },
    ["gxj"] = {
      function() require("trouble").next { skip_groups = true, jump = true } end,
      noremap = true,
      silent = true,
    },
    ["gxk"] = {
      function() require("trouble").previous { skip_groups = true, jump = true } end,
      noremap = true,
      silent = true,
    },
    ["gxf"] = {
      function() require("trouble").first { skip_groups = true, jump = true } end,
      noremap = true,
      silent = true,
    },
    ["gxl"] = {
      function() require("trouble").last { skip_groups = true, jump = true } end,
      noremap = true,
      silent = true,
    },
```

### ブックマークを付けて検索したい

TODOはある意味でブックマークですが、何もかもTODOではなく、単純に複数のファイルの間に行き来することをシンプルにしたいときにブックマークをつけます。2つのプラグインを借ります。

```lua
-- plugins/init.lua
  { "MattesGroeger/vim-bookmarks" },
  ["tom-anders/telescope-vim-bookmarks.nvim"] = {
    after = "telescope.nvim",
    config = function() require("telescope").load_extension "vim_bookmarks" end,
  },
-- telescope.lua
return {
  -- ...
  extensions = {
    -- ...
    "vim_bookmarks",
  },
}
```

デフォルトでは`mm`でマークをon/offできます。もう一つは`mi`でマークする上でメモを入れることが可能です。

![](https://storage.googleapis.com/zenn-user-upload/8473c60116dc-20220925.gif)

### コピペの履歴を保存・検索したい(23/03/12追加)

vimネーティブな機能のレジスター（[registers](https://www.brianstorti.com/vim-registers/)）があります。これをシンプルに考えると、一つのハッシュマップとして、key:valueの形でテキストデータを保存しています。

![](https://storage.googleapis.com/zenn-user-upload/430772cf6dd5-20230312.png)

デフォルトのレジスター`"`には、普段削除、コピーしたテキストが保存されています。コピペの履歴を保存したいなら、他のレジスターも活用して、保存できるのでは？と思ったりするかもしれません。確かにある程度対応できますが、[neoclip](https://github.com/AckslD/nvim-neoclip.lua)というよりやりやすい解決案があります（そもそもレジスター自体はマクロとかもっと別の機能が含まれているのでこのためのものではない）。

neoclipでやっているのは、デフォルトのレジスター（設定次第別のレジスターを使うのも可能）を拡張し、本来の`":String`のマッピングを、`":String[]`にしました。その履歴をローカルでsqliteに保存し、セッションを再開しても保持することが可能になります。

![](https://storage.googleapis.com/zenn-user-upload/6fd355b6d4f8-20230312.gif)

設定は[こちら](https://github.com/convers39/astro-nvim-config/blob/master/plugins/neoclip.lua)です。デフォルトのキーマップでは、インサートモードで`<c-k>`がデフォルトの上下移動を上書きしているので、何も変更しないと使いづらい（他のtelescopeプラグインと別軸で覚えないといけない）と感じています。また、AstroNvimとの相性の問題かと思いますが、neoclipのコンフィグをロードする場所は、`plugins/init.lua`だと効果がなく、`telescope.lua`ファイルに置くと機能する、との問題が存在します。公式ドキュメントでのインストール方法に従うと、登録できない、履歴表示されないとの問題が確認できていました。

### コードの構造がわかるようなスケルトンを見たい

telescopeを使う場合、内蔵の`telescope.builtin.lsp_document_symbols()`で見られます（他にも似ている機能持っているtreesitterピッカーがある）。class, function, variableなどのキーワードでクラス、関数、変数の検索をすることが可能です。

また、ナビゲーションとして使う場合、[`Aerial`](https://github.com/stevearc/aerial.nvim)を使うことになります。

![](https://storage.googleapis.com/zenn-user-upload/fc446c6cc07e-20220928.png)

AerialはAstroNvimのプラグインリストに入っているのでそのまま使えます。自分はキーマップだけ変更しています。

```lua
-- mappings.lua
    -- Aerial
    ["<C-b>"] = { "<cmd>:AerialToggle<cr>" },
```

## Git周り

telescopeではgit関連のピッカーはいくつかあり、[こちら](https://github.com/AstroNvim/AstroNvim/blob/main/lua/core/mappings.lua#L136)のマッピングを確認すると良いでしょう。特にブランチのピッカーはかなり使いやすくて、リモートのブランチも簡単にチェックアウトできます。コードレビューなどにぜひ活用したい機能です。

### Gitでdiffしたい

こちらの[ diffview ](https://github.com/sindrets/diffview.nvim)を導入します。

ファイルツリーや変更箇所わかりやすくハイライトできますのでデフォルトのdiffviewより見やすい気がします。マージ時の競合も対応可能ですが、また実際に試していません。

このプラグインはdiffを一つのtabとして作っていて、gitsignsのdiffはbufferを作っています。なので画面を切り替えしたい時は、`:tabnext`とかで簡単にできます。

余談ですが、AstroNvimのデフォルトキーマップには、space+g+gで[`lazygit`](https://github.com/kdheepak/lazygit.nvim)があります。使うには別途lazygitをインストールする必要があります。diffに限らず、ステータス、ログ、コマンドなど全般あるので、自分は下記の`vim-fugitive`と併用しています。

### 競合解決したい

上記のdiff-viewで十分かもしれないが、他にこれ専用の[プラグイン](https://github.com/akinsho/git-conflict.nvim)もあります。これも公式の動画があるので省略します。

下記のようにchoose ours/theirsなどのショートカット、競合のコードブロックの間のナビゲーションのキーマップを定義しています。

```lua
    -- git conflict
    ["gco"] = { "<cmd>GitConflictChooseOurs<CR>" },
    ["gct"] = { "<cmd>GitConflictChooseTheirs<CR>" },
    ["gcb"] = { "<cmd>GitConflictChooseBoth<CR>" },
    ["gcn"] = { "<cmd>GitConflictChooseNone<CR>" },
    ["gcj"] = { "<cmd>GitConflictNextConflict<CR>" },
    ["gck"] = { "<cmd>GitConflictPrevConflict<CR>" },
    ["gcq"] = { "<cmd>GitConflictListQf<CR>" },
```

### vimコマンドに直接gitコマンド打ちたい

毎回ターミナルを呼び出してgitコマンド実行するのは面倒ですね。。というときに[`vim-fugitive`](https://github.com/tpope/vim-fugitive)が非常に助けになります。インストールするだけで、`:Git`もしくは`:G`プラスgitコマンドで行けます。

![](https://storage.googleapis.com/zenn-user-upload/bbbea161a658-20220925.gif)

個人的に必須レベルのプラグインです。

## UI

見た目重要ですね。NvChadの方が好みですが、機能面ではAstroの方が勝っていると感じています。

### colorschemeをプレビューしながら変更したい

これはNvChadのデフォルト設定で、Space+t+tでtelescopeでテーマのピッカーを立ち上げます。探したら確かにbuiltinのピッカーにありました。プレビューしながら変更するには、少しコンフィグ変更が必要です。

```lua
-- telescope.lua
return {
  -- ...
  pickers = { colorscheme = { enable_preview = true } },
}
```

自分はNvChadのように、space+t+tにしています。

![](https://storage.googleapis.com/zenn-user-upload/c5d969f10e7b-20220925.gif)


### Dashboardのヘッダーとコマンドを変えたい

ダッシュボードの有無は好みによりますが、なんか儀式感あるので良いとしています。

冒頭のヘッダーについて、ascii art generatorで生成したものです。[他の方のセットアップ](https://github.com/goolord/alpha-nvim/discussions/16)も参考になるかも。

また、コマンドについてもカスタマイズ可能です。

```lua
-- alpha.lua
local alpha_button = astronvim.alpha_button
return {
  layout = {
    { type = "padding", val = vim.fn.max { 2, vim.fn.floor(vim.fn.winheight(0) * 0.2) } },
    {
      type = "text",
      val = astronvim.user_plugin_opts("header", {
        "███████ ███████ ███    ██ ███    ██",
        "   ███  ██      ████   ██ ████   ██",
        "  ███   █████   ██ ██  ██ ██ ██  ██",
        " ███    ██      ██  ██ ██ ██  ██ ██",
        "███████ ███████ ██   ████ ██   ████",
        " ",
        " ███    ██ ██    ██ ██ ███    ███",
        " ████   ██ ██    ██ ██ ████  ████",
        " ██ ██  ██ ██    ██ ██ ██ ████ ██",
        " ██  ██ ██  ██  ██  ██ ██  ██  ██",
        " ██   ████   ████   ██ ██      ██",
      }, false),
      opts = { position = "center", hl = "DashboardHeader" },
    },
    { type = "padding", val = 5 },
    {
      type = "group",
      val = {
        alpha_button("LDR S .", "  Last Session  "),
        alpha_button("LDR S f", "  Find Session  "),
        alpha_button("LDR f f", "  Find File  "),
        alpha_button("LDR f o", "  Find Recent  "),
        alpha_button("LDR f w", "  Find Word  "),
        alpha_button("LDR m a", "  Bookmarks  "),
      },
      opts = { spacing = 1 },
    },
  },
}
```

AstroNvimには、`SessionManager`をデフォルトインストールしているので、一つのdirectoryに対して自動で保存してくれます。起動後エンターキー押すだけで、cwdに対してのセッションをロードします。ただ初めての場合はセッションがないのでロードが失敗します。

セッション管理とは別で、プロジェクト管理のtelescope用のエクステンションの[`telescope-project`](https://github.com/nvim-telescope/telescope-project.nvim)もありますが、今のところneo-tree+SessionManagerで十分な気がします。というのは、find sessionコマンドで事実上cwdに関わらず飛ぶことが可能です。

![](https://storage.googleapis.com/zenn-user-upload/39421af38e8e-20220925.gif)

### アイコンを色々と変更したい

上記のダッシュボードだけでなく、telescopeなどのプラグインの設定でもアイコンの変更が可能です。できるとしても、どこからアイコン候補を探せば良いか。。[unicodeのサイト](https://unicode-table.com/en/)で探してみましたが、結局そちらのキャラクターが表示できないものが多く使い物になりません。emojiが使えますが、カラー付きなので若干合わない気もします（と言いつつtelescopeはemoji）。

補足のフォントのところと関係ありますが、[こちら](https://www.nerdfonts.com/cheat-sheet)でキーワードを検索して、好きなアイコンが見つかるかもしれません。Font Awesome, Devicon, Material Design, Codiconsなどを含めて全部で4000以上あるらしいです。

![](https://storage.googleapis.com/zenn-user-upload/34596aa1c825-20220930.png)

### コマンド入力時のみコマンドインプットを真ん中に表示したい(23/03/12追加)

人によって問題にはならないはずですね。vscodeにはコマンドパレットがあると思いますが、入力する場所は一番上になっています。それに対してvimデフォルトでは一番下になっていますが、いずれにしても目線を移動させる必要があります（この問題意識は強引すぎるw）。で、どっちも正直自分が好きではありません。理想は入力時だけ、真ん中にポップアップの形で入力ボックスを表示することです。

それを探してみたら、やはりプラグインがありました（だから自分だけの問題意識ではないって）。最初に使ったのは[findCmdline](https://github.com/VonHeikemen/fine-cmdline.nvim)で、まさにこのためのプラグインになっています。しかし、ウィンドウ間移動する時にバグがあり、コマンドインプットのポップアップから、`<c-l>`とかで移動すると、ポップアップが消えず、消すためにもう一度コマンドインプットに入って何か実行する必要があります（実在しないコマンドでも、`<CR>`があればOK）。

これで不便なので、前にスターしたプラグインで探ったら、同じ機能を提供する[noice](https://github.com/folke/noice.nvim)が見つかりました。このプラグインも多少AstroNvimとの相性の問題がありましたが、何日か模索した結果、これを採用するようにしました。設定は[こちら](https://github.com/convers39/astro-nvim-config/blob/master/plugins/noice.lua#L46)です。

![](https://storage.googleapis.com/zenn-user-upload/70681138a253-20230312.gif)

このプラグインでは、コマンドインプットを真ん中にするだけではなく、候補のボックスもきれいに備えることができます。本来は通知機能をよりカスタマイズするためのプラグインですので、これを利用してついでに別の問題(次に説明)を修復しました。

ちなみに、vscodeのような一番上にコマンドパレット式に表示するには、プリセットの設定を使えば便利です。

```lua
require("noice").setup({
  presets = {
    command_palette = true, -- position the cmdline and popupmenu together
  },
})
```

### メッセージ表示とcmdheightの互換性問題を解決したい(23/03/12追加)

この問題は、nvim0.8でコマンドインプットを常時隠すことができるようになって（`cmdheight=0`）、一番下にステータスバーのみ表示する、との素晴らしい機能に由来するものです。

この機能の副作用として、メッセージとコマンドラインが共通の場所を使うので、何かしらの操作によってメッセージが表示される場合、一度`<CR>`押さないと、フォーカスがずっとメッセージにあります。例えば、`mm`でブックマークを追加する時に、`cmdheight=1`の場合だと、メッセージが表示されても、フォーカスはファイルにあるので特に問題ありませんが、`cmdheight=0`にすると、`mm`の後にもう一度`<cr>`を押さないと、フォーカスがファイルにないので、ファイルでの操作ができなくなります。

前の`mm`のみで良いパターンに慣れていると、急に`mm<cr>`に変えるのが少し辛いかもしれません。この解決策として、先ほどのNoiceを利用し、messageチャンネルの内容を別のチャンネルに表示することで解決できます。

```lua
require("noice").setup({
  -- ...
  messages = {
    enabled = true,
    view = "mini", -- redirect messages to mini
    view_error = "notify",
    view_warn = "notify",
    view_history = "messages",
    view_search = false,
  },
})
```

![](https://storage.googleapis.com/zenn-user-upload/8c3c9c26f65b-20230312.gif)

デフォルトのviewは`cmdline`となっているので、これを別のもの（例えば`mini`、`notify`とか）に変えると、`cmdline`を占めることがなくなり、本来の`mm`のパターンでフォーカスが無くならずに済みます。


# 終わりに

Vimはコミュニティーと共に進化し続けていくので、一つのコンフィグも変わり続けていくと思います。また何か面白い使い方があったら更新しようと思います。何かコメント、ご指摘とかあったら是非お願いいたします。

ではでは！

# 補足

## optionキーが使えない場合

optionキーはMacのメタキーであり、Windows/LinuxのAltキーと相当します。キーマップ設定するときに、`map('n', '<A-x>', 'xxx')`もしくは`<M-x>`のいずれかが効きます。もしこのキーが使えない場合は、エミュレータでの設定を変更する必要があるかもしれません。

例えばiTermの場合、Left Option Keyを`Normal`から`Esc+`にすることで使えるようになります。

![](https://storage.googleapis.com/zenn-user-upload/eb014d35cff5-20220927.png)

## アイコンが口口または??の場合

[Nerd Font](https://www.nerdfonts.com/)を使ってみましょう。

例えば、iTermでは2つのフォント設定ができますが、一個目を自分が好きなやつにして、二個目はアイコン対応のNerd Fontにすると表示ができるかと思います。筆者はFiraCode Nerdにしています。 

![](https://storage.googleapis.com/zenn-user-upload/9db18aba3e1b-20220927.png)

## プラグインまたは変更内容が反映しない場合

手順としては、
1. プラグインなどのファイル変更
2. neovim再起動
3. `:PackerSync`実行またはデフォルトの`space+p+s`
4. neovim再起動

再起動2回必要です。

## キー列完了判定が早すぎる場合

人によってデフォルトの1000ms判定が早すぎる可能性もあります（体感だと1000msない気もしますが）。その場合は判定時間を伸ばすように設定変更してみてください。
```lua
-- polish.lua
return function()
  -- timeout判定を3秒に伸ばす
  vim.opt.tm = 3000
end
```
詳細は`:h timeoutlen`で。300msに設定する猛者もいるらしいですが自分はデフォルトで良いです。

## アンインストールしたい場合

通常インストールの場合は：

```bash
rm -rf ~/.config/nvim
rm -rf ~/.cache/nvim
rm -rf ~/.local/share/nvim
```

ただもし`.config/nvim`ではなく、`.config/astronvim`にインストールした場合は適宜調整必要。

自分は切り替えを考慮して`rm`より`mv`でしています。