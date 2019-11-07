
[toc]

AIDL是一种跨进程通信的方式，通信基于Binder。

直接继承Binder也可以实现跨进程通信。

# Binder
Binder是一个类，它实现了IBinder接口，而IBinder接口定义了与远程对象的交互协议。通常在进行跨进程通信时，不需要实现IBinder接口，直接从Binder派生即可。

除了实现IBinder接口外，Binder中还提供了两个重要的接口。
- transact()，客户端调用，用于发送调用请求
- onTransact()，服务端响应，用于接收调用请求

因为以上的原因，Binder成为了客户端与服务端的通信媒介，其主要用在Service组件应用中。 

# AIDL
## 机制
AIDL的通信基于Binder。

下面写一个AIDL的demo讲解他的通信机制。

新建Book.aidl文件

```java
package com.competition.pdking.ipcdemo;

parcelable Book;
```

然后创建一个IBookManager.aidl文件

```java
// IBookManager.aidl
package com.competition.pdking.ipcdemo;

// Declare any non-default types here with import statements

import com.competition.pdking.ipcdemo.Book;
interface IBookManager {

    List<Book> getBookList();
    void addBook(in Book book);
}

```
Android会生成一个java文件

```java
/*
 * This file is auto-generated.  DO NOT MODIFY.
 * Original file: C:\\Projects\\AndroidProjects\\AndroidDemoProject\\IPCDemo\\app\\src\\main
 * \\aidl\\com\\competition\\pdking\\ipcdemo\\IBookManager.aidl
 */
package com.competition.pdking.ipcdemo;

public interface IBookManager extends android.os.IInterface {
    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements com.competition.pdking.ipcdemo.IBookManager {
        private static final java.lang.String DESCRIPTOR = "com.competition.pdking.ipcdemo" +
                ".IBookManager";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an com.competition.pdking.ipcdemo.IBookManager interface,
         * generating a proxy if needed.
         */
        public static com.competition.pdking.ipcdemo.IBookManager asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.competition.pdking.ipcdemo.IBookManager))) {
                return ((com.competition.pdking.ipcdemo.IBookManager) iin);
            }
            return new com.competition.pdking.ipcdemo.IBookManager.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply,
                                  int flags) throws android.os.RemoteException {
            java.lang.String descriptor = DESCRIPTOR;
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(descriptor);
                    return true;
                }
                case TRANSACTION_getBookList: {
                    data.enforceInterface(descriptor);
                    java.util.List<com.competition.pdking.ipcdemo.Book> _result =
                            this.getBookList();
                    reply.writeNoException();
                    reply.writeTypedList(_result);
                    return true;
                }
                case TRANSACTION_addBook: {
                    data.enforceInterface(descriptor);
                    com.competition.pdking.ipcdemo.Book _arg0;
                    if ((0 != data.readInt())) {
                        _arg0 = com.competition.pdking.ipcdemo.Book.CREATOR.createFromParcel(data);
                    } else {
                        _arg0 = null;
                    }
                    this.addBook(_arg0);
                    reply.writeNoException();
                    return true;
                }
                default: {
                    return super.onTransact(code, data, reply, flags);
                }
            }
        }

        private static class Proxy implements com.competition.pdking.ipcdemo.IBookManager {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            @Override
            public java.util.List<com.competition.pdking.ipcdemo.Book> getBookList() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<com.competition.pdking.ipcdemo.Book> _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.createTypedArrayList(com.competition.pdking.ipcdemo.Book.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }

            @Override
            public void addBook(com.competition.pdking.ipcdemo.Book book) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    if ((book != null)) {
                        _data.writeInt(1);
                        book.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
                    mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
        }

        static final int TRANSACTION_getBookList = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_addBook = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    }

    public java.util.List<com.competition.pdking.ipcdemo.Book> getBookList() throws android.os.RemoteException;

    public void addBook(com.competition.pdking.ipcdemo.Book book) throws android.os.RemoteException;
}

```
- IBookManager.java这个类他继承IInterFace接口，同时他也是一个接口，所有可以在Binder中传输的接口都需要继承IInterface接口。

- 首先他声明了getBookList和addBook两个方法。这显然就是我们在aidl文件中声明的方法。同时还声明了两个id用于标记这两个方法。
- 然后，他声明了一个内部类Stub，这个类就是一个Binder类，当位于一个进程时，方法调用不会走跨进程的transact过程；当位于不同的进程时，会走transact过程，这个逻辑由Stub的内部类Proxy完成。

然后逐个解释作用：

- DESCRIPTOR——Binder的唯一标识，用来表示当前Binder的类名表示。
- asInterface——将服务端的Binder对象转换为客户端所需的AIDL接口类型，这个转换过程是区分进程的。如果位于同一个进程那么此方法返回的就是服务端的Stub对象本身，否则返回系统封装的Stub.Proxy对象。
- asBinder——返回当前Binder对象。
- onTransact——这个方法运行在服务端的Binder线程池中，当客户端发起远程请求后，会经系统底层封装后交此方法来处理。        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) 服务端通过code可以确定客户端请求的目标方法，接着从data中取出目标方法的参数，然后执行目标方法，目标方法执行完后，就向reply中写入返回值。
- Proxy#getBookList——这个方法运行在客户端，调用流程如下：创建Parcel对象_data和_reply，接着调用transact方法发起远程调用请求，同时线程挂起，然后服务端的onTransact方法会调用，知道调用过程返回，当前线程继续执行，并从_reply中取出返回的结果，最后返回结果。
- Proxy#addBook——和getBookList过程一样，只不过没有返回值。

## 使用
### 服务端
```java
/**
 * @author liupeidong
 * Created on 2019/11/7 21:40
 */
public class MyService extends Service {

    IBookManager.Stub mBinder = new IBookManager.Stub() {
        @Override
        public List<Book> getBookList() throws RemoteException {
            return null;
        }

        @Override
        public void addBook(Book book) throws RemoteException {

        }
    };

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }
}
```
### 客户端

```java
public class MainActivity extends AppCompatActivity {

    IBookManager aidl;

    ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            aidl = IBookManager.Stub.asInterface(service);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Intent intent = new Intent(this, MyService.class);
        bindService(intent, connection, Context.BIND_AUTO_CREATE);

    }
}
```

这样就通过AIDL完成了进程间通信。

# 继承Binder实现继承间通信
## 服务端

服务端只要继承一个Binder，并完成重写onTransact方法，在onTransact方法里面对不同的方法进行判断即可。

```java
	private MyBinder mBinder = new MyBinder();
 
	private class MyBinder extends Binder
	{
		@Override
		protected boolean onTransact(int code, Parcel data, Parcel reply,
				int flags) throws RemoteException
		{
			switch (code)
			{
			case 0x110:
			{
				data.enforceInterface(DESCRIPTOR);
				int _arg0;
				_arg0 = data.readInt();
				int _arg1;
				_arg1 = data.readInt();
				int _result = _arg0 * _arg1;
				reply.writeNoException();
				reply.writeInt(_result);
				return true;
			}
			case 0x111:
			{
				data.enforceInterface(DESCRIPTOR);
				int _arg0;
				_arg0 = data.readInt();
				int _arg1;
				_arg1 = data.readInt();
				int _result = _arg0 / _arg1;
				reply.writeNoException();
				reply.writeInt(_result);
				return true;
			}
			}
			return super.onTransact(code, data, reply, flags);
		}
 
	};
 
```

## 客户端

创建ServiceConnection 绑定IBinder 

```java
	private IBinder mPlusBinder;
	private ServiceConnection mServiceConnPlus = new ServiceConnection()
	{
		@Override
		public void onServiceDisconnected(ComponentName name)
		{
			Log.e("client", "mServiceConnPlus onServiceDisconnected");
		}
 
		@Override
		public void onServiceConnected(ComponentName name, IBinder service)
		{
 
			Log.e("client", " mServiceConnPlus onServiceConnected");
			mPlusBinder = service;
		}
	}; 
```
然后调用Binder的方法时需要注意，需要加上自己的方法标志或ID来区分不同的方法，然后的流程就可AIDL的差不多了，创建Parcel _data 和_reply，接着调用transact方法进行远程请求，最后从_reply读取结果即可。
```java
	public void mulInvoked(View view)
	{
 
		if (mPlusBinder == null)
		{
			Toast.makeText(this, "未连接服务端或服务端被异常杀死", Toast.LENGTH_SHORT).show();
		} else
		{
			android.os.Parcel _data = android.os.Parcel.obtain();
			android.os.Parcel _reply = android.os.Parcel.obtain();
			int _result;
			try
			{
				_data.writeInterfaceToken("CalcPlusService");
				_data.writeInt(50);
				_data.writeInt(12);
				mPlusBinder.transact(0x110, _data, _reply, 0);
				_reply.readException();
				_result = _reply.readInt();
				Toast.makeText(this, _result + "", Toast.LENGTH_SHORT).show();
 
			} catch (RemoteException e)
			{
				e.printStackTrace();
			} finally
			{
				_reply.recycle();
				_data.recycle();
			}
		}
 
	}
	
	public void divInvoked(View view)
	{
 
		if (mPlusBinder == null)
		{
			Toast.makeText(this, "未连接服务端或服务端被异常杀死", Toast.LENGTH_SHORT).show();
		} else
		{
			android.os.Parcel _data = android.os.Parcel.obtain();
			android.os.Parcel _reply = android.os.Parcel.obtain();
			int _result;
			try
			{
				_data.writeInterfaceToken("CalcPlusService");
				_data.writeInt(36);
				_data.writeInt(12);
				mPlusBinder.transact(0x111, _data, _reply, 0);
				_reply.readException();
				_result = _reply.readInt();
				Toast.makeText(this, _result + "", Toast.LENGTH_SHORT).show();
 
			} catch (RemoteException e)
			{
				e.printStackTrace();
			} finally
			{
				_reply.recycle();
				_data.recycle();
			}
		}
 
	}

```
