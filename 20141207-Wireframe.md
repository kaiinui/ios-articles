Wireframe
===

Wireframe は別名 View Controller Factory です。

Why?
---

通常であれば、Segue や Storyboard からの View Controller 初期化を行い、必要なパラメータを渡して View Controller をインスタンス化します。

```objc
SomeViewController *nextViewController = [[UIStoryboard storyboardWithName:@"Main" inBundle:[NSBundle mainBundle]] instantiateViewControllerWithStoryboardIdentifier:@"SomeViewController"];
nextViewController.someparam = @"somevalue"

[self.navigationController pushViewController:nextViewController animated:YES];
```

これは次の問題をはらんでいます。

1. 今の View Controller が遷移先である `SomeViewController` について知りすぎている。
2. Storyboard を分割したいときに、全ての同様のコードを書き換えなければならない。
3. Storyboard Identifier 及び Storyboard name が文字列となっており、常に Storyboard との不一致の脅威がある。

本来であれば、今の View Controller は遷移先に何のパラメータを渡せばいいかだけを知っていれば遷移出来るようになるべきです。
また、View Controller は遷移先の View Controller を生成する責務を負うべきでは有りません。
ですから、次のようなコードで遷移を行えるようなレイヤ分けを行います。

```objc
SomeViewController *nextViewController = [MYWireframe someViewControllerWithSomeParam:@"somevalue"];

[self.navigationController pushViewController:nextViewController animated:YES];
```

How?
---

Wireframe は非常に単純です。クラスメソッドに先ほどのようなコードを持つだけです。

```objc
@class SomeViewController;

@interface MYWireframe : NSObject

/**
 *  SomeViewController をインスタンス化します。
 *  
 *  @param someParam ○○に使うパラメータです。
 *
 *  @return 初期化された SomeViewController
 */
+ (SomeViewController *)someViewControllerWithSomeParam:(NSString *)someParam;

@end

@implementation MYWireframe

+ (SomeViewController *)someViewControllerWithSomeParam:(NSString *)someParam {
    YOURParameterAssert(someParam); // ここに Assert を入れるかどうかは好みです。

    SomeViewController *someViewController = [[self mainStoryboard] instantiateViewControllerWithStoryboardIdentifier:@"SomeViewController"];
    someViewController.param_someParam = someParam;
    
    return someViewController;
}

# pragma mark - Helpers

+ (UIStoryboard *)mainStoryboard {
    return [UIStoryboard storyboardWithName:@"Main" inBundle:[NSBundle mainBundle]];
}

@end
```

`SomeViewController` のパラメータのためのプロパティが `param_someParam` となっていますが、View Controller において外から渡されるパラメータには `param_` Prefix を付けると明示的で良いです。ここは好みでしょう。

インスタンスメソッドにして、DI が出来るようにしても良いですが、実際的には View Controller を Stub することは少ないです。
ですので、クラスメソッドで構成されたヘルパーメソッド群の構成で十分でしょう。

このように一カ所に View Controller の生成をまとめることで、各 View Controller は遷移先の View Controller を具体的にインスタンス化する責務から開放されます。

また、`MYWireframe` の各メソッドについて、最低限 `nil` が返ってないかを返すテストを書いておくことで、Storyboard 上での変更にも強くなります。

ここで、`YOURAssert` は次のように定義されたマクロです。このように `NSParameterAssert` を一段ラップして使う利点は、マクロ の章を参照して下さい。

```objc
#ifdef YOUR_DEBUG
    #define YOURParameterAssert(condition) NSParameterAssert(condition)
#else
    #define YOURParameterAssert(condition)
#endif
```

Testing Concerns
---

最低限次のようなテストを書いておけば、Storyboard Identifier の不一致でのインスタンス化失敗に強くなります。

```objc
- (void)testItInstantiateSomeViewController {
    XCTAssertNotNil([MYWireframe someViewControllerWithSomeParam:@"somevalue"]);
}
```

実際はパラメータも含めてテストを書いておきます。

```objc
- (void)testItInstantiateSomeViewControllerWithParams {
    XCTAssertNotNil([MYWireframe someViewControllerWithSomeParam:@"somevalue"]);
    XCTAssertEqualObjects([MYWireframe someViewControllerWithSomeParam:@"somevalue"].param_someParam, @"someValue");
}
```

Advanced Techniques
---

### `UIActivityViewController`

`UIActivityViewController` は View 層から呼び出されることが常のため、常にテスト困難な存在です。

Wireframe を使えば、いくらかテスト可能になります。

```objc
+ (UIActivityViewController *)activityViewControllerWithActivityItems:(NSArray *)activityItems completion:^(NSString *activityType, BOOL success)completionBlock {
    NSArray *applicationActivities = ...; // 自前の UIActivity
    UIActivityViewController *activityViewController = [[UIActivityViewController alloc] initWithActivityItems:activityItems applicationActivities:applicationActivities];
    activityViewController.excludedActivityTypes = @[...]; // 不必要な UIActivity
    activityViewController.completionBlock = ^(NSString *activityType, BOOL success) {
        [MYTracking trackActivity:activityType success:success];
    
        completionBlock(activityType, success);
    };
    
    return activityViewController;
}
```

これで利用する側 (= View) からは、次のようなコードで利用可能になります。

```objc
UIActivityViewController *activityViewController = [MYWireframe activityViewControllerWithActivityItems:@[...] completion:^ (NSString *activityType, BOOL success){
    // DO SOMETHING WITH ACTIVITY_TYPE AND SUCCESS
}];

[self presentViewController:activityViewController animated:YES completion:nil];
```

Activity をトラッキングするために、`MYTracking` クラスを用いていますが、Measurement Events を使っていることが前提です。
AQSEvent で Measurement Events を運用していることを前提に解説します。

```objc
// 不要な UIActivity を表示してないかテスト
- (void)testItExcludesUnnecessaryActivity {
    UIActivityViewController *activityViewController = [MYWireframe activityViewControllerWithActivityItems:nil completion:nil];
    
    XCTAssertEqualObjects(activityViewController.excludedActivityTypes, @[
        UIActivityTypePrint,
        UIActivityTypeCopyToPasteboard,
        UIActivityTypeAssignToContact
    ]);
}

// Measurement Event を post してるかテスト
- (void)testItPostsMeasurementEvent {
    UIActivityViewController *activityViewController = [MYWireframe activityViewControllerWithActivityItems:nil completion:nil];
    
    XCTestExpectation *expectation = [self expectationForNotification:kAQSEvent object:nil handler:nil];
    
    activityViewController.completionBlock(@"sometype", YES);
    
    [self waitForNotificationWithTimeout:1.0 handler:nil];
}

// 渡した Completion Block が正常に呼ばれているかのテスト
- (void)testItCallsPassedCompletionBlockOnCompletion {
    __block BOOL called = NO;
    UIActivityViewController *activityViewController = [MYWireframe activityViewControllerWithActivityItems:nil completion:^(NSString *activityType, BOOL success) {
        called = YES;
    }];
    
    activityViewController.completionBlock(@"sometype", nil);
    
    XCTAssertTrue(called);
}
```

### Segue

Segue は `- prepareForSegue:sender:` を実装することで、Storyboard 上で設定した遷移に対して、遷移時の処理（パラメータ渡し等）を行うことが出来るメカニズムです。

Segue にも対応することが出来ます。

```objc
+ (void)handleSegueForYOURViewController:(UIUIStoryboardSegue *)segue sender:(id)sender withSomeParam:(NSString *)someParam {
    if ([segue.destinationViewController isKindOfClass:[SomeViewController class]]) {
        SomeViewController *someViewController = segue.destinationViewController;
        someViewController.param_someParam = someParam;
    }
}
```

（実際にはパラメータを設定する部分は共通化しておきます。）

これを View Controller の `- prepareForSegue:sender:` で呼び出します。

```objc
// YOURViewController

- (void)prepareForSegue:(UIStoryboardSegue *)segue sender:(id)sender {
    [MYWireframe handleSegueForYOURViewController:segue sender:sender withSomeParam:@"somevalue"];
}
```

References
---

- UIActivityViewController Class Reference : https://developer.apple.com/library/ios/documentation/UIKit/Reference/UIActivityViewController_Class/index.html
- UIActivity Class Reference : https://developer.apple.com/library/ios/documentation/UIKit/Reference/UIActivity_Class/index.html
- Xcode6で追加された、XCTestの新機能 - Qiita : http://qiita.com/nomadmonad/items/d2c283f9b9ad33a2b32a
