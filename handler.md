




# Handler源码解析

handler

handler是Android系统中重要的消息机制,四大组件的消息传递都离不开handler,我们在日常的开发中也经常用handler来传递消息,那么handler是怎么传递消息的

1. 首先通过handler.sendMessage发送一个消息,最终这个消息会进入messagequeue中存储,而messagequeue是一个优先级队列,其数据结构是链表的形式,按message的执行顺序排序
2. 接着进入handler. enqueueMessage()方法中,然后调用MessageQueue的enqueueMessage方法将数据插入到messagequeue的队列中,紧接着唤醒并处理消息
3. 唤醒的机制是通过epoll机制来进行的,在NativeMessageQueue中我们最终发现其实是将 "1"写入到pipe管道中,触发WakeEvent事件唤醒通知Looper处理当前的message
4. 而我们知道Looper处理消息实际上是通过Looper.loop中的一个死循环for(;;)来处理的,如果没有消息的时候会进入休眠状态,而我们在上文(步骤3中提到发送一个新的消息的时候会通过epoll机制唤醒Looper处理msg)提到会唤醒looper也就是通过对管道写入"1"触发wakeEvent时间
5. 当Looper被唤醒之后即Looper的loop函数的for(;;)不被阻塞则在内部调用messagequeu.next方法获取消息队列中的msg,最终通过msg.target.dispatchMessage();来将消息分发下去此处的,msg就是上文sendMessage发送的msg,而msg.target就是发送msg的handler
6. 接下来就是对msg的分发了其内部就是调用despatchMessage方法,而该方法内部又调用了 handleMessage方法,handleMessage就是我们创建这个handler的时候自己实现的handleMessage函数,这样我们发送的message就被返回到handler的handleMessage方法中了

接下来我们来进一步讨论消息的发送接收,Looper的阻塞和唤醒,首先来看一下java 层 Looper MessageQueue 和Native Looper NativeMessageQueue的相关创建 

# Looper的创建

我们知道一个应用的main方法是在ActivityThread里面的,首先我们来看ActivityThread的main方法,我们省略无关代码,可以看到在注释1出系统通过prepareMainLooper->prepare->new Looper初始化了Looper对象,并将初始化后的对象给了sThreadLocal,而且Looper只能被初始化一次,多次初始化会抛出Only one Looper may be created per thread的RunTimeException,在创建Looper的时候同时创建了MessageQueue,这也是为啥一个线程只有一个MessageQueue的原因(一个线程只能创建一个Looper,也就只能有一个MessageQueu了)

	
	public static void main(String[] args) {
	Looper.prepareMainLooper();//1 初始化Looper
	
	long startSeq = 0;
	if (args != null) {
	    for (int i = args.length - 1; i >= 0; --i) {
	        if (args[i] != null && args[i].startsWith(PROC_START_SEQ_IDENT)) {
	            startSeq = Long.parseLong(
	                    args[i].substring(PROC_START_SEQ_IDENT.length()));
	        }
	    }
	}
	ActivityThread thread = new ActivityThread();
	thread.attach(false, startSeq);
	
	if (sMainThreadHandler == null) {
	    sMainThreadHandler = thread.getHandler();
	}
	 
	Looper.loop();//2 looper开始循环运行
	throw new RuntimeException("Main thread loop unexpectedly exited");
	}
	
	

prepareMainLooper中其实就是调用Prepare函数

	
	@Deprecated
	public static void prepareMainLooper() {
	    prepare(false);
	    synchronized (Looper.class) {
	        if (sMainLooper != null) {
	            throw new IllegalStateException("The main Looper has already been prepared.");
	        }
	        sMainLooper = myLooper();
	    }
	}


prepare函数中创建Looper并将Looper函数给到ThreadLocal中

	private static void prepare(boolean quitAllowed) {
	    if (sThreadLocal.get() != null) {
	        throw new RuntimeException("Only one Looper may be created per thread");
	    }
	    sThreadLocal.set(new Looper(quitAllowed);
	}


创建java层Looper的过程至此告一段落,接下来我们来看一下MessageQueue的创建过程

	private Looper(boolean quitAllowed) {
	    mQueue = new MessageQueue(quitAllowed);
	    mThread = Thread.currentThread();
	}


# MessageQueue的创建

上一章节的结束我们知道了创建Looper的时候同时创建了MessageQueue,而Looper是每个线程只有一个的,在创建Looper的时候创建了MessageQueue函数,则可以推断出每个线程也只能有一个MessageQueue,下面看一下MessageQueue的具体创建,我们可以看到关键函数nativeInit(),那么这个函数做了什么,我们来看一下,这个函数是在android_os_MessageQueue中定义的


	MessageQueue(boolean quitAllowed) {
	        mQuitAllowed = quitAllowed;
	        mPtr = nativeInit();//1. 将更多的初始化工作放到Native层
	    }
	

通过下面的nativeInit()函数我们知道,这里首先在注释1 处创建了NativeMessageQueue,然后在2处将创建好的NativeMessageQueue返回到了java层


	static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
	    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();//1创建NativeMessageQueu
	    if (!nativeMessageQueue) {
	        jniThrowRuntimeException(env, "Unable to allocate native queue");
	        return 0;
	    }
	
	    nativeMessageQueue->incStrong(env);
	    return reinterpret_cast<jlong>(nativeMessageQueue);//2. 将创建的NativeMessageQueue返回到Java层
	}


接下来我们看一下NativeMessageQueue函数中具体初始化了那些内容,从下面的代码我们可以看到,其实NativeMessageQueue函数内部创建了Native层的Looper(注释1处)

	
	NativeMessageQueue::NativeMessageQueue() :
	mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
	    mLooper = Looper::getForThread();
	    if (mLooper == NULL) {
	        mLooper = new Looper(false);//1. 创建Looper
	        Looper::setForThread(mLooper);
	    }
	}


至此java层的MessageQueue和NativeMessageQueue的创建也结束了,接下来我们看一下Native的Looper的创建过程

# Native Looper的创建

在nativeMessageQueue的创建过程中我们发现Native层的Looper随之创建,我们看一下Looper的创建都做了什么,从下面的代码我们可以发现关键函数rebuildEpollLocked()


	 Looper::Looper(bool allowNonCallbacks)
	: mAllowNonCallbacks(allowNonCallbacks),
	            mSendingMessage(false),
	            mPolling(false),
	            mEpollRebuildRequired(false),
	            mNextRequestSeq(0),
	            mResponseIndex(0),
	            mNextMessageUptime(LLONG_MAX) {
	        mWakeEventFd.reset(eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC));
	        LOG_ALWAYS_FATAL_IF(mWakeEventFd.get() < 0, "Could not make wake event fd: %s", strerror(errno));
	
	        AutoMutex _l(mLock);
	        rebuildEpollLocked();//1. 初始化Looper创建epoll
	    }



rebuildEpollLocked()函数中我们可以看到

1. 首先在注释1处创建了epoll,并将创建返回的fd写入到mEpollFd中
2. 在注释2 出设置了WakeEvent事件的监听,这个跟Looper的唤醒有关
3. 注释3处创建了其他的监听事件


        void Looper::rebuildEpollLocked() {
            
            // Allocate the new epoll instance and register the wake pipe.
            mEpollFd.reset(epoll_create1(EPOLL_CLOEXEC));//1. epoll_createl
            LOG_ALWAYS_FATAL_IF(mEpollFd < 0, "Could not create epoll instance: %s", strerror(errno));

            struct epoll_event eventItem;
            memset(& eventItem, 0, sizeof(epoll_event)); // zero out unused members of data field union
            eventItem.events = EPOLLIN;
            eventItem.data.fd = mWakeEventFd.get();
            int result = epoll_ctl(mEpollFd.get(), EPOLL_CTL_ADD, mWakeEventFd.get(), &eventItem);//2. 注册WakeEvent事件的监听,后续Looper的唤醒跟这个有关
            LOG_ALWAYS_FATAL_IF(result != 0, "Could not add wake event fd to epoll instance: %s",
                    strerror(errno));

            for (size_t i = 0; i < mRequests.size(); i++) {
        const Request& request = mRequests.valueAt(i);
                struct epoll_event eventItem;
                request.initEventItem(&eventItem);

                int epollResult = epoll_ctl(mEpollFd.get(), EPOLL_CTL_ADD, request.fd, &eventItem);//创建了一些其他的监听事件
                if (epollResult < 0) {
                    ALOGE("Error adding epoll events for fd %d while rebuilding epoll set: %s",
                            request.fd, strerror(errno));
                }
            }
        }


至此Native的Looper的创建完成,并且我们直达了在Native层的Looper中通过Epoll机制监听了WakeEvent事件,这个事件就是下文中我们新添加一个Message到MessageQueue中唤醒Looper的关键点

## 消息的发送

1. 我们平常在使用handler发送Message的时候一般都是通过sendMessage 发送消息,当然也有其他方式,但是最终其实都调到一个方法,我们讨论常见的这个使用方式.


		public final boolean sendMessage(@NonNull Message msg) {
		    return sendMessageDelayed(msg, 0);//立即发送一个message
		}

2. sendMessage常用的有两种 1. 是立即发送即 delaytime == 0 2. 延迟发送 即delayTime >0,我们目前只对delayTime == 0 的情况做讨论

		public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
		    if (delayMillis < 0) {//如果延迟时间小于0 则将延迟时间设置为0
		        delayMillis = 0;
		    }
		    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
		}


3. 将Message   enqueue 到MessageQueue中,

	
		public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
		    MessageQueue queue = mQueue;
		    if (queue == null) {
		        RuntimeException e = new RuntimeException(
		                this + " sendMessageAtTime() called with no mQueue");
		        Log.w("Looper", e.getMessage(), e);
		        return false;
		    }
		    return enqueueMessage(queue, msg, uptimeMillis);//将Messag enqueue 到MessageQueue中
		}



4. **enqueueMessage方法,这里需要大家注意**后面还会提到

   在enqueueMessage方法中我们可以确定的几点是

   1. 这个方法是由我们使用handler的时候的Handler handler = new Handler()创建的handler对象象调用的
   2. 当前enqueueMessage方法实际上就是我们创建的handler调用sendMessage(msg)调用而来的

   那么从上面可以推出两点

   1. 注释1处的this就是我们创建handler
   2. 注释1处的msg就是我们handler.sendMessage(msg)中的msg

   从而我们得出结论msg持有了sendMessage的handler的引用,也就是target


	   private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
	           long uptimeMillis) {
	       msg.target = this;//1. 我们创建的handler
	       msg.workSourceUid = ThreadLocalWorkSource.getUid();//获取UID
	   
	       if (mAsynchronous) {
	           msg.setAsynchronous(true);
	       }
	       return queue.enqueueMessage(msg, uptimeMillis);//enqueue到Message中
	   }
	

   接下来进入发送消息最关键的将消息插入到消息队列的调用中,在下面的方法中首先讨论注释0 处

   如果p ==null (由注释1可知P其实就是上一次添加进来的msg),或者when==0(也就是立即执行)或者执行时间要比上一次的msg的执行时间还要早,也就是这一次的比上一次的要先执行,则将本次的msg置于head位置,也就是下一个最先执行的msg的位置,然后将之前的head作为本次替换的msg的next节点,然后唤醒线程消费当前msg(p == null 的时候loop一定是被休眠的,所以需要唤醒,其他两种需要唤醒吗,不一定: when ==0 的时候loop不一定休眠了,when<p.when的时候 when>0 也不一定需要唤醒,因为时间还没到)	

   ​		当然如果当前的Head节点不是空的或者when>p.when,也就是说当前待插入的msg不是需要插在头节点的,那么需要为当前待插入的msg找一个位置插入(根据待执行时间when),注释4处解释了是如何查找当前msg的位置的,最后将当前msg插入到p节点的前面,通过注释5处msg完成了enqueue的操作,此时当前线程可能是休眠状态也可能是唤醒状态,接下来我们讨论当looper没有被阻塞的时候是怎么去处理消息的

	
	   boolean enqueueMessage(Message msg, long when) {
	       //省略一些异常检查eg msg 是否已经在用了 发送线程是否已经dead
	       synchronized (this) { 
	           msg.markInUse();
	           msg.when = when;
	           Message p = mMessages;
	           boolean needWake;
	           if (p == null || when == 0 || when < p.when) {// 0
	               // New head, wake up the event queue if blocked.
	               msg.next = p;
	               mMessages = msg;//1
	               needWake = mBlocked;
	           } else {
	   
	               needWake = mBlocked && p.target == null && msg.isAsynchronous();
	               Message prev;
	               for (;;) {
	                   prev = p;//2 保存当前节点
	                   p = p.next;//3 将当前节点的后继给到当前节点
	                   if (p == null || when < p.when) { 4 //如果后继节点不null的,当前待插入的msg则需要插在最后跳出查找,或者当前待插入的msg的when要比后继节点的优先级高,则跳出查找,且p节点是当前待插入的msg的next节点
	                       break;
	                   }
	                   if (needWake && p.isAsynchronous()) {
	                       needWake = false;
	                   }
	               }
	               //5 下面两句是将msg插入到原先p节点之前
	               msg.next = p; 
	               prev.next = msg;
	           }
	   
	           // We can assume mPtr != 0 because mQuitting is false.
	           if (needWake) {
	               nativeWake(mPtr);//唤醒Looper去处理Message
	           }
	       }
	       return true;
	   }
	   
	   //初始化MessageQueue
	   MessageQueue(boolean quitAllowed) {
	          mQuitAllowed = quitAllowed;
	          mPtr = nativeInit();
	      }
	   
	   static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
	       NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
	       if (!nativeMessageQueue) {
	           jniThrowRuntimeException(env, "Unable to allocate native queue");
	           return 0;
	       }
	   
	       nativeMessageQueue->incStrong(env);
	       return reinterpret_cast<jlong>(nativeMessageQueue);
	   
	   }
	



## 处理Message 关键的loop函数

接下来我们看看Looper是怎么处理Message的

从上文我们可以看到现在Looper已经创建好了,并且已经在Messagequeue中插入了一条message,那么系统是怎么处理message的呢,从上面的代码中我们可以发现loop函数,而这个函数就是用来循环处理消息的,在loop函数中存在一个for(;;)死循环,接下来着重分析loop函数,当Looper创建的时候MessageQueue中肯定是没有Message 的,难道loop就一直在忙等待吗,这样岂不是很消耗性能,答案是否定的,**其实这个时候我们的loop函数是被阻塞的,当然这个我们方案后面单独讲**现在先讲Looper是如何处理消息,假设当前Looper中已经存在一个消息了,马上要执行,那么消息具体是怎么分发到我们的handler中的呢,具体看下面的代码,由于代码过长我们删除与当前处理无关的代码

1. 可以看到当存在一个message的时候我们MessageQueue会从queue中取出一个Message,见注释1.处
2. 将当前取出的msg分发到对应的handler中,见注释2处,这里你可能会问我们并没有见到向handler分发的啊,我们来看一下msg.target是什么,如果还记得我们在**消息的发送**中的第4点讲的话就很容易理解这里的msg其实就是我们handler.sendMessage(msg)中的msg,而msg.target就是我们的handler,那么这里就很容易理解了,这里的注释2 其实就是调用了我们Handler handler = new Handler();创建的handler的dispatchMessage方法,接下来看一下该方法


		/**
		 * Run the message queue in this thread. Be sure to call
		 * {@link #quit()} to end the loop.
		 */
		public static void loop() {
		    for (;;) {
		        Message msg = queue.next(); //1. 从MessageQueue中取出一个Message might block
		        
		        try {
		            msg.target.dispatchMessage(msg);//将msg分发到对应的handler
		            if (observer != null) {
		                observer.messageDispatched(token, msg);
		            }
		            dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
		        } catch (Exception exception) {
		          
		        }
		         
		    }
		}
	

从上文知道dispatchMessage就是对消息的具体的分发了,我们可以看到下面的代码趋势最终就是调用了handleMessage方法,而这个方法就是我们在初始化handler的时候自己实现的那个handleMessage函数,至此handler消息的发送与处理我们分析完成了
	
	public void dispatchMessage(@NonNull Message msg) {
	    if (msg.callback != null) {
	        handleCallback(msg);
	    } else {
	        if (mCallback != null) {
	            if (mCallback.handleMessage(msg)) {
	                return;
	            }
	        }
	        handleMessage(msg);
	    }
	}


# Looper的休眠

当然还记得上文我们提到过的如果说当前MessageQueue 中没有消息loop函数难道一直会循环的执行吗,这样岂不是很浪费CPU,答案是否定的,loop当然不会一直忙等待message的到来,那么loop函数到底是怎么被阻塞的呢,又是怎么被唤醒的呢,接下来我们具体分析,首先我们分析loop的阻塞.

在上面的loop处理消息中我们删除了部分代码,下面我们看有关被阻塞的关键代码,下面的代码是loop获取消息的关键代码,我们删除了无关代码,从下面的代码可以看到,其实我们是在循环的通过messagequeue.next()来获取一个Message


	/**
	 * Run the message queue in this thread. Be sure to call
	 * {@link #quit()} to end the loop.
	 */
	public static void loop() {
	    for (;;) {
	        Message msg = queue.next(); // 通过messageQueue.next获取一个Message
	    }
	}
	 

接下来看看next()函数中具体做了什么,在next函数中首先调用了nativePollOnce函数,这个函数是一个native函数,则根据Android的native的函数命名规范我们可得知其native的完整方法名是android_os_messagequeue_nativePollOnce,我们找到这个方法 


	Message next() {
	    
	    for (;;) {
	
	        nativePollOnce(ptr, nextPollTimeoutMillis);
	
	    }
	}


android_os_MessageQueue_nativePollOnce被java层调用,此处的ptr其实就是native层的NativeMessageQueue的引用,该方法将ptr转成NativeMessageQueue并调用poolOnce,下面看pollOnce函数

	static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
	        jlong ptr, jint timeoutMillis) {
	    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
	    nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
	}
	


NativeMessageQueue的pollonce函数又调用了Looper的pollOnce函数


	void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
	    mPollEnv = env;
	    mPollObj = pollObj;
	    mLooper->pollOnce(timeoutMillis);
	    mPollObj = NULL;
	    mPollEnv = NULL;
	}


Looper的pollOnce函数继续调用pollOnce函数

	int pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData);
	inline int pollOnce(int timeoutMillis) {
	    return pollOnce(timeoutMillis, nullptr, nullptr, nullptr);
	}


可以看到pollOnce函数中首先处理了还没有完成回调的message,如果不存在还没有回调的message的话则进入pollInner函数

	
	        int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
	            int result = 0;
	            for (;;) {
	                while (mResponseIndex < mResponses.size()) {//处理当前还没完成回调的Message
	            const Response& response = mResponses.itemAt(mResponseIndex++);
	                    int ident = response.request.ident;
	                    if (ident >= 0) {
	                        int fd = response.request.fd;
	                        int events = response.events;
	                        void* data = response.request.data;
	#if DEBUG_POLL_AND_WAKE
	                        ALOGD("%p ~ pollOnce - returning signalled identifier %d: "
	                                "fd=%d, events=0x%x, data=%p",
	                                this, ident, fd, events, data);
	#endif
	                        if (outFd != nullptr) *outFd = fd;
	                        if (outEvents != nullptr) *outEvents = events;
	                        if (outData != nullptr) *outData = data;
	                        return ident;
	                    }
	                }
	
	                result = pollInner(timeoutMillis);
	            }
	        }
       


pollinner函数相对比较复杂一些,我们只看最关键的代码,从下面可以看到在pollInner函数中调用了epoll_wait函数,而这个函数其实是epoll机制的重要函数,这个函数代表着 **等待事件的产生**,即在这里Looper的loop被阻塞了,陷入等待消息进入MessageQueue的状态


        int Looper::pollInner(int timeoutMillis) {


            // Poll.
            int result = POLL_WAKE;
            mResponses.clear();
            mResponseIndex = 0;

            // We are about to idle.
            mPolling = true;

            struct epoll_event eventItems[EPOLL_MAX_EVENTS];
            int eventCount = epoll_wait(mEpollFd.get(), eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

            
            return result;
        }


至此Looper的阻塞已经讨论完毕

接下来我们看看Looper是怎么被唤醒的

# Looper的唤醒

上文已经分析了Looper是如何陷入阻塞状态的,即MessageQueue中没有消息了,那么怎么唤醒Looper呢,其实很简单,发送一个消息,我们重新看一下消息的发送过程handler.sendMessage()->sendMessageDelayed()->sendMessageAtTime()->enqueueMessage->Messagequeue.enqueueMessage(),而在enqueueMessage函数中我们可以看到在将message添加进入MessageQueue的之后会调用 nativeWake(mPtr)函数,而这个函数就是用于唤醒Looper去处理message的,接下来我们看一下nativeWake函数,nativeWake的全名根据Android的定义为android_os_messagequeu_nativeWake,从下面代码可以看到是调用了nativeMessageQueue的wake函数


	static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jlong ptr) {
	    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
	    nativeMessageQueue->wake();
	}


接下来看NativeMessageQueue的wake函数,实际上是调用了Looper的wake()函数,接下来看Looper的wake函数

	void NativeMessageQueue::wake() {
	    mLooper->wake();
	}

从Looper的wake函数我们可以看到ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd.get(), &inc, sizeof(uint64_t)));而这一行其实就是向pipe管道中写入一个int 类型的 1 触发wakeEvent事件,下面是具体的函数调用,注释1处就是通过向管道写入1触发WakeEvent事件

	
	void Looper::wake() {
	#if DEBUG_POLL_AND_WAKE
	            ALOGD("%p ~ wake", this);
	#endif
	
        uint64_t inc = 1;
        ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd.get(), &inc, sizeof(uint64_t)));//1.通过向pipe中写入 "1" 来唤醒epoll的wakeEvent实践
        if (nWrite != sizeof(uint64_t)) {
            if (errno != EAGAIN) {
                LOG_ALWAYS_FATAL("Could not write wake signal to fd %d (returned %zd): %s",
                        mWakeEventFd.get(), nWrite, strerror(errno));
            }
        }
    }


前面说到Looper::pollInner的函数会调用epoll_wait函数进入阻塞状态,当有wakeevent事件的时候解除了在Looper:pollInner中epoll_wait的阻塞,则继续向下执行代码,接下来我们看Looper::pollInner的后续代码,下面的代码很复杂,但是其实主要就是做了两件是

1. 由于触发了wakeEvent事件 epoll_wait函数解除阻塞状态,代码继续执行
2. 处理当前的数据将当前的message封装在Response中返回



		
		
		int Looper::pollInner(int timeoutMillis) {
		#if DEBUG_POLL_AND_WAKE
		            ALOGD("%p ~ pollOnce - waiting: timeoutMillis=%d", this, timeoutMillis);
		#endif
		
		            // Adjust the timeout based on when the next message is due.
		            if (timeoutMillis != 0 && mNextMessageUptime != LLONG_MAX) {
		                nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
		                int messageTimeoutMillis = toMillisecondTimeoutDelay(now, mNextMessageUptime);
		                if (messageTimeoutMillis >= 0
		                        && (timeoutMillis < 0 || messageTimeoutMillis < timeoutMillis)) {
		                    timeoutMillis = messageTimeoutMillis;
		                }
		#if DEBUG_POLL_AND_WAKE
		                ALOGD("%p ~ pollOnce - next message in %" PRId64 "ns, adjusted timeout: timeoutMillis=%d",
		                        this, mNextMessageUptime - now, timeoutMillis);
		#endif
		            }
		
		            // Poll.
		            int result = POLL_WAKE;
		            mResponses.clear();
		            mResponseIndex = 0;
		
		            // We are about to idle.
		            mPolling = true;
		
		            struct epoll_event eventItems[EPOLL_MAX_EVENTS];
		            int eventCount = epoll_wait(mEpollFd.get(), eventItems, EPOLL_MAX_EVENTS, timeoutMillis);//
		
		            // No longer idling.
		            mPolling = false;
		
		            // Acquire lock.
		            mLock.lock();
		
		            // Rebuild epoll set if needed.
		            if (mEpollRebuildRequired) {
		                mEpollRebuildRequired = false;
		                rebuildEpollLocked();
		        goto Done;
		            }
		
		            // Check for poll error.
		            if (eventCount < 0) {
		                if (errno == EINTR) {
		            goto Done;
		                }
		                ALOGW("Poll failed with an unexpected error: %s", strerror(errno));
		                result = POLL_ERROR;
		        goto Done;
		            }
		
		            // Check for poll timeout.
		            if (eventCount == 0) {
		#if DEBUG_POLL_AND_WAKE
		                ALOGD("%p ~ pollOnce - timeout", this);
		#endif
		                        result = POLL_TIMEOUT;
		        goto Done;
		            }
		
		            // Handle all events.
		#if DEBUG_POLL_AND_WAKE
		            ALOGD("%p ~ pollOnce - handling events from %d fds", this, eventCount);
		#endif
		
		            for (int i = 0; i < eventCount; i++) {
		                int fd = eventItems[i].data.fd;
		                uint32_t epollEvents = eventItems[i].events;
		                if (fd == mWakeEventFd.get()) {
		                    if (epollEvents & EPOLLIN) {
		                        awoken();
		                    } else {
		                        ALOGW("Ignoring unexpected epoll events 0x%x on wake event fd.", epollEvents);
		                    }
		                } else {
		                    ssize_t requestIndex = mRequests.indexOfKey(fd);
		                    if (requestIndex >= 0) {
		                        int events = 0;
		                        if (epollEvents & EPOLLIN) events |= EVENT_INPUT;
		                        if (epollEvents & EPOLLOUT) events |= EVENT_OUTPUT;
		                        if (epollEvents & EPOLLERR) events |= EVENT_ERROR;
		                        if (epollEvents & EPOLLHUP) events |= EVENT_HANGUP;
		                        pushResponse(events, mRequests.valueAt(requestIndex));
		                    } else {
		                        ALOGW("Ignoring unexpected epoll events 0x%x on fd %d that is "
		                                "no longer registered.", epollEvents, fd);
		                    }
		                }
		            }
		            Done: ;
		
		            // Invoke pending message callbacks.
		            mNextMessageUptime = LLONG_MAX;
		            while (mMessageEnvelopes.size() != 0) {
		                nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
		        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
		                if (messageEnvelope.uptime <= now) {
		                    // Remove the envelope from the list.
		                    // We keep a strong reference to the handler until the call to handleMessage
		                    // finishes.  Then we drop it so that the handler can be deleted *before*
		                    // we reacquire our lock.
		                    { // obtain handler
		                        sp<MessageHandler> handler = messageEnvelope.handler;
		                        Message message = messageEnvelope.message;
		                        mMessageEnvelopes.removeAt(0);
		                        mSendingMessage = true;
		                        mLock.unlock();
		
		#if DEBUG_POLL_AND_WAKE || DEBUG_CALLBACKS
		                        ALOGD("%p ~ pollOnce - sending message: handler=%p, what=%d",
		                                this, handler.get(), message.what);
		#endif
		                        handler->handleMessage(message);
		                    } // release handler
		
		                    mLock.lock();
		                    mSendingMessage = false;
		                    result = POLL_CALLBACK;
		                } else {
		                    // The last message left at the head of the queue determines the next wakeup time.
		                    mNextMessageUptime = messageEnvelope.uptime;
		                    break;
		                }
		            }
		
		            // Release lock.
		            mLock.unlock();
		
		            // Invoke all response callbacks.
		            for (size_t i = 0; i < mResponses.size(); i++) {
		                Response& response = mResponses.editItemAt(i);
		                if (response.request.ident == POLL_CALLBACK) {
		                    int fd = response.request.fd;
		                    int events = response.events;
		                    void* data = response.request.data;
		#if DEBUG_POLL_AND_WAKE || DEBUG_CALLBACKS
		                    ALOGD("%p ~ pollOnce - invoking fd event callback %p: fd=%d, events=0x%x, data=%p",
		                            this, response.request.callback.get(), fd, events, data);
		#endif
		                    // Invoke the callback.  Note that the file descriptor may be closed by
		                    // the callback (and potentially even reused) before the function returns so
		                    // we need to be a little careful when removing the file descriptor afterwards.
		                    int callbackResult = response.request.callback->handleEvent(fd, events, data);
		                    if (callbackResult == 0) {
		                        removeFd(fd, response.request.seq);
		                    }
		
		                    // Clear the callback reference in the response structure promptly because we
		                    // will not clear the response vector itself until the next poll.
		                    response.request.callback.clear();
		                    result = POLL_CALLBACK;
		                }
		            }
		            return result;
		        }
		
		


接下来我们看一下java层的Looper中是怎么处理的,我们知道在java层中Looper的loop函数是一直在循环的读取message的,并且queue.next()方法会阻塞当前looper( 其实是epoll_wait()函数导致的阻塞,也就是上面提到的epoll::pollInner函数中 ),当next()函数获取到一个Message的时候,则进入我们最开始说的**关键的loop函数**这节内容所讲的Message的处理流程



	/**
	 * Run the message queue in this thread. Be sure to call
	 * {@link #quit()} to end the loop.
	 */
	public static void loop() {
	    
	    for (;;) {
	        Message msg = queue.next(); // 此时next()函数由于wakeEvent时间从阻塞中解除阻塞,从而获取到一个message
	         
	        try {
	            msg.target.dispatchMessage(msg);
	            if (observer != null) {
	                observer.messageDispatched(token, msg);
	            }
	            dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
	        } catch (Exception exception) {
	            if (observer != null) {
	                observer.dispatchingThrewException(token, msg, exception);
	            }
	            throw exception;
	        } finally {
	            ThreadLocalWorkSource.restore(origWorkSource);
	            if (traceTag != 0) {
	                Trace.traceEnd(traceTag);
	            }
	        }
	       
	    }
	}




至此,所有的消息的创建,分发,Looper的阻塞与唤醒均分析完毕