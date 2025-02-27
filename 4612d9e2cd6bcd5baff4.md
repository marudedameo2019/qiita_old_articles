---
title: signalを使ってみる
tags: C++ UNIX POSIX Linux
author: dameyodamedame
slide: false
---
signalって何だろう？って思う人も、`kill 12345`みたいなことはしたことあるかもしれない。Unix系OSを使う人なら割とよく使うコマンドだと思う。指定したプロセスを文字通りやってしまう(kill)コマンドです。

このとき使われてるのがsignalです。指定したPIDを持つプロセスにSIGTERMというsignalを送っています。他だと、端末からCtrl+Cをしたときに出るのがSIGINTというsignalです。これらのsignalはプロセスが受け取り、処理して終了したりしています(SIGKILLなど受け取れないものもある)。

今日はコレを使ってみようという試みです。

# 前提

https://qiita.com/rarul/items/090920b850acc4b7e910

まずはコレを読みましょう。signalは割り込みというとても原始的でsensitiveな仕組みなので、他の仕組みと比較してアプリが守られていないし、自衛するための機能も不十分です。製品などで扱う場合には細心の注意を払って使うか、安易に使わないべきということです。

UnixライクなOSの中でもLinuxではGNU C Libraryがよく使用されます。私がよく使用しているUbuntu22.04でも使用されており、バージョンは現在2.35なので、そのマニュアルを見ると…

https://sourceware.org/glibc/manual/2.35/html_node/Signal-Handling.html

シグナルについては色々記述があります。よく分からないから怖い！だけで済まない人(使う必要がある人)は、ご自分が使用するライブラリの当該バージョンのマニュアルを読みましょう。

なお実際使用する際は、POSIX標準に合わせると移植性が上がります。

# Ctrl-Cを受け取る

```c++
#include <iostream>
#include <csignal>

struct sigmask {
    bool result;
    sigset_t oldset;
};

void get_sigmask(sigmask* mask) {
    mask->result = sigprocmask(SIG_BLOCK, nullptr, &mask->oldset) == 0;
}

void print_is_sig_masked(int sig, const sigmask& mask) {
    if (mask.result) std::cout << "signal " << sig << (sig==SIGINT ? "(SIGINT)" : "") << " is " << (sigismember(&mask.oldset, SIGINT) ? "set" : "not set") << std::endl;
    else std::cerr << "sigprocmask was failed" << std::endl;
}

sigmask g_mask;
volatile bool g_interrupted = false;
void callback(int sig) {
    g_interrupted = true;
    get_sigmask(&g_mask);
}

int main() {
    sigmask mask;
    std::cout << "before sigaction: ";
    get_sigmask(&mask);
    print_is_sig_masked(SIGINT, mask);

    struct sigaction sa;
    sa.sa_handler = callback;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    
    if (sigaction(SIGINT, &sa, NULL) != 0) {
        perror("sigaction");
        exit(1);
    }

    std::cout << "after sigaction: ";
    get_sigmask(&mask);
    print_is_sig_masked(SIGINT, mask);

    char buff[0x1000];
    while(! g_interrupted) {
        auto r = read(STDIN_FILENO, buff, sizeof(buff));
        if (r < 0) {
            perror("read");
            std::cerr << "errno: " << errno << ((errno==EINTR) ? "(EINTR)" : "") << std::endl;
        }
    }
    std::cout << "when signal handler was called: ";
    print_is_sig_masked(SIGINT, g_mask);

    return 0;
}
```

ただただ無限に標準入力から読み込みを続けるだけのプログラムに、SIGINTのシグナルハンドラーを付けただけです。実行するとこんな感じ。

```console
before sigaction: signal 2(SIGINT) is not set
after sigaction: signal 2(SIGINT) is not set
^Cread: Interrupted system call
errno: 4(EINTR)
when signal handler was called: signal 2(SIGINT) is set
```

sigactionでシグナルハンドラーを設定する前もした後もSIGINTはマスクされていません。しかしSIGINTシグナルハンドラ呼び出し中はSIGINTがマスクされているのが分かります。シグナルハンドラー呼び出し中の自動マスクは、シグナルハンドラーを呼び出し中に同じシグナルのシグナルハンドラーを再度呼び出させないためです。

シグナルは非同期処理なわけですが、同期？に使える手段はこのマスクしかありません。シグナルハンドラーと通常のスレッドで共有する情報を保護するにはこのマスクを使います。マスクされているシグナルはハンドラー呼び出しを保留されるので、同時アクセスを回避できるわけです。今回はSIGINTなので横着してメインスレッド側はマスクしていません(Ctrl+Cを押して処理されるまでに再度Ctrl+Cは押せないだろうと考える＝横着。シグナルは機械的に送れるので不可能ではない)。

なお、注意すべきはメインスレッド側の処理にはシグナルハンドラー側から能動的に関与していないことです。メインスレッド側の処理は、システムコール呼び出し中のシグナル受信でEINTRが出て失敗するので、能動的に止める必要がありません。ただこういう方式だと、全てのシステムコールはシグナル受信で失敗する可能性があることを意味します。シグナル自体はいつでも送れるので、この方式だと全てのシステムコールやそれを呼び出す関数の処理にEINTRなどの判定が必要になり、その対処はあまりにも大変だということです。

また、システムコールがないときも定期的にシグナルを受信していないかチェックが必要になってしまいます。これは完全な対処が不可能ということです。この例で言えば、ピッタリ4KB読んだり、端末からEnterキーで1行分が確定するなどしてreadから戻ってきて、whileの判定後次のread呼び出しの間にシグナルが割り込んだ場合、再度readが呼び出されてしまうということです。この場合同様にreadが終われば次のwhile判定では終了できるのですが、ユーザーからしてみればCtrl-Cが次の入力まで効かなかった、ということになります。

これらの回避方法は、前提で紹介したリンク先にあると思うので興味がある人は読んでください。

# signalハンドラをC++で書いてみる

さっきまでのコードはシステムコールがメインだったのでほとんどCのコードでした。次は無限ループ中にCtrl-Cで何か表示して終わるC++のコードを書いてみます[^1]。

```c++
#include <vector>
#include <functional>
#include <string>
#include <iostream>
#include <type_traits>
#include <cstring>
#include <signal.h>

using namespace std;

template<typename R, typename Error>
struct result {
    bool success;
    union Value {
        R r;
        Error e;
        Value(){}
        Value(const R& r): r(r) {}
        Value(const Error& e): e(e) {}
        ~Value() {}
    } v;
    result(const R& r): success(true), v(r) {}
    result(const Error& e): success(false), v(e) {}
    result(const result& org): success(org.success) {
        if (success) new (&v.r) R(org.v.r);
        else new (&v.e) Error(org.v.e);
    }
    ~result() {
        if (success) v.r.~R();
        else v.e.~Error();
    }
    operator bool() {return success;}
    R operator *() {return v.r;}
    Error error() {return v.e;}
};

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
        if (owned) {func();}
    }
    bool owned;
    Func func;
};

template <typename Func>
deferred_call<Func> defer(Func&& f) {
    return deferred_call<Func>(std::forward<Func>(f));
}

struct signal_manager {
    static signal_manager instance;
    vector<function<void(int)>> handlers;
    signal_manager(): handlers(SIGRTMAX) {}
    static void callback(int signal) {
        instance.handlers[signal](signal);
    }
    struct signal_snapshot {
        int signal;
        struct sigaction sa;
    };
    template <typename Func>
    result<signal_snapshot, string> regist(int signal, Func func){
        handlers[signal] = func;
        struct sigaction sa;
        signal_snapshot snapshot;
        snapshot.signal = signal;
        sa.sa_handler = callback;
        sigemptyset(&sa.sa_mask);
        sa.sa_flags = 0;
        auto r = sigaction(signal, &sa, &snapshot.sa);
        if (r == 0) return snapshot;
        else return string(strerror(errno)); // cannot use in signal handler
    }
    result<bool, string> unregist(const signal_snapshot& snapshot) {
        auto r = sigaction(snapshot.signal, &snapshot.sa, nullptr);
        if (r == 0) return true;
        else return string(strerror(errno)); // cannot use in signal handler
    }
};
signal_manager signal_manager::instance;

int main() {
    volatile bool stop = false;
    auto r = signal_manager::instance.regist(SIGINT, [&](int signal)->void{
        cout << "signal: sigint comming..." << endl;
        stop = true;
    });
    cout << "signal handler is registered" << endl;
    auto def = defer([&]{
        if (r) {
            signal_manager::instance.unregist(*r);
            cout << "signal handler is unregistered" << endl;
        }
    });
    if (r) {
        while(!stop){}
    } else {
        cerr << r.error() << endl;
    }
    cout << "finished!" << endl;
    return 0;
}
```

resultについては、前々回
https://qiita.com/dameyodamedame/items/e05feb1391a6ce59ad9f

deferについては、前回
https://qiita.com/dameyodamedame/items/a9cc39caefa63c4cf041

説明したので省略します。

今回はsignal_managerを実装して、signal handlerをlambdaで書いてみたということです。lambdaは特に参照キャプチャすると、コードの合間に挟んでカジュアルに書けて、**かゆいところに手が届く**反面、signal handlerのようなグローバルに動くもので使うと危なっかしいのですが、そこをdeferでさらに蓋をしていくコードです。毒を喰らわば皿まで。。。

実際のところ、コンパイラが吐くコードまで見ないと(もちろん見てません)安心できませんね。

# 本物のC++によるsignalの扱い方

有名なBoostライブラリによるsignalの実装は以下のようです。

https://beta.boost.org/doc/libs/1_82_0/doc/html/boost_asio/overview/signals.html

実装例はコチラ

https://stackoverflow.com/a/25577250

ハンドラを定義して登録してrunみたいなコードです。JavaScriptの`setTimeout()`みたいな再登録をすることで再度イベントを受け取るという仕組みになっていて、2度目に再登録せず終了する、という作りのようです。bindの仕組みが若干読みにくいかもしれません。

全体的な仕組みとしてはsignalが非同期I/Oをまとめて扱うパッケージに組み込まれていて、他と同様に使用できるみたいです。boost::asio::signal_setは中身読んでないのでどんな作りなのか分かりませんが、確かに非同期という括りにすべきなんでしょうね。

(追記)
ちょっと読んでみました。作りは前提で紹介したリンク先で触れられているself pipe trickと、epoll(select/pollのpoll改良版)を使用した非同期イベント処理でした。調べたのはubuntu 22.04にパッケージされているBoost 1.74(libboost-devパッケージ)です。

初期化や登録は記事内コードの`boost::asio::signal_set signals(io, SIGINT);`からで

```c++:boost/asio/detail/impl/signal_set_service.ipp
void signal_set_service::add_service(signal_set_service* service)
```
の`open_descriptors()`でパイプを作成。同ファイル

```c++:boost/asio/detail/impl/signal_set_service.ipp
boost::system::error_code signal_set_service::add(
    signal_set_service::implementation_type& impl,
    int signal_number, boost::system::error_code& ec)
```
のsigactionでハンドラ登録。ハンドラは同ファイル boost_asio_signal_handler で、中身は主に作成したパイプに書き込みを実施。run中は以下がループ本体

```c++:boost/asio/detail/impl/scheduler.ipp
std::size_t scheduler::do_run_one(mutex::scoped_lock& lock,
    scheduler::thread_info& this_thread,
    const boost::system::error_code& ec)
```
で、ループ内
```c++
        task_->run(more_handlers ? 0 : -1, this_thread.private_op_queue);
```
でepoll_waitを、
```c++
        o->complete(this, ec, task_result);
```

でpipeからのリードを実施している。イベント待ち本体であるepoll_waitは以下。

```c++:boost/asio/detail/impl/epoll_reactor.ipp
void epoll_reactor::run(long usec, op_queue<operation>& ops)
```

# まとめ

- C++で形ばかりシグナルを受け取ってみた
- シグナルは原始的な仕組みなので取り扱い注意

[^1]: Ctrl-Cをgdbでデバッグしたり、straceで追っかけたりするときは、gdbだとコマンドから`handle SIGINT noprint nostop pass`をしておかないといけません(gdbに奪われる)。vscode上のgdbの場合は`-exec `が先頭に必要です。straceの場合は、Ctrl-Cするとstraceも止まってしまうので、`kill -INT [pid]`で外からSIGINTを送りましょう。

