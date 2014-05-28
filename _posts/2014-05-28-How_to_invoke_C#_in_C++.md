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
  #using "..\release\CSharpLibrary.dll" 
* C++工程中必须开启公共语言运行时/clr的支持
* 在C++使用using namespace来访问C#中的类和方法，采用gcnew访问托管对象，注意使用帽子‘^’，而不是星星‘*’
  using namespace CSharpLibrary; 
	CSharpClass ^dll = gcnew CSharpClass(); 

#第一种实现

C#

public void CSharpSimpleInterface(ref IntPtr results, int num)
{
    for (int i = 0; i < num; i++)
    {
        CSStruct b = new CSStruct();
        b.a = i * i * i;
        b.b = (byte)(i);
        b.c = (short)(i * i);
        b.d = String.Format("C#:{0}", i);

        Marshal.StructureToPtr(b, (IntPtr)(results.ToInt32() + i * Marshal.SizeOf(b)), true);
    }
}
    

C++

int num = 10;
struct CPPStruct* sAA = new struct CPPStruct[num];
dll->CSharpSimpleInterface((System::IntPtr)sAA, num);
//
...UI显示等
for(int i = 0; i < num; i++)
		printf("%d: (%d,%d,%d,%s)\n", i, sAA[i].a, sAA[i].b, sAA[i].c, sAA[i].d);
		
#第二种实现
如果num数很大，或者每条信息的生成都比较耗时(比如Web异步调用)，这样将导致CSharpSimpleInterface的调用时间变长，另一种可行的方法是使用回调函数来处理。

C++定义了callback函数，处理单条数据
int WINAPI cpp_callback(void* p, void* p2)
{
	struct CPPStruct* a = (struct CPPStruct*)p;
	vector<CPPStruct>* v = (vector<CPPStruct>*)p2;

	printf("C++: (%d,%d,%d,%s)\n", a->a, a->b, a->c, a->d);
	v->push_back(struct CPPStruct(*a));
	//UI显示等
	//::MessageBoxA(NULL, "fuckyou", a->d, 0);
	return 0;
}

vector<struct CPPStruct> sAA;
dll->CSharpInterface((System::IntPtr)&cpp_callback, (System::IntPtr)&sAA, 20);

C#的委托和C/C++的函数指针都描述了方法/函数的签名，并通过统一的接口调用不同的实现。但二者又有明显的区别，简单说来，委托对象是真正的对象，而函数指针变量只是函数的入口地址。

与cpp_callback函数相对应，定义如下的一个委托
  [UnmanagedFunctionPointerAttribute(CallingConvention.Cdecl)]
  public delegate int CALLBACK(IntPtr a, IntPtr b);
  
采用Marshal.GetDelegateForFunctionPointer来转换一个函数指针为一个委托
 CALLBACK callback = (CALLBACK)Marshal.GetDelegateForFunctionPointer(func, typeof(CALLBACK));
 
 完整代码如下
  [UnmanagedFunctionPointerAttribute(CallingConvention.Cdecl)]
  public delegate int CALLBACK(IntPtr a, IntPtr b);

  public void CSharpCallbackInterface(IntPtr func, IntPtr arg, int num)
  {
      CALLBACK callback = (CALLBACK)Marshal.GetDelegateForFunctionPointer(func, typeof(CALLBACK));
      
      for (int i = 0; i < num; i++)
      {
          CSStruct b = new CSStruct();
          b.a = i*i*i;
          b.b = (byte)(i);
          b.c = (short)(i * i);
          b.d = String.Format("C#:{0}", i);
          IntPtr p2 = Marshal.AllocHGlobal(Marshal.SizeOf(b));
          Marshal.StructureToPtr(b, p2, true);
          Console.WriteLine("C#: addr={0}", p2);
          callback(p2, arg);
          Marshal.FreeHGlobal(p2);
      } 
  }
  
  OK,现在转到C++，
  vector<struct CPPStruct> sAA;
	dll->CSharpCallbackInterface((System::IntPtr)&callback, (System::IntPtr)&sAA, 1000);