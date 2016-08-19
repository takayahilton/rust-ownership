# メモリの安全性とリソース管理とRust
________



### Rustとは

Mozillaによって開発されたシステムプログラミング用の言語
手続き型・関数型・オブジェクト指向をサポートしている。
ゼロコスト抽象化(=オーバーヘッドなしの抽象化)も目標としている。



## 基本構文


## hello world
```rust
let hello = "hello world!";
println!("{}", hello);
```


## 変数束縛
```rust
let one: u32 = 1; //再代入不可
let mut one: u32 = 1;//再代入化
let one = 1; //変数宣言では型推論できる
```


## 関数定義
```rust
//仮変数と帰り型の型推論はできない。returnはいらない
fn add(x: u32, y: u32) -> u32 {
    a + b
}
```

## 構造体と列挙型
```rust
struct Point {
    x: i32,
    y: i32,
}

enum Color {
    Red,
    Green,
    Blue,
}
```



## メモリの安全性とは
不正なメモリにアクセスしない事

```cpp
auto user = new User();
delete user;
user.name; // 不正なメモリアクセス!
```


他にも
- 解放したメモリをもう一回解放してしまう
- 確保していないメモリを解放する
- 確保していないメモリにアクセスするとか
- マルチスレッドでのデータ競合など


プログラムが管理していないメモリに対する操作をしないようにする


GCがある言語ならユーザーがメモリの管理しなくてすむので安全
がGCにはオーバーヘッドあるので低レイヤの処理ではなるべく避けたい

とはいえ自分でメモリ管理するのめんどくさい！というかバグりそう



# 解放処理
変数のスコープを抜けたらその変数が持っているメモリを解放する処理をコンパイラが付け加えれば良い
```rust
{
    let one = Box::new(1);
}// drop(one); 
```


これでメモリ管理しなくていいな！


しかしこういう場合はどうする?
```rust
let one = Box::new(1);
{
    let inner_one = one;
}// drop(one); 
```


外のスコープのoneがまだ有効なのに解放してしまっている
メモリを複数の変数が参照している状態だと上手く動かない
どうする?



## 所有権

```rust
let one = 1;
let _one = one;
println!("{}", one); error: use of moved value: `one`
```


代入したり関数の引数に渡したりする度に所有権が移動

所有権を失った変数は以後使う事ができない

rustで構造体を定義するとデフォルトで所有権が移動する

(Copy型を実装すると所有権の移動ではなくコピー)


一つのメモリを参照するのは一つの変数だけにする事によって
不正なアクセスは起こらない
が　これは関数に変数を渡すと前の変数は使えなくなるという事
```rust
fn print<A: std::fmt::Debug>(a: A) {
    println!("{:?}", a)
}

let one = Box::new(1);
print(one);
print(one); //error: use of moved value
```
使い辛い。。。


いちいちmoveが起きて前の変数が使えなくなる
と云う事態は避けたい

=> どうするか



# 参照とlifetime
`&`をつけると変数を参照として借用できる
```rust
fn print<A: std::fmt::Debug>(a: A) {
    println!("{:?}", a)
}

let one = Box::new(1);
print(&one);
print(&one);//ok!
```


参照は型としてlifetimeを持っていて
lifetimeに違反するとコンパイルエラーになる
```rust
fn one()-> &u32 {
    let one = 1;
    &one //error: missing lifetime specifier
}
```
内側のスコープで宣言したものを外側のスコープに返してるのでエラー

(スタックで確保したポインタを返すとc言語でも警告はでるけど)


返している参照のlifetimeが外のスコープのものと同じなのでこっちは大丈夫
```rust
fn add_one(i: &u32)-> &u32 {
    *i = *i + 1;
    i
}
```



## 参照を含む構造体
参照をメンバに持つ構造体を作る

Rustには複数のポインタ型があるが統一的な
```rust
struct Wrapper<A> {
    a: &A
}
```


error: missing lifetime specifier 
???


こう書くのが正解


```rust
struct Wrapper<'a, A: 'a> {
    a: &'a A
}
```


なにこれ。。。


`&x`という書き方は実は`&'a x`の省略　'aは寿命
関数や変数に指定する時は推論してくれるが構造体作る時は推論してくれない


`'a` <- lifetime 

`A: 'a` <- 型引数のAの寿命を指定

`a: &'a A` <- 参照の型を指定



## まとめ
- Rustは借用とlifetimeのおかげでオーバーヘッドなしでリソースの自動管理ができる
- 借用とlifetimeらへん学習コストは高め　慣れるまでコンパイラと戦う
- [プログラミング言語Rust](https://rust-lang-ja.github.io/the-rust-programming-language-ja/1.6/book/README.html)を読みましょう







