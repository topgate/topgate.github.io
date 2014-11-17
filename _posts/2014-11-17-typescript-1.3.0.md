TypeScript 1.3.0 変更点

TypeScriptリファレンスお買い上げありがとうございます！ [Amazon](http://www.amazon.co.jp/gp/product/484433588X?ie=UTF8&camp=1207&creative=8411&creativeASIN=484433588X&linkCode=shr&tag=damenako-22) [達人出版会](http://tatsu-zine.com/books/typescript-reference)
1.0.0の書籍なのですが、基本は変わってないので僕のQiitaの記事をフォローしてもらえればだいたいオッケーです(きり

TypeScript1.1.0-1が出て、それの[変更点紹介](http://qiita.com/vvakame/items/5e53c392867ebc604267)を書いてから1ヶ月強で1.3.0リリースとなりました。早いものです。

昨日夜、[Connect();](http://channel9.msdn.com/Events/Visual-Studio/Connect-event-2014)なるイベントがあり、[ニコ生](http://live.nicovideo.jp/watch/lv199630620)で実況もしてたらしいですが、Google信者の僕は酒かっくらってpull requestのレビューして寝てました。不覚。
今度何かMS系イベントがあるときは誰かpingください(小声

公式のブログ記事は[こちら](http://blogs.msdn.com/b/typescript/archive/2014/11/12/announcing-typescript-1-3.aspx)。

## ありがとうMicrosoftのTypeScriptチーム！

TypeScriptチームはコミュニティの意見をよく聞き、柔軟に対応してくれます。
以前はdocxしかなかった仕様も、[Markdownに変換したもの](https://github.com/Microsoft/TypeScript/blob/master/doc/spec.md)をホストしてくれるようになりました。
今回入ったprotectedもコミュニティからの要望によるものですし今後も今のような運営を続けていってくれると嬉しいです。

## protectedのサポート

[作業場所](https://github.com/Microsoft/TypeScript/pull/688)
[仕様](https://github.com/Microsoft/TypeScript/blob/release-1.3/doc/spec.md#8.2.2)

JavaとかC#とかにあるprotected修飾子そのものです。
クラスに子孫にしか見えない要素やメソッドなどを作成することができます。

[Playground](http://goo.gl/rKFCLP)

```typescript
class Base {
	protected hello() {
		return "Hello, world!";
	}
}

class Inherit extends Base {
	greeting() {
		return this.hello();
	}
}

var objA = new Base();
var objB = new Inherit();
var objC: any = new Base();

// protectedなので外からはアクセスできない！
objA.hello();

// Inherit#greeting内部では普通にhelloにアクセスできる
objB.greeting();

// anyにしちまえば関係ないのさ！
objC.hello();
```

しかしまぁご覧のとおり`any`を使うと普通にアクセスできてしまうので、僕個人の感想としてはあんまり意味がないですね。
privateも使わないで書こうマンだよ〜。
`--target es6`とかでは[Symbol](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol)とかを使って真にprivateにしてくれたりすると嬉しいですね。

protectedは今までpublicとかprivateとか書ける場所で使える感じです。

[Playground](http://goo.gl/1zyP0e)

```typescript
class Base {
	protected static str: string = "static";
	protected str: string = "instance";
	constructor( protected num: number ) { }
	protected static method() { }
	protected method() { }
}

class Inherit extends Base {
	static str2 = Base.str;
	static method() { Base.method(); }
}
```

## tuple typesのサポート

[作業場所](https://github.com/Microsoft/TypeScript/pull/428)
[仕様1](https://github.com/Microsoft/TypeScript/blob/release-1.3/doc/spec.md#3.3.3) [仕様2](https://github.com/Microsoft/TypeScript/blob/release-1.3/doc/spec.md#3.6.5)

tuple(たぷる)は、任意の数の要素の組です。
JavaScriptではtupleはサポートされていないため、TypeScriptでのtupleはただのArrayです。

[Playground](http://goo.gl/eGFuKz)

```typescript
var array = [1, "str", true]; // これはいままで通りの {}[]
var tuple: [number, string, boolean] = [1, "str", true]; // tuple!

array[1].charAt(0); // {} は charAt を持たない
tuple[1].charAt(0); // string は charAt を持つ！

// 普通にArrayでもあるのだ
tuple.forEach(v => {
	console.log(v);
});
```

[Playground](http://goo.gl/KUzzY2)

```
// 要素が多い分にはOKだ！
var tuple: [string, number] = ["str", 1, true];

// 型指定されていない要素は BCT(Best Common Type) つまりここでは {} になる
var value = tuple[2];
```

[Playground](http://goo.gl/nkZISk)

```typescript
// Genericsを使ってtupleを生成して返す
function zip<T1, T2>( v1: T1, v2: T2 ): [T1, T2] {
	return [v1, v2];
}

var tuple = zip( "str", { hello: () => "Hello!" });
tuple[0].charAt( 0 ); // おー、静的に検証される！
tuple[1].hello(); // おー、静的に検証される！
```

tupleがなかった今までは、オブジェクト型リテラルで頑張るしかありませんでした。
でも、これって辛いよね。

[Playground](http://goo.gl/X8KI11)

```typescript
var tuple: {0: number; 1: string; 3: boolean;} = <any>[1, "str", true];

tuple[1].charAt(0); // string は charAt を持つ！

// だがしかし(型のうえでは)Arrayではない…
tuple.forEach(v => {
	console.log(v);
});
```

とはいえ、使うのがめんどくさいし今はあまり便利なfeatureとは言えないでしょう。
[spread operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_operator)のサポートが入ってからが真の活躍が始まるのかもしれません。

## その他の変更点

### enumの要素に数値ぽいものが許されなくなった

今までは以下のようなコードのコンパイルが通ってたけど許可されなくなった。

[Playground](http://goo.gl/jA5IRa)

```typescript
enum Sample {
	'0' = 10,
	'2' = 0
}

alert( Sample['0'] ); // 10がとれてほしいけど2がとれる
```

でも半角スペースとかタブとか、まぁ許容されてほしいものも許容されなくなってたのでとりあえず[報告した](https://github.com/Microsoft/TypeScript/issues/1144)。

### 関数の返り値が暗黙的にanyになるのが許されなくなった

のかな？

```typescript
function test(v: any) {
  return test(v);
}
```

```
$ tsc -v
message TS6029: Version 1.3.0.0
$ tsc --noImplicitAny test.ts
test.ts(1,10): error TS7023: 'test' implicitly has return type 'any' because it does not have a return type annotation and is referenced directly or indirectly in one of its return expressions.

$ tsc -v
message TS6029: Version 1.1.0.1
$ tsc --noImplicitAny test.ts
# 普通に成功する
```

### 再帰的に自分を参照する変数の型推論がanyになるのが許されなくなった

のかな？

```typescript
var recursive = {
    rec: recursive
};
```

```
$ tsc -v
message TS6029: Version 1.3.0.0
$ tsc --noImplicitAny test.ts
test.ts(1,5): error TS7022: 'recursive' implicitly has type 'any' because it is does not have a type annotation and is referenced directly or indirectly in its own initializer.

$ tsc -v
message TS6029: Version 1.1.0.1
$ tsc --noImplicitAny test.ts
# 普通に成功する
```

## これからのTypeScript

[Roadmap](https://github.com/Microsoft/TypeScript/wiki/Roadmap)

次の1.4で構文的なES6サポートが入り始めたり、TypeScriptコンパイラの内部APIが公開されstableになったりする予定みたいです。
他にも、union typesが入ったりtype aliasが入ったりするため、本格的なお祭りは1.4をお待ちください。
