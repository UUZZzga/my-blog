+++
title = "lua parser 负数小bug"
date = "2021-09-19T11:48:40+08:00"
tags = ["c","lua"]
image = ""
comments = true	# set false to hide Disqus
share = true	# set false to hide share buttons
menu= ""		# set "main" to add this content to the main menu
author = "uzzzzz"
description = "lua 解析负数 最小值 bug"
+++
bug代码
```lua
local a=-9223372036854775808
local b=-9223372036854775807-1

print(a==b) -- true
print(a) -- -9.2233720368548e+18
print(b) -- -9223372036854775808
```
查找官方社区
http://lua-users.org/lists/lua-l/2020-03/msg00112.html
http://lua-users.org/lists/lua-l/2020-03/msg00114.html

经阅读代码发现有写解析代码（详见 llex.c read_numeral）
观察调用堆栈
lparser.c subexpr
```c
static BinOpr subexpr (LexState *ls, expdesc *v, int limit) {
  BinOpr op;
  UnOpr uop;
  enterlevel(ls);
  uop = getunopr(ls->t.token);
  if (uop != OPR_NOUNOPR) {  /* prefix (unary) operator? */
    int line = ls->linenumber;
    luaX_next(ls);  /* skip operator */
    subexpr(ls, v, UNARY_PRIORITY);
    luaK_prefix(ls->fs, uop, v, line);
  }
  else simpleexp(ls, v);
  // ...
```
luaX_next 执行 llex 进行处理字符串转 整数和小数
luaK_prefix 内处理过常量计算优化

简单写了个修复
lex.h
```c
// ...
typedef struct LexState {
  // ...
  int prefixMinus;  /* prefix equal minus */
} LexState;
// ...
```
lparser.c
```c
// ...
static BinOpr subexpr (LexState *ls, expdesc *v, int limit) {
  BinOpr op;
  UnOpr uop;
  enterlevel(ls);
  uop = getunopr(ls->t.token);
  if(uop == OPR_MINUS){ /* 一元运算符 符号为 - */
    // 修改部分
    int line = ls->linenumber;
    ls->prefixMinus = 1;// 为负数
    luaX_next(ls);
    isPrefix = ls->prefixMinus;//a=-(-1)
							   //   ^
    subexpr(ls, v, UNARY_PRIORITY);
	// 不执行 luaK_prefix(ls->fs, uop, v, line);
	// 不能使用 a=-(-1)
	
	if(isPrefix){
      luaK_prefix(ls->fs, uop, v, line);
      ls->prefixMinus = 0;
    }
  }
  else if (uop != OPR_NOUNOPR) {  /* prefix (unary) operator? */
    int line = ls->linenumber;
    luaX_next(ls);  /* skip operator */
    subexpr(ls, v, UNARY_PRIORITY);
    luaK_prefix(ls->fs, uop, v, line);
  }
  else simpleexp(ls, v);
  // ...
```
lex.c
```c
static int llex (LexState *ls, SemInfo *seminfo) {
      // ...
      case '0': case '1': case '2': case '3': case '4':
      case '5': case '6': case '7': case '8': case '9': {
	    // 修改部分
        if(ls->prefixMinus){
          save(ls, '-');// 向buffer写入 '-'
		  ls->prefixMinus = 0;
        }
        return read_numeral(ls, seminfo);//能处理 负数情况
      }
	  // ...
}
```
lparser.c
```c
LClosure *luaY_parser (lua_State *L, ZIO *z, Mbuffer *buff,
                       Dyndata *dyd, const char *name, int firstchar) {
  LexState lexstate;
  FuncState funcstate;
  LClosure *cl = luaF_newLclosure(L, 1);  /* create main closure */
  setclLvalue2s(L, L->top, cl);  /* anchor it (to avoid being collected) */
  luaD_inctop(L);
  lexstate.prefixMinus = 0;//初始化
  // ...
}
```

编译测试
```lua
local a=-9223372036854775808
local b=-9223372036854775807-1
-- 试一下修改会不会影响小数
local c=-1.2345678
c=c-99

local d=1-a
local e=0.0 + a

print(a==b) -- true
print(a) -- -9223372036854775808
print(b) -- -9223372036854775808
print(c) -- -100.2345678
print(d) -- -9223372036854775807
-- 原因
-- -9223372036854775808 * (-1) = -9223372036854775808
-- 64为整数最大表示范围 9223372036854775807 刚好溢出 1
-- 1 - (-9223372036854775808) = 1 + (-9223372036854775808)
-- -9223372036854775808 = 0x8000000000000000
-- 1 + 0x8000000000000000 = 0x8000000000000001 = -9223372036854775807
print(e==a) -- true
print(e==b) -- true
print(e) -- -9.2233720368548e+18
```
再换一段代码测试 优化是否能正常工作
```lua
local a=-9223372036854775808 + 9223372036854775807
print(a)
```
查看指令
```
0+ params, 3 slots, 1 upvalue, 1 local, 1 constant, 0 functions
        1       [1]     VARARGPREP      0
        2       [17]    LOADI           0 -1
        3       [18]    GETTABUP        1 0 0   ; _ENV "print"
        4       [18]    MOVE            2 0
        5       [18]    CALL            1 2 1   ; 1 in 0 out
        6       [18]    RETURN          1 1 1   ; 0 out
```
看结果没问题 用一段时间试试

补充一下代码测速
```lua
local integerMin = -9223372036854775808
-- local integerMin = -9223372036854775807 - 1

local io = require "io"

local fp = io.open("tmp.lua", "w")
fp:write("local tmp = {")
fp:write(integerMin)

for index = -9223372036854775807, -9223372036854000000 do
    fp:write(",")
    fp:write(index)
end
fp:write("}\n")
fp:write("return tmp")
fp:close()

start_time = os.clock()
local tmp = require "tmp"
end_time = os.clock()
print(#tmp)
print(end_time - start_time)
```
没有修改的输出
```
775809
0.197
```
修改过的输出
```
775809
0.177
```
我修改过的竟然还要快？
分析原因
1. struct LexState 的结构我也顺便优化了一下把指针放结构体到前面，这样改
```c
typedef struct LexState {
  struct FuncState *fs;  /* current function (parser) */
  struct lua_State *L;
  ZIO *z;  /* input stream */
  Mbuffer *buff;  /* buffer for tokens */
  Table *h;  /* to avoid collection/reuse strings */
  struct Dyndata *dyd;  /* dynamic structures used by the parser */
  TString *source;  /* current source name */
  TString *envn;  /* environment variable name */
  Token t;  /* current token */
  Token lookahead;  /* look ahead token */
  int current;  /* current character (charint) */
  int linenumber;  /* input line counter */
  int lastline;  /* line of last token 'consumed' */
  int prefixMinus;  /* prefix equal minus */
} LexState;
```
注意代码只改过先后顺序,编译运行
```
775809
0.187
```
还有0.01的差距
2. 测试集问题
我的测试集是选用负数来测
修改的代码部分负数部分没有执行 luaK_prefix -> constfolding 这样的优化
换个正负数差不多的测试集试试
```lua
-- ...
for index = -9223372036854775807, -9223372036854000000 do
    fp:write(",")
    fp:write(index)
    fp:write(",") 			-- +++++++
    fp:write(index * -1)	-- +++++++
end
-- ...
```
我修改的代码速度
```
1551617
0.356
```
官方代码优化内存布局
```
1551617
0.37
```
还是能快0.01秒

那都是正数情况呢
```lua
local begin = 9223372036854000000

local io = require "io"

local fp = io.open("tmp.lua", "w")
fp:write("local tmp = {")
fp:write(begin)

for index = begin+1, 9223372036854775807 do
    fp:write(",")
    fp:write(index)
end
-- ...
```
我修改过的
```
775808
0.169
```
官方代码优化内存布局
```
775808
0.169
```
得出结论
修改内存布局就可以提高 5%
在此基础上
在只负数时 我修改的快约 6%
在正负数各占一半 我修改的快约 4%
在只有正数时 性能几乎一样