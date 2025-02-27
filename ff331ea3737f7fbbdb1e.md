---
title: デッドロックとconditional wait
tags: C++
author: dameyodamedame
slide: false
---
# 序

前回はマルチスレッドについて簡単に説明しました。大抵のモノは入口さえ説明すれば詳細で詰まることは少ないのですが、マルチスレッドについては、デッドロックの説明が必要かもしれない、ということで少々細かいことを説明します。

# マルチスレッドのデッドロックとは

複数のスレッドがロックを取得しようとしたときに、それら全てのスレッドがロックを取得できずに止まってしまう状態のことです。以下に示すコードを実行すると起こります。

## コード

```C++
#include <thread>
#include <mutex>
#include <chrono>
using namespace std;
using namespace std::chrono;
int main() {
    mutex m1, m2;
    thread t1([&]{
        lock_guard<mutex> l1(m1);
        this_thread::sleep_for(seconds(1));
        {lock_guard<mutex> l2(m2);}
    });
    thread t2([&]{
        lock_guard<mutex> l1(m2);
        this_thread::sleep_for(seconds(1));
        {lock_guard<mutex> l2(m1);}
    });
    t1.join();
    t2.join();
    return 0;
}
```

## 解説

t1スレッドはm1をロックして1秒停止し、t2スレッドはm2をロックして1秒停止します。
1秒後両スレッドは目覚めて処理を再開するのですが、t1は次のm2をロックしようとしてt2スレッドが既にロックを持っているのでロック開放を待ちます。
t2も同様にm1をロックしようとしてt1スレッドが既にロックを持っているのでロック開放を待ちます。
両スレッドが互いのロック開放を待ってしまうため、デッドロックする、という話です。

これは複数のmutexのロックが必要な処理でよく起こる問題です。簡単な回避方法としては、mutexのロック順序を同じにする、なのですが、実装の自由度を極端に下げてしまいますし、何よりチェックが大変です。

# conditional wait

そこで登場するのが条件付きのwaitです。mutexは共有されるデータを操作するときにロックを取る必要があるオブジェクトですが、共有されるデータを操作するための条件を表す変数そのものをmutexでロックするための機構です。

今回の要件ではm1とm2を同時にロックするという処理を、m1相当をロックしているスレッドのIDである条件変数id1とm2相当をロックしているスレッドのIDである条件変数id2を自分のスレッドIDに変更することでロックしている状態と見立てます。

## コード
```c++
#include <thread>
#include <mutex>
#include <condition_variable>
#include <chrono>
using namespace std;
using namespace std::chrono;
int main() {
    mutex m;
    condition_variable cond;
    const thread::id null_thread_id;
    thread::id id1 = null_thread_id, id2 = null_thread_id;
    auto f = [&]{
        const auto this_thread_id = this_thread::get_id();

        unique_lock<mutex> l(m);
        cond.wait(l, [&]{return id1 == null_thread_id && id2 == null_thread_id;});
        id2 = id1 = this_thread_id;
        cond.notify_all();
        l.unlock();
        
        this_thread::sleep_for(seconds(1));
        
        l.lock();
        cond.wait(l, [&]{return id1 == this_thread_id && id2 == this_thread_id;});
        id2 = id1 = null_thread_id;
        cond.notify_all();
    };
    thread t1(f);
    thread t2(f);
    t1.join();
    t2.join();
    return 0;
}
```

## 解説

条件変数を書き換えるためにcond.wait()を実施し、第1引数は書き換えのためにロックするmutex、その第2引数で実際にwaitする条件(条件変数から算出される真偽値)を記述しています。
書き換え終わったらnotify～するのがお約束です。これをすると、同じ条件変数をcondで待っていたスレッドが起き出して変更された条件変数を元に自身の条件が満たされているかチェックします。満たされていなければ再度待ち続けます。各waitが同じ条件とは限らないので、条件変数を書き換えたら原則notify_allします。
すると、最小限の同期で複雑な条件を満たす必要なロックを確保できるというカラクリです。

実装上ロックの位置が妙な感じがするかもしれませんが、cond.waitの際、実際にはmがロックされておらず、condがロックされているためです。条件変数の書き換え自体にはロックが必要なのですが、mをロックしっぱなしにすると他のスレッドがcond.waitできなくなってしまうので…。しかし実際にnotifyするまではmのロックが必要なのです。

# まとめ

- mutexを複数使用すると、デッドロックを起こす可能性がある
- 複数のmutexを使用し、複雑な条件のロックを実現したい場合はconditional waitを使用した方がいいかもしれない

# おまけ

ロック順序によるデッドロックの回避方法は他にもいくつかあります(std::adopt_lock, std::scoped_lock)。それらで済む場合はそちらを使うのも簡単です。

