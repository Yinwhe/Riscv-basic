# 接受参数的宏

> 下列例中的 `define` 都没有加 `#` 源于代码着色的问题，这样更清楚，注意加上

**`##` 连接符**

- 功能是在带参数的宏定义中将两个子串联接起来，从而形成一个新的子串，应该注意到是，**这里指的不是字符串**。

```c
define test(n) Thistest##n //那么
test(1) => Thistest1

define D(name, type)  type name_##type##_type 
D(a, int);  /* 等价于: int name_int_type; */
```

**Note**

- `##` 前后的空格有无都可

- 连接后形成的字串应该能够被识别

  

**`#` 符号**

- 把传递过来的参数当成字符串进行替代。其只能用于有传入参数的宏定义中，且必须置于宏定义体中的参数名前。

```c
define tran(str) #str

tran(abc) => "abc"
```

**Note**

- 会忽略传入参数名前面和后面的空格
- 当传入参数名间存在空格时，编译器将会自动连接各个子字符串，用每个子字符串中只以一个空格连接，忽略其中多余一个的空格。如： `str=tran( abc   def);` 将会被扩展成 `str="abc def";`



**`@#` 字符化操作符**

- 只能用于有传入参数的宏定义中，且必须置于宏定义体中的参数名前。作用是**将传的单字符参数名转换成字符**，以一对单引用括起来。

```c
define makechar(x)  #@x
a = makechar(b);
//a = 'b'
```



### 在写riscv内核时的应用

因为需要一个能在C里更改寄存器，而如果每一个寄存器都定义一个单独的宏展开的话，会很占位置，所以考虑使用带参数的宏：

```c
define w_csr(csr, para) asm("csrw " #csr ", %0"::"r"(para))
define r_csr(csr, para) asm("csrr %0, " #csr :"=r"(para))
define c_csr(csr, para) asm("csrc " #csr ", %0"::"r"(para))
define s_csr(csr, para) asm("csrs " #csr ", %0"::"r"(para))
```

- 关于内联汇编参见之前