---
title: deferとlambdaとRVOの話
tags: C++
author: dameyodamedame
slide: false
---
C23でGoのdeferが入ったんだか入るんだかという話をどこかで聞いた気がします。
今回はC++11でdeferを簡易実装しようという話です。
ついでにlambdaとRVOの話もします。

### deferとは

> defer ステートメントは、 defer へ渡した関数の実行を、呼び出し元の関数の終わり(returnする)まで遅延させるものです。
> defer へ渡した関数の引数は、すぐに評価されますが、その関数自体は呼び出し元の関数がreturnするまで実行されません。

だそうです([A tour of Go](https://go-tour-jp.appspot.com/flowcontrol/12)より)。
自分の言葉で言えば、スコープから外れたときに呼ばれるクリーンアップ用のコールバック定義です[^1]。
例外のfinally節とかに近い感じ。

### C++で実装すると…

https://stackoverflow.com/questions/50182244/simple-way-to-execute-code-at-the-end-of-function-scope/50182372#50182372

これを参考にして書いたものが以下です。

```c++
#include <utility>
#include <iostream>
#include <algorithm>

using namespace std;

template <typename Func>
struct deferred_call {
    deferred_call(const deferred_call&) = delete;
    deferred_call(deferred_call&& org)
        : func(move(org.func))
        , owned(true)
    {
        std::cout << "moved" << std::endl;
        org.owned = false;
    }
    deferred_call& operator=(const deferred_call&) = delete;
    deferred_call& operator=(deferred_call&&) = delete;
    deferred_call(Func&& f)
        : func(forward<Func>(f))
        , owned(true)
    {}
    ~deferred_call() {
        if (owned) func();
    }
    bool owned;
    Func func;
};

template <typename Func>
deferred_call<Func> defer(Func&& f) {
    return deferred_call<Func>(std::forward<Func>(f));
}

int main() {
    try {
        char* buff = new char[0x1000];
        cout << "created" << endl;
        auto hoge = defer([&]{
            delete[] buff;
            cout << "deleted" << endl;
        });
        fill(buff, buff + 0x1000, 1);
        cout << "filled" << endl;
        throw "threw";
        fill(buff, buff + 0x1000, 2);
        cout << "2nd filled" << endl;
    }
    catch (const char* s) {
        cerr << s << endl;
    }
    return 0;
}
```

中身はほぼ同じですね。実に簡単な実装です。多少並列処理に弱くなってますが、そもそも同期が必要なタイミングで使ってはいけないと思うので気にしないことにしました。

実行するとこんな感じに出力されます。

```console
created
filled
deleted
threw
```

処理的には4KB確保した(created)バッファにfill処理をした(filled)後、例外が上がってスコープが外れると、deferで書かれたlambda式が動いてバッファが削除されて(deleted)、catchされた例外が捕捉されています。

実装としてはdeferに渡されたlambda式が、defered_call構造体に格納されて返されています。そのデストラクタで渡したlambda式が呼ばれる形です。**注意すべきポイントは以下の3つです**。

1. 生成されたdefered_callオブジェクトは消えるとlambda式を実行すること
1. 作成されたオブジェクトは戻り値として返され、コピーしたり移動したりする可能性があること
1. lambda式にはキャプチャされている変数があること

1.と2.からdefered_callオブジェクトが複数生成される可能性があり、その際は**複数回lambda式が呼ばれないようにしないといけない**ということになります。3.は今回参照キャプチャなので特に生存期間に注意しないと呼び出したときに参照した変数が生きていることが必要になります。

以下でそれぞれ細かく見ていきます。

### defered_callオブジェクトは複数生成されるのか？

結論から言うと上のコードでは通常複数生成されません。理由はコンパイラがRVO(Return Value Optimization)をしているために、1つしか生成されないからです。特にC++17以降では言語仕様からそうなっており、処理系に依存せず生成されません。

https://cpprefjp.github.io/lang/cpp17/guaranteed_copy_elision.html

今回はC++11をターゲットにしているので、処理系に依存していますが、gccなどは相当昔からRVOが入っているので、最適化を指定しなくても生成されないのです(ただしcopy/moveコンストラクタがない場合コンパイルエラーとなる。呼び出されないのに。。。)。

gcc/clangでRVOを止めるにはコンパイル時に`-fno-elide-constructors`オプションをつける必要があります。実際につけて実行すると仕込んでおいた出力が出てきます。

```console
created
moved
moved
filled
deleted
threw
```

movedという出力がそうなのですが、これはdefered_call構造体のmoveコンストラクタで出しているものです。2回出ているので、3つのオブジェクトが生成されています。ちなみに以下のように変更してownedメンバ変数に関係なくlambda式を実行すると、普通に実行してしまいます(gccの場合)。

```diff
@@ -21,7 +21,8 @@ struct deferred_call {
         , owned(true)
     {}
     ~deferred_call() {
-        if (owned) func();
+        //if (owned) func();
+        func();
     }
     bool owned;
     Func func;
@@ -37,7 +38,8 @@ int main() {
         char* buff = new char[0x1000];
         cout << "created" << endl;
         auto hoge = defer([&]{
-            delete[] buff;
+            // delete[] buff;
+            cout << static_cast<void*>(buff) << std::endl;
             cout << "deleted" << endl;
         });
         fill(buff, buff + 0x1000, 1);
```
```console
created
moved
0x55ba09b152b0
deleted
moved
0x55ba09b152b0
deleted
filled
0x55ba09b152b0
deleted
threw
```

つまりlambda式はstd::moveして右辺値にしても中身が移動しません(gccの場合)。

lambda式は

https://cpprefjp.github.io/lang/cpp11/lambda_expressions.html

によると実体は関数オブジェクトだそうなので、gccの場合は中身が移動できない(恐らくコピーになる)関数オブジェクトが生成されるようです。したがってownedメンバ変数は必要ということになります。

### 参照キャプチャのあるlambda式を移動しても動くのか？

gccを`-fsanitize=address`とかしても動いていたので、動くようです。

```console
created
moved
0x621000001500
deleted
moved
0x621000001500
deleted
filled
0x621000001500
deleted
threw

=================================================================
==746==ERROR: LeakSanitizer: detected memory leaks

Direct leak of 4096 byte(s) in 1 object(s) allocated from:
    #0 0x7f5e64937357 in operator new[](unsigned long) ../../../../src/libsanitizer/asan/asan_new_delete.cpp:102
    #1 0x560aeff69603 in main (/home/a.out+0x2603)
    #2 0x7f5e63af2d8f in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:58

SUMMARY: AddressSanitizer: 4096 byte(s) leaked in 1 allocation(s).
```
(※二重freeをさせないために故意にリークさせてる4KBしか報告がない上に全てのlambda式でキャプチャしてる変数のアドレスが同じなので)

ただ如何せん参照なので、参照先の生存期間を超えていると動きません。C++もこの辺は得意ではないので、今回のように生存期間が明示的にわかっている場合を除き、shared-pointerをコピーキャプチャして延命させたり、weak-referenceをコピーキャプチャして、参照の度に生存を確認する方が良いと思います。

とはいえ、前述したように **lambda式の実体は関数オブジェクト** だそうなので、難しく考えずにそういう前提で普通にコードを書けば問題ないと思います。

### まとめ

- C++でもdeferは実装できる
- RVOのおかげで戻り値はコピーも移動もしない場合がある
- lambda式の実体は関数オブジェクト

[^1]: @SaitoAtsushi さんの[コメント指摘](#comment-31c977d9dfed6fe39551)にあるとおり、deferはスタックもできるので、deferそのものをC++で実装することは難しく、この記事では出来ていません。またこの記事のdeferは標準で提案中の scope_exit や unique_resource で実現できるそうです。

