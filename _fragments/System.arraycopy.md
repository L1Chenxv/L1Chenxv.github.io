---
layout: fragment
title: System.arraycopy() 的拷贝方式
tags: [Java]
description: some word here
keywords: java
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---



直接上结论：**浅拷贝**！

## 分析

**对象数组的复制**

```java
public static void main(String[] args) {
    User[] users = new User[] { 
            new User(1, "seven", "seven@qq.com"), 
            new User(2, "six", "six@qq.com"),
            new User(3, "ben", "ben@qq.com") };// 初始化对象数组
    
    User[] target = new User[users.length];// 新建一个目标对象数组
    
    System.arraycopy(users, 0, target, 0, users.length);// 实现复制
    
    System.out.println("源对象与目标对象的物理地址是否一样：" + (users[0] == target[0] ? "浅复制" : "深复制"));  //浅复制
    
    target[0].setEmail("admin@sina.com");
    
    System.out.println("修改目标对象的属性值后源对象users：");
    for (User user : users) {
        System.out.println(user);
    }
}

```

最终输出`true`，说明System.arraycopy() 在拷贝数组的时候，采用的使用潜复制，复制结果是一维的引用变量传递给副本的一维数组，修改副本时，会影响原来的数组。

**一维数组和多维数组的复制的区别**

```java
    String[] st  = {"A","B","C","D","E"};
    String[] dt  = new String[5];
    System.arraycopy(st, 0, dt, 0, 5);
    
    //改变dt的值
    dt[3] = "M";
    dt[4] = "V";
    
    System.out.println("两个数组地址是否相同：" + (st == dt)); //false
    
    for(String str : st){
        System.out.print(" " + str +" ");   // A  B  C  D  E 
        
    }
    System.out.println(); 
    for(String str : dt){
        System.out.print(" " + str +" ");   // A  B  C  M  V 
    }

```

使用该方法对一维数组在进行复制之后，目标数组修改不会影响原数据，这种复制属性值传递，修改副本不会影响原来的值。

但是，请重点看以下代码：

```java
    String[] st  = {"A","B","C","D","E"};
    String[] dt  = new String[5];
    System.arraycopy(st, 0, dt, 0, 5);

   for(String str : st){
        System.out.print(" " + str +" ");   // A  B  C  D  E 
        
    }
    System.out.println(); 
    for(String str : dt){
        System.out.print(" " + str +" ");   // A  B  C  D  E 
    }

    System.out.println("数组内对应位置的String地址是否相同:" + st[0] == dt[0]); // true

```

为什么会出现以上的情况呢？

通过以上代码可以推断，在System.arraycopy()进行复制的时候，首先检查了字符串常量池是否存在该字面量，一旦存在，则直接返回对应的内存地址，如不存在，则在内存中开辟空间保存对应的对象。

**二维数组的复制**

```java
    String[][] s1 = {
                {"A1","B1","C1","D1","E1"},
                {"A2","B2","C2","D2","E2"},
                {"A3","B3","C3","D3","E3"}
                    };
    String[][] s2 = new String[s1.length][s1[0].length];  
    
    System.arraycopy(s1, 0, s2, 0, s2.length);  
    
    for(int i = 0;i < s1.length ;i++){ 
     
       for(int j = 0; j< s1[0].length ;j++){  
          System.out.print(" " + s1[i][j] + " ");
       }  
       System.out.println();  
    }  
    
    //  A1  B1  C1  D1  E1 
    //  A2  B2  C2  D2  E2 
    //  A3  B3  C3  D3  E3 
    
    
    s2[0][0] = "V";
    s2[0][1] = "X";
    s2[0][2] = "Y";
    s2[0][3] = "Z";
    s2[0][4] = "U";
    
    System.out.println("----修改值后----");  
    
    
    for(int i = 0;i < s1.length ;i++){  
           for(int j = 0; j< s1[0].length ;j++){  
              System.out.print(" " + s1[i][j] + " ");
           }  
           System.out.println();  
     }  

    //  Z   Y   X   Z   U 
    //  A2  B2  C2  D2  E2 
    //  A3  B3  C3  D3  E3 

```

上述代码是对二维数组进行复制，数组的第一维装的是一个一维数组的引用，第二维里是元素数值。对二维数组进行复制后后，第一维的引用被复制给新数组的第一维，也就是两个数组的第一维都指向相同的“那些数组”。而这时改变其中任何一个数组的元素的值，其实都修改了“那些数组”的元素的值，所以原数组和新数组的元素值都一样了。

## 结论

System.arraycopy() 在拷贝数组的时候，采用的使用潜复制，复制结果是一维的引用变量传递给副本的一维数组，修改副本时，会影响原来的数组。