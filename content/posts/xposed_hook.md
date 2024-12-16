---
title: "Xposed 如何Hook方法和变量总结"
date: '2024-12-16T18:44:56+08:00'
# weight: 1
# aliases: ["/first"]
tags: ["转载","Android","Xposed"]
summary: " "
author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: ""
canonicalURL: "https://canonical.url/to/page"
disableShare: true
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/<path_to_repo>/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---
## 1、概述
Xposed是非常牛叉的一款hook框架，本人也是刚刚接触，在网上搜索一些资料，发现写的都不是太全面，于是搜集该框架的用法，总结出该文。如有纰漏，还请轻拍，主要内容包括
> 1、如何Hook静态变量  
> 2、如何Hook构造方法  
> 3、如何Hook复杂参数的方法  
> 4、如何替换函数执行内容  
> 5、如何Hook内部类中的函数  
> 6、如何Hook匿名类的函数  
> 7、如何获取调用对象去调用函数，或者新建新建示例去调用方法  

学会这些方法，在结合逆向smail的一些知识，应该可以满足大多数java层的hook了。话不多说，上代码！

## 2、Hook目标程序源码
HookDemo.java
```Java
abstract class Animal{
    int anonymoutInt = 500;
    public abstract void eatFunc(String value);
}

public class HookDemo {
    private String Tag = "HookDemo";
    private static  int staticInt = 100;
    public  int publicInt = 200;
    private int privateInt = 300;

    public HookDemo(){
        this("NOHook");
        Log.d(Tag, "HookDemo() was called|||");
    }

    private HookDemo(String str){
        Log.d(Tag, "HookDemo(String str) was called|||" + str);
    }

    public void hookDemoTest(){
        Log.d(Tag, "staticInt = " + staticInt);
        Log.d(Tag, "PublicInt = " + publicInt);
        Log.d(Tag, "privateInt = " + privateInt);
        publicFunc("NOHook");
        Log.d(Tag, "PublicInt = " + publicInt);
        Log.d(Tag, "privateInt = " + privateInt);
        privateFunc("NOHook");
        staticPrivateFunc("NOHook");

        String[][] str = new String[1][2];
        Map map = new HashMap<String, String>();
        map.put("key", "value");
        ArrayList arrayList = new ArrayList();
        arrayList.add("listValue");
        complexParameterFunc("NOHook", str, map, arrayList);

        repleaceFunc();
        anonymousInner(new Animal() {
            @Override
            public void eatFunc(String value) {
                Log.d(Tag, "eatFunc(String value)  was called|||" + value);
                Log.d(Tag, "anonymoutInt =  " + anonymoutInt);
            }
        }, "NOHook");

        InnerClass innerClass = new InnerClass();
        innerClass.InnerFunc("NOHook");
    }

    public void publicFunc(String value){
        Log.d(Tag, "publicFunc(String value) was called|||" + value);
    }

    private void privateFunc(String value){
        Log.d(Tag, "privateFunc(String value) was called|||" + value);
    }

    static private void staticPrivateFunc(String value){
        Log.d("HookDemo", "staticPrivateFunc(Strin value) was called|||" + value);
    }

    private void complexParameterFunc(String value, String[][] str, Map<String,String> map, ArrayList arrayList)
    {
        Log.d("HookDemo", "complexParameter(Strin value) was called|||" + value);
    }

    private void repleaceFunc(){
        Log.d(Tag, "repleaceFunc will be replace|||");
    }

    public void anonymousInner(Animal dog, String value){
        Log.d(Tag, "anonymousInner was called|||" + value);
        dog.eatFunc("NOHook");
    }

    private void hideFunc(String value){
        Log.d(Tag, "hideFunc was called|||" + value);
    }

    class InnerClass{
        public int innerPublicInt = 10;
        private int innerPrivateInt = 20;
        public InnerClass(){
            Log.d(Tag, "InnerClass constructed func was called");
        }
        public void InnerFunc(String value){
            Log.d(Tag, "InnerFunc(String value) was called|||" + value);
            Log.d(Tag, "innerPublicInt = " + innerPublicInt);
            Log.d(Tag, "innerPrivateInt = " + innerPrivateInt);
        }
    }
}
```
## 3、XposedHook程序源码
```Java
public class XposedHook implements IXposedHookLoadPackage {

    @Override
    public void handleLoadPackage(XC_LoadPackage.LoadPackageParam loadPackageParam) throws Throwable {
        if (loadPackageParam.packageName.equals("com.example.xposedhooktarget")) {
            final Class<?> clazz = XposedHelpers.findClass("com.example.xposedhooktarget.HookDemo", loadPackageParam.classLoader);
            //getClassInfo(clazz);

            //不需要获取类对象，即可直接修改类中的私有静态变量staticInt
            XposedHelpers.setStaticIntField(clazz, "staticInt", 99);

            //Hook无参构造函数，啥也不干。。。。
            XposedHelpers.findAndHookConstructor(clazz, new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    XposedBridge.log("Haha, HookDemo constructed was hooked" );
                    //大坑，此时对象还没有建立，即不能获取对象，也不能修改非静态变量的值
                    //XposedHelpers.setIntField(param.thisObject, "publicInt", 199);
                    //XposedHelpers.setIntField(param.thisObject, "privateInt", 299);
                }
            });

            //Hook有参构造函数，修改参数
            XposedHelpers.findAndHookConstructor(clazz, String.class,  new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    param.args[0] = "Haha, HookDemo(str) are hooked";
                }
            });

            //Hook有参构造函数，修改参数------不能使用XC_MethodReplacement()替换构造函数内容，
            //XposedHelpers.findAndHookConstructor(clazz, String.class, new XC_MethodReplacement() {
            //    @Override
            //    protected Object replaceHookedMethod(MethodHookParam methodHookParam) throws Throwable {
            //        Log.d("HookDemo" , "HookDemo(str) was replace");
            //    }
            //});

            //Hook公有方法publicFunc，
            // 1、修改参数
            // 2、修改下publicInt和privateInt的值
            // 3、再顺便调用一下隐藏函数hideFunc
            //XposedHelpers.findAndHookMethod("com.example.xposedhooktarget.HookDemo", clazz.getClassLoader(), "publicFunc", String.class, new XC_MethodHook()
            XposedHelpers.findAndHookMethod(clazz, "publicFunc", String.class, new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    param.args[0] = "Haha, publicFunc are hooked";
                    XposedHelpers.setIntField(param.thisObject, "publicInt", 199);
                    XposedHelpers.setIntField(param.thisObject, "privateInt", 299);
                    // 让hook的对象本身去执行流程
                    Method md = clazz.getDeclaredMethod("hideFunc", String.class);
                    md.setAccessible(true);
                    //md.invoke(param.thisObject, "Haha, hideFunc was hooked");
                    XposedHelpers.callMethod(param.thisObject, "hideFunc", "Haha, hideFunc was hooked");

                    //实例化对象，然后再调用HideFunc方法
                    //Constructor constructor = clazz.getConstructor();
                    //XposedHelpers.callMethod(constructor.newInstance(), "hideFunc", "Haha, hideFunc was hooked");
                }
            });

            //Hook私有方法privateFunc，修改参数
            //XposedHelpers.findAndHookMethod("com.example.xposedhooktarget.HookDemo", clazz.getClassLoader(), "privateFunc", String.class, new XC_MethodHook()
            XposedHelpers.findAndHookMethod(clazz, "privateFunc", String.class, new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    param.args[0] = "Haha, privateFunc are hooked";
                }
            });

            //Hook私有静态方法staticPrivateFunc, 修改参数
            XposedHelpers.findAndHookMethod(clazz, "staticPrivateFunc", String.class, new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    param.args[0] = "Haha, staticPrivateFunc are hooked";
                }
            });

            //Hook复杂参数函数complexParameterFunc
            Class fclass1 = XposedHelpers.findClass("java.util.Map", loadPackageParam.classLoader);
            Class fclass2 = XposedHelpers.findClass("java.util.ArrayList", loadPackageParam.classLoader);
            XposedHelpers.findAndHookMethod(clazz, "complexParameterFunc", String.class,
                    "[[Ljava.lang.String;", fclass1, fclass2, new XC_MethodHook() {
                        @Override
                        protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                            param.args[0] = "Haha, complexParameterFunc are hooked";
                        }
                    });

            //Hook私有方法repleaceFunc, 替换打印内容
            XposedHelpers.findAndHookMethod(clazz, "repleaceFunc", new XC_MethodReplacement() {
                @Override
                protected Object replaceHookedMethod(MethodHookParam methodHookParam) throws Throwable {
                    Log.d("HookDemo", "Haha, repleaceFunc are replaced");
                    return null;
                }
            });
            //Hook方法, anonymousInner， 参数是抽象类，先加载所需要的类即可
            Class animalClazz  = loadPackageParam.classLoader.loadClass("com.example.xposedhooktarget.Animal");
            XposedHelpers.findAndHookMethod(clazz, "anonymousInner", animalClazz, String.class, new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    XposedBridge.log("HookDemo This is test");
                    param.args[1] = "Haha, anonymousInner are hooked";
                }
            });

            //Hook匿名类的eatFunc方法，修改参数，顺便修改类中的anonymoutInt变量
            XposedHelpers.findAndHookMethod("com.example.xposedhooktarget.HookDemo$1", clazz.getClassLoader(),
                    "eatFunc", String.class, new XC_MethodHook() {
                        @Override
                        protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                            param.args[0] = "Haha, eatFunc are hooked";
                            XposedHelpers.setIntField(param.thisObject, "anonymoutInt", 499);
                        }
                    });

            //hook内部类的构造方法失败，且会导致hook内部类的InnerFunc方法也失败，原因不明
//            XposedHelpers.findAndHookConstructor(clazz1, new XC_MethodHook() {
//                        @Override
//                        protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
//                            XposedBridge.log("Haha, InnerClass constructed was hooked" );
//                        }
//                    });

            //Hook内部类InnerClass的InnerFunc方法，修改参数，顺便修改类中的innerPublicInt和innerPrivateInt变量
            final Class<?> clazz1 = XposedHelpers.findClass("com.example.xposedhooktarget.HookDemo$InnerClass", loadPackageParam.classLoader);
            XposedHelpers.findAndHookMethod(clazz1, "InnerFunc", String.class, new XC_MethodHook() {
                        @Override
                        protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                            param.args[0] = "Haha, InnerFunc was hooked";
                            XposedHelpers.setIntField(param.thisObject, "innerPublicInt", 9);
                            XposedHelpers.setIntField(param.thisObject, "innerPrivateInt", 19);
                        }
                    });
        }
    }

    private void getClassInfo(Class clazz) {
        //getFields()与getDeclaredFields()区别:getFields()只能访问类中声明为公有的字段,私有的字段它无法访问，
        //能访问从其它类继承来的公有方法.getDeclaredFields()能访问类中所有的字段,与public,private,protect无关，
        //不能访问从其它类继承来的方法
        //getMethods()与getDeclaredMethods()区别:getMethods()只能访问类中声明为公有的方法,私有的方法它无法访问,
        //能访问从其它类继承来的公有方法.getDeclaredFields()能访问类中所有的字段,与public,private,protect无关,
        //不能访问从其它类继承来的方法
        //getConstructors()与getDeclaredConstructors()区别:getConstructors()只能访问类中声明为public的构造函数
        //getDeclaredConstructors()能访问类中所有的构造函数,与public,private,protect无关

        //XposedHelpers.setStaticObjectField(clazz,"sMoney",110);
        //Field sMoney = clazz.getDeclaredField("sMoney");
        //sMoney.setAccessible(true);
        Field[] fs;
        Method[] md;
        Constructor[] cl;
        fs = clazz.getFields();
        for (int i = 0; i < fs.length; i++) {
            XposedBridge.log("HookDemo getFiled: " + Modifier.toString(fs[i].getModifiers()) + " " +
                    fs[i].getType().getName() + " " + fs[i].getName());
        }
        fs = clazz.getDeclaredFields();
        for (int i = 0; i < fs.length; i++) {
            XposedBridge.log("HookDemo getDeclaredFields: " + Modifier.toString(fs[i].getModifiers()) + " " +
                    fs[i].getType().getName() + " " + fs[i].getName());
        }
        md = clazz.getMethods();
        for (int i = 0; i < md.length; i++) {
            Class<?> returnType = md[i].getReturnType();
            XposedBridge.log("HookDemo getMethods: " + Modifier.toString(md[i].getModifiers()) + " " +
                    returnType.getName() + " " + md[i].getName());
            //获取参数
            //Class<?> para[] = md[i].getParameterTypes();
            //for (int j = 0; j < para.length; ++j) {
            //System.out.print(para[j].getName() + " " + "arg" + j);
            //if (j < para.length - 1) {
            //    System.out.print(",");
            //}
            //}
        }
        md = clazz.getDeclaredMethods();
        for (int i = 0; i < md.length; i++) {
            Class<?> returnType = md[i].getReturnType();
            XposedBridge.log("HookDemo getDeclaredMethods: " + Modifier.toString(md[i].getModifiers()) + " " +
                    returnType.getName() + " " + md[i].getName());
        }
        cl = clazz.getConstructors();
        for (int i = 0; i < cl.length; i++) {
            XposedBridge.log("HookDemo getConstructors: " + Modifier.toString(cl[i].getModifiers()) + " " +
                    md[i].getName());
        }
        cl = clazz.getDeclaredConstructors();
        for (int i = 0; i < cl.length; i++) {
            XposedBridge.log("HookDemo getDeclaredConstructors: " + Modifier.toString(cl[i].getModifiers()) + " " +
                    md[i].getName());
        }
    }
}
```

转自： https://cloud.tencent.com/developer/article/1774189
