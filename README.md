## [LightWeightRunLoop](https://github.com/wuyunfeng/LightWeightRunLoop.git)

> Using BSD kqueue realize iOS RunLoop and some Runloop-Relative Fundation API such as perform selector(or delay some times) on other thread , Timer, URLConnection, LWStream(LWInputStream、LWOutputStream) etc..

## Main Features

* **NSObject** - *postSelector:onThread:withObject:afterDelay:modes* etc.

* **LWTimer** - Extremely like *NSTimer*, `nanosecond` precision.

* **LWURLConnection** - Extremely like *NSURLConnection*.

* **LWInputStream** - Extremely like *NSInputStream* (`Design Failure according to Java InputStream`).

* **LWOutputStream** - Extremely like *NSOutputStream(`Design Failure according to Java InputStream`)*.


## Future Features

* **LWPort** - Extremely like *NSMachPort*.

* **Socket Pools** - reuse socket.

## Get Started
Each NSThread object, `excluding the application’s main thread`, can own an `LWRunLoop` object. You can get the current thread’s  `LWRunLoop`, through the class method *`currentLWRunLoop`*. Subsequently code snippet shows how configure LWRunLoop for NSThread and make the NSThread `_lwRunLoopThread` entering into `Event-Driver-Mode:`


     NSThread *_lwRunLoopThread = [[NSThread alloc] initWithTarget:self selector:@selector(lightWeightRunloopThreadEntryPoint:) object:nil];
        - (void)lightWeightRunloopThreadEntryPoint:(id)data {
    @autoreleasepool {
        LWRunLoop *looper = [LWRunLoop currentLWRunLoop];
        [looper run];
        
        //or
        [[LWRunLoop currentLWRunLoop] run];
        }
    }
       
##To enqueue a selector to be performed on a different thread than your own &&  schedule a selector to be executed at some point in the future
you can use the category of NSObject(post)

* -(void)postSelector:(SEL)aSelector onThread:(NSThread *)thread withObject:(id)arg;

* -(void)postSelector:(SEL)aSelector onThread:(NSThread *)thread withObject:(id)arg afterDelay:(NSInteger)delay

* -(void)postSelector:(SEL)aSelector onThread:(NSThread *)thread withObject:(id)arg afterDelay:(NSInteger)delay modes:(NSArray<NSString *> modes)

lwrunloop modes are:



	NSString * const  LWDefaultRunLoop = @"LWDefaultRunLoop";
	NSString * const  LWRunLoopCommonModes = @"LWRunLoopCommonModes";
	NSString * const  LWRunLoopModeReserve1 = @"LWRunLoopModeReserve1";
	NSString * const  LWRunLoopModeReserve2 = @"LWRunLoopModeReserve2";
	NSString * const  LWTrackingRunLoopMode = @"LWTrackingRunLoopMode";

you can switch runloop mode at any time you can like this:

		    LWRunLoop *runLoop = [_lwModeRunLoopThread looper];
            [runLoop changeRunLoopMode:LWRunLoopModeReserve2];
you can use the category of NSObject(post):


      [self postSelector:@selector(execute) onThread:_lwRunLoopThread withObject:nil];
      [self postSelector:@selector(execute) onThread:_lwRunLoopThread withObject:nil afterDelay:5000];
      [self postSelector:@selector(execute) onThread:_lwRunLoopThread withObject:nil afterDelay:5000 modes:@[LWDefaultRunLoop]];
      


##You use the LWTimer class to create timer objects or, more simply, timers. 
A timer waits until a certain time interval has elapsed and then fires, sending a specified message to a target object. 


* +(LWTimer *)scheduledLWTimerWithTimeInterval:(NSTimeInterval)interval target:(id)aTarget selector:(SEL)aSelector userInfo:(id)userInfo repeats:(BOOL)yesOrNo;

* +(LWTimer *)timerWithTimeInterval:(NSTimeInterval)interval target:(id)aTarget selector:( SEL)aSelector userInfo:(nullable id)userInfo repeats:(BOOL)yesOrNo;


fire the LWTimer using `- (void)fire` and invalidate the LWTimer using `- (void)invalidate`

#####For example:

run the `- (void)genernateLWTimer` on `_lwRunLoopThread`:

    [self postSelector:@selector(genernateLWTimer) onThread:_lwRunLoopThread withObject:nil];
    
    - (void)genernateLWTimer
    {
        _count = 0;
        LWTimer *timer = [LWTimer timerWithTimeInterval:1000 target:self selector:@selector(bindLWTimerWithSelector:) userInfo:nil repeats:YES];
    [timer fire];
       //gTimer = [LWTimer scheduledLWTimerWithTimeInterval:2000 target:self selector:@selector(bindLWTimerWithSelector:) userInfo:nil repeats:YES];
    }
    
the selector for `LWTimer` to be executed:
    
    - (void)bindLWTimerWithSelector:(LWTimer *)timer
    {
        _count++;
        NSLog(@"* [ LWTimer : %@ performSelector: ( %@ ) on Thread : %@ ] *", [self class], NSStringFromSelector(_cmd), [NSThread currentThread].name);
        if (_count == 4) {
            [timer invalidate];
        }
    }   
##An LWURLConnection object lets you load the contents of a URL by providing a URL request object.
Step1: Perform `LWURLConnection` on `_lwRunLoopThread`:

	- (void)executeURLConnection:(UIButton *)button
	{
       [self postSelector:@selector(performURLConnectionOnRunLoopThread) onThread:_lwRunLoopThread withObject:nil];
	}
	
Step2: Create `LWURLConnection`, schedule `LWURLConnection` to `_lwRunLoopThread` such as following code snippet:

	- (void)performURLConnectionOnRunLoopThread
	{
        NSLog(@"[%@ %@]", [self class], NSStringFromSelector(_cmd));
        NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:@"http://192.168.1.8:8888/post.php"]];
       request.HTTPMethod = @"POST";
       NSString *content = @"name=john&address=beijing&mobile=140005";
       request.HTTPBody = [content dataUsingEncoding:NSUTF8StringEncoding];
       LWURLConnection *conn = [[LWURLConnection alloc] initWithRequest:request delegate:self startImmediately:NO];
       [conn scheduleInRunLoop:_lwRunLoopThread.looper];
       [conn start];
    }
Step3: Implement the delegate methods on Receiver:

	@protocol LWURLConnectionDataDelegate <NSObject>

	- (void)lw_connection:(LWURLConnection * _Nonnull)connection didReceiveData:(NSData * _Nullable)data;
	- (void)lw_connection:(LWURLConnection * _Nonnull)connection didFailWithError:(NSError * _Nullable)error;
	- (void)lw_connectionDidFinishLoading:(LWURLConnection * _Nonnull)connection;
	@end

Step4: You can use `LWURLResponse` to format Http response
   
    LWURLResponse *response = [[LWURLResponse alloc] initWithData:_responseData];
## LWInputStream & LWOutputStream(later maybe refator some code)
######Initilize LWInputStream or LWOutputStream : 
		
		  _lwInputStream = [[LWInputStream alloc]initWithFileAtPath:filePath];
		  _lwInputStream.delegate = self;
    	  [_lwInputStream scheduleInRunLoop:[_thread looper] forMode:LWDefaultRunLoop];
    	  [_lwInputStream open];
or

          _lwOutputStream = [[LWOutputStream alloc]initWithFileAtPath:filePath];
    	  _lwOutputStream.delegate = self;
          [_lwOutputStream scheduleInRunLoop:[_thread looper] forMode:LWDefaultRunLoop];
          [_lwOutputStream open];
          
######Then read or write data on the selector of delegate,such as:

          
	 switch (eventCode) {
            case LWStreamEventOpenCompleted:
                 break;
            case LWStreamEventHasBytesAvailable:
            {
                uint8_t buffer[1];
                NSUInteger len = [(LWInputStream *)aStream read:buffer maxLength:sizeof(buffer)];
                [_inputStreamData appendBytes:buffer length:len];
            }
                break;
            case LWStreamEventEndEncountered:
            {
                NSString *content = [[NSString alloc] initWithData:_inputStreamData encoding:NSUTF8StringEncoding];
                NSLog(@"content = %@", content);
                [(LWInputStream *)aStream close];
            }
                break;
            default:
                break;
        }
or 

		        switch (eventCode) {
            case LWStreamEventOpenCompleted:
                break;
            case LWStreamEventHasSpaceAvailable:
            {
                uint8_t buffer[50] = "abcdefghijklmnopqrstuvwxyz";
                NSUInteger len = [(LWOutputStream *)aStream write:buffer maxLength:50];
                [_inputStreamData appendBytes:buffer length:len];
                [(LWOutputStream *)aStream close];
            }
                break;
            case LWStreamEventEndEncountered:
            {
            }
                break;


##[Links](https://github.com/wuyunfeng/PHPMApi.git)
