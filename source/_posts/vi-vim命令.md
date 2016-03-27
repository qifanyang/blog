---
title: vi/vim命令
date: 2016-03-24 09:40:38
tags: 杂七杂八
---
## vi/vim
在linux下编辑文本文件,vi是首选,系统都自带,在linux下编写c/java等程序文件,查找,替换,删除一行,复制一行,这些操作比较常用,所以记录下命令方便查找,网上搜索的命令太多但不一定适合自己.所以记录一些可以达到方便编辑的命令就可以了

//删除
x   - delete current character
dw  - delete current word
dd  - delete current line
5dd - delete five lines
d$  - delete to end of line
d0  - delete to beginning of line

//退出
:x<Return>	quit vi, writing out modified file to file named in original invocation
:wq<Return>	quit vi, writing out modified file to file named in original invocation
:q<Return>	quit (or exit) vi
:q!<Return>	quit vi even though latest changes have not been saved for this vi call

//移动光标
j or <Return> [or down-arrow]	move cursor down one line
k [or up-arrow]	move cursor up one line
h or <Backspace> [or left-arrow]	move cursor left one character
l or <Space> [or right-arrow]	move cursor right one character
0 (zero)	move cursor to start of current line (the one with the cursor)
$	move cursor to end of current line
w	move cursor to beginning of next word
b	move cursor back to beginning of preceding word
:0<Return> or 1G	move cursor to first line in file
:n<Return> or nG	move cursor to line n
:$<Return> or G	move cursor to last line in file


//剪切和拷贝,这个使用频率高
yy	copy (yank, cut) the current line into the buffer
Nyy or yNy	copy (yank, cut) the next N lines, including the current line, into the buffer
p	put (paste) the line(s) in the buffer into the text after the current line

//搜索文本
/string	search forward for occurrence of string in text
?string	search backward for occurrence of string in text
n	move to next occurrence of search string
N	move to next occurrence of search string in opposite direction

//显示行号
 : set nu
 vim ~/.vimrc  在其中输入 set nu

http://www.cs.colostate.edu/helpdocs/vi.html