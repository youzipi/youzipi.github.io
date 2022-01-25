---
title: java源码学习
date: 2014-04-08 10:38:13
categories: 
- 他山之玉
tags: 
- java
comments: true
description:  
keywords: java源码,parseInt,valueOf

---

<!--more-->

在51CTO上看java视频的时候，看到了`parseInt`这个函数，是`Integer`的方法，
看到了`valueOf`这个函数，是`String`的方法，

```java
  int num = 100;
  //String s = String(num);
  String s = String.valueOf(num);
```

在`String.java`中：

```java
        public static String valueOf(int i) {
                return Integer.toString(i);
                }
```

`Integer.java`:

```java
//Integer.java
        public static String toString(int i) {
                if (i == Integer.MIN_VALUE)
                return "-2147483648";
                int size = (i < 0) ? stringSize(-i) + 1 : stringSize(i);
                char[] buf = new char[size];
                getChars(i, size, buf);
                return new String(buf, true);
                }


        static int stringSize(int x) {
                for (int i=0; ; i++)
                    if (x <= sizeTable[i])
                        return i+1;//进位
                }
		static void getChars(int i, int index, char[] buf) {
                int q, r;
                int charPos = index;//index <-size
                char sign = 0;

                if (i < 0) {
                sign = '-';
                i = -i;
                }
                // Fall thru to fast mode for smaller numbers
                // assert(i <= 65536, i);
                for (;;) {
                q = (i * 52429) >>> (16+3);//q = i*(52429/2^16)/2^3 = 52429/524288 ?= 0.1//2^19
                r = i - ((q << 3) + (q << 1));  // r = i-(q*10) ...
                buf [--charPos] = digits [r];//逆序存入数组
                i = q;
                if (i == 0) break;
                }
                if (sign != 0) {
                buf [--charPos] = sign;
                }
                }


```
为了提高效率，使用了`查表(空间换时间)`和`位运算`

用到的数组

```java
//Integer.java
        final static int [] sizeTable = { 9, 99, 999, 9999, 99999, 999999, 9999999,
                99999999, 999999999, Integer.MAX_VALUE };

        final static char [] DigitOnes = {
                '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
                '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
                '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
                '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
                '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
                '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
                '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
                '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
                '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
                '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
                } ;

        final static char [] DigitTens = {
                '0', '0', '0', '0', '0', '0', '0', '0', '0', '0',
                '1', '1', '1', '1', '1', '1', '1', '1', '1', '1',
                '2', '2', '2', '2', '2', '2', '2', '2', '2', '2',
                '3', '3', '3', '3', '3', '3', '3', '3', '3', '3',
                '4', '4', '4', '4', '4', '4', '4', '4', '4', '4',
                '5', '5', '5', '5', '5', '5', '5', '5', '5', '5',
                '6', '6', '6', '6', '6', '6', '6', '6', '6', '6',
                '7', '7', '7', '7', '7', '7', '7', '7', '7', '7',
                '8', '8', '8', '8', '8', '8', '8', '8', '8', '8',
                '9', '9', '9', '9', '9', '9', '9', '9', '9', '9',
                } ;
```