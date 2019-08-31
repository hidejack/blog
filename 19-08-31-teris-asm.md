# 基于x86汇编实现的俄罗斯方块

## 一、实验环境

- 文本编辑器

        vscode + PlatformIO IDE插件

- 编译器

        MASM 8.0

- 运行环境

        DosBox 0.74

## 二、需求分析

> 如图所示，我们需要在DosBox终端的显存“画出” 10X20 的游戏界面。

![BDE164FD-A4DA-4920-AB3D-FBA6F4231185.png](https://i.loli.net/2019/08/31/Yj9VG5QB6Zyi4Wd.jpg)

### 1.分解游戏步骤
1. 清屏（clear_screen）
2. 初始化游戏边框（init_screen）
3. 显示右边游戏状态栏（init_var）
4. 初始化变量：分数、level、消除行、下一个方块（show_number）
5. 创建俄罗斯方块（create_teris）
6. 显示俄罗斯方块 （show_teris）
7. 由于我们要用调用键盘中断，所以还需要定义自己的中断函数，最后还需要将系统的int9中断保存起来，向中断向量表写入自己定义的中断函数（save_sys_int9、set_new_int9）

到这里游戏开发步骤基本分析完毕，为了开发便利，我们还需要将常用的变量进行存储。如：分数，边框显示，俄罗斯方块显示及颜色，下一个降落方块。

### 2.数据模型分析

- 游戏边框
        
        这里为了方便辨认，我选择上下边框使用字符“=”，左右边框使用 “|”。
        这两个字符都定义在了内存的data区，方便随时修改。
- 俄罗斯方块
    
        俄罗斯方块定义了7种类型：O,L,J,S,Z,I,T
        俄罗斯方块显示字符为3，每个方块的颜色不一样。
        颜色变量，显示字符，都定义在了内存的data区。

        每个方块固定需要4个显存元素即8个显存空间。为了方便后续移动旋转操作，我在内存data 段中预存了每个俄罗斯方块当前的显存位置和移动旋转之后的显存坐标。
- 游戏状态栏

        分数、游戏难度、消除行、下一个俄罗斯方块，需要实时更新，因此这部分变量我也都放到了内存中。
        ps：为了方便显示，我把不同的状态显示的显存位置也存放到了内存中。
>   内存data 段代码定义如下：
```
data 		segment
            ;存放常量内存区域
            db	'SCORE:',0
            db	'LEVEL:',0
            db	'DELROW:',0
            db	'NEXT:',0
            db	'=','|',0,0

            ;数字字符串，用来显示分数
num_str	    db	'0','1','2','3','4','5','6','7','8','9'

            ;E黄色D 红色B 亮绿色A 绿色6 橙色7 暗黄色8 灰色
color		db	0EH,0DH,0BH,0AH,6,7,3

            ;0显示方块1分数2难度3消除行数4当前方块5下一个方块E旋转状态
            ;67分数显存地址89难度显存AB消除行数显示CD时间显存
var		    db	3,0,1,0,0,6,0,0,0,0,0,0,0,0,0,1

teris 		dw	5	dup	(0)	;俄罗斯方块的4个点显存位置
teris_old	dw  5	dup (0)	;移动旋转之前的方块显存位置

sys_address	dw	2	dup	(0)	;用来保存系统int9中断的cs，ip

data		end
```
## 三、开始写代码吧！

这里不具体叙述具体代码，只是大体写下思路。<br>
- 随机生成俄罗斯方块

        因为俄罗斯方块是随机生成的，因此这里需要一个随机函数。
        这里取系统的时钟计数器，调用int 1AH中断实现。
> 代码如下：
```
;取随机数
;参数： dl
;返回值： bl
;说明： 取0-dl 范围随机数，取时钟计时器值，取余得随机数
rand:					push ax
						push dx
						push cx

						push dx
						sti
						mov ah,0	;读时钟计数器
						int 1AH
						mov ax,dx
						mov ah,0	;高位清0 此时 al 范围 0-255
						pop dx
						div dl		;除dl 获取 0-dl 范围内的余数

						mov bx,0
						mov bl,ah 	;ah 中余数 存 bl中，作为随机数使用

						pop cx
						pop dx
						pop ax
						ret
```
- 定时下落
        
        在游戏界面初始化完成之后，需要实现一个定时下落的功能。
        延迟下落，写一个循环，不断执行 move_down 函数即可，执行时需要一个延迟1秒执行。
        因此这里需要实现一个延迟执行的函数。
        这里调用系统 int 15h进行中断延迟。
        
        ps：空循环也能实现，太暴力，浪费系统资源，代码看起来不舒服。

> 延迟函数代码如下：
```
delay:					push ax
						push cx
						push dx

						mov ah,86H
						mov cx,0FH
						mov dx,2420
						int 15H

						pop dx
						pop cx
						pop ax
						ret
```
-  俄罗斯方块的旋转

        这里也是整个俄罗斯方块的核心部分了。这里解决了基本俄罗斯方块完成了一大半。
        我的具体思路为：选中俄罗斯方块左下方的点作为每一次旋转的参考点，进行旋转。

        如一个T型方块，显存坐标为：
        A点：160*n+22
        B点：160*n+24
        C点：160*n+26
        D点：160*(n+1)+24.

        四个点的坐标可以分解为（x，y），x为列，y为行。

        则四个点的坐标转换过来就是：
        A点：（n,22）
        B点：(n,24)
        C点：(n,26)
        D点：(n+1,24)

        此时取左下方的参考点则是取：列值最大，行值最小。
        因此左下方的参考点坐标为：（n+1，22）

        旋转算法：
        取当前俄罗斯方块的每个点的坐标，与参考点坐标进行差值运算，取绝对值。
        如上例子，进行运算并取绝对值后的结果为：
        A：（1，0）
        B：（1，2）
        C：（1，4）
        D：（0，2）

        取到的绝对值加上参考点的坐标之后，即为旋转之后俄罗斯方块各点的坐标：
        A:(n+2,22)
        B:(n+2,24)
        C:(n+2,26)
        D:(n+1,24)

        最后把取到的坐标进行显存地址的转换即可，转换的地址为：
        A:(n+2)*160+22
        B:(n+2)*160+24
        C:(n+2)*160+26
        D:(n+1)*160+24
        
        最后，显示即可！

> 代码如下
```
revolve_teris:			push ax
						push bx
						push cx
						push dx
						push si

						mov ax,word ptr teris[0]	;取出参考点显存地址
						mov dx,ax
						mov bl,160
						div bl						;此时获取al 为行 ah 为列

						mov cx,4
						mov si,2

revolveTeris:			push ax						;保存参考点坐标
						mov bx,word ptr teris[si]
						mov word ptr teris_old[si],bx
						push ax
						mov ax,bx
						mov bl,160
						div bl
						mov bx,ax					;获取方块坐标存入bx
						pop ax						;取出ax 即参考点坐标

						cmp ah,bh					;获取列差绝对值
						jb bh_ah
						sub ah,bh
						mov bh,ah
						jmp next_value

bh_ah:					sub bh,ah

next_value:				cmp al,bl					;获取行 差绝对值
						jb bl_al
						sub al,bl
						mov bl,al
						jmp get_revolve_address

bl_al:					sub bl,al					;此时绝对值在bx中 bh:列 bl:行

get_revolve_address:	mov ah,0					;获取移动后坐标，此处要进行转换坐标
						mov al,bh
						mov bh,80
						mul bh
						add bl,bl
						mov bh,0
						add ax,bx

						add ax,dx
						mov word ptr teris[si],ax

						add si,2
						pop ax
						loop revolveTeris

						mov word ptr teris_old[0],dx		;将参考点坐标存入old Teris 区域 计算旋转后 参考点
						call get_ref_point
						mov word ptr teris[0],ax

						pop si
						pop dx
						pop cx
						pop bx
						pop ax
						ret
```

- 检查碰撞

        几种情况的碰撞：
        1.下落碰撞
        2.左右移动碰撞
        3.旋转碰撞

        碰撞这里只要检查，旋转移动之后的显存坐标是否存在边框字符串即可。
        如果存在，则取消这次操作。如果不存在，则操作生效。
        上面data段中定义的两个内存地址作用就是在这里用的。

        teris:当前显示的俄罗斯方块显存地址
        old_teris:操作之前俄罗斯方块的显存地址。

        每一次操作都将old_teris存储起来，当检查操作合法，则取出teris中的显存地址显示。如果操作不合法，则将odl_teris 返回给teris 进行显示。这样就做到了边框的碰撞。

        左右和旋转碰撞只要简单的检查不合法，操作不生效即可。
        下落发生碰撞，则要重新生成俄罗斯方块，显示。
> 代码如下
```
;检查边界
;参数：无
;返回值：上下边框碰撞：bx=7DH 左右边框碰撞：bx=7EH 碰撞方块：bx=7FH 无碰撞:2FH
;检查teris 区中的teris 是否产生碰撞
check_boundary:			push cx
						push si
						push di

						mov cx,4
						mov si,2
						mov bx,0

checkBoundary:			mov di,word ptr teris[si]
						mov al,byte ptr es:[di]
						mov ah,ds:[1CH]
						cmp al,ah
						je bottomCollision
						mov ah,ds:[1DH]
						cmp al,ah
						je sideCollision
						mov ah,var[0]
						cmp al,ah
						je terisCollision

						add si,2
						loop checkBoundary

						mov al,'0'
						jmp checkBoundaryRet

terisCollision:			mov al,'1'
						jmp checkBoundaryRet

bottomCollision:		mov al,'2'
						jmp checkBoundaryRet

sideCollision:			mov al,'3'
						jmp checkBoundaryRet

checkBoundaryRet:		pop di
						pop si
						pop cx
						ret
```
- 消除

        在每次下落事件触底时，从最低一行检查每一个显存单元是否存在俄罗斯字符方块即可。
        如果每一行全都存在，则执行消除。
        如果不存在，则向上检查。

        消除时需要注意，消除当前代码后，当前行以上的行需要下移，下移之后需要消除上一行。
> 代码如下：
```
;检查清除填满的行
;参数：无
;返回值：无
;四个点的坐标分别为：160*2+62  160*2+80  160*21+62  160*21+80
eliminate:				push ax
						push bx
						push cx
						push dx
						push di

						mov bl,byte ptr var[5]				;随机生成俄罗斯方块
						mov byte ptr var[4],bl
						mov dx,0
						mov dl,7
						call rand
						mov byte ptr var[5],bl

						mov di,160*21+62
						mov dx,0						;存储消除的行
						mov cx,20

check_row:				push cx
						mov bx,0
						mov cx,10

check_one:				mov al,byte ptr es:[bx+di]
						mov ah,byte ptr var[0]
						cmp al,ah
						jne check_next_row
						add bx,2
						loop check_one
						call clear_row
						call move_down_all
						add di,160						;检查完如果消除1行则需要从这一行继续检查
						pop cx
						inc cx
						push cx
						inc dx

check_next_row:			sub di,160
						pop cx
						loop check_row

						mov bl, byte ptr var[1]			;更新分数和消除行数
						mov al,byte ptr var[3]
						add bl,dl
						add al,dl
						mov byte ptr var[1],bl
						mov byte ptr var[3],al


						pop di
						pop dx
						pop cx
						pop bx
						pop ax
						ret
```
## 四、总结

- 缺点

    1. 旋转的代码不够优美，每次旋转都要下落1行。这里可以采用图像旋转的算法来解决。
    2. 随机生成俄罗斯方块，由于代码执行的问题，第一次的俄罗斯方块会产生第一个方块和下一个方块相同的问题。
    3. 缺少暂停，重新开始功能。

以上，基本完成了俄罗斯方块的大概游戏流程。<br>

以上缺点都有办法解决，因为只是拿来练手x86汇编，因此不想花太多精力。<br>

不过有兴趣的小伙伴可以尝试着优化改进～

代码托管在git：<a href='https://github.com/hidejack/teris.git
' >https://github.com/hidejack/teris.git
</a>

<br>

完结～撒花花～滚去学win32汇编啦～