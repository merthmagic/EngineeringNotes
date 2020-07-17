## IDisposable接口的实现 ##

`IDisposable` 接口是为了释放非托管资源，非托管资源包括了数据库连接，文件，网络socket等等。一个区分托管资源与非托管资源的简单方法是，托管资源基本上都是.net class所创建出来的，但如果这个类是实现了IDisposable接口的，那么这个类应该是引用有非托管资源，我们在引用这样的类的时候，应该保证及时的对这些类所创建的对象进行dispose.

托管资源的释放，是由GC来完成，同时，GC也能进行非托管资源的释放，但是这个释放的时机是不确定的（GC何时启动回收是不确定的），而非托管资源常常稀有而昂贵，我们应该在使用完非托管资源后，尽快自行释放。

GC释放非托管资源的方法是调用~Finnalize方法，这个方法可以认为是析构函数. GC在垃圾回收时，会检查该对象是否有~Finnalize方法，如果有，则调用之，进行非托管资源的释放.标准的代码实现方法可以参考MSDN.

### 无finalize ###

	using Microsoft.Win32.SafeHandles;
	using System;
	using System.Runtime.InteropServices;
	
	class BaseClass : IDisposable
	{
	   // Flag: Has Dispose already been called?bool disposed = false;
	   // Instantiate a SafeHandle instance.
	   SafeHandle handle = new SafeFileHandle(IntPtr.Zero, true);
	
	   // Public implementation of Dispose pattern callable by consumers.publicvoid Dispose()
	   { 
	      Dispose(true);
	      GC.SuppressFinalize(this);           
	   }
	
	   // Protected implementation of Dispose pattern.protectedvirtualvoid Dispose(bool disposing)
	   {
	      if (disposed)
	         return; 
	
	      if (disposing) {
	         handle.Dispose();
	         // Free any other managed objects here.//
	      }
	
	      // Free any unmanaged objects here.//
	      disposed = true;
	   }
	}

### 有finalize ###

	using System;
	
	class BaseClass : IDisposable
	{
	   // Flag: Has Dispose already been called?bool disposed = false;
	
	   // Public implementation of Dispose pattern callable by consumers.publicvoid Dispose()
	   { 
	      Dispose(true);
	      GC.SuppressFinalize(this);           
	   }
	
	   // Protected implementation of Dispose pattern.protectedvirtualvoid Dispose(bool disposing)
	   {
	      if (disposed)
	         return; 
	
	      if (disposing) {
	         // Free any other managed objects here.//
	      }
	
	      // Free any unmanaged objects here.//
	      disposed = true;
	   }
	
	   ~BaseClass()
	   {
	      Dispose(false);
	   }
	}