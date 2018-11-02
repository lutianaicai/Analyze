# SVProgressHUD源码分析

众所周知 [SVProgressHUD](https://github.com/SVProgressHUD/SVProgressHUD) 是一个简洁易用的 HUD 库，我想探寻简洁易用背后的原理。

##Singleton

`SVProgressHUD` 同 `MBProgressHUD` 一样，都是 `UIView` 的子类，不同与 `MB` 的高度定制化, `SV` 提供的是单例，这也是它简洁的一大因素。

```objectivec
+ (SVProgressHUD*)sharedView {
    static dispatch_once_t once;
    
    static SVProgressHUD *sharedView;
#if !defined(SV_APP_EXTENSIONS)
    dispatch_once(&once, ^{ sharedView = [[self alloc] initWithFrame:[[[UIApplication sharedApplication] delegate] window].bounds]; });
#else
    dispatch_once(&once, ^{ sharedView = [[self alloc] initWithFrame:[[UIScreen mainScreen] bounds]]; });
#endif
    return sharedView;
}
```

常规的单例创建，根据是否是`App Extension`进行了判断以确定`Frame`大小

##Show Methods

`SV` 不像 `MB` 可以有较高的自由度定制，所以为了简洁，暴露的全是类方法

* `+ (void)show;`
* `+ (void)showWithStatus:(nullable NSString*)status;`
* `+ (void)showProgress:(float)progress;`
* `+ (void)showProgress:(float)progress status:(nullable NSString*)status;`

没有列举全部，因为都不是重点，他们不添加图片的最终会调用主方法:

```objectivec
- (void)showProgress:(float)progress status:(NSString*)status { }
```

有图的调用:

```objectivec
- (void)showImage:(UIImage*)image status:(NSString*)status duration:(NSTimeInterval)duration { }
```

以无图为例:

```objectivec
- (void)showProgress:(float)progress status:(NSString*)status {
    __weak SVProgressHUD *weakSelf = self;
    [[NSOperationQueue mainQueue] addOperationWithBlock:^{
        __strong SVProgressHUD *strongSelf = weakSelf;
        if(strongSelf){
            if(strongSelf.fadeOutTimer) {
                strongSelf.activityCount = 0;
            }
            
            // Set properties
            strongSelf.fadeOutTimer = nil;
            strongSelf.graceTimer = nil;
            ...
            
            // Choose the "right" indicator depending on the progress
            if(progress >= 0) {
                ...
                // Add ring to HUD
                if(!strongSelf.ringView.superview){
                    [strongSelf.hudView.contentView addSubview:strongSelf.ringView];
                }
                if(!strongSelf.backgroundRingView.superview){
                    [strongSelf.hudView.contentView addSubview:strongSelf.backgroundRingView];
                }
                ...
            } else {
                ...
                // Add indefiniteAnimatedView to HUD
                [strongSelf.hudView.contentView addSubview:strongSelf.indefiniteAnimatedView];
                if([strongSelf.indefiniteAnimatedView respondsToSelector:@selector(startAnimating)]) {
                    [(id)strongSelf.indefiniteAnimatedView startAnimating];
                }
                ...
            }
            
            // Fade in delayed if a grace time is set
            if (self.graceTimeInterval > 0.0 && self.backgroundView.alpha == 0.0f) {
                strongSelf.graceTimer = [NSTimer timerWithTimeInterval:self.graceTimeInterval target:strongSelf selector:@selector(fadeIn:) userInfo:nil repeats:NO];
                [[NSRunLoop mainRunLoop] addTimer:strongSelf.graceTimer forMode:NSRunLoopCommonModes];
            } else {
                [strongSelf fadeIn:nil];
            }
            
            ...
        }
    }];
}
```

首先是`weakSelf` `strongSelf`标准写法，老生常谈了避免循环引用以及提前释放。

然后设置一些属性，根据progress判断选择合适的指示符。

然后根据是否设置了`gracetime`调用或者延时调用`fadeIn`方法。

```objectivec
- (void)fadeIn:(id)data {
    // Update the HUDs frame to the new content and position HUD
    [self updateHUDFrame];
    [self positionHUD:nil];
    
    ...
    
    // Show if not already visible
    if(self.backgroundView.alpha != 1.0f) {
        ...
        
        __block void (^animationsBlock)(void) = ^{
            // Zoom HUD a little to make a nice appear / pop up animation
            self.hudView.transform = CGAffineTransformIdentity;
            
            // Fade in all effects (colors, blur, etc.)
            [self fadeInEffects];
        };
        
        __block void (^completionBlock)(void) = ^{
            // Check if we really achieved to show the HUD (<=> alpha)
            // and the change of these values has not been cancelled in between e.g. due to a dismissal
            if(self.backgroundView.alpha == 1.0f){
                // Register observer <=> we now have to handle orientation changes etc.
                [self registerNotifications];
                
                // Post notification to inform user
                [[NSNotificationCenter defaultCenter] postNotificationName:SVProgressHUDDidAppearNotification
                                                                    object:self
                                                                  userInfo:[self notificationUserInfo]];
                
                ...
                
                // Dismiss automatically if a duration was passed as userInfo. We start a timer
                // which then will call dismiss after the predefined duration
                if(duration){
                    self.fadeOutTimer = [NSTimer timerWithTimeInterval:[(NSNumber *)duration doubleValue] target:self selector:@selector(dismiss) userInfo:nil repeats:NO];
                    [[NSRunLoop mainRunLoop] addTimer:self.fadeOutTimer forMode:NSRunLoopCommonModes];
                }
            }
        };
        
        // Animate appearance
        if (self.fadeInAnimationDuration > 0) {
            // Animate appearance
            [UIView animateWithDuration:self.fadeInAnimationDuration
                                  delay:0
                                options:(UIViewAnimationOptions) (UIViewAnimationOptionAllowUserInteraction | UIViewAnimationCurveEaseIn | UIViewAnimationOptionBeginFromCurrentState)
                             animations:^{
                                 animationsBlock();
                             } completion:^(BOOL finished) {
                                 completionBlock();
                             }];
        } else {
            animationsBlock();
            completionBlock();
        }
        
        // Inform iOS to redraw the view hierarchy
        [self setNeedsDisplay];
    } else {
        // Update accessibility
        UIAccessibilityPostNotification(UIAccessibilityScreenChangedNotification, nil);
        UIAccessibilityPostNotification(UIAccessibilityAnnouncementNotification, self.statusLabel.text);
        
        // Dismiss automatically if a duration was passed as userInfo. We start a timer
        // which then will call dismiss after the predefined duration
        if(duration){
            self.fadeOutTimer = [NSTimer timerWithTimeInterval:[(NSNumber *)duration doubleValue] target:self selector:@selector(dismiss) userInfo:nil repeats:NO];
            [[NSRunLoop mainRunLoop] addTimer:self.fadeOutTimer forMode:NSRunLoopCommonModes];
        }
    }
}
```






