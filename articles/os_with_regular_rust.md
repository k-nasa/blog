---
title: "Rustを使ったOS開発 - 普段のRustと同じ様にOSを実装したい"
emoji: "📖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["os自作", "rust"]
published: false
---

## まえがき

こんにちは、[@nasa](https://twitter.com/nasa_desu)です。

この記事は、[Wantedly Advent Calendar 2023](https://qiita.com/advent-calendar/2023/wantedly)10日目の記事です。
前回の記事は、「[Wantedly での SLO 運用の現状とこれから](https://www.wantedly.com/companies/wantedly/post_articles/876525)」でした！

最近、趣味でRustを用いてOSを開発しています。通常のRustでCLIツールを開発する際とは異なり、OSを書く際には多くの異なる考慮事項があります。
ここでは、ベアメタル環境（OSが存在しない環境）で動作するプログラムをRustで実装する方法を紹介します。

ゴールは普段のRustと同じ様にOSを実装できる状態にすることです。

## 普段のRustとは

タイトルで普段のRustと書きましたが、普段のRustとはどのようなものでしょうか？
本記事では下記の条件を満たす環境で実装できる状態を普段のRustと呼んでいます。

- main関数が動作する
- 関数呼び出しができる
- 動的メモリ確保ができる
- printfが使える

コードにすると以下のような感じですね。
vecを使ってhello worldしていて普段というには半ば強引ですが。。。

```rust
fn main() {
    hello();
}

fn hello() {
    let hello_world = vec!['H', 'e', 'l', 'l', 'o', ' ', 'W', 'o', 'r', 'l', 'd', '!', '\n'];

    for c in hello_world {
            print!("{}", c);
    }
}
```

OSを書くときにはベアメタル環境(OSがない環境)で動作するプログラムを書くことになります。
よってOSが提供するさまざま機能を使うことができないので、上記の条件を満たす環境を自分で用意する必要があります。

また、先程挙げた4つ以外にもOSが提供していた機能は多くありこれらを実現しても「普段の」とは程遠いかもしれませんが、今回はこの4つを実現することを目標にしていきます。

## 動作環境

本記事ではQEMU上で動作するRISC-V向けのプログラムを記述していきます。
QEMUは仮想マシンを動作させるためのソフトウェアで、今回はRISC-V向けの仮想マシンを動作させます。

## mainを動かす

まずはmain関数を動かしてみましょう。

OSがない環境ではstdクレートを使うことが出来ないのでこれを無効化する必要があります。

下記のようにmain.rsの先頭にno_std属性をつけることでstdクレートを無効化することができます。
また、ビルドターゲットをRISC-V向けにするために`.cargo/config.toml`を用意します。これで`cargo build`実行時にターゲットを指定する手間を省くことができます。

```rust:main.rs
#![no_std]
#![no_main]

#[panic_handler]
fn panic(_info: &core::panic::PanicInfo) -> ! {
    loop {}
}
```

```toml:.cargo/config.toml
[build]
target = "riscv64gc-unknown-none-elf"
```

stdクレートを無効化してビルドできるようになったので次はmain関数を動かしてみましょう。

まずは動作するプログラムを下記に示します。

```rust:src/main.rs
#![no_std]
#![no_main]

#[no_mangle]
#[link_section = ".entry"]
pub unsafe extern "C" fn _entry() {
    main();
}

#[inline]
fn main() {
    let a = 1;
    let b = 2;
    let c = a + b;

    loop {}
}

#[panic_handler]
fn panic(_info: &core::panic::PanicInfo) -> ! {
    loop {}
}
```

```rust:build.rs
fn main() {
    println!("cargo:rustc-link-arg-bin=code_for_blog=--script=src/link.ld");
}
```

```toml:.cargo/config.toml
[build]
target = "riscv64gc-unknown-none-elf"

[target.riscv64gc-unknown-none-elf]
runner = """
qemu-system-riscv64 \
-machine virt \
-bios default \
-nographic \
-serial mon:stdio \
-kernel
"""
```

多くのコードを提示しましたが一つずつ説明していきます。

最初に`link.ld`についてです。これはリンカスクリプトと呼ばれるもので、プログラムがメモリ上にどの様に配置されるかを定義するものです。
今回は`_entry`というシンボルを0x80200000番地に配置するように定義しています。


```link.ld
OUTPUT_ARCH( "riscv" )
ENTRY( _entry )

SECTIONS
{
  . = 0x80200000;
```

0x80200000番地をエントリーポイントとして定義している理由を説明します。
今回のプログラムが動作する前にSBIというソフトウェアが動作します。(これはqemuのオプションに`-bios default`を指定しているためです)
SBIはデバッグコンソールへの入出力やタイマーなどのOS開発に利用できる便利関数を提供してくれます。これ無しで開発することも出来るのですが楽をしたいので乗っかることにします。

このSBIは0x80200000番地に配置されたプログラムを実行するようになっているため、0x80200000番地をエントリーポイントとして定義しています。

次に`build.rs`についてです。これはビルド時に実行されるスクリプトで、今回はリンカスクリプトをリンカに渡すために使用しています。

```rust:build.rs
    println!("cargo:rustc-link-arg-bin=code_for_blog=--script=src/link.ld");
```

次に`.cargo/config.toml`についてです。これはcargoの設定ファイルで、今回は`cargo run`実行時にQEMUを起動するために使用しています。

```toml:.cargo/config.toml
[build]
target = "riscv64gc-unknown-none-elf"

[target.riscv64gc-unknown-none-elf]
runner = """
qemu-system-riscv64 \
-machine virt \
-bios default \
-nographic \
-serial mon:stdio \
-kernel
"""
```

最後に`main.rs`についてです。これはエントリーポイントとなるプログラムで、今回は`_entry`というシンボルを定義しています。
Rustの関数はデフォルトだと名前が変更される(マングリング)ので、`#[no_mangle]`をつけることでマングリングを無効化しています。
これでリンカスクリプトに定義した`_entry`というシンボルが存在するようになりました。
`#[inline]`はインライン展開を行う属性です。これをつけることで関数呼び出しを行わずに関数の中身を展開してくれます。

インライン展開を行っている理由は後述します。

```rust:src/main.rs
#[no_mangle]
#[link_section = ".entry"]
pub unsafe extern "C" fn _entry() {
    main();
}

#[inline]
fn main() {
    let a = 1;
    let b = 2;
    let c = a + b;

    loop {}
}
```

これで`cargo run`を実行するとQEMUが起動し、main関数が動作するはずです！

![opensbi](/images/opensib_view.png)

## 関数呼び出し

最初に関数呼び出しを行えるようにします。
関数呼び出し時にはスタック領域が確保されます。これは関数の引数やローカル変数などが格納される領域です。

先程のプログラムをディスアセンブルするとスタックポインタを操作していることが分かります。

```
# 名前が分かりづらいがmain関数の先頭
0000000080200492 <_ZN13code_for_blog4main17h0ff467490ac93fe7E>:
80200492: 5d 71        	addi	sp, sp, -80
```

スタックポインタは下位アドレスに伸びていきます。
しかし悲しいことに初期値が0なので減算することが出来ません。。。(余談ですがこういう場合ってCPU例外は発生しないのですね)

なのでスタックポインタの初期値を設定しスタック領域を確保しておきましょう
リンカスクリプトで設定できるんでしょうが、今回はリンカスクリプトを変更せずにスタックポインタの初期値を設定する方法を取ります。

```rust:src/main.rs
#[no_mangle]
static INIT_SP: [u8; 4096 * 1024] = [0; 4096 * 1024];

#[no_mangle]
static STACK_SIZE: usize = 4096 * 1024; // 4MB

#[link_section = ".entry"]
#[no_mangle]
pub unsafe extern "C" fn _entry() {
    // NOTE: スタックポインタの初期値を設定する
    // NOTE: スタックは下位に伸びていくのでINIT_SP + STACK_SIZEを設定しSTACK_SIZE分の領域を確保
    asm!("la sp, INIT_SP", "ld a0, STACK_SIZE", "add sp, sp, a0",);
    main();
}
```

INIT_SPという配列を定義しそのアドレス + STACK_SIZEをスタックポインタに設定しています。
何らかの理由でスタック領域が書き換わってしまうのを防ぐためにINIT_SPは配列にしています。これでINIT_SPを変更しない限りスタック領域は書き換わらないようになります。

ここまでで偉大なる力、関数呼び出しを手に入れることができました！

ちなみに先程説明を省いたインライン展開についてですが、あれは関数をインライン展開しスタックを確保しないように設定していました。

## printfを使う

次にprintfを使えるようにします。
これがあるとデバッグが捗りますね。

殆どは`format_args!`にお任せすれば良いのですが文字の出力部分は自分で実装する必要があります。

今回はOpenSBIのputcharを使って文字を表示していきます。

putchatは`a0`レジスタに表示したい文字の文字コード、`a6`レジスタに1、`a7`レジスタに0を設定してecall命令を実行することで呼び出せます。
`a6`レジスタに設定したのが関数番号で、`a7`レジスタに設定したのが拡張機能番号です。

実装としては下記の様になります。

```rust:src/main.rs
fn main() {
    println!("Hello, world!");
    loop {}
}

#[macro_export]
macro_rules! print {
    ($($arg:tt)*) => ({
        use core::fmt::Write;
        let _ = write!(crate::Writer, $($arg)*);
    });
}

#[macro_export]
macro_rules! println {
    ($($arg:tt)*) => ({
        print!("{}\n", format_args!($($arg)*));
    });
}

pub struct Writer;

impl core::fmt::Write for Writer {
    fn write_str(&mut self, s: &str) -> core::fmt::Result {
        for c in s.bytes() {
            unsafe {
                asm!(
                    "ecall",
                    in("a0") c,
                    in("a6") 0,
                    in("a7") 1,
                );
            }
        }
        Ok(())
    }
}
```

## 動的メモリ確保

最後に動的メモリ確保を行います。
これでVecやStringなど便利な型を使って実装できるようになります

今回はBump allocatorというメモリアロケータにより動的メモリ確保を実現します。
「Bump Allocator」とは何かを簡単に説明します。Bump Allocatorは、メモリの割り当てを、割り当て可能なメモリ領域の先頭から順に行うシンプルなメモリアロケータです。メモリ解放は行われませんがそのシンプルさから実装が容易です。

メモリの再利用を行いたい場合は[Writing an OS in Rust](https://os.phil-opp.com/allocator-designs/ja/)あたりを参考に別のアロケーターを実装してみてください。


Bump allocatorの実装は下記のようになります。

```rust:src/main.rs
use core::{
    alloc::{GlobalAlloc, Layout},
    cell::{Cell, RefCell},
};

#[global_allocator]
static mut ALLOCATOR: BumpAllocator = BumpAllocator::new();

const ARENA_SIZE: usize = 32 * 1024 * 1024; // 32MB

pub struct BumpAllocator {
    arena: RefCell<[u8; ARENA_SIZE]>,
    next: Cell<usize>,
}

impl BumpAllocator {
    const fn new() -> Self {
        Self {
            arena: RefCell::new([0; ARENA_SIZE]),
            next: Cell::new(0),
        }
    }
}

unsafe impl GlobalAlloc for BumpAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        let next = self.next.get();

        let size = layout.size();
        let align = layout.align();

        let alloc_start = aligned_addr(next, align);

        let mut arena = self.arena.borrow_mut();
        let alloc_end = alloc_start + size;

        if alloc_end > arena.len() {
            panic!("out of memory");
        }

        self.next.set(alloc_end);

        let ptr = arena.as_mut_ptr();

        return ptr.add(alloc_start);
    }

    // BumpAllocatorはメモリを開放しない
    unsafe fn dealloc(&self, _ptr: *mut u8, _layout: Layout) {}
}

fn aligned_addr(addr: usize, align: usize) -> usize {
    if addr % align == 0 {
        addr
    } else {
        addr + align - (addr % align)
    }
}
```

RustではGlobalAllocトレイトを実装することで自作のメモリアロケータを実装することが出来ます。
また`#[global_allocator]`属性をつけることで自作のメモリアロケータをデフォルトのメモリアロケータとして使用することが出来ます。

次にBumpAllocator構造体の実装についてです。
BumpAllocatorは`arena`フィールドと`next`フィールドを持っています。
`arena`フィールドはメモリ領域を保持するための配列で`next`フィールドは次に割り当てるアドレスを保持するための変数です。

```rust
pub struct BumpAllocator {
    arena: RefCell<[u8; ARENA_SIZE]>,
    next: Cell<usize>,
}

impl BumpAllocator {
    const fn new() -> Self {
        Self {
            arena: RefCell::new([0; ARENA_SIZE]),
            next: Cell::new(0),
        }
    }
}
```

`alloc`メソッドは`&self`を受け取る、つまり不変参照を受け取るメソッドですが`next`は都度変更する必要がありますし、`alloc`では`arena`の可変参照を返す必要があります。

```rust
pub unsafe trait GlobalAlloc {
    // selfは不変参照
    // 返り値の*mut u8は可変参照。ここでarenaのポインタを返したい
    unsafe fn alloc(&self, layout: Layout) -> *mut u8;
}
```

そのためBumpAllocatorは内部可変性を持つ必要があります。
RefCellやCellを使ってこれを実現しています。内部可変性とその実装については[こちら](https://doc.rust-jp.rs/book-ja/ch15-05-interior-mutability.html)を参考にしてみてください。


RefCell, Cellを使っていること以外は特筆することは無さそうですね。配列の先頭からメモリを切り盛りしています。

```rust
unsafe impl GlobalAlloc for BumpAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        let next = self.next.get();

        let size = layout.size();
        let align = layout.align();

        // アラインメントを考慮したアドレスを計算する
        let alloc_start = aligned_addr(next, align);

        let mut arena = self.arena.borrow_mut();
        let alloc_end = alloc_start + size;

        if alloc_end > arena.len() {
            panic!("out of memory");
        }

        self.next.set(alloc_end);

        let ptr = arena.as_mut_ptr();

        return ptr.add(alloc_start);
    }

    // BumpAllocatorはメモリを開放しない
    unsafe fn dealloc(&self, _ptr: *mut u8, _layout: Layout) {}
}
```

## まとめ

今回は普段のRustと同じ様にOSを実装するまでの道のりを書いていきました。
無事に関数呼び出し、printf、動的メモリ確保ができるようになりました。

```rust
fn main() {
    hello();
}

fn hello() {
    let hello_world = vec!['H', 'e', 'l', 'l', 'o', ' ', 'W', 'o', 'r', 'l', 'd', '!', '\n'];

    for c in hello_world {
            print!("{}", c);
    }
}
```

最後に今回の記事で実装したコードを公開しておきます。

https://github.com/k-nasa/blog_sample_code

## もっと詳しく知りたい方へ

### [Writing an OS in Rust](https://os.phil-opp.com/)

Rustでx86_64向けのOSを実装するチュートリアルです。
日本語訳もありかなり詳しく書かれているのでおすすめです.

### [octox](https://github.com/o8vm/octox)

Rustで実装されたRISC-V向けのOSです。RustでOSを実装する際の参考になると思います。
僕はチュートリアルだと飽きちゃうのでoctoxを参考にしつつOSを実装しています。

OpenSBIを利用せずに実装しているので全部自分でやりたい人には良いかもしれません。

## 最後に

最後まで読んでいただきありがとうございました。
OSやハードウェアへの知識があまりない状態でOSを実装しているのですが、実際に作ってみると色々と学べることがあり楽しいです。
変な(?)バグを大量に踏むのでそれを解決するのも楽しいです(ツライ)

今作っているOSはまだまだ実装できていない機能が多くありますが、完成したらまた記事にしたいと思います。
SPの初期値を設定する必要があるってのに気づくまでに時間がかかった思い出です。。。あとプロセスのコンテキストスイッチムズい。。。

みなさんも是非OSを実装してみてください！