## 插件：[Awesome Emacs Keymap](https://marketplace.visualstudio.com/items?itemName=tuttieee.emacs-mcx)
-----------

## 常用快捷键

Meta=Alt/Cmd

<!-- ← ↑ → ↓ -->

### 光标移动
- Ctrl+f 向前移动一个字符
- Ctrl+b 向后移动一个字符
- Ctrl+n 移动到下一行
- Ctrl+p 移动到上一行
- Ctrl+a 移动到行首
- Ctrl+e 移动到行尾
- Meta+f 向前移动一个单词
- Meta+b 向后移动一个单词
- Ctrl+→或Meta+→ 向前移动一个单词(相当于Meta+f)
- Ctrl+←或Meta+← 向后移动一个单词(相当于Meta+b) 
- Meta+m 移动到光标所在行的第一个非空白字符处
- Ctrl+v 向下翻动一个屏幕单位
- Meta+v 向上翻动一个屏幕单位
- Meta+Shift+\[或Meta+\{或Ctrl+↑ 移动到上一段落的开头
- Meta+Shift+\]或Meta+\}或Ctrl+↓ 移动到下一段落的结尾
- Meta+Shift+,或Meta+< 移动到缓冲区的顶部
- Meta+Shift+.或Meta+> 移动到缓冲区的底部
- Meta+g+g(即Meta g + Meta g) 跳转到第几行
- Meta+g+n(即Meta g + Meta n) 跳转到下一处error
- Meta+g+p(即Meta g + Meta p) 跳转到上一处error
- Ctrl+l 将光标所在行置于屏幕中央

### 搜索和替换
- Ctrl+s 向前搜索
- Ctrl+r 向后搜索
- Ctrl+Meta+s 正则表达式搜索
- Ctrl+Meta+r 反向正则表达式搜搜
- Meta+Shift+5 替换
- Ctrl+Meta+5 使用正则表达式替换
- Ctrl+Meta+n 将下一处匹配项加入选中
- Ctrl+Meta+p 将上一处匹配项加入选中

### 编辑
- Ctrl+d 向右删除字符
- Ctrl+h 向左删除字符
- Meta+d 向右删除单词
- Meta+Backspace 向左删除单词
- Ctrl+k 删除从光标处到当前所在行的末尾的所有字符
- Ctrl+Shift+Backspace 删除整行
- Ctrl+w 删除区域文本
- Meta+w 将区域文本添加到kill-ring中，方便找回
- Ctrl+y 粘贴
- Meta+y(即yank-pop) 此命令只能在刚用完Ctrl+y后使用。它的作用是用kill-ring中再前一个内容替换掉刚用Ctrl+y粘贴出来的内容
- Ctrl-o 新开启一行(类似于回车)
- Ctrl+j或Ctrl+m 新建一行(类似于回车)
- Ctrl+x+o(即Ctrl+x Ctrl+o) 删除文本上面的空行
- Ctrl+x h 全选
- Ctrl+x u或Ctrl+/或Ctrl+Shift+- 撤回
- Ctrl+; 行注释和非行注释之间切换
- Meta+; 块注释和非块注释之间切换
- Ctrl+x Ctrl+l或者Meta+l 转换为小写
- Ctrl+x Ctrl+u或者Meta+u 转换为大写
- Meta+c 转换为标题格式
- Meta+Shift+6 合并上一行和当前行

### 标记
- Ctrl+Space或Ctrl+Shift+2 设置标记点并激活
- Ctrl+Space Ctrl+Space 设置标记点，存入mark-ring，不激活
- Ctrl+u Ctrl+Space 移动到标记点。并读取mark-ring中的下一标记点。
- Ctrl+x Ctrl+x 设置标记点并激活。然后移动到默认标记点。

### 文本寄存器
- Ctrl+x r sr 将区域文本复制到文本寄存器r
- Ctrl+x r ir 使用文本寄存器r中的内容进行插入

### 矩形
- Ctrl+x Space 在矩形标记模式和普通模式之间切换
- Ctrl+x r k 删除矩形区域，将区域文本存入"last-killed-rectangle"中
- Ctrl+x r Meta+w 将矩形区域存入"last-killed-rectangle"
- Ctrl+x r d 删除矩形区域
- Ctrl+x r y 粘贴"last-killed-rectangle"到当前光标所在位置
- Ctrl+x r p 如果"last-kill-ring"的顶层只有一行，则用它替换矩形区域中的每一行
- Ctrl+x r o 用空行填充矩形区域
- Ctrl+x r c 清空矩形区域(用空格填充)
- Ctrl+x r t 用字符串替换矩形区域的每一行

### 其他
- Ctrl+g或Esc 取消
- Ctrl+'或者Meta+/ 显示智能提示内容
- Meta+x 打开VSCode命令面板
- Ctrl+Meta+Space VSCode侧边栏显示与隐藏切换
- Ctrl+x z Zen模式切换
- Ctrl+x Ctrl+c 关闭窗口(包含保存文件、关闭终端)

### 文件
- Ctrl+x Ctrl+f 快速打开文件
- Ctrl+x Ctrl+s 保存文件
- Ctrl+x Ctrl+w 文件保存为
- Ctrl+x s 保存所有文件
- Ctrl+x Ctrl+n 打开一个新窗口

### 标签页(Tab)或者缓冲区(buffer)
- Ctrl+x b 切换到另一个已打开的缓冲区
- Ctrl+x k 关闭当前标签页或者缓冲区
- Ctrl+x 0 关闭当前分组(split)的所有Editors(标签页)
- Ctrl+x 1 关闭其他分组(split)的所有Editors(标签页)
- Ctrl+x 2 水平分组(split horizontal)
- Ctrl+x 3 垂直分组(split vertical)
- Ctrl+x 4 水平和垂直分组之间切换
- Ctrl+x o 切换焦点到其他分组(split)
- Ctrl+x ← 选择上一个标签页或者缓冲区
- Ctrl+x → 选择下一个标签页或者缓冲区
