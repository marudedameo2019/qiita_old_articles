---
title: ext3はfsync(2)が必要なのか？
tags: Linux ext4 ext3
author: dameyodamedame
slide: false
---
# 序

この記事は、以下の記事

https://qiita.com/Kenta_Omura/items/777cdd0f233c64ace8ac

のコメント欄で @angel_p_57 さんが記事主さんに聞かれていた質問

>> ※補足ですが、ext3のファイルシステムではfsyncの使用は不要です。ext3デフォルトではfsync相当の動きになります。
>
> 流石にそれはないはずですが、どこからの情報でしょうか。
>「デフォルトでfsync相当の動き」は、mountオプションsyncもしくは、個別のファイル属性(chattr(1)manページ参照)の'S'で設定されるものです。

に関して、調査した結果です。

# 1. 背景

## 1-1. 記事を書いた理由と想定読者

[ext3](#ext3)やLinuxの前提知識が少ない人でもある程度読めるように記事にしてみました。LinuxでC/C++/pythonを使った開発をし始めた/できる程度の人を想定しています。カーネルコードを読める人にはかなり物足りない記事になります。が、件のコメント欄では誰にも伝わってる感じがしなかったので、改めて書いてみた次第です。私もそんな詳しくないので、詳しい方はご指摘等して頂けたらありがたいです。

# 2. 目的

ext3は[fsync](#fsync)(2)が必要なのかどうかを調査し結論を出すこと

# 3. 調査の方法

1. 10MiBの適当なファイルAを作成し、その中にext3を構築、[ループマウント](#ループマウント)する
1. マウントしたところにファイルを作成し、色々なタイミング(B)でAの**コピーをして**スナップショットを取る
1. 各スナップショットをマウントして中身を確認する
1. 各スナップショットをマウントせずに直接解析するソフト(C)を使用し、中身を確認する

※通常スナップショットはコピーで取るものではないのですが、他に手段もなく、サイズも小さいのでこういう手段にしています

(B)のタイミングは以下の10種類

- write(2)で4KiB書き込みを1回行ってプロセス終了した後
- write(2)で4KiB書き込みを1回行い6秒sleepし(※1)プロセス終了した後
- write(2)で4KiB書き込みを1回行い31秒sleepし(※2)プロセス終了した後
- write(2)で4KiB書き込みを1回行いfsync(2)しプロセス終了した後
- write(2)で4KiB書き込みを1回行いfsync(2)して6秒sleepし(※1)プロセス終了した後
- write(2)で4KiB書き込みを1回行い[syncfs](#sync)(2)しプロセス終了した後
- write(2)で4KiB書き込みを1回行い[sync](#sync)(2)しプロセス終了した後
- 上記の後umountした後(計7回)

(※1)6秒の理由はext3マウント時の[commitオプション](
#ext3%E3%81%AE%E3%83%9E%E3%82%A6%E3%83%B3%E3%83%88%E3%82%AA%E3%83%97%E3%82%B7%E3%83%A7%E3%83%B3commit)がデフォルトで5だからです
(※2)31秒の理由は[カーネルパラメータのvm.dirty_expire_centisecs](#%E3%82%AB%E3%83%BC%E3%83%8D%E3%83%AB%E3%83%91%E3%83%A9%E3%83%A1%E3%83%BC%E3%82%BF-dirty_expire_centisecs)がデフォルトで3000だからです

(C)のソフトは以下の3種類

- [dumpe2fs](#dumpe2fs)(Ubuntuでは標準装備)
- [debugfs](#debugfs)(Ubuntuでは標準装備)

今回はここまでです。

# 4. 準備

Ubuntu 24.04前提で準備するものを記述します。

## 4-1. パッケージインストール

```console
$ sudo apt install build-essential
```

4KiB書き込み用プログラムのビルドにg++が必要なので。あまり入ってない人もいないと思いますが…

## 4-2. 4KiB書き込み用プログラム

これは実行用スクリプトに同梱されています。が、内容だけ説明しておきます。

C++でファイルに4KiB書き込むとなると、普通はC/C++の標準ライブラリを使って、Cならfwrite(3)、C++ならstd::ofstreamなどを使用するのですが、今回確認したいのはシステムコールのfsync(2)なので、標準関数より低レベルなシステムコールopen(2)/close(2)/write(2)を使います。

<details><summary>ソースコード(押すと開きます)</summary>

```c++:tmp/test.cpp
#include <fcntl.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>

#include <cstdio>
#include <string>

const size_t BUFFER_SIZE = 4096;
const size_t FILE_SIZE = 4 * 1024;  // 4KiB

char buffer[BUFFER_SIZE] = {};

int main(int argc, char *argv[]) {
  int fd = open("mnt/dummy", O_CREAT | O_WRONLY | O_TRUNC, 0666);
  if (fd == -1) {
    std::perror(argv[0]);
    return 1;
  }
  for (size_t i = 0; i < static_cast<size_t>(FILE_SIZE / BUFFER_SIZE); ++i) {
    int r = write(fd, buffer, BUFFER_SIZE);
    if (r != BUFFER_SIZE) {
      std::perror(argv[0]);
      return 1;
    }
  }
  if (argc > 1) {
    std::string arg1(argv[1]);
    int r = 0;
    if ((arg1 == "--fsync") || (arg1 == "--fsync6")) {
      r = fsync(fd);
    } else if (arg1 == "--syncfs") {
      r = syncfs(fd);
    }
    if (r == -1) {
      std::perror(argv[0]);
      return 1;
    }
    close(fd);
    if (arg1 == "--sleep") {
      sleep(6);
    } else if (arg1 == "--sleep31") {
      sleep(31);
    } else if (arg1 == "--fsync6") {
      sleep(6);
    } else if (arg1 == "--sync") {
      sync();
    }
  } else {
    close(fd);
  }
  return 0;
}
```

</details>

4KiBの0データをdummyファイルに書き込んで、オプションにより7パターンの動作をして終了するだけのプログラムです。
|引数|動作パターン|
|:--|:--|
|引数なし|何もしない|
|--sleep|sleep(6)|
|--sleep31|sleep(31)|
|--fsync|fsync(fd)|
|--fsync6|fsync(fd)とsleep(6)|
|--syncfs|syncfs(fd)|
|--sync|sync()|

これらの引数により、方法の章で書いたパターンを実施します。

## 4-3. 実行用スクリプト

上のtest.cppを使って、方法の章で書いたパターンを実施するスクリプトです。

<details><summary>ソースコード(押すと開きます)</summary>

test.cppを内包してるのでやや長めです。

```sh:test.sh
set -eux
do_umount() {
    sudo sync
    sudo sync
    sudo sync
    while ! sudo umount $*; do
        sleep 1
    done
}
do_test() {
    cp -p loopdisk.img.org loopdisk.img
    sudo mount ./loopdisk.img -o loop ./mnt
    sudo ./test $2
    ls -lai mnt/ | sed '/ \.\.$/d' >$1.ls.0.log
    cp -p loopdisk.img loopdisk.img.$1.snapshot1
    do_umount mnt
    mv loopdisk.img loopdisk.img.$1.snapshot2
    cp -p loopdisk.img.$1.snapshot1 loopdisk.img
    sudo mount ./loopdisk.img -o loop ./mnt
    cp -p loopdisk.img loopdisk.img.$1.snapshot3
    do_umount mnt
    for ss in 1 2 3;do
        cp -p loopdisk.img.$1.snapshot$ss loopdisk.img
        sudo mount ./loopdisk.img -o loop ./mnt
        ls -lai mnt/ | sed '/ \.\.$/d' >$1.ls.$ss.log
        do_umount mnt
    done
}

if [ -e tmp ]; then
    echo "tmp is already exist."
    return 1
fi

sudo true

mkdir tmp
cd tmp
mkdir mnt
dd if=/dev/zero of=loopdisk.img.org bs=1MiB count=10
mkfs.ext3 ./loopdisk.img.org

cat >test.cpp <<EOF
#include <fcntl.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>

#include <cstdio>
#include <string>

const size_t BUFFER_SIZE = 4096;
const size_t FILE_SIZE = 4 * 1024;  // 4KiB

char buffer[BUFFER_SIZE] = {};

int main(int argc, char *argv[]) {
  int fd = open("mnt/dummy", O_CREAT | O_WRONLY | O_TRUNC, 0666);
  if (fd == -1) {
    std::perror(argv[0]);
    return 1;
  }
  for (size_t i = 0; i < static_cast<size_t>(FILE_SIZE / BUFFER_SIZE); ++i) {
    int r = write(fd, buffer, BUFFER_SIZE);
    if (r != BUFFER_SIZE) {
      std::perror(argv[0]);
      return 1;
    }
  }
  if (argc > 1) {
    std::string arg1(argv[1]);
    int r = 0;
    if ((arg1 == "--fsync") || (arg1 == "--fsync6")) {
      r = fsync(fd);
    } else if (arg1 == "--syncfs") {
      r = syncfs(fd);
    }
    if (r == -1) {
      std::perror(argv[0]);
      return 1;
    }
    close(fd);
    if (arg1 == "--sleep") {
      sleep(6);
    } else if (arg1 == "--sleep31") {
      sleep(31);
    } else if (arg1 == "--fsync6") {
      sleep(6);
    } else if (arg1 == "--sync") {
      sync();
    }
  } else {
    close(fd);
  }
  return 0;
}
EOF
g++ -g -Wall -pedantic -std=c++17 test.cpp -o test

do_test case1 "" 2>&1 >case1.log
do_test case2 "--sleep" 2>&1 >case2.log
do_test case2a "--sleep31" 2>&1 >case2a.log
do_test case3 "--fsync" 2>&1 >case3.log
do_test case3a "--fsync6" 2>&1 >case3a.log
do_test case4 "--syncfs" 2>&1 >case4.log
do_test case5 "--sync" 2>&1 >case5.log

for f in loopdisk.img*; do
    dumpe2fs $f 2>&1 > $f.dumpe2fs.log
    sed '/time/d;/^Last mounted on:/d;/^Mount count:/d' $f.dumpe2fs.log > $f.dumpe2fs.notime.log
    debugfs -R 'logdump -a' $f 2>&1 > $f.logdump_not_committed.log;
    debugfs -R 'logdump -Oa' $f 2>&1 > $f.logdump_all.log;
    debugfs -R 'ls -l /' $f 2>&1 > $f.ls.log
done
```
</details>

それほど説明が必要な部分はないと思いますが、少しだけ

- dumpe2fsのログを出力した後、sedで結果をフィルタしているのは時間やマウント回数などdiffを取る際に邪魔な部分を消したいからです
- debugfsの最初の行はメタデータのディスク反映が終わっていないジャーナルの情報を抽出しています
- debugfsの2番目の行はディスク反映済含めて残っている全ジャーナルの情報を抽出しています
- debugfsの3番目の行はマウントしたディスクの/をlsした結果を抽出しています
- mountにはmount -o loopを使っていますが、こうした場合losetup --direct-io=offと同等になり、マウントした[ファイルシステム](#ファイルシステム)への読み書きに1枚、ループ対象のファイルへの読み書きにもう1枚、計2枚の[ページキャッシュ](#ページキャッシュ)が使用されます。実際スナップショットを取る際のコピーは2枚目のページキャッシュからになるでしょう。

実行すると分かりますが、このスクリプトは結果がバラついてあまり再現性がありません。そのため測定が1度だけでは意味がなく、繰り返し実行して頻度を調べる必要があります。今回は各ケース200回実行させたこともあり、目視確認を嫌って出力自動解析までコードに入れたのでちょっと長くなってしまいました。これらは記事の末尾に全コードまとめて置いてあります。以下はそのトップレベルのスクリプトです。

<details><summary>ソースコード(押すと開きます)</summary>

```sh:all.sh
set -eux
if [ $# -gt 0 ]; then
    count=$1
else
    count=200
fi
logdir=logs
mkdir -p $logdir
for i in $(seq $count); do
    s=$(date +%+4Y%m%d%H%M%S)
    sh test.sh
    mkdir -p "$logdir/$s"
    mv tmp/*.log "$logdir/$s/"
    rm -rf tmp
    sudo sync
done
lsdir=ls
dumpe2fsdir=dumpe2fs
debugfsdir=debugfs
sh ls_diff.sh $logdir $lsdir
sh dumpe2fs_diff.sh $logdir $dumpe2fsdir
sh debugfs_diff.sh $logdir $debugfsdir
sh report.sh $logdir $lsdir $dumpe2fsdir $debugfsdir

python=python3
cat $logdir/ls_report.txt | $python qiita_ls.py
cat $logdir/dumpe2fs_report.txt | $python qiita_dumpe2fs.py
cat $logdir/debugfs_ls_report.txt | $python qiita_debugfs_ls.py
cat $logdir/debugfs_log_report.txt | $python qiita_debugfs_log.py
```

</details>

# 5. 調査の実施と結果

## 5-1. スクリプトを実行

`sh all.sh [試行回数]`で実行します。

## 5-2. lsの結果を確認

以下のタイミングで取得したスナップショットイメージをマウントし、test.cppを実行した直後のlsの結果と比較しています。
- umount直前に取得したsnapshot1
- umount直後に取得したsnapshot2
- snapshot1を再マウントして取得したsnapshot3

私の環境で集計した結果は以下のとおりです。

### 5-2-1. /dummyの差分

|スナップショット|引数なし|sleep(6)|sleep(31)|fsync|fsync+sleep(6)|syncfs|sync|
|:--|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
|snapshot1|差分あり|差分なし(73.0%)<br>差分あり(27.0%)|差分なし|差分なし|差分なし|差分なし|差分なし|
|snapshot2|差分なし|差分なし|差分なし|差分なし|差分なし|差分なし|差分なし|
|snapshot3|差分あり|差分なし(73.0%)<br>差分あり(27.0%)|差分なし|差分なし|差分なし|差分なし|差分なし|

### 5-2-2. /の差分

|スナップショット|引数なし|sleep(6)|sleep(31)|fsync|fsync+sleep(6)|syncfs|sync|
|:--|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
|snapshot1|差分なし(98.5%)<br>差分あり(1.5%)|差分なし(97.0%)<br>差分あり(3.0%)|差分なし|差分なし|差分なし|差分なし|差分なし|
|snapshot2|差分なし|差分なし|差分なし|差分なし|差分なし|差分なし|差分なし|
|snapshot3|差分なし(98.5%)<br>差分あり(1.5%)|差分なし(97.0%)<br>差分あり(3.0%)|差分なし|差分なし|差分なし|差分なし|差分なし|

### 5-2-3. ポイント

ここで注目すべきは、fsyncなどがなく、sleepもしていないケースでのsnapshotでは/dummyに差分がある、つまりファイル/dummyが存在していないということです。そして、6秒待っているケースでは73%の確率でファイル/dummyが存在していて、31秒待っているケースでは100%の確率でファイル/dummyが存在、fsync/syncfd/syncを呼んでいるケースでも100%存在しているという点です。つまり、ext3では**書き込み(open→write→close)後、マウントオプションcommitのデフォルト値5秒以上待っていても、ファイルに書き込まれている保証がない**ということです。そして**vm.dirty_expire_centisecsのデフォルト値30秒以上待っていると書き込まれています**。意味するところは明確ではありませんが、理由はいくつか推測も出来ます。しかし実験の結果からは私の環境ではRHEL7で言及されているような楽観的な仕様

https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/storage_administration_guide/ch-ext4#ch-ext4
> By default, ext3 automatically forces newly created files to disk almost immediately even without fsync()

ではないように見えます。

/側にあまり差分がない理由はよく分かりません。

## 5-3. dumpe2fsのログを比較

dumpe2fsから取得できるいくつかの属性に違いが検出されます。比較の対象は初期状態(mkfs直後)と各スナップショットになります。あとついでに初期状態のブロック情報が出ているのでそれもリストアップしておきます。

### 5-3-1. 初期状態のブロック種別

tmp/loopdisk.img.org.dumpe2fs.log から得られる情報になります。

ブロックグループは0のみ

|ブロック番号|タイプ|
|--:|:--:|
|0|Primary superblock|
|1|Group descriptors|
|2|Block bitmap|
|3|Inode bitmap|
|4|Inode table|

### 5-3-2. umount直前(snapshot1)

|属性|引数なし|sleep(6)|sleep(31)|fsync|fsync+sleep(6)|syncfs|sync|
|:--|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
|Filesystem features(needs_recovery)|+|+|+|+|+|+|+|
|Journal sequence|-|0x00000002|0x00000002|0x00000002|0x00000002|0x00000002|0x00000002|
|増加ブロック|-|-|1195(61.0%)<br>-(39.0%)|-|1195(59.0%)<br>-(41.0%)|1195|1195|
|増加inode|-|-|12(61.0%)<br>-(39.0%)|-|12(59.0%)<br>-(41.0%)|12|12|

### 5-3-3. umount直後(snapshot2)

|属性|引数なし|sleep(6)|sleep(31)|fsync|fsync+sleep(6)|syncfs|sync|
|:--|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
|Filesystem features(needs_recovery)|-|-|-|-|-|-|-|
|Journal sequence|0x00000004|0x00000004(23.0%)<br>0x00000005(77.0%)|0x00000004(0.5%)<br>0x00000005(99.5%)|0x00000005|0x00000005|0x00000005|0x00000005|
|増加ブロック|1195(99.5%)<br>1536(0.5%)|1195|1195|1195|1195|1195|1195|
|増加inode|12|12|12|12|12|12|12|

### 5-3-4. ポイント

umount直前で注目すべき点は、通常/sleep6秒/sleep31秒/fsync/fsync+sleep6秒で**inodeが増加していないケースがあること**。**inodeが増加していないということはdummyファイルが存在しない**ことを意味しています。

しかしlsの結果(/dummyのsnapshot1)からは、**再mount時**これらのケースのうちsleep31秒/fsync/fsync+sleep6秒では100%、sleep6秒でも73%で**inodeが出来ており、/dummyファイルが復元**されていました。これらは、再マウント時に**リカバリが機能していることを示唆**しています。

umount直前ではfsyncのinode増加は100%になっておらず、syncfs/syncだと100%になっているのは、sync/syncfsがファイルシステム単位のsyncであり、メタデータのsyncも同時に行っているからではないかと推測しています。

umount直後ではJournal sequenceにも4になるものと5になるものが出ていまずが、ここでは気にしないことにします。また増加ブロックにもわずかな差異が発生していますが、これも無視します。増加ブロックの1195,1536は後で調べていて、1195が/dummyの実データ保管ブロックであることが分かっています。1536はレアケースなので調べていませんが、恐らく同じです。

## 5-4. debugfsのログを比較

debugfsを使うことにより、mount前のsnapshotを解析して状態を取得することが出来ます。ここではdumpe2fsのときと同様に初期状態のイメージと各snapshotを比較した結果を見ていきます。

### 5-4-1. lsの差分をチェック

debugfsがlsのようなことをエミュレーションしてくれる機能の出力をチェックした結果です。

#### 5-4-1-1. umount直前(snapshot1)

|ケース|/dummy|/|
|:--|:--:|:--:|
|引数なし|差分なし|差分なし|
|sleep(6)|差分なし|差分なし|
|sleep(31)|差分なし(39.0%)<br>差分あり(61.0%)|差分なし(78.5%)<br>差分あり(21.5%)|
|fsync|差分なし|差分なし|
|fsync+sleep(6)|差分なし(41.0%)<br>差分あり(59.0%)|差分なし(41.0%)<br>差分あり(59.0%)|
|syncfs|差分あり|差分あり|
|sync|差分あり|差分あり|

#### 5-4-1-2. umount直後(snapshot2)

|ケース|/dummy|/|
|:--|:--:|:--:|
|引数なし|差分あり|差分なし(98.5%)<br>差分あり(1.5%)|
|sleep(6)|差分あり|差分なし(86.0%)<br>差分あり(14.0%)|
|sleep(31)|差分あり|差分なし(64.0%)<br>差分あり(36.0%)|
|fsync|差分あり|差分なし(2.5%)<br>差分あり(97.5%)|
|fsync+sleep(6)|差分あり|差分あり|
|syncfs|差分あり|差分あり|
|sync|差分あり|差分あり|

#### 5-4-1-3. snapshot1の再mount直後(snapshot3)

|ケース|/dummy|/|
|:--|:--:|:--:|
|引数なし|差分なし|差分なし|
|sleep(6)|差分なし(27.0%)<br>差分あり(73.0%)|差分なし(89.0%)<br>差分あり(11.0%)|
|sleep(31)|差分あり|差分なし(64.0%)<br>差分あり(36.0%)|
|fsync|差分あり|差分なし(2.5%)<br>差分あり(97.5%)|
|fsync+sleep(6)|差分あり|差分あり|
|syncfs|差分あり|差分あり|
|sync|差分あり|差分あり|

#### 5-4-1-4. ポイント

umount直前のスナップショットを直接解析することにより、fsyncを入れるケースで/dummyファイルが作成されていないことがより明確に示されています。対照的にsyncfs/syncは再マウント前に全てが書き込まれていることが分かります。ただし、fsyncを使用した場合でも、再マウントしたときには普通にumountした直後のスナップショットと同じデータに復元しているので、ジャーナルへの書き込みは終わっていることを示しており、sleep6秒/31秒したケースでも73%以上が同じデータに復元出来ている点は興味深いところです。

### 5-4-2. transaction logの差分をチェック

debugfsがtransaction log、つまりjournalを解析した差分をチェックしたものです。{}がトランザクションを示していて、Cがついているものは直前のトランザクションがコミットされています。{}内の数字はブロックを示しています。Cがついているところまでが復元される対象になります。

#### 5-4-2-1. umount直前(snapshot1)

|ケース|all|not_committed|
|:--|:--:|:--:|
|引数なし|-|-|
|sleep(6)|{3,1,4,164,0,2},C(73.0%)<br>{3,1,4,164,0,2}(27.0%)|{3,1,4,164,0,2},C(73.0%)<br>{3,1,4,164,0,2}(27.0%)|
|sleep(31)|{4,3,1,164,0,2},C(0.5%)<br>{3,1,4,164,0,2},C(99.5%)|{4,3,1,164,0,2},C(0.5%)<br>{3,1,4,164,0,2},C(99.5%)|
|fsync|{3,1,4,164,0,2},C|{3,1,4,164,0,2},C|
|fsync+sleep(6)|{3,1,4,164,0,2},C|{3,1,4,164,0,2},C|
|syncfs|{3,1,4,164,0,2},C|{3,1,4,164,0,2},C|
|sync|{3,1,4,164,0,2},C|{3,1,4,164,0,2},C|

#### 5-4-2-2. umount直後(snapshot2)

|ケース|all|not_committed|
|:--|:--:|:--:|
|引数なし|{3,1,4,164,0,2},C|-|
|sleep(6)|{3,1,4,164,0,2},C(23.0%)<br>{3,1,4,164,0,2},C,{4},C(77.0%)|-|
|sleep(31)|{4,3,1,164,0,2},C(0.5%)<br>{3,1,4,164,0,2},C,{4},C(99.5%)|-|
|fsync|{3,1,4,164,0,2},C,{4},C|-|
|fsync+sleep(6)|{3,1,4,164,0,2},C,{4},C|-|
|syncfs|{3,1,4,164,0,2},C,{4},C|-|
|sync|{3,1,4,164,0,2},C,{4},C|-|

#### 5-4-2-3. snapshot1の再マウント直後(snapshot3)

|ケース|all|not_committed|
|:--|:--:|:--:|
|sleep(6)|{3,1,4,164,0,2},C(73.0%)<br>{3,1,4,164,0,2}(27.0%)|-|
|sleep(31)|{4,3,1,164,0,2},C(0.5%)<br>{3,1,4,164,0,2},C(99.5%)|-|
|fsync|{3,1,4,164,0,2},C|-|
|fsync+sleep(6)|{3,1,4,164,0,2},C|-|
|syncfs|{3,1,4,164,0,2},C|-|
|sync|{3,1,4,164,0,2},C|-|

#### 5-4-2-4. ポイント

トランザクション内の対象ブロックは最初にdumpe2fsの結果を見たときに0〜4までは一覧だけしましたが、今回は一応意味も少しだけ記述しておきます。

- 0はスーパーブロックで最初からあるやつです
- 1はブロックグループ0のデスクリプタが保管されています

今回は小さいファイルシステムなので、ブロックグループやスーパーブロックは複数出てきません。なので、**0,1は無視**します。

- 2,3はグループ0内の、ブロックとinodeの確保/未確保ビットマップです

これらはブロックやinodeを新たに確保したり削除したりしない限り変更されません。今回の操作では/dummyを作成するときにブロックやinodeが必要なので変更が入ります。

- 4はinodeテーブルです

各ファイル(ディレクトリなども含む)自体の基本属性(inode)を格納している固定長レコードの配列です。各ファイルのinode番号やブロック番号は以下の方法で確認できます。

```console
$ debugfs -R 'blocks /' tmp/loopdisk.img.case3.snapshot3
debugfs 1.47.0 (5-Feb-2023)
164 
$ debugfs -R 'blocks /dummy' tmp/loopdisk.img.case3.snapshot3
debugfs 1.47.0 (5-Feb-2023)
1195 
$ debugfs -R 'icheck 164' tmp/loopdisk.img.case3.snapshot3
debugfs 1.47.0 (5-Feb-2023)
Block	Inode number
164	2
$ debugfs -R 'icheck 1195' tmp/loopdisk.img.case3.snapshot3
debugfs 1.47.0 (5-Feb-2023)
Block	Inode number
1195	12
```

つまり各トランザクションで記述されているブロックは0〜4の始めからあるブロックと、/のディレクトリエントリを保管しているブロック164だったことが分かります。/dummyの実データが保管されているブロック1195には触れられていません。以上から現設定のext3のジャーナルにはメタデータしか含まれていないことが確認できました。

そして記録されているパターンとしては、3,1,4で始まるトランザクションと4,3,1で始まるトランザクションがあることが分かっています。1は無視するので、違いは3,4と4,3の順番だけになります。3はinodeの確保用bitmapだし、4はinodeテーブルなので、両方書き込まれないと意味がなく、トランザクションでもあり、この順番は重要でないと考えます。

他のパターンとしては、2つ目のトランザクションとして4だけ追加で書き込まれるパターンがありましたが、これは恐らく、/の最終更新時間など属性の更新が必要になったケースではないかと推測しています。しかし裏は取れていません。

これらのトランザクションはブロック1195が書き込まれた後である必要があり、fsyncではトランザクションの書き込みまで行うが、メタデータの書き込みまでは待たず、syncfsやsyncではメタデータの書き込みまで待つことになる、と考えると全ての辻褄が合うのではないかと考えています。

## 5-5. 結果まとめ

ここまでのポイントから

- open→write→closeだけの実施では、データは即時ファイルシステムに書き込まれているわけではない
- ext3ではデータの書き込み後5秒(デフォルト)でcommitするように設計されているが、close後6秒待っても書き込まれていないケースがあった
- ext3ではデータの書き込み後fsyncすると即時commitされるように見え、commitされる場合は再マウント時リカバリでメタデータが書き込まれる
- ページキャッシュの書き込みタイミングは30秒(デフォルト)以内になるが、close後31秒待てばfsyncせずとも書き込まれる

ことが分かっています。以上から**ext3は書き込み後5秒待っても100%fsync不要とは言えない**という結果になりました。

# 6. 考察

- 5秒の件が正確に何を指しているのか
- 本体データ、ジャーナル、メタデータの書き込みタイミング

が、最後まで分かりませんでした。これらはカーネルコードに仕込みを入れて実際にログを追ってみないと分かりません。ただその前に5秒や30秒だけでなく、それらの情報の書き込み時間の正確な分布が現象ベースで欲しいと思います。

また、今回は10MiBのファイルシステムに4KiBのファイルを作ってるだけですが、作成するファイルの大きさに対してどういう分布の変化をするのかも知りたいところです。

# 7. まとめ

今回の調査結果からは、

- ext3は書き込み後5秒待っても100%fsync不要とは言えない

という結論になった。

# 付録

## 用語集

### ext3

Linuxで使われていたファイルシステムの1つです。この系譜で現在使われているのは[ext4](https://ja.wikipedia.org/wiki/Ext4)で、ext4は2025年1月現在恐らくLinuxで一番標準的なファイルシステムです。[ext2](#ext2)というジャーナルのないシンプルなファイルシステムにジャーナルを追加して耐久性を上げたのが[ext3](https://ja.wikipedia.org/wiki/Ext3)というのが私の理解です。

### ファイルシステム

Linuxでは/で始まる階層構造に割り振られたファイルを管理するためのシステムです。何をファイルとして扱うかはOSに依りますが、多くはデータファイルであることが多く、フラットなデータ領域をいかにして階層を持ったファイル(データ)集合として扱うか、つまりフォーマットをどうするかの仕様とも言えます。連続データであればメモリ上でも1ファイルの中にも[ファイルシステム](https://ja.wikipedia.org/wiki/%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E3%82%B7%E3%82%B9%E3%83%86%E3%83%A0)は構築可能です。一般的にOSが直に使うものなので、ドライバで実装されるのが普通です。LinuxではVFS APIを実装したドライバを書けば、新しいファイルシステムを実装できます。

### ジャーナル

ファイルシステムの文脈で出てくるジャーナルとは、[トランザクションログ](https://ja.wikipedia.org/wiki/%E3%83%88%E3%83%A9%E3%83%B3%E3%82%B6%E3%82%AF%E3%82%B7%E3%83%A7%E3%83%B3%E3%83%AD%E3%82%B0)のことです。データベースに詳しい人はこの言葉だけでもうピンと来てると思います。

ファイルを書いたり編集したりする操作をジャーナルに記録していき、コミットした時点でその操作を確定するために使用します。コミット前までの操作は確定していないので、内容が保証されません。例えばファイルを書き込み中にUPS未使用のLinuxで急にコンセントを抜いたとき、再起動後にコミット後の書き込みは復旧できないという意味です(雷による停電だとそもそもディスクが壊れてたりして保証できませんが…)。

本記事で扱うのはext3ですが、今は実装がext4ドライバに含まれており、資料もext4のものになりますが、私が知る限り一番詳しい資料は以下になります。

https://www.kernel.org/doc/html/v5.7/filesystems/ext4/globals.html#journal-jbd2

### ext2

先に書いたとおりシンプルなファイルシステムです。ext2はまず全体をブロックという単位に分け、それをいくつかの種類に分類して管理しています。詳細は面倒なので英語版の[ext2 wikipedia](https://en.wikipedia.org/wiki/Ext2)に任せます。一番詳しい資料は以下になります。

https://www.nongnu.org/ext2-doc/ext2.html

### ページキャッシュ

Linuxカーネル2.4未満にあったバッファキャッシュが現在どうなってるのか(メタデータをどう扱っているのか)よく知らないですが、バッファキャッシュはページキャッシュと同じであるとして書いています。この不明点は今現在も私が結論を明確に出せない理由の一つになっています。

ファイルからデータを読み取ったり、ファイルにデータを書き込んだりするとき、Linuxカーネルでは一旦カーネル内のメモリ領域に書き込んでからユーザーのプログラムに渡したり、ディスクに書き込んだりしています。このメモリ領域をページキャッシュと言います。ディスクへの書き込みはpdflush(Dirty Page FLUSH)というカーネルスレッドが担当していて、カーネルパラメータにより若干動作を変えます。pdflushによるディスクへの書き込みは小さなファイルなら一般的にユーザープログラムがclose(2)した後行われる動作になります。書き込んだファイルをユーザープログラムが読むとページキャッシュから読まれるのでディスクまで書き込まれてるように見えるのですが、実際にはユーザープログラムが終了した後でもpdflushによるディスクへの書き込みが行われていなければディスクには書き込まれていません。

正直ユーザープログラムから見た内容まで解説してるカーネル側の資料は見当たらず、カーネル側から見た、ちょっと古いけどそこそこ読みやすい資料は以下になります。

https://lwn.net/Articles/326552/

### fsync

ファイル単位でページキャッシュのフラッシュをしてくれるシステムコールです。詳細は`man 2 fsync`で見てください。

### sync

sync(2)/syncfs(2)はファイルシステム単位でページキャッシュのフラッシュをしてくれるシステムコールです。sync(2)は対象を選べず全てになります。詳細は`man 1 sync`、`man 2 syncfs`、`man 2 sync`で見てください。

### ループマウント

通常mountコマンドはブロックデバイスを指定してそれを指定のマウントポイントにマウントするのですが、ブロックデバイスの代わりに普通のファイルを指定してmountすると、ループマウントと呼ばれます。このマウントにはループデバイスが必要となります。Linuxではmountオプションにloop指定を入れるだけでループマウントでき、ループデバイスが自動で割り当てられます。ループマウントをすると、そのファイルシステムの内容は全て指定された普通のファイルに記録されます。

詳細(?)は以下を見てください。

https://en.wikipedia.org/wiki/Loop_device

### ext3のマウントオプションcommit

ジャーナルのコミット間隔を秒数で指定します。デフォルトは5秒です。詳細は`man 5 ext3`で見てください。

### カーネルパラメータ dirty_expire_centisecs

sysctlコマンドで設定したり出来るカーネルパラメータのvm.dirty_expire_centisecsです。ページキャッシュに書き込んでからの経過時間がこれを過ぎるとディスクへ書き込むべき状態になります(条件の1つで、他の条件でも同じ状態になります)。デフォルトは30秒です。

詳細は各カーネルバージョンのドキュメントを見てください。私が使っている6.8.0-51のドキュメントは以下です。

https://github.com/torvalds/linux/blob/v6.8/Documentation/admin-guide/sysctl/vm.rst#dirty_expire_centisecs

### dumpe2fs

ext2～4ファイルシステム用のdumpツールです。といってもrestoreできるようなものではありません。ただの診断ツールで大雑把な情報を教えてくれます。今回は各スナップショットから情報を取り出して、主にブロック情報とiノードを比較するために使っています。

詳細は`man dumpe2fs`で見てください。

### debugfs

dumpe2fsと似たようなext2～4ファイルシステム用のdebugツールです。dumpe2fsよりは細かい情報を出力でき、書き込んだりも出来るようになっているinteractiveな使い方のツールです。今回は各スナップショットから情報を取り出して、主にジャーナルの情報とディレクトリ走査情報を比較するために使っています。

詳細は`man debugfs`で見てください。

## 全スクリプト

Ubuntu 22.04/24.04用環境構築スクリプトです(測定はしません)。tarアーカイブの代わりです。

|ファイル|説明|
|:--|:--|
|all.sh|測定実行スクリプト。引数で測定回数指定(デフォルト200回)|
|test.sh|実際の測定用スクリプト1回分|
|classify.sh|ケース分類用ライブラリ|
|ls_diff.sh|test.shの出力のうち、lsの結果から差分抽出するスクリプト|
|dumpe2fs_diff.sh|test.shの出力のうち、dumpe2fsの結果から差分抽出するスクリプト|
|debugfs_diff.sh|test.shの出力のうち、debugfsの結果から差分抽出するスクリプト|
|report.sh|各抽出された差分を目視確認しやすい結果に整形するスクリプト|
|qiita_base_ls.py|複数のスクリプトから抽出された共通部分|
|qiita_ls.py|report.shが出力した結果のうちls分を解析しQiitaの表形式にするスクリプト|
|qiita_dumpe2fs.py|report.shが出力した結果のうちdumpe2fs分を解析しQiitaの表形式にするスクリプト|
|qiita_debugfs_ls.py|report.shが出力した結果のうちdebugfsのls分を解析しQiitaの表形式にするスクリプト|
|qiita_debugfs_log.py|report.shが出力した結果のうちdebugfsのログ分を解析しQiitaの表形式にするスクリプト|

<details><summary>ソースコード(押すと開きます)</summary>

```sh:create_env.sh
cat >all.sh <<CREATE_ENV_EOF
set -eux
if [ \$# -gt 0 ]; then
    count=\$1
else
    count=200
fi
logdir=logs
mkdir -p \$logdir
for i in \$(seq \$count); do
    s=\$(date +%+4Y%m%d%H%M%S)
    sh test.sh
    mkdir -p "\$logdir/\$s"
    mv tmp/*.log "\$logdir/\$s/"
    rm -rf tmp
    sudo sync
done
lsdir=ls
dumpe2fsdir=dumpe2fs
debugfsdir=debugfs
sh ls_diff.sh \$logdir \$lsdir
sh dumpe2fs_diff.sh \$logdir \$dumpe2fsdir
sh debugfs_diff.sh \$logdir \$debugfsdir
sh report.sh \$logdir \$lsdir \$dumpe2fsdir \$debugfsdir

python=python3
cat \$logdir/ls_report.txt | \$python qiita_ls.py
cat \$logdir/dumpe2fs_report.txt | \$python qiita_dumpe2fs.py
cat \$logdir/debugfs_ls_report.txt | \$python qiita_debugfs_ls.py
cat \$logdir/debugfs_log_report.txt | \$python qiita_debugfs_log.py
CREATE_ENV_EOF
cat >classify.sh <<CREATE_ENV_EOF
classify() {
    classify_files=\$*
    classify_caseno=0
    for classify_f in \$classify_files;do
        if [ -e \$classify_f ];then
            classify_caseno=\$(expr \$classify_caseno + 1)
            mkdir "case\$classify_caseno"
            mv \$classify_f "case\$classify_caseno/"
            classify_h="case\$classify_caseno/\$(basename \$classify_f)"
            if [ ! -e \$classify_h ]; then
                exit 1
            fi
            for classify_g in \$classify_files;do
                if [ -e \$classify_g ];then
                    if cmp \$classify_h \$classify_g; then
                        mv \$classify_g "case\$classify_caseno/"
                    fi
                fi
            done
        fi
    done
}
CREATE_ENV_EOF
cat >debugfs_diff.sh <<CREATE_ENV_EOF
. ./classify.sh

logdir=\$1
tooldir=\$2

for d in \$logdir/20*;do
    echo \$d
    cd \$d
    d=\$(basename \$d)
    for ss in 1 2 3;do
        for c in 1 2 2a 3 3a 4 5;do
            for logtype in logdump_all logdump_not_committed ls;do
                diff loopdisk.img.org.\$logtype.log loopdisk.img.case\$c.snapshot\$ss.\$logtype.log | sed 's/[0-9]*-[A-Za-z]*-\\([0-9]*\\) [0-9]*:[0-9]*/\\1/' >../debugfs.\$d.case\$c.snapshot\$ss.\$logtype.diff
            done
        done
    done
    cd ../..
done

mkdir -p \$logdir/\$tooldir/log
cd \$logdir/\$tooldir/log
for ss in 1 2 3;do
    mkdir snapshot\$ss
    cd snapshot\$ss
    for c in 1 2 2a 3 3a 4 5;do
        mkdir case\$c
        cd case\$c
        for logtype in logdump_all logdump_not_committed;do
            mkdir \$logtype
            cd \$logtype
            classify ../../../../../debugfs.*.case\$c.snapshot\$ss.\$logtype.diff
            cd ..
        done
        cd ..
    done
    cd ..
done
cd ../../..
mkdir -p \$logdir/\$tooldir/ls
cd \$logdir/\$tooldir/ls
for ss in 1 2 3;do
    mkdir snapshot\$ss
    cd snapshot\$ss
    for c in 1 2 2a 3 3a 4 5;do
        mkdir case\$c
        cd case\$c
        classify ../../../../debugfs.*.case\$c.snapshot\$ss.ls.diff
        cd ..
    done
    cd ..
done
cd ../../..
CREATE_ENV_EOF
cat >dumpe2fs_diff.sh <<CREATE_ENV_EOF
. ./classify.sh

logdir=\$1
tooldir=\$2

for d in \$logdir/20*;do
    echo \$d
    cd \$d
    d=\$(basename \$d)
    for c in 1 2 2a 3 3a 4 5;do
        for ss in 1 2;do
            diff loopdisk.img.org.dumpe2fs.notime.log loopdisk.img.case\$c.snapshot\$ss.dumpe2fs.notime.log >../dumpe2fs.\$d.case\$c.snapshot\$ss.diff
        done
    done
    cd ../..
done

for ss in 1 2;do
    mkdir -p \$logdir/snapshot\$ss
    cd \$logdir/snapshot\$ss
    for c in 1 2 2a 3 3a 4 5;do
        mkdir case\$c
        cd case\$c
        classify ../../dumpe2fs.*.case\$c.snapshot\$ss.diff
        cd ..
    done
    cd ../..
done

mkdir -p \$logdir/\$tooldir
mv \$logdir/snapshot* \$logdir/\$tooldir
CREATE_ENV_EOF
cat >ls_diff.sh <<CREATE_ENV_EOF
. ./classify.sh

logdir=\$1
tooldir=\$2

for d in \$logdir/20*;do
    echo \$d
    cd \$d
    d=\$(basename \$d)
    for c in 1 2 2a 3 3a 4 5;do
        for ss in 1 2 3;do
            diff case\$c.ls.0.log case\$c.ls.\$ss.log | sed 's/[0-9]*月 [0-9]* [0-9]*:[0-9]*//' >../ls.\$d.case\$c.snapshot\$ss.diff
        done
    done
    cd ../..
done

for ss in 1 2 3;do
    mkdir -p \$logdir/snapshot\$ss
    cd \$logdir/snapshot\$ss
    for c in 1 2 2a 3 3a 4 5;do
        mkdir case\$c
        cd case\$c
        classify ../../ls.*.case\$c.snapshot\$ss.diff
        cd ..
    done
    cd ../..
done

mkdir -p \$logdir/\$tooldir
mv \$logdir/snapshot* \$logdir/\$tooldir
CREATE_ENV_EOF
cat >report.sh <<CREATE_ENV_EOF
logdir=\$1
lsdir=\$2
dumpe2fsdir=\$3
debugfsdir=\$4

total=\$(ls -1 \$logdir/\$lsdir/snapshot1/case1/*/*|wc -l)
for i in \$logdir/\$lsdir/snapshot*/case*/*;do
    echo "============================================================================"
    echo "[\$i]"
    count=\$(ls -1 \$i/*|wc -l)
    percent=\$(echo "scale=2;\$count*100/\$total"|bc)
    echo "Count: \$count/\$total(\$percent%)"
    cat \$(ls -1 \$i/*|head -1)
done | tee \$logdir/ls_report.txt

total=\$(ls -1 \$logdir/\$dumpe2fsdir/snapshot1/case1/*/*|wc -l)
for i in \$logdir/\$dumpe2fsdir/snapshot*/case*/*;do
    echo "============================================================================"
    echo "[\$i]"
    count=\$(ls -1 \$i/*|wc -l)
    percent=\$(echo "scale=2;\$count*100/\$total"|bc)
    echo "Count: \$count/\$total(\$percent%)"
    cat \$(ls -1 \$i/*|head -1)
done | tee \$logdir/dumpe2fs_report.txt

total=\$(ls -1 \$logdir/\$debugfsdir/ls/snapshot1/case1/*/*|wc -l)
for i in \$logdir/\$debugfsdir/ls/snapshot*/case*/*;do
    echo "============================================================================"
    echo "[\$i]"
    count=\$(ls -1 \$i/*|wc -l)
    percent=\$(echo "scale=2;\$count*100/\$total"|bc)
    echo "Count: \$count/\$total(\$percent%)"
    cat \$(ls -1 \$i/*|head -1)
done | tee \$logdir/debugfs_ls_report.txt

total=\$(ls -1 \$logdir/\$debugfsdir/log/snapshot1/case1/logdump_all/*/*|wc -l)
for i in \$logdir/\$debugfsdir/log/snapshot*/case*/*/*;do
    echo "============================================================================"
    echo "[\$i]"
    count=\$(ls -1 \$i/*|wc -l)
    percent=\$(echo "scale=2;\$count*100/\$total"|bc)
    echo "Count: \$count/\$total(\$percent%)"
    cat \$(ls -1 \$i/*|head -1)
done | tee \$logdir/debugfs_log_report.txt
CREATE_ENV_EOF
cat >test.sh <<CREATE_ENV_EOF
set -eux
do_umount() {
    sudo sync
    sudo sync
    sudo sync
    while ! sudo umount \$*; do
        sleep 1
    done
}
do_test() {
    cp -p loopdisk.img.org loopdisk.img
    sudo mount ./loopdisk.img -o loop ./mnt
    sudo ./test \$2
    ls -lai mnt/ | sed '/ \\.\\.\$/d' >\$1.ls.0.log
    cp -p loopdisk.img loopdisk.img.\$1.snapshot1
    do_umount mnt
    mv loopdisk.img loopdisk.img.\$1.snapshot2
    cp -p loopdisk.img.\$1.snapshot1 loopdisk.img
    sudo mount ./loopdisk.img -o loop ./mnt
    cp -p loopdisk.img loopdisk.img.\$1.snapshot3
    do_umount mnt
    for ss in 1 2 3;do
        cp -p loopdisk.img.\$1.snapshot\$ss loopdisk.img
        sudo mount ./loopdisk.img -o loop ./mnt
        ls -lai mnt/ | sed '/ \\.\\.\$/d' >\$1.ls.\$ss.log
        do_umount mnt
    done
}

if [ -e tmp ]; then
    echo "tmp is already exist."
    return 1
fi

sudo true

mkdir tmp
cd tmp
mkdir mnt
dd if=/dev/zero of=loopdisk.img.org bs=1MiB count=10
mkfs.ext3 ./loopdisk.img.org

cat >test.cpp <<EOF
#include <fcntl.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>

#include <cstdio>
#include <string>

const size_t BUFFER_SIZE = 4096;
const size_t FILE_SIZE = 4 * 1024;  // 4KiB

char buffer[BUFFER_SIZE] = {};

int main(int argc, char *argv[]) {
  int fd = open("mnt/dummy", O_CREAT | O_WRONLY | O_TRUNC, 0666);
  if (fd == -1) {
    std::perror(argv[0]);
    return 1;
  }
  for (size_t i = 0; i < static_cast<size_t>(FILE_SIZE / BUFFER_SIZE); ++i) {
    int r = write(fd, buffer, BUFFER_SIZE);
    if (r != BUFFER_SIZE) {
      std::perror(argv[0]);
      return 1;
    }
  }
  if (argc > 1) {
    std::string arg1(argv[1]);
    int r = 0;
    if ((arg1 == "--fsync") || (arg1 == "--fsync6")) {
      r = fsync(fd);
    } else if (arg1 == "--syncfs") {
      r = syncfs(fd);
    }
    if (r == -1) {
      std::perror(argv[0]);
      return 1;
    }
    close(fd);
    if (arg1 == "--sleep") {
      sleep(6);
    } else if (arg1 == "--sleep31") {
      sleep(31);
    } else if (arg1 == "--fsync6") {
      sleep(6);
    } else if (arg1 == "--sync") {
      sync();
    }
  } else {
    close(fd);
  }
  return 0;
}
EOF
g++ -g -Wall -pedantic -std=c++17 test.cpp -o test

do_test case1 "" 2>&1 >case1.log
do_test case2 "--sleep" 2>&1 >case2.log
do_test case2a "--sleep31" 2>&1 >case2a.log
do_test case3 "--fsync" 2>&1 >case3.log
do_test case3a "--fsync6" 2>&1 >case3a.log
do_test case4 "--syncfs" 2>&1 >case4.log
do_test case5 "--sync" 2>&1 >case5.log

for f in loopdisk.img*; do
    dumpe2fs \$f 2>&1 > \$f.dumpe2fs.log
    sed '/time/d;/^Last mounted on:/d;/^Mount count:/d' \$f.dumpe2fs.log > \$f.dumpe2fs.notime.log
    debugfs -R 'logdump -a' \$f 2>&1 > \$f.logdump_not_committed.log;
    debugfs -R 'logdump -Oa' \$f 2>&1 > \$f.logdump_all.log;
    debugfs -R 'ls -l /' \$f 2>&1 > \$f.ls.log
done
CREATE_ENV_EOF
cat >qiita_base_ls.py <<CREATE_ENV_EOF
import re

count_regex = re.compile(r'^Count: (\\d+)/(\\d+)\\((\\d*.\\d+)%\\)\$')

def parse_head(line, head_regex):
    m = head_regex.match(line)
    if m:
        return (m.group(1), m.group(2))
    else:
        raise RuntimeError('Error!!!')

def parse_count(line):
    m = count_regex.match(line)
    if m:
        return (int(m.group(1)), int(m.group(2)), float(m.group(3)))
    else:
        raise RuntimeError('Error!!!')

def parse_case(lines, head_regex, file_regex):
    it = iter(lines)
    head = next(it)
    snapshot, cs = parse_head(head, head_regex)
    countline = next(it)
    count, total, percent = parse_count(countline)

    dummy_diff = False
    root_diff = False
    file = None
    for line in it:
        m = file_regex.match(line)
        if m:
            file = m.group(1)
            if file == 'dummy':
                dummy_diff = True
            elif file == '.':
                root_diff = True
            else:
                raise RuntimeError('Error!!!')
    return (snapshot, cs, count, total, percent, dummy_diff, root_diff)

case2idx = {
    'case1': 0,
    'case2': 1,
    'case2a': 2,
    'case3': 3,
    'case3a': 4,
    'case4': 5,
    'case5': 6,
}

def record(result, snapshot, cs, count, total, percent, dummy_diff, root_diff):
    if not snapshot in result.keys():
        result[snapshot] = {'dummy': [0, 0, 0, 0, 0, 0, 0], 'root': [0, 0, 0, 0, 0, 0, 0]}
    dict_file = result[snapshot]
    idx = case2idx[cs]
    if dummy_diff:
        dict_file['dummy'][idx] += percent
    if root_diff:
        dict_file['root'][idx] += percent

def table_out(fin, fout, head_regex, file_regex, print_result):
    result = {}
    it = iter(fin)
    next(it)
    lines = []
    for line in it:
        line = line.rstrip()
        if line == '============================================================================':
            snapshot, cs, count, total, percent, dummy_diff, root_diff = parse_case(lines, head_regex, file_regex)
            record(result, snapshot, cs, count, total, percent, dummy_diff, root_diff)
            lines = []
        else:
            lines.append(line)
    snapshot, cs, count, total, percent, dummy_diff, root_diff = parse_case(lines, head_regex, file_regex)
    record(result, snapshot, cs, count, total, percent, dummy_diff, root_diff)

    print_result(fout, result)
CREATE_ENV_EOF
cat >qiita_debugfs_log.py <<CREATE_ENV_EOF
import re

count_regex = re.compile(r'^Count: (\\d+)/(\\d+)\\((\\d*.\\d+)%\\)\$')

def parse_head(line, head_regex):
    m = head_regex.match(line)
    if m:
        return (m.group(1), m.group(2), m.group(3))
    else:
        raise RuntimeError('Error!!!')

def parse_count(line):
    m = count_regex.match(line)
    if m:
        return (int(m.group(1)), int(m.group(2)), float(m.group(3)))
    else:
        raise RuntimeError(f'Error!!!: {line}')



skip_switch_regex = re.compile(r'^-.*\$')
skip_diff_line_regex = re.compile(r'^\\d.*\$')
skip_journal_starts_regex = re.compile(r'^[<>] Journal starts at block.*\$')
end_of_journal_regex = re.compile(r'^[<>] (Found sequence \\d+ \\(not \\d+\\)|No magic number) at block \\d+|: end of journal.\$')
commit_regex = re.compile(r'^> Found expected sequence \\d+, type \\d+ \\(commit block\\) at block \\d+\$')
skip_found_descriptor_regex = re.compile(r'^> Found expected sequence \\d+, type \\d+ \\(descriptor block\\) at block \\d+\$')
start_transaction_regex = re.compile(r'^> Dumping descriptor block, sequence \\d+, at block \\d+:\$')
transaction_fs_block_regex = re.compile(r'^>   FS block (\\d+) logged at journal block \\d+ \\(flags 0x[0-9a-f]+\\)\$')

def parse_case(lines, head_regex, file_regex):
    it = iter(lines)
    head = next(it)
    snapshot, cs, logtype = parse_head(head, head_regex)
    countline = next(it)
    count, total, percent = parse_count(countline)

    transactions = []
    transaction = None
    for line in it:
        m = skip_switch_regex.match(line)
        if m:
            continue
        m = skip_diff_line_regex.match(line)
        if m:
            continue
        m = skip_journal_starts_regex.match(line)
        if m:
            continue
        m = skip_found_descriptor_regex.match(line)
        if m:
            continue
        m = end_of_journal_regex.match(line)
        if m:
            if transaction is not None and len(transaction) > 0:
                transactions.append(transaction)
                transaction = None
            continue
        m = commit_regex.match(line)
        if m:
            if transaction is not None and len(transaction) > 0:
                transactions.append(transaction)
                transaction = None
            transactions.append('C')
            continue
        m = start_transaction_regex.match(line)
        if m:
            if transaction is None:
                transaction = []
            else:
                raise RuntimeError('Error')
            continue
        m = transaction_fs_block_regex.match(line)
        if m:
            transaction.append(m.group(1))
            continue
        raise RuntimeError(f'unknown line: {line}')

    return (snapshot, cs, logtype, count, total, percent, transactions)

def record(result, snapshot, cs, logtype, count, total, percent, transactions):
    if snapshot not in result.keys():
        result[snapshot] = {}
    dict_snapshot = result[snapshot]
    if cs not in dict_snapshot.keys():
        dict_snapshot[cs] = {}
    dict_case = dict_snapshot[cs]
    if logtype not in dict_case.keys():
        dict_case[logtype] = []
    dict_case[logtype].append((percent, transactions))

case2name = {
    'case1': '引数なし',
    'case2': 'sleep(6)',
    'case2a': 'sleep(31)',
    'case3': 'fsync',
    'case3a': 'fsync+sleep(6)',
    'case4': 'syncfs',
    'case5': 'sync',
}

def table_out(fin, fout, head_regex, file_regex):
    result = {}
    it = iter(fin)
    next(it)
    lines = []
    for line in it:
        line = line.rstrip()
        if line == '============================================================================':
            snapshot, cs, logtype, count, total, percent, transactions = parse_case(lines, head_regex, file_regex)
            record(result, snapshot, cs, logtype, count, total, percent, transactions)
            lines = []
        else:
            lines.append(line)
    snapshot, cs, logtype, count, total, percent, transactions = parse_case(lines, head_regex, file_regex)
    record(result, snapshot, cs, logtype, count, total, percent, transactions)
    
    for snapshot, cases in result.items():
        print(f'[{snapshot}]')
        for case, cols in cases.items():
            print(f'|{case2name[case]}|', end='', file=fout)
            for logtype, pat_ary in cols.items():
                newpat = {}
                for percent, transactions in pat_ary:
                    if len(transactions) > 0:
                        transactions = ','.join(x if not isinstance(x, list) else '{'+','.join(x)+'}' for x in transactions)
                    else:
                        transactions = '-'
                    if transactions not in newpat:
                        newpat[transactions] = percent
                    else:
                        newpat[transactions] += percent
                text = '<br>'.join(f'{transactions}({percent}%)' if percent < 100 else transactions for transactions, percent in newpat.items())
                print(f'{text}|', end='', file=fout)
            print(file=fout)

import re

head_regex = re.compile(r'^\\[[^/]*/[^/]*/[^/]*/([^/]*)/([^/]*)/([^/]*)/[^/]*\\]\$')
file_regex = re.compile(r'^>\\s*\\d+\\s+\\d+\\s+\\(\\d+\\)\\s+\\d+\\s+\\d+\\s+\\d+\\s+\\d+\\s+(dummy|\\.)\$')

import sys
table_out(sys.stdin, sys.stdout, head_regex, file_regex)
CREATE_ENV_EOF
cat >qiita_debugfs_ls.py <<CREATE_ENV_EOF
from qiita_base_ls import table_out
import re

head_regex = re.compile(r'^\\[[^/]*/[^/]*/[^/]*/([^/]*)/([^/]*)/[^/]*\\]\$')
file_regex = re.compile(r'^>\\s*\\d+\\s+\\d+\\s+\\(\\d+\\)\\s+\\d+\\s+\\d+\\s+\\d+\\s+\\d+\\s+(dummy|\\.)\$')

casenames = [
    '引数なし',
    'sleep(6)',
    'sleep(31)',
    'fsync',
    'fsync+sleep(6)',
    'syncfs',
    'sync',
]

def print_result(fout, result):
    for snapshot, dict_cases in result.items():
        print(f'[{snapshot}]', file=fout)
        for case_number in range(len(casenames)):
            print(f'|{casenames[case_number]}|', end='', file=fout)
            for file in ['dummy', 'root']:
                col = dict_cases[file][case_number]
                if col == 0.:
                    print('差分なし|', end='', file=fout);
                elif col == 100.:
                    print('差分あり|', end='', file=fout);
                else:
                    print(f'差分なし({100. - col}%)<br>', end='')
                    print(f'差分あり({col}%)|', end='')
            print(file=fout)
    fout.flush()

import sys
table_out(sys.stdin, sys.stdout, head_regex, file_regex, print_result)
CREATE_ENV_EOF
cat >qiita_dumpe2fs.py <<CREATE_ENV_EOF
import re

head_regex = re.compile(r'^\\[[^/]*/[^/]*/([^/]*)/([^/]*)/[^/]*\\]\$')
count_regex = re.compile(r'^Count: (\\d+)/(\\d+)\\((\\d*.\\d+)%\\)\$')
param_regex = re.compile(r'^([<>])\\s([^:]+):\\s+(.*)\$')

def parse_head(line):
    m = head_regex.match(line)
    if m:
        return (m.group(1), m.group(2))
    else:
        raise RuntimeError('Error!!!')

def parse_count(line):
    m = count_regex.match(line)
    if m:
        return (int(m.group(1)), int(m.group(2)), float(m.group(3)))
    else:
        raise RuntimeError(f'Error!!!: {line}')

def diff_ranges(left_ranges, right_ranges):
    li = 0
    ri = 0
    left_range = left_ranges[li]
    right_range = right_ranges[ri]
    left0 = left_range[0]
    right0 = right_range[0]
    left_only = []
    right_only = []
    while True:
        if left0 < right0:
            if left_range[1] < right0:
                left_only.extend(range(left0, left_range[1]+1))
                li += 1
                if li >= len(left_ranges):
                    break
                else:
                    left_range = left_ranges[li]
                    left0 = left_range[0]
            else:
                left_only.extend(range(left0, right0))
                left0 = right0
                if left_range[1] < right_range[1]:
                    right0 = left_range[1]+1
                    li += 1
                    if li >= len(left_ranges):
                        right_only.extend(range(left_range[1]+1, right_range[1]+1))
                        ri += 1
                        break
                    else:
                        left_range = left_ranges[li]
                        left0 = left_range[0]
                else:
                    left0 = right_range[1]+1
                    ri += 1
                    if ri >= len(right_ranges):
                        left_only.extend(range(right_range[1]+1, left_range[1]+1))
                        li += 1
                        break
                    else:
                        right_range = right_ranges[ri]
                        right0 = right_range[0]
        else:
            if right_range[1] < left0:
                right_only.extend(range(right0, right_range[1]+1))
                ri += 1
                if ri >= len(right_ranges):
                    break
                else:
                    right_range = right_ranges[ri]
                    right0 = right_range[0]
            else:
                right_only.extend(range(right0, left0))
                right0 = left0
                if right_range[1] < left_range[1]:
                    left0 = right_range[1]+1
                    ri += 1
                    if ri >= len(right_ranges):
                        left_only.extend(range(right_range[1]+1, left_range[1]+1))
                        li += 1
                        break
                    else:
                        right_range = right_ranges[ri]
                        right0 = right_range[0]
                else:
                    right0 = left_range[1]+1
                    li += 1
                    if li >= len(left_ranges):
                        right_only.extend(range(left_range[1]+1, right_range[1]+1))
                        ri += 1
                        break
                    else:
                        left_range = left_ranges[li]
                        left0 = left_range[0]
    while li < len(left_ranges):
        left_range = left_ranges[li]
        left_only.extend(left_range[0], left_range[1]+1)
        li += 1
    while ri < len(right_ranges):
        right_range = right_ranges[ri]
        right_only.extend(right_range[0], right_range[1]+1)
        ri += 1
    return (tuple(left_only), tuple(right_only))

def diff_ary(left, right):
    li = 0
    ri = 0
    l_only = []
    r_only = []
    while ri < len(right):
        lelm = left[li]
        found = False
        for ristart in range(ri, len(right)):
            if right[ristart] == lelm:
                found = True
                break
        if found:
            r_only.extend(right[ri:ristart])
            ri = ristart
            li += 1
            ri += 1
            if li >= len(left):
                break
        else:
            l_only.add(lelm)
            li += 1
            if li >= len(left):
                break
    if ri < len(right):
        r_only.extend(right[ri:])
    return (tuple(l_only), tuple(r_only))

def parse_case(lines):
    it = iter(lines)
    head = next(it)
    snapshot, cs = parse_head(head)
    countline = next(it)
    count, total, percent = parse_count(countline)

    left_params = {}
    params = {}
    for line in it:
        m = param_regex.match(line)
        if m:
            left_right = m.group(1)
            name = m.group(2)
            values = re.split(r',? +', m.group(3))
            if left_right == '<':
                left_params[name] = values
            elif left_right == '>':
                if name == 'Free blocks':
                    pass
                elif name == 'Free inodes':
                    pass
                elif name == '  Free blocks':
                    left_ranges = [[int(x) for x in e.split('-')] for e in left_params[name]]
                    right_ranges = [[int(x) for x in e.split('-')] for e in values]
                    params['増加ブロック'], dummy = diff_ranges(left_ranges, right_ranges);
                    if len(dummy) != 0:
                        raise RuntimeError('Error!!!')
                elif name == '  Free inodes':
                    left_ranges = [[int(x) for x in e.split('-')] for e in left_params[name]]
                    right_ranges = [[int(x) for x in e.split('-')] for e in values]
                    params['増加inode'], dummy = diff_ranges(left_ranges, right_ranges);
                    if len(dummy) != 0:
                        raise RuntimeError('Error!!!')
                elif name == 'Journal start':
                    pass
                elif name == 'Journal sequence':
                    params[name] = values[0]
                elif name == 'Filesystem features':
                    minus, plus = diff_ary(left_params[name], values)
                    for e in plus:
                        params[f'{name}({e})'] = '+'
                    for e in minus:
                        params[f'{name}({e})'] = '-'
                else:
                    raise RuntimeError(f'Unexpected parameter "{name}"')
            else:
                raise RuntimeError('Error!!!')
    return (snapshot, cs, count, total, percent, params)

def record(result, snapshot, cs, count, total, percent, params):
    if not snapshot in result.keys():
        result[snapshot] = {}
    dict_cases = result[snapshot]
    if not cs in dict_cases.keys():
        dict_cases[cs] = []
    ary_params = dict_cases[cs]
    ary_params.append({'percent': percent, 'params': params})

import sys
f = sys.stdin
result = {}
it = iter(f)
next(it)
lines = []
for line in it:
    line = line.rstrip()
    if line == '============================================================================':
        snapshot, cs, count, total, percent, params = parse_case(lines)
        record(result, snapshot, cs, count, total, percent, params)
        lines = []
    else:
        lines.append(line)
snapshot, cs, count, total, percent, params = parse_case(lines)
record(result, snapshot, cs, count, total, percent, params)

rowids=[]
for snapshot, value in result.items():
    for cs, ary in value.items():
        for obj in ary:
            params = obj['params']
            percent = obj['percent']
            for name in params.keys():
                if name not in rowids:
                    rowids.append(name)

for snapshot, value in result.items():
    print(f'[{snapshot}]')
    for name in rowids:
        print(f'|{name}|', end='')
        for cs, ary in value.items():
            ary2 = ((obj["params"], obj["percent"]) for obj in ary)
            ary3 = ((params[name] if name in params.keys() else '-', percent) for params, percent in ary2)
            ary4 = {}
            for pv, percent in ary3:
                if isinstance(pv, (list, tuple)):
                    pv = ','.join(str(x) for x in pv)
                if pv not in ary4.keys():
                    pv = str(pv)
                    ary4[pv] = percent
                else:
                    ary4[pv] += percent
            if len(ary4) > 1:
                col = '<br>'.join(f'{pv}({percent}%)' for pv, percent in ary4.items())
            else:
                col = next(iter(ary4.keys()));
            print(f'{col}|', end='')
        print()
CREATE_ENV_EOF
cat >qiita_ls.py <<CREATE_ENV_EOF
from qiita_base_ls import table_out
import re

head_regex = re.compile(r'^\\[[^/]*/[^/]*/([^/]*)/([^/]*)/[^/]*\\]\$')
file_regex = re.compile(r'^<\\s*\\d+\\s+[-rwxdst]+\\s+\\d+\\s+\\w+\\s+\\w+\\s+\\d+\\s+(dummy|\\.)\$')

def print_result(fout, result):
    for file in ['dummy', 'root']:
        print(f'[{file}]', file=fout)
        for key, value in result.items():
            print(f'|{key}|', end='', file=fout)
            for col in value[file]:
                if col == 0.:
                    print('差分なし|', end='', file=fout);
                elif col == 100.:
                    print('差分あり|', end='', file=fout);
                else:
                    print(f'差分なし({100. - col}%)<br>', end='')
                    print(f'差分あり({col}%)|', end='')
            print(file=fout)
    fout.flush()

import sys
table_out(sys.stdin, sys.stdout, head_regex, file_regex, print_result)
CREATE_ENV_EOF

```

</details>

