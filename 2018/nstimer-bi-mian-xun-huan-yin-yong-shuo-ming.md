# NSTimer避免循环引用说明

## 添加 Category/Extension \(iOS10之前\)

#### Objective-C

1. 为 NSTimer 添加一个 category，添加一个传入 block 参数的接口。==将此 block 作为 NSTimer 的 userInfo 参数传入，而 NSTimer 的 target 为 timer 自己==，从而避免 NSTimer 持有 ViewController，避免了循环引用。代码如下:

```swift
// NSTimer+BlocksSupport.h
import <Foundation/Foundation.h>
@interface NSTimer (BlocksSupport)
(NSTimer *)hs_scheduledTimerWithTimeInterval:(NSTimeInterval)interval 
                                 repeats:(BOOL)repeats
                                   block:(void(^)())block;
@end
// NSTimer+BlocksSupport.m
import "NSTimer+BlocksSupport.h"
@implementation NSTimer (BlocksSupport)
(NSTimer *)hs_scheduledTimerWithTimeInterval:(NSTimeInterval)interval
                                 repeats:(BOOL)repeats
                                   block:(void(^)())block;
{
  return [self scheduledTimerWithTimeInterval:interval
                                   target:self
                                 selector:@selector(hs_invokeBlock:)
                                 userInfo:[block copy]
                                  repeats:repeats];
}
(void)hs_invokeBlock:(NSTimer *)timer {
  void (^block)() = timer.userInfo; 
  if(block) { block(); } 
} 
@end
```

创建NSTimer

```text
- (void)startTimer
{
    __weak typeof(self)weakSelf = self;
    self.timer = [NSTimer hs_scheduledTimerWithTimeInterval:5.0 repeats:YES handlerBlock:^void(void){
        __strong typeof(weakSelf)strongSelf = weakSelf;
        [strongSelf excuteAction];
    }];
}

- (void)excuteAction
{
    //Todo...;
}
```

销毁NSTimer对象

```text
- (void)stopTimer
{
    [self.timer invalidate];
    self.timer = nil;
}

- (void)dealloc
{
    [self.timer invalidate];
}
```

#### Swift

```swift
extension Timer {

    public class func hs_scheduledTimer(withTimeInterval interval: TimeInterval, repeats: Bool, block: @escaping (Timer) -> Void) -> Timer {
        return self.scheduledTimer(timeInterval: interval, target: self, selector: #selector(self.excuteTimerBlock(_:)), userInfo: block, repeats: true)
    }

    @objc private class func excuteTimerBlock(_ timer: Timer) {
        if let block = timer.userInfo as? (Timer) -> Void {
            block(timer)
        }
    }
}
```

## 使用新的 API \(iOS10 之后\)

iOS 10 之后， NSTimer 添加了支持传入 block 类型参数的 API

Objective-C

```text
// Creates a timer and schedules it on the current run loop in the default mode.
+ (NSTimer *)scheduledTimerWithTimeInterval:(NSTimeInterval)interval 
                                    repeats:(BOOL)repeats
                                      block:(void (^)(NSTimer *timer))block;

// Initializes a timer object with the specified time interval and block.
+ (NSTimer *)timerWithTimeInterval:(NSTimeInterval)interval
                           repeats:(BOOL)repeats
                             block:(void (^)(NSTimer *timer))block;

// Initializes a timer for the specified date and time interval with the specified block.
- (instancetype)initWithFireDate:(NSDate *)date 
                        interval:(NSTimeInterval)interval 
                         repeats:(BOOL)repeats 
                           block:(void (^)(NSTimer *timer))block;
```

Swift

```swift
/// Creates and returns a new NSTimer object initialized with the specified block object. This timer needs to be scheduled on a run loop (via -[NSRunLoop addTimer:]) before it will fire.
@available(iOS 10.0, *)
public /*not inherited*/ init(timeInterval interval: TimeInterval, repeats: Bool, block: @escaping (Timer) -> Swift.Void)


/// Creates and returns a new NSTimer object initialized with the specified block object and schedules it on the current run loop in the default mode.
@available(iOS 10.0, *)
open class func scheduledTimer(withTimeInterval interval: TimeInterval, repeats: Bool, block: @escaping (Timer) -> Swift.Void) -> Timer


// Initializes a new NSTimer object using the block as the main body of execution for the timer. This timer needs to be scheduled on a run loop (via -[NSRunLoop addTimer:]) before it will fire.
@available(iOS 10.0, *)
public convenience init(fire date: Date, interval: TimeInterval, repeats: Bool, block: @escaping (Timer) -> Swift.Void)
```

