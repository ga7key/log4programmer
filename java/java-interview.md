
### try-catch-finally 的执行顺序
1. 不管有木有出现异常，finally 块中代码都会执行；
2. 当 try 和 catch 中有 return 时，finally 仍然会执行；
3. finally 是在 return 后面的表达式运算后执行的（此时并没有返回运算后的值，而是先把要返回的值保存起来，不管 finally 中的代码怎么样，返回的值都不会改变，仍然是之前保存的值），所以函数返回值是在 finally 执行前确定的；
4. finally 中最好不要包含 return，否则程序会提前退出，返回值不是 try 或 catch 中保存的返回值。

举例：

情况 1：
```java
try{} 
    catch(){}
    finally{} 
    return;
```

显然程序按顺序执行。

情况 2:
```java
try{ return; }
    catch(){} 
    finally{} 
    return;
```

程序执行 try 块中 return 之前（包括 return 语句中的表达式运算）代码；  
再执行 finally 块，最后执行 try 中 return;  
finally 块之后的语句 return，因为程序在 try 中已经 return 所以不再执行。

情况 3:
```java
try{ } 
    catch(){ return; } 
    finally{} 
    return;
```

程序先执行 try，如果遇到异常执行 catch 块，  
有异常：则执行 catch 中 return 之前（包括 return 语句中的表达式运算）代码，再执行 finally 语句中全部代码，最后执行 catch 块中 return. finally 之后的代码不再执行。  
无异常：执行完 try 再 finally 再 return.

情况 4:
```java
try{ return; }
    catch(){} 
    finally{ return; }
```

程序执行 try 块中 return 之前（包括 return 语句中的表达式运算）代码；  
再执行 finally 块，因为 finally 块中有 return 所以提前退出。

情况 5:
```java
try{} 
    catch(){ return; }
    finally{ return; }
```

程序执行 catch 块中 return 之前（包括 return 语句中的表达式运算）代码；  
再执行 finally 块，因为 finally 块中有 return 所以提前退出。

情况 6:
```java
try{ return; }
    catch(){ return; } 
    finally{ return; }
```

程序执行 try 块中 return 之前（包括 return 语句中的表达式运算）代码；  
有异常：执行 catch 块中 return 之前（包括 return 语句中的表达式运算）代码；则再执行 finally 块，因为 finally 块中有 return 所以提前退出。  
无异常：则再执行 finally 块，因为 finally 块中有 return 所以提前退出。

 

<span style="color: red;font-weight: bold;">最终结论：</span>  
任何执行 try 或者 catch 中的 return 语句之前，都会先执行 finally 语句，如果 finally 存在的话。  
如果 finally 中有 return 语句，那么程序就 return 了，所以 finally 中的 return 是一定会被返回的，编译器把 finally 中的 return 实现为一个 warning。

下面是个测试程序
```java
public class FinallyTest {
    public static void main(String[] args) {
        System.out.println(new FinallyTest().test());
    }

    static int test() {
        int x = 1;
        try { x++; return x; }
        finally { ++x; }
    }
}
```

执行结果是 2。

分析：  
在 try 语句中，在执行 return 语句时，要返回的结果已经准备好了，就在此时，程序转到 finally 执行了。  
在转去之前，try 中先把要返回的结果存放到不同于 x 的局部变量中去，执行完 finally 之后，在从中取出返回结果。  
因此，即使 finally 中对变量 x 进行了改变，但是不会影响返回结果（返回值保存在栈中了）。

### 父子类执行顺序
Java 中，new 一个类的对象，类里面的静态代码块、非静态代码、无参构造方法、有参构造方法、类的一般方法等部分，它们的执行顺序相对来说比较简单，用程序也很容易验证。比如新建一个测试父类。

```java
public class FatherTest {

    private String name;

    FatherTest(){
       System.out.println ("-- 父类的无参构造函数 --");
    }

    FatherTest(String name){
       this.name=name;
       System.out.println ("-- 父类的有参构造函数 --"+this.name);
    }

    static{
       System.out.println ("-- 父类的静态代码块 --");
    }

    {
       System.out.println ("-- 父类的非静态代码块 --");
    }

    public void speak(){
       System.out.println ("-- 父类的方法 --");
    }
}
//用main()方法调用FatherTest类
public static void main(String[] args) {
    System.out.println ("-- 父类主程序 --");
    FatherTest father=new FatherTest ("父亲的名字");
    father.speak();
}
```

执行结果为：

    -- 父类的静态代码块 --
    -- 父类主程序 --
    -- 父类的非静态代码块 --
    -- 父类的有参构造函数 -- 父亲的名字
    -- 父类的方法 --

执行顺序如下：  
静态代码块 — 主程序 — 非静态代码块 — 构造函数 — 一般方法

如果加入子类的继承以后，情况就会变得复杂些。比如再新建一个测试子类。

```java
public class SonTest extends FatherTest {

    private String name;

    static{
       System.out.println ("-- 子类的静态代码块 --");
    }

    {
       System.out.println ("-- 子类的非静态代码块 --");
    }

    SonTest(){
       System.out.println ("-- 子类的无参构造函数 --");
    }

    SonTest(String name){
       this.name=name;
       System.out.println ("-- 子类的有参构造函数 --"+this.name);
    }

    @Override
    public void speak(){
       System.out.println ("-- 子类 Override 了父类的方法 --");
    }  
}
//用main()方法调用FatherTest类和SonTest类
public static void main(String[] args) {
       System.out.println ("-- 子类主程序 --");

       FatherTest father=new FatherTest ("父亲的名字");
       father.speak();

       SonTest son=new SonTest ("儿子的名字");
       son.speak();
}
```

执行结果为：

    -- 父类的静态代码块 --
    -- 子类的静态代码块 --
    -- 子类主程序 --
    -- 父类的非静态代码块 --
    -- 父类的有参构造函数 -- 父亲的名字
    -- 父类的方法 --
    -- 父类的非静态代码块 --
    -- 父类的无参构造函数 --
    -- 子类的非静态代码块 --
    -- 子类的有参构造函数 -- 儿子的名字
    -- 子类 Override 了父类的方法 --

加入了子类以后，执行顺序有了新的变化：  
1. 首先执行的是 父类的静态代码块 — 子类的静态代码块 — 主程序。这一部分都是执行一次，与建立多少对象没有关系。
2. 新建了一个父类对象，并调用speak()方法。执行了 父类的非静态代码块 — 构造函数 — 一般方法。
3. 新建了一个子类的对象，并调用speak()方法。执行了 父类的非静态代码块 — 父类的无参构造函数，然后是 子类的非静态代码块 — 子类构造函数 — 子类的方法。