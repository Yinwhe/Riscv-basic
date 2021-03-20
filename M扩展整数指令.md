# Multiplication Operations

![M%E6%89%A9%E5%B1%95%E6%95%B4%E6%95%B0%E6%8C%87%E4%BB%A4%207010bdca4b674a95b9f2d7181c185887/Screen_Shot_2021-03-20_at_6.31.22_PM.png](https://tva1.sinaimg.cn/large/008eGmZEly1goqleu86kqj314w06qmyg.jpg)

**mul rd, rs1, rs2**

- 将`XLEN`的`rs1`和`rs2`相乘，并将结果的**低**`XLEN`位放到`rd`中，忽略算术溢出

**mulh rd, rs1, rs2**

- 将乘积的**高**`XLEN`位放到`rd`中

**mulhsu rd, rs1, rs2**

- 同mulh，但是是符号数和非符号数相乘

**mulhu rd, rs1, rs2**

- 同mulh，但是是无符号数相乘

如果乘法的高低位都需要的话，建议使用这样的顺序：

```
MULH[[S]U] rdh, rs1, rs2;
MUL rdl, rs1, rs2;
```

- 源寄存器的顺序必须一样，rdh也不能是rs1或者rs2，这样的条件下，微架构会将上述指令调整为**一次操作**以加快速度。

**mulw rd, rs1, rs2 (RV64 Only)**

- 将乘积的**低**`32`位进行符号扩展后写入`rd`

# Division Operations

![M%E6%89%A9%E5%B1%95%E6%95%B4%E6%95%B0%E6%8C%87%E4%BB%A4%207010bdca4b674a95b9f2d7181c185887/Screen_Shot_2021-03-20_at_6.57.24_PM.png](https://tva1.sinaimg.cn/large/008eGmZEly1goqlezprkxj30qw047dgd.jpg)

**div rd, rs1, rs2**

- 将`rs1`除以rs2，结果向0舍入，然后将商存到rd中

**divu rd, rs1, rs2**

- 同`div`，但是是无符号数除法

**rem rd, rs1, rs2**

- 将`div`的余数写入`rd`中

**remu rd, rs1, rs2**

- 将`divu`的余数写入`rd`中

对于有符号数的除法，余数的符号和被除数的符号保持一致

如果同时需要除法的商和余数，建议写为：

```
DIV[U] rdq, rs1, rs2;
REM[U] rdr, rs1, rs2;
```

- 这样微架构就会将这两条指令优化为一次操作，加快速度；当然要求和上面的乘法是类似的

**divw rd, rs1, rs2 (RV64 Only)**

- 用`rs1`的**低32位**除以`rs2`的**低32位**，向0舍入，将32位的商符号扩展后存入rd中

**divuw rd, rs1, rs2 (RV64 Onlu)**

- 同divw，是无符号数的除法

**remw rd, rs1, rs2**

- 将`divw`的余数符号扩展后写入`rd`

**remuw rd, rs1, rs2**

- 将`divuw`的余数符号扩展写入`rd`

当除0时，无论是符号数还是非符号数的除法，都将把商全部bit置1，而余数和被除数是相等的；Overflow当且仅当最大的负数$-2^{L-1}$除以(-1)的时候hui f

![M%E6%89%A9%E5%B1%95%E6%95%B4%E6%95%B0%E6%8C%87%E4%BB%A4%207010bdca4b674a95b9f2d7181c185887/Screen_Shot_2021-03-20_at_7.20.54_PM.png](https://tva1.sinaimg.cn/large/008eGmZEly1goqlf0xt0lj30q4039mxi.jpg)