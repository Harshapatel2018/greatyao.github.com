---
layout: post
title: "How_to_invoke_C#_in_C++"
tagline: null
category: null
tags: []
published: true---
如何在C++中调用C#的dll和方法？

我们将场景简化为:开发人员A用C#语言实现了一个根据用户的输入生成若干条数据的接口，另外一个开发人员B使用了C++进行UI开发，他需要动态将这些数据逐条显示在UI控件中。

# C++和C#的结构体定义

C++中的结构体定义
struct CPPStruct
{
	int a;
	char b;
	short c;
	char d[32];
};

C#中的结构体定义
namespace CSharpLibrary
{
    [StructLayoutAttribute(LayoutKind.Sequential)]
    public struct CSStruct
    {
        public int a;
        public byte b;
        public short c;
        [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 32)]
        public string d; 
    }
}

* C++和C#的结构体的内存布局必须一致
* 在C++中使用#using引用C#生成的DLL
* 在C++使用using namespace来访问C#中的类和方法
* C++工程中必须开启公共语言运行时/clr的支持
* 在C++中采用正确的形式访问托管对象，使用帽子‘^’，而不是星星‘*’

#第一种实现
