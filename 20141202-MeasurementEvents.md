Measurement Events
===

Google Analytics などを始めとしたトラッキングツールをアプリに組み込むことはよくあることです。
しかし、単純に実装するとテストが困難になったり、コードが散らばったりして、メンテナンスが困難になる原因になりがちです。

これを解決するため、Analytics のためのイベント発行を統一的に、テスト容易に実装するパターン Measurement Events をご紹介します。

Why
---

iOS アプリで、イベントをトラッキングするコードは例えば以下のようになるかと思います。

```objc
[SomeAnalytics trackEvent:@"someevent" args:@{
	@"somearg": @"somevalue"
}];
```

このコード自体は自明なのですが、このようにメソッド呼び出しを用いてトラッキングイベントを発行すると面倒がいくつかあります。

1. メソッド呼び出しがテストし辛い。これを真面目にやろうとすると、全モジュールへの Analytics モジュールの DI が必要になります。
2. イベントをフィルタリングし辛い。（「このイベントは送信するけど、このイベントは要らない。」といったことがし辛い。）
3. このコードを用いているモジュール全てで、特定ベンダーの Analytics モジュールへの依存が必要
4. どのようなイベントがどこでトラッキングされているかが、コードのあちこちに分散する。
5. トラッキングイベントの送信は頻出タスクなので、特定のベンダーに依存したコードがあちこちに散らばることになる。

この中で、2, 5 は Analytics Helper クラスを作って、ベンダーのコードをラップすればある程度解決出来ますが、他はダメです。

(3)のモジュール依存が必要になる問題も面倒で、アプリケーションを複数の CocoaPods モジュールで分けるといったモジュラーな設計を阻害する原因にもなります。

これらを解決するため、Event に `NSNotification` を用いることでイベント送信とトラッキング部分を分離し、PubSub なトラッキングを実現するパターンをご紹介します。

How
---

Measurement Events は、以下の特徴を持った `NSNotification` です。

1. name が統一されている
2. object は nil である
3. userInfo に `NSString` でイベントの名前を含んでいる
4. userInfo に `NSDictionary<NSString, id>` でイベントのパラメータを含んでいる

(1)の name が統一されているということがポイントで、この `NSNotification` は単一の Observer で受け取ることが可能です。この Observer でイベントをアグリゲートし、`userInfo` から必要な情報を取り出し、実際にトラッキングを行います。

Observer で受け取るイベントの名前をリストしておくことで、アプリケーションがトラッキングするイベントを一カ所に集めることが出来ます。このリストを見れば、このアプリケーションがどんなイベントをトラッキングしているかが一目瞭然となります。

トラッキングイベントを生成するモジュールは可能な限り多くの Measurement Events の発行に徹し、Observer はこれらをフィルタしてトラッキングを行う責務分担になります。

モジュールの依存の問題では、イベントを発行する部分はただの `NSNotification` なので依存はゼロです。
（実際的には Measurement Event を発行する軽い汎用的なモジュールに依存します。）

モジュールは発行する Measurement Events の名前のリストを公開することで、イベントを発行することを宣言することが出来ます。これは、一つのヘッダファイルにまとめておくと良いでしょう。

Testing Concerns
---

Measurement Events を使わない場合では、メソッド呼び出しをテストしなければならないので、テストが困難です。やろうと思えば全てのモジュールに Analytics を DI する必要が有ります。

Measurement Events はただの `NSNotification` なので、XCTestExpectation, Expecta や OCMock を使うことで簡単にテストすることが出来ます。

```objc
// XCTestExpectation

- (void)testItPostsMeasurementEvent {
    XCTestExpectation *expectation = [self expectationForNotification:kMeasurementEventsNotificationName object:nil handler:nil];
    
    [someModule someMethod];
    
    [self waitForExpectationsWithTimeout:1.0 handler:nil];
}
```

```objc
// Expecta

expect(^{
	[someModule someMethod];
}).to.notify(kMeasurementEventsNotificationName);
```

パラメータのテストは適宜行ってください。

Best Practices
---

### AQSEvent, AQSEventAggregator を用いる

`AQSEvent` を用いることで簡単に Measurement Event を発行することが出来ます。

```objc
[AQSEvent postEvent:@"somename" args:@{
	@"somearg": @"somevalue"
}];
```

また、`AQSEventAggregator` を用いることで、Measurement Events をフィルタリングしての購読が可能です。

```objc
@interface AppEventAggregator : AQSEventAggregator
@end

@implementation

// @override
- (NSArray *)whiteListForEvents {
    return @{
        @"app/run_at_first",
        @"app/did_become_active",
        @"app/permission/photo_library/granted"
    };
}

- (void)didReceiveEvent:(NSString *)eventName args:(NSDictionary *)eventArgs {
    [SomeAnalytics trackEvent:eventName args:eventArgs];
}

@end
```

```objc
[[AppEventAggregator sharedAggregator] startAggregation];
```

- AquaSupport/AQSEvent : https://github.com/AquaSupport/AQSEvent
- AquaSupport/AQSEventAggregator : https://github.com/AquaSupport/AQSEventAggregator

### イベント送信を Helper クラスでラップする

実際的には、イベントの名前、パラメータのスキーマやキーに間違いが生じやすいため、ヘルパークラスでラップすると良いでしょう。

```objc
+ (void)postSomeEventWithValue:(NSString *)value {
	[AQSEvent event:@"somename" args:@{
		@"somearg": value
	}];
}
```

### モジュールでは、発行する Event を　Header でリストしておく

Measurement Events をリストした Header であることを明示的にするため、`events_` Prefix を付け、モジュールの名称を次に続けることが推奨されます。

例: AFNetworking であれば、`events_AFNetworking.h`

また、定数名も `events_` Prefix を付けるとより分かりやすく、補完のしやすい定数になります。

```objc
// events_SomeModule.h

NSString *const events_SomeModuleSomeEventName = @"somename";
```

モジュールを使う側は、このヘッダーから必要なイベント名を取捨選択し、Aggregator のホワイトリストに含めていく形になります。

Conclusion
---

- トラッキングイベントを `NSNotification` で発行することで、テストが容易になる。
- トラッキングイベントの送信と、実際にトラッキングを行う部分を分離することが出来る。
- アプリケーションで扱うイベントを一カ所に集めて記述することが出来る。
- トラッキングイベントを `NSNotification` で発行することで、汎用的なトラッキングイベントの発行が可能になる。

References
---

- AquaSupport/AQSEvent : https://github.com/AquaSupport/AQSEvent
- AquaSupport/AQSEventAggregator : https://github.com/AquaSupport/AQSEventAggregator
- Xcode6で追加された、XCTestの新機能 - Qiita : http://qiita.com/nomadmonad/items/d2c283f9b9ad33a2b32a
