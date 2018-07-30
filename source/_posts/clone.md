---
title: java的浅拷贝和深拷贝
date: 2018-7-26 14:11:52
tags: Java
---

> 浅拷贝

所谓浅拷贝，是指原对象与拷贝对象公用一份实体，仅仅是对象名字不同而已（类似引用，即对原对象起别名），其中任何一个对象改变都会导致其他的对象也跟着它变。

> 深拷贝

所谓深拷贝，就是为新对象开辟一块新的空间，并将原对象的内容拷贝给新开的空间，释放时就不会牵扯到多次析构的问题。

所有对象都继承了父类Object，Object有一个protected修饰的clone()方法：

> protected native Object clone() throws CloneNotSupportedException;

<!-- more -->

当我们想实现一个类的浅拷贝，只需实现Cloneable接口，override父类的clone()方法

```
import java.io.Serializable;
import java.util.List;

public class Man implements Cloneable, Serializable {
    private int age;

    private String name;

    private List<Man> girls;

    public Man(int age, String name, List<Man> girls) {
        this.age = age;
        this.name = name;
        this.girls = girls;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public List<Man> getGirls() {
        return girls;
    }

    public void setGirls(List<Man> girls) {
        this.girls = girls;
    }

    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }

    @Override
    public String toString() {
        return "Man{" +
                "age=" + age +
                ", name='" + name + '\'' +
                ", girls=" + girls +
                '}';
    }
}
```

> 测试如下

```
import java.util.ArrayList;
import java.util.List;

public class Main {

    public static void main(String[] args) {
        Man girl = new Man(18, "美女", null);
        List<Man> girls = new ArrayList<>(1);
        girls.add(girl);
        Man man = new Man(25, "帅哥", girls);
        Man clone = null;
        try {
            clone = (Man) man.clone();
        } catch (CloneNotSupportedException e) {
            e.printStackTrace();
        }
        man.setAge(20);
        man.setName("美女");
        man.getGirls().get(0).setAge(20);
        man.getGirls().get(0).setName(null);
        System.out.println(man);
        System.out.println(clone);
    }
}
```

> 运行结果如下

Man{age=20, name='美女', girls=[Man{age=20, name='null', girls=null}]}

Man{age=25, name='帅哥', girls=[Man{age=20, name='null', girls=null}]}

> 分析一下，age是int基础数据类型，clone的时候应该是值传递，name是String类型对象，查看SString的源码,没有实现Cloneable接口，但按照运行结果来看对象clone的时候String类型的属性应该是new了一个新的String对象和原来的String属性一模一样，并指向clone对象的String类型的属性，但List<T>确实浅拷贝，只拷贝了原对象的引用，并没有拷贝原对象的值.

```
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
```
> 这里有个地方要注意：Arrays.asList(T... a)返回的的ArrayList对象并不是java.util.ArrayList,而是Arrays的一个一个静态内部类，
java.util.ArrayList实现了Cloneable接口，而Arrays.asList(T... a)返回的的ArrayList没有实现Cloneable接口。

```
    public static <T> List<T> asList(T... a) {
        return new ArrayList<>(a);
    }
    
    private static class ArrayList<E> extends AbstractList<E>
        implements RandomAccess, java.io.Serializable

```

> 从ArrayList的源码可以看出实现的是浅拷贝

```
    /**
     * Returns a shallow copy of this <tt>ArrayList</tt> instance.  (The
     * elements themselves are not copied.)
     *
     * @return a clone of this <tt>ArrayList</tt> instance
     */
    public Object clone() {
        try {
            ArrayList<?> v = (ArrayList<?>) super.clone();
            v.elementData = Arrays.copyOf(elementData, size);
            v.modCount = 0;
            return v;
        } catch (CloneNotSupportedException e) {
            // this shouldn't happen, since we are Cloneable
            throw new InternalError(e);
        }
    }
```

> 假如在开发环境的时候需要clone集合的时候，就很容易出现隐形的bug，下面给出集合的深拷贝的解决方案：就是使用序列化和反序列化，要考虑的对象要实现Serializable接口

```
import java.io.*;
import java.util.ArrayList;
import java.util.List;

public class Main {

    public static void main(String[] args) {
        Man girl = new Man(18, "美女", null);
        List<Man> girls = new ArrayList<>(2);
        girls.add(girl);
        Man man = new Man(25, "帅哥", girls);
        Man clone = null;
        try {
            clone = (Man) man.clone();
            clone.setGirls(depCopy(clone.getGirls()));
        } catch (CloneNotSupportedException e) {
            e.printStackTrace();
        }
        man.setAge(20);
        man.setName("美女");
        man.getGirls().get(0).setAge(20);
        man.getGirls().get(0).setName(null);
        System.out.println(man);
        System.out.println(clone);
    }

    public static <T> List<T> depCopy(List<T> srcList) {
        ByteArrayOutputStream byteOut = new ByteArrayOutputStream();
        try (ObjectOutputStream out = new ObjectOutputStream(byteOut)) {
            out.writeObject(srcList);
            ByteArrayInputStream byteIn = new ByteArrayInputStream(byteOut.toByteArray());
            ObjectInputStream inStream = new ObjectInputStream(byteIn);
            List<T> destList = (List<T>) inStream.readObject();
            return destList;
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        return null;
    }
}

```
> 测试结果如下

Man{age=20, name='美女', girls=[Man{age=20, name='null', girls=null}]}

Man{age=25, name='帅哥', girls=[Man{age=18, name='美女', girls=null}]}
