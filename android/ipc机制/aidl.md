# 自动生成的java 代码
<pre>
/*
 * This file is auto-generated.  DO NOT MODIFY.
 * Original file: /Users/longzhenhao/Desktop/Server/app/src/main/aidl/com/example/aidlserver/IRemote.aidl
 */
package com.example.aidlserver;
// Declare any non-default types here with import statements

public interface IRemote extends android.os.IInterface
{
/** Local-side IPC implementation stub class. */
    public static abstract class Stub extends android.os.Binder implements com.example.aidlserver.IRemote
    {
       private static final java.lang.String DESCRIPTOR = "com.example.aidlserver.IRemote";
    /** Construct the stub at attach it to the interface. */
       public Stub()
       {
          this.attachInterface(this, DESCRIPTOR);
       }
    /**
     * Cast an IBinder object into an com.example.aidlserver.IRemote interface,
     * generating a proxy if needed.
     */
        public static com.example.aidlserver.IRemote asInterface(android.os.IBinder obj)
        {
            if ((obj==null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin!=null)&&(iin instanceof com.example.aidlserver.IRemote))) {
                return ((com.example.aidlserver.IRemote)iin);
            }
            return new com.example.aidlserver.IRemote.Stub.Proxy(obj);
        }
        @Override public android.os.IBinder asBinder()
        {
            return this;
        }
        @Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
        {
        switch (code)
        {
            case INTERFACE_TRANSACTION:
            {
                reply.writeString(DESCRIPTOR);
                return true;
            }
            case TRANSACTION_multi:
            {
                data.enforceInterface(DESCRIPTOR);
                int _arg0;
                _arg0 = data.readInt();
                int _arg1;
                _arg1 = data.readInt();
                int _result = this.multi(_arg0, _arg1);
                reply.writeNoException();
                reply.writeInt(_result);
                return true;
            }
        }
            return super.onTransact(code, data, reply, flags);
        }
        private static class Proxy implements com.example.aidlserver.IRemote
        {
            private android.os.IBinder mRemote;
            Proxy(android.os.IBinder remote)
            {
                mRemote = remote;
            }
            @Override public android.os.IBinder asBinder()
            {
                return mRemote;
            }
            public java.lang.String getInterfaceDescriptor()
            {
                return DESCRIPTOR;
            }
            @Override public int multi(int a, int b) throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                int _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(a);
                    _data.writeInt(b);
                    mRemote.transact(Stub.TRANSACTION_multi, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.readInt();
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
        }
        static final int TRANSACTION_multi = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
    }
    public int multi(int a, int b) throws android.os.RemoteException;
}

</pre>
- DESCRIPTOR
Binder的唯一标识，一般用Binder的类名表示。
asInterface(android.os.IBinder obj)
将服务端的Binder对象转换为客户端所需的AIDL接口类型的对象，如果C/S位于同一进
程，此方法返回就是服务端的Stub对象本身，否则返回的就是系统封装后的Stub.proxy对
象。
 - asBinder 返回当前Binder对象。
- onTransact 这个方法运行在服务端的Binder线程池中，由客户端发起跨进程请求时，远程请求会通过
系统底层封装后交由此方法来处理。该方法的原型是
 java public Boolean onTransact(int code,Parcelable data,Parcelable reply,int flags)
服务端通过code确定客户端请求的目标方法是什么，
接着从data取出目标方法所需的参数，然后执行目标方法。
执行完毕后向reply写入返回值（ 如果有返回值） 。
如果这个方法返回值为false，那么服务端的请求会失败，利用这个特性我们可以来做权限验证。
- Proxy# 代理类 
运行在客户端，内部实现过程如下：
首先创建该方法所需要的输入型对象Parcel对象_data，输出型Parcel对象_reply和返回值对象List。
然后把该方法的参数信息写入_data（ 如果有参数）
接着调用transact方法发起RPC（ 远程过程调用） ，同时当前线程挂起
- 然后服务端的onTransact方法会被调用知道RPC过程返回后，当前线程继续执行，并从_reply中取出RPC过程的返回结果，最后返回_reply中的数据。
