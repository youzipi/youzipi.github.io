---
title: java学习笔记（1）
date: 2014-03-08 21:24:23
categories: 
- java
tags: 
description: 数组初始化相关错误
keywords: java,java数组,无法从静态上下文中引用非静态变量&nbsp;this,intelliJ IDEA,简单的学生成绩管理程序
---
> 数组初始化相关错误

<!--more-->

错误处理：





## Exception in thread "main" java.lang.NullPointerException
	
[stackoverflow上的解答](http://stackoverflow.com/questions/5958012/exception-in-thread-main-java-lang-nullpointerexception)

`It means that you are trying to access / call a method on an object which is null.`



错误原因：

初始化了学生数组，但没有初始化数组中的每一个元素

解决方法：

重新new一个Student对象，对其录入数据，再将其付给当前的stu[i],
或者直接`stu[i] = new Student();`
对应代码块如下：
```
	Student[] stu = new Student[count];	//建立了引用，但并没有实例化
	int i;
	for(i=0;i < count;i++){
		System.out.println("input id:");
		stu[i].id = in.next();		//调用stu[i],报错
		pass
	}
```


[stackoverflow上的提问](http://stackoverflow.com/questions/22278892/whats-the-differences-between-initializing-an-array-of-objects-and-initializing)
 	关于对象数组初始化的问题，


    
```
	Student[] stu = new Student[count];
```
包含了两部分：

---- 
 
```java
	Student[] stu;	
```
- 声明。
- `stu`是Student[]的引用，也就是学生数组的引用，stu的值为`NULL`


---- 
```java
stu = new Student[count];
```
- 初始化。
- 分配`count`个Student类的内存空间，`stu`是其引用。
- 每一个元素都是`object`类型，值为`NULL`
- 此时`stu[i]`对象并未生成

对于单个对象的情况：
```java
Student stu1 = new Student();
```
- `stu1`是Student的一个引用，调用`Student()`创建一个Student对象，同时返回给`stu1`一个内存引用
- 此时，对象已生成，可使用其属性和方法。

`constructor`:`构造函数`
<br/>

## java: 无法从静态上下文中引用非静态 变量 this
	用static修饰的成员是属于类的，在static的方法里可以用类名直接调用；
	不用static修饰的成员是属于具体实例对象的，需要用对象名调用，且在static的方法里不可以调用。

```      
	public static void main(String[] args) {
        ...
	for(i=0;i < count;i++){
    	stu[i] = new Student();
```
<br/>

## Exception in thread "main" java.lang.ArrayIndexOutOfBoundsException: -1

错误原因：数组越界



IntelliJ IDEA：

* 类头注释编辑：`IDE Settings->File and Code Templates`修改`Includes->File Header`
* 自定义代码高亮，分类很细
* 	Structure:
* 自带的版本控制功能还不错
```
所有类，方法，变量的结构图：
类/方法/元素是否为静态
类/方法/元素名称
方法的返回值类型,参数/变量的数据类型，初始值
```
```java
package student_score;

import java.util.Scanner;

/**
 * project_name:student2
 * package_name:student_score
 * user: Administrator
 * date: 14-3-5
 * 成绩由外部输入
 * 对成绩排序，输出平均成绩
 * 四级：优，良，中，差
 * -没有对学生属性进行封装
 * -无错误检测：
 * --学号重复
 * --分数超出范围
 */
public class student2{
    static float total = 0;
    static int count = 0;
    //Students类定义
    public static class Student{
        String name;
        String id;
        int score;
        String grade;
        //score_to_grade
        public void setGrade() {
            switch(this.score/10){
                case 10:
                case 9:this.grade = "A";break;
                case 8:this.grade = "B";break;
                case 7:
                case 6:this.grade = "C";break;
                default:this.grade = "D";break;
            }
        }
        //
        public void showInfo() {
            this.setGrade();
            System.out.println(this.id +"  "+this.name +"  "+this.score+"  "+this.grade);
        }
    }
    //排序
    public static void sort(Student[] stu){
        Student temp = new Student();
        for(int i=1;i < count;i++){
            for(int j=0;j < i;j++){
                if(stu[i].score > stu[j].score)
                    temp = stu[i];
                    stu[i] = stu[j];
                    stu[j] = temp;
            }
        }
    }
    //main
    public static void main(String[] args) {
        System.out.println("Input Students' Number:");
        Scanner in = new Scanner(System.in);
        count = in.nextInt();
        Student[] stu = new Student[count];
        int i;
        for(i=0;i < count;i++){
            stu[i] = new Student();
            System.out.println("input id:");
            stu[i].id = in.next();
            System.out.println( "input name:");
            stu[i].name = in.next();
            System.out.println( "input score:");
            stu[i].score = in.nextInt();
            //stu[i] = stu1;
        }
        sort(stu);
        System.out.println( "ID NAME SCORE GRADE");
        for(Student s:stu){
            total += s.score;
            s.showInfo();
        }
        System.out.println( "AVERAGE："+total/count);
    }
}
```
