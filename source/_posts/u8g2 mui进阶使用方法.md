---
title: u8g2 mui进阶使用方法
date: 2025.10.15 13:24:53
toc: true
categories: 
  - 嵌入式
tags:
  - u8g2
  - STM32
---
u8g2作为著名的OLED显示资源库，在嵌入式系统中有着广泛应用。但是，大多数人使用u8g2，往往局限于其基本功能，例如`u8g2_DrawStr`，`u8g2_DrawLine`等。如果想用u8g2做多级菜单，大多数人会额外编写很多代码或者引用另外一个库，却忽略了u8g2自带一个多级菜单库——mui。网络上对mui的介绍少之又少，这里我结合最近的一个项目，详细介绍一下mui的用法，并且扩展一些进阶用法，实现更加复杂的功能。
<!-- more -->
## 开发环境
我在STM32上移植了u8g2的所有库，使用Keil MDK+STM32CubeMX共同开发。使用的OLED是SSD1306，通过i2C接口与之相连，IIC速率为400kbps。为了减少CPU占用，尽可能使用DMA方式传输OLED数据。
## u8g2使用
u8g2的使用方法，网上介绍的已经很多了。移植重点分为以下几点：
1. 把所有.c文件加入Keil MDK中
2. 选择合适的初始化函数，并注释掉其余函数。我选择的是`u8g2_Setup_ssd1306_128x64_noname_f`
3. 选择合适的缓存函数，并注释掉其余函数。初始化函数中，只调用了`u8g2_m_16_8_f`这一个缓存函数，因此注释掉其他的。
4. 编写`u8x8_gpio_and_delay`函数和`u8x8_byte_xxx`函数。如果是硬件IIC/SPI，则需要自己编写后者，否则可以使用u8g2自带的软件IIC和SPI函数。
本篇博文重点在于mui的使用，因此不再赘述。

## mui使用
### mui介绍
官方Github介绍，mui是u8g2的一个小插件，目的是方便用u8g2创建一个能够与用户交互的界面。对于嵌入式应用来说，它体量很小，适合用在嵌入式系统中。当然，体量小意味着开发起来是有一定难度的。好在mui内置的界面组件够多，能够满足简单的需求，因此学会了之后，用起来还是挺方便的。
mui与u8g2是共同使用的，必须先把u8g2调通了之后，再开发mui。
mui的菜单界面、功能主要由两个东西决定，分别叫做MUIF和FDS。我们先把两个名词弄清楚。MUIF即MUI Function，决定了一个UI组件的所有功能（包括绘制！！！），与组件的位置、排布等无关。FDS即“Form Definition String”，决定了UI组件的排列、页面布局等设计内容，与功能关系不大。
### MUIF介绍
MUIF决定了组件的功能。它的组成方式，与word中的样式很相似。一个MUIF可以对应多个组件，这些组件的功能是一样的，类似于word中，不同段落的文字，只要是同一个样式，那么格式就是一样的。
有关MUIF。我举几个例子。
1. 假设一个Label类型的组件，他不能直接显示在屏幕上，想要把它写在屏幕上，需要调用对应的MUIF，MUIF调用u8g2的函数，使其显示在屏幕上。但是，Label的内容不是MUIF决定的。Label的内容对MUIF来说，就是一个函数参数，直接传给u8g2显示出来而已。
2. 假设一个Button类型的组件，他的显示与Label相同，需要调用MUIF。同时，它的按下、抬起等交互功能，也需要调用MUIF实现。比如，按下跳转到另一个页面，本质上是按下按钮后，调用`mui_SendSelect`，让mui知道按钮被按下了。mui就会寻找这个按钮的MUIF，调用MUIF中的有关按钮按下的代码，实现用户需要的各种功能。
3. 假设有两个Button位于同一页面，一个是跳转到下一页面，一个是返回上一页面。他俩的功能是不一样的，因此，他俩应该使用不同的MUIF。每个MUIF都有一个独特的、自定义的ID，不同按钮的可以选择不同的ID，即对应到不同的MUIF。
4. 假设两个Button，功能相同，都是跳转页面，但是两个按钮跳转到不同的页面。那么他们应当调用同一个MUIF，即使用同一个ID。只不过给MUIF的参数不同，跳转页面1，就传入1，跳转页面2，就传入2。
这几个例子，我觉得能够把MUIF的作用解释清楚了。
下面我们实际看一个MUIF如何设计。这是一个mui内置的MUIF，能够实现常见的按下跳转功能。
```c
uint8_t mui_u8g2_btn_goto_wm_fi(mui_t *ui, uint8_t msg)
{
  switch(msg)
  {
    case MUIF_MSG_DRAW:
      mui_u8g2_draw_button_utf(ui, U8G2_BTN_HCENTER |mui_u8g2_get_fi_flags(ui), 0, 1, MUI_U8G2_V_PADDING, ui->text);
      break;
    case MUIF_MSG_FORM_START:
      break;
    case MUIF_MSG_FORM_END:
      break;
    case MUIF_MSG_CURSOR_ENTER:
      break;
    case MUIF_MSG_CURSOR_SELECT:
    case MUIF_MSG_VALUE_INCREMENT:
    case MUIF_MSG_VALUE_DECREMENT:
      //return mui_GotoForm(ui, ui->arg, 0);
      mui_SaveForm(ui);
      return mui_GotoFormAutoCursorPosition(ui, ui->arg);
    case MUIF_MSG_CURSOR_LEAVE:
      break;
    case MUIF_MSG_TOUCH_DOWN:
      break;
    case MUIF_MSG_TOUCH_UP:
      break;    
    
  }
  return 0;
}
```
可以看到，MUIF是以msg为框架的。不同的msg会对应到不同的代码。如果输入一个`MUIF_MSG_DRAW`，那么`mui_u8g2_draw_button_utf`这个函数就会被调用，从而在屏幕上绘制一个按钮出来。
我们常用的msg是`MUIF_MSG_CURSOR_SELECT`，即按钮按下事件。现在这个MUIF里面，设计的是按下后，保存当前表单，跳转到另一个表单。
### 进阶用法1
假设，我们需要，按下按钮后，让某一个变量+1，然后跳转到另一个表单，怎么做呢？
```c
uint8_t variable;

uint8_t mui_u8g2_btn_add_goto_wm_fi(mui_t *ui, uint8_t msg)
{
	switch(msg)
	{
		case MUIF_MSG_CURSOR_SELECT:
			variable++;
		default:
			return mui_u8g2_btn_goto_wm_fi(ui, msg);
	}
}
```
自己定义一个函数，这个函数也仿照内置的MUIF编写，对`msg`进行switch，如果`msg`是`MUIF_MSG_CURSOR_SELECT`，那么就让`variable`自增。而且，自增完后，因为没加`break`，他还会调用本来的MUIF，实现跳转功能。除了`MUIF_MSG_CURSOR_SELECT`以外的`msg`，也会直接进入default，让本来的MUIF进行处理。我们只需要添加“按下按钮变量+1”的功能，因此只需要修改`msg`是`MUIF_MSG_CURSOR_SELECT`时的代码，其他的都可以交给原来的MUIF进行处理。
构造好想要的MUIF函数之后，就可以构造一个MUIF数组，与FDS建立关联。仍然是举个例子
```c
muif_t muif_list[] = {  
  MUIF_U8G2_FONT_STYLE(0, u8g2_font_helvR08_tr),   /* define style 0 */
  MUIF_U8G2_LABEL(),                               /* allow MUI_LABEL macro */
  MUIF_BUTTON("BN", mui_u8g2_btn_add_goto_wm_fi)       /* define exit button */
};
```
这个`muif_list`首先定义了一个字体样式，样式ID为0，字体为`u8g2_font_helvR08_tr`。启用`MUI_LABEL`这一FDS描述符。定义了一个按钮样式，样式ID为"BT"，样式的回调函数是`mui_u8g2_btn_add_goto_wm_fi`。这样一个MUIF就把很多特性全部定义好了。只要是ID为"BT"的组件，都会按照这个MUIF定义的功能执行。只要是ID为0的字体，都使用`u8g2_font_helvR08_tr`这个字体。  
这里需要澄清一下两个ID的关系。一种ID是两位字符型变量。例如`"BT"`。这个ID与我们平时理解的数字ID，有显著区别。另一个ID是一个8位无符号二进制数，用在Form的ID、字体样式的ID等地方。这就限制了mui最多可以定义255个字体、255个表单，字体选择、表单之间的跳转也是根据ID进行的。
### FDS介绍
FDS我认为是mui中比较败笔的一个部分，虽然能够实现功能，但是实现起来太难理解、维护困难、代码冗长、逻辑混乱。
FDS是一个字符串，不能把它理解为数组。为FDS初始化时，不能在两个macro之间加逗号，必须紧连着，实现一个连续不断的字符串的效果。
```c
fds_t fds_data[] = 
	MUI_FORM(1)      // no comma allowed, because fds_data is a string
	MUI_STYLE(0)
	MUI_XYT("BN",64, 30, " Select Me ")
;
```
这样一个FDS字符串定义了一个表单（Form）。表单就是我们日常理解的页面，跳转到另一个表单，就是跳转到另一个页面。这个FDS只定义了一个Form。`MUI_STYLE(0)`代表了使用第0号字体，`MUI_XYT`定义了一个具有横坐标，纵坐标，内部文字三个属性的元素，且该元素使用ID为“BN”的MUIF。
FDS直接决定页面布局。定义了上述这一个FDS后，第一个页面中，就会在(64,30)的地方出现一个元素，内部有“Select Me”文字。不过，这个元素的外貌并不取决于FDS，而是取决于MUIF中，msg为`MUIF_MSG_DRAW`时的回调函数。这个回调函数画出来的就是这个元素的外貌。FDS决定了位置、文字和可能的参数（使用`MUI_XYA`或`MUI_XYAT`时可以定义参数，参数一般都是跳转目标页面号）。 
### 进阶用法2
显然，FDS很难被修改。这就导致mui生成的多级菜单比较固定，很难做到动态修改菜单内容。不过，这一点官方也提供了一种解决方法。先看代码。下面这段代码来自于[https://github.com/olikraus/u8g2/wiki/muiref#scrollable-jump-table-dynamic]
```c
uint16_t menu_get_cnt(void *data) {
  return 10;    /* number of menu entries */
}

const char *menu_get_str(void *data, uint16_t index) {
  static const char *menu[] = { 
    MUI_1 "Goto Main Menu",
    MUI_10 "Enter a number",
    MUI_11 "Parent/Child Selection",
    MUI_13 "Checkbox",
    MUI_14 "Radio Selection",
    MUI_15 "Text Input",
    MUI_16 "Single Line Selection",
    MUI_17 "List Line Selection",
    MUI_18 "Parent/Child List",
    MUI_20 "Array Edit",
  };
  return menu[index];
}
uint16_t selection = 0;

muif_t muif_list[] = {
	MUIF_U8G2_FONT_STYLE(0, u8g2_font_helvR08_tr),   /* define style 0 */
	MUIF_U8G2_LABEL(),
	MUIF_U8G2_U16_LIST("ID", &selection, NULL, menu_get_str, menu_get_cnt, mui_u8g2_u16_list_goto_w1_pi),
};

fds_t fds_data[] = 
	MUI_FORM(3)
	MUI_STYLE(0)
	MUI_XYA("ID", 5, 25, 0) 
	MUI_XYA("ID", 5, 37, 1) 
	MUI_XYA("ID", 5, 49, 2) 
	MUI_XYA("ID", 5, 61, 3) 
;
```
代码实现的核心，是一个叫做`MUIF_U8G2_U16_LIST`的MUIF。这个函数具体的实现我也不懂，不过借助这个函数可以构造出一个可滚动列表，由于FDS定义了四个该ID的对象，因此这个列表只会显示四个元素，元素会随着滚动而变化。如果元素数量小于四，程序也会自动禁用滚动功能。所以只要改变这个列表，就可以改变菜单的元素。官方的`menu_get_cnt`和`menu_get_str`实现了动态获取列表长度和列表元素的功能。虽然官方给的这俩函数，返回值是固定的，但是放到自己的项目上，只要满足返回值的规则，可以自己随意的修改这个函数，来返回自己想要显示在屏幕上的内容。
`menu_get_str`返回的字符串可以拆分为两部分，第一个字节所代表的数字，是这一行按下后会跳转到的目标表单序号。第一个字节后的内容，就是该行的显示文字。这里一定要注意，目标表单序号是字符串的第一个字节，以二进制数的形式存储，而不是字符形式（ASCII码）。
下面我举一个例子，把一个链表的所有元素显示到屏幕上。这里只实现`menu_get_cnt`和`menu_get_str`两个函数，FDS和MUIF与上面的代码相同。
```c
// 定义链表数据结构体
typedef struct {
	char str[32];
	node_t *next;
} node_t;

extern node_t *head;

/* 
 假设已经实现了
 uint32_t List_GetLength(node_t* head);
 char* List_GetStr(node_t* head, uint32_t index);
*/

uint16_t menu_get_cnt(void *data) {
	return List_GetLength(head);    /* number of menu entries */
}

const char *menu_get_str(void *data, uint16_t index) {
	// 必须定义成静态变量，因为该变量会以指针的形式返回到函数外
	static char str[32];
	// 假设第一个按钮跳转到1号表单，第二个按钮跳转到2号表单。向第一个元素写入表单ID
	str[0] = index+1;
	// 从str的第二个元素开始，把链表字符串复制到str中。
	strcpy(str+1, (const char*)List_GetStr(head, index));
	// 返回str指针
	return (const char*)str;
}
```
每次调用`menu_get_str`函数时，都会返回对应的链表的字符串。随着链表的修改，增加，删除，得到的字符串和总长度也会随之变化，这样就实现了一个符合自己需求的动态菜单。

## 结语

mui作为一个伴随u8g2出现的轻量级多级菜单库，它的功能已经远超我的预期，配合自己编写的回调函数，能够满足绝大多数的需求。但是，轻量化所带来的抽象，使得理解和编写代码变得困难。本文算是填补了网络上有关mui的空白，尤其是介绍了重写mui回调函数的基本方法，实现mui与已有项目的良好融合。