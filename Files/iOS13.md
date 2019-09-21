iOS 13 问题解决以及苹果登录

* ##### KVC修改私有属性可能Crash(不是所有)，需要用别的姿势替代。
    
    例如,UITextField的私有属性_placeholderLabel，如果 ``` [textField setValue:color forKeyPath:@"_placeholderLabel.textColor"]; ``` 会crash。
    
    可采用富文本形式 ``` _textField.attributedPlaceholder = [[NSMutableAttributedString alloc] initWithString:placeholder attributes:@{NSForegroundColorAttributeName : color}];```

* ##### 模态弹出时 modalPresentationStyle 改变了默认值
 
 在iOS13之前的版本中, UIViewController的UIModalPresentationStyle属性默认是UIModalPresentationFullScreen，而在iOS13中变成了UIModalPresentationPageSheet。
所以我们需要在presentViewController时，设置一下UIModalPresentationStyle，就可以达到旧的效果。


* ##### 获取DeviceToken姿势改变
 
 iOS13之前
 
    ```
    NSString *dt = [deviceToken description];
    dt = [dt stringByReplacingOccurrencesOfString: @"<" withString: @""];
    dt = [dt stringByReplacingOccurrencesOfString: @">" withString: @""];
    dt = [dt stringByReplacingOccurrencesOfString: @" " withString: @""];
    ```
 iOS13之后
 
     ```
     - (void)application:(UIApplication *)application didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken {
        if (![deviceToken isKindOfClass:[NSData class]]) return;
        const unsigned *tokenBytes = [deviceToken bytes];
        NSString *hexToken = [NSString stringWithFormat:@"%08x%08x%08x%08x%08x%08x%08x%08x",
                              ntohl(tokenBytes[0]), ntohl(tokenBytes[1]), ntohl(tokenBytes[2]),
                              ntohl(tokenBytes[3]), ntohl(tokenBytes[4]), ntohl(tokenBytes[5]),
                              ntohl(tokenBytes[6]), ntohl(tokenBytes[7])];
        NSLog(@"deviceToken:%@",hexToken);
    }
     ```
    
* ##### UIWebView


    苹果已经从iOS13禁止UIWebView方式了，需要更换WKWebView
    
* ##### 即将废弃的 LaunchImage    

    从iOS 8 苹果引入了 LaunchScreen，我们可以设置LaunchScreen来作为启动页。当然，现在你还可以使用LaunchImage来设置启动图。不过使用LaunchImage的话，要求我们必须提供各种屏幕尺寸的启动图，来适配各种设备，随着苹果设备尺寸越来越多，这种方式显然不够 Flexible。而使用 LaunchScreen的话，情况会变的很简单， LaunchScreen是支持AutoLayout+SizeClass的，所以适配各种屏幕都不在话下。
    从2020年4月开始，所有使⽤ iOS13 SDK的 App将必须提供 LaunchScreen，LaunchImage即将退出历史舞台。


* ##### MPMoviePlayerController 被禁止

    这个用的人应该不多了，如果是则更换姿势，如用AVPlayer
    
* ##### 增加苹果登录    

1. 去[开发者网站](https://developer.apple.com) 在 Sign in with Apple 开启功能
2. Xcode 里面 Signing & Capabilities 开启 Sign in with Apple 功能
3. 利用 ASAuthorizationAppleIDButton
    
    ```
    ASAuthorizationAppleIDButton *button = [ASAuthorizationAppleIDButton buttonWithType:ASAuthorizationAppleIDButtonTypeSignIn style:ASAuthorizationAppleIDButtonStyleWhite];
    [button addTarget:self action:@selector(signInWithApple) forControlEvents:UIControlEventTouchUpInside];
    button.center = self.view.center;
    button.bounds = CGRectMake(0, 0, 40, 40); // 宽度过小就没有文字了，只剩图标
    [self.view addSubview:button];
    ```
    
4. ASAuthorizationControllerPresentationContextProviding

    * ASAuthorizationControllerPresentationContextProviding 就一个方法，主要是告诉 ASAuthorizationController 展示在哪个 window 上  

    ```
    #pragma mark - ASAuthorizationControllerPresentationContextProviding
    
    - (ASPresentationAnchor)presentationAnchorForAuthorizationController:(ASAuthorizationController *)controller API_AVAILABLE(ios(13.0))
    {
        return self.view.window;
    }
    ```  

5. Authorization 发起授权登录请求

    ```
    - (void)signInWithApple API_AVAILABLE(ios(13.0)) {
        ASAuthorizationAppleIDProvider *provider = [[ASAuthorizationAppleIDProvider alloc] init];
        ASAuthorizationAppleIDRequest *request = [provider createRequest];
        request.requestedScopes = @[ASAuthorizationScopeFullName, ASAuthorizationScopeEmail];
        
        ASAuthorizationController *vc = [[ASAuthorizationController alloc] initWithAuthorizationRequests:@[request]];
        vc.delegate = self;
        vc.presentationContextProvider = self;
        [vc performRequests];
    }
    ```
    
    * ASAuthorizationAppleIDProvider 这个类比较简单，头文件中可以看出，主要用于创建一个 ASAuthorizationAppleIDRequest 以及获取对应 userID 的用户授权状态。在上面的方法中我们主要是用于创建一个 ASAuthorizationAppleIDRequest ，用户授权状态的获取后面会提到。
    * 给创建的 request 设置 requestedScopes ，这是个 ASAuthorizationScope 数组，目前只有两个值，ASAuthorizationScopeFullName 和 ASAuthorizationScopeEmail ，根据需求去设置即可。
    * 然后，创建 ASAuthorizationController ，它是管理授权请求的控制器，给其设置 delegate 和 presentationContextProvider ，最后启动授权 performRequests

6. 授权回调处理

    ```
    #pragma mark - ASAuthorizationControllerDelegate
    
    - (void)authorizationController:(ASAuthorizationController *)controller didCompleteWithAuthorization:(ASAuthorization *)authorization API_AVAILABLE(ios(13.0))
    {
        if ([authorization.credential isKindOfClass:[ASAuthorizationAppleIDCredential class]])       {
            ASAuthorizationAppleIDCredential *credential = authorization.credential;
            
            NSString *state = credential.state;
            NSString *userID = credential.user;
            NSPersonNameComponents *fullName = credential.fullName;
            NSString *email = credential.email;
            NSString *authorizationCode = [[NSString alloc] initWithData:credential.authorizationCode encoding:NSUTF8StringEncoding]; // refresh token
            NSString *identityToken = [[NSString alloc] initWithData:credential.identityToken encoding:NSUTF8StringEncoding]; // access token
            ASUserDetectionStatus realUserStatus = credential.realUserStatus;
            
            NSLog(@"state: %@", state);
            NSLog(@"userID: %@", userID);
            NSLog(@"fullName: %@", fullName);
            NSLog(@"email: %@", email);
            NSLog(@"authorizationCode: %@", authorizationCode);
            NSLog(@"identityToken: %@", identityToken);
            NSLog(@"realUserStatus: %@", @(realUserStatus));
        }
    }
    
    - (void)authorizationController:(ASAuthorizationController *)controller didCompleteWithError:(NSError *)error API_AVAILABLE(ios(13.0))
    {
        NSString *errorMsg = nil;
        switch (error.code) {
            case ASAuthorizationErrorCanceled:
                errorMsg = @"用户取消了授权请求";
                break;
            case ASAuthorizationErrorFailed:
                errorMsg = @"授权请求失败";
                break;
            case ASAuthorizationErrorInvalidResponse:
                errorMsg = @"授权请求响应无效";
                break;
            case ASAuthorizationErrorNotHandled:
                errorMsg = @"未能处理授权请求";
                break;
            case ASAuthorizationErrorUnknown:
                errorMsg = @"授权请求失败未知原因";
                break;
        }
        NSLog(@"%@", errorMsg);
    }
    ```
    
    * User ID: Unique, stable, team-scoped user ID，苹果用户唯一标识符，该值在同一个开发者账号下的所有 App 下是一样的，开发者可以用该唯一标识符与自己后台系统的账号体系绑定起来。

    * Verification data: Identity token, code，验证数据，用于传给开发者后台服务器，然后开发者服务器再向苹果的身份验证服务端验证本次授权登录请求数据的有效性和真实性，详见 Sign In with Apple REST API。如果验证成功，可以根据 userIdentifier 判断账号是否已存在，若存在，则返回自己账号系统的登录态，若不存在，则创建一个新的账号，并返回对应的登录态给 App。

    * Account information: Name, verified email，苹果用户信息，包括全名、邮箱等。

    * Real user indicator: High confidence indicator that likely real user，用于判断当前登录的苹果账号是否是一个真实用户，取值有：unsupported、unknown、likelyReal。

    * 失败情况会走 authorizationController:didCompleteWithError: 这个方法，具体看代码吧

7. 其他情况的处理

    * 用户终止 App 中使用 Sign in with Apple 功能
    * 用户在设置里注销了 AppleId
    
    这些情况下，App 需要获取到这些状态，然后做退出登录操作，或者重新登录。     
    我们需要在 App 启动的时候，通过 getCredentialState:completion: 来获取当前用户的授权状态

    ```
    - (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
        
        if (@available(iOS 13.0, *)) {
            NSString *userIdentifier = 钥匙串中取出的 userIdentifier;
            if (userIdentifier) {
                ASAuthorizationAppleIDProvider *appleIDProvider = [ASAuthorizationAppleIDProvider new];
                [appleIDProvider getCredentialStateForUserID:userIdentifier
                                                  completion:^(ASAuthorizationAppleIDProviderCredentialState credentialState,
                                                               NSError * _Nullable error)
                {
                    switch (credentialState) {
                        case ASAuthorizationAppleIDProviderCredentialAuthorized:
                            // The Apple ID credential is valid
                            break;
                        case ASAuthorizationAppleIDProviderCredentialRevoked:
                            // Apple ID Credential revoked, handle unlink
                            break;
                        case ASAuthorizationAppleIDProviderCredentialNotFound:
                            // Credential not found, show login UI
                            break;
                    }
                }];
            }
        }
        
        return YES;
    }
    ```
    
    ASAuthorizationAppleIDProviderCredentialState 解析如下：

    * ASAuthorizationAppleIDProviderCredentialAuthorized 授权状态有效；
    * ASAuthorizationAppleIDProviderCredentialRevoked 上次使用苹果账号登录的凭据已被移除，需解除绑定并重新引导用户使用苹果登录；
    * ASAuthorizationAppleIDProviderCredentialNotFound 未登录授权，直接弹出登录页面，引导用户登录

8. 还可以通过通知方法来监听 revoked 状态，可以添加 ASAuthorizationAppleIDProviderCredentialRevokedNotification 这个通知

    ```
    - (void)observeAppleSignInState {
        if (@available(iOS 13.0, *)) {
            [[NSNotificationCenter defaultCenter] addObserver:self
                                                     selector:@selector(handleSignInWithAppleStateChanged:)
                                                         name:ASAuthorizationAppleIDProviderCredentialRevokedNotification
                                                       object:nil];
        }
    }
    
    - (void)handleSignInWithAppleStateChanged:(NSNotification *)notification {
        // Sign the user out, optionally guide them to sign in again
        NSLog(@"%@", notification.userInfo);
    }
    ```
    
 9. One more thing

    苹果还把 iCloud KeyChain password 集成到了这套 API 里，我们在使用的时候，只需要在创建 request 的时候，多创建一个 ASAuthorizationPasswordRequest ，这样如果 KeyChain 里面也有登录信息的话，可以直接使用里面保存的用户名和密码进行登录。代码如下  
    
    ```
    - (void)perfomExistingAccountSetupFlows API_AVAILABLE(ios(13.0))
    {
        ASAuthorizationAppleIDProvider *appleIDProvider = [ASAuthorizationAppleIDProvider new];
        ASAuthorizationAppleIDRequest *authAppleIDRequest = [appleIDProvider createRequest];
        ASAuthorizationPasswordRequest *passwordRequest = [[ASAuthorizationPasswordProvider new] createRequest];
        
        NSMutableArray <ASAuthorizationRequest *>* array = [NSMutableArray arrayWithCapacity:2];
        if (authAppleIDRequest) {
            [array addObject:authAppleIDRequest];
        }
        if (passwordRequest) {
            [array addObject:passwordRequest];
        }
        NSArray <ASAuthorizationRequest *>* requests = [array copy];
        
        ASAuthorizationController *authorizationController = [[ASAuthorizationController alloc] initWithAuthorizationRequests:requests];
        authorizationController.delegate = self;
        authorizationController.presentationContextProvider = self;
        [authorizationController performRequests];
    }
    
    #pragma mark - ASAuthorizationControllerDelegate
    
    - (void)authorizationController:(ASAuthorizationController *)controller didCompleteWithAuthorization:(ASAuthorization *)authorization API_AVAILABLE(ios(13.0))
    {
        if ([authorization.credential isKindOfClass:[ASPasswordCredential class]]) {
            ASPasswordCredential *passwordCredential = authorization.credential;
            NSString *userIdentifier = passwordCredential.user;
            NSString *password = passwordCredential.password;
            
            NSLog(@"userIdentifier: %@", userIdentifier);
            NSLog(@"password: %@", password);
        }
    }
    ``` 