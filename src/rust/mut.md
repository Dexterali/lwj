# mut 和 &mut

```rust
#![allow(unused)]
fn main() {
    // a拥有所有权
    let a = String::from("hello rust!");
    // 这行代码会报错，因为a不可变，即不可更新绑定
    // a = "foo".to_string();
    
    // mut var 和 &var
    // b拥有所有权，并且b可以更新绑定
    let mut b = "Hello Rust".to_string();
    b.push_str("foo");
    b = "foo".to_string(); // b可以更新绑定

    let c = b;
    // 这行代码会报错，因为b的所有权转移给了c
    // println!("{}", b);

    // c绑定的资源借给d使用，d只有资源c的读权限
    let d = &c;
    // 这行代码会报错，因为d只有资源c的读权限
    // d.push_str("bar");

    // -------------------------------------
    let mut e = "Hello Rust".to_string();

    // e绑定的资源借给f使用，f有资源的读写权限，这要求e必须本身可读写，即可变
    let f = &mut e;
    f.push_str("hi");
    // 这行代码会报错，因为f不可变，即不可更新绑定
    // f = &mut "foo".to_string();

    // ----------------------------------------
    let mut g = "Hello Rust".to_string();

    // g绑定的资源借给h使用，h有资源的读写权限，这要求e必须本身可读写，即可变。且h可以更新绑定，即h可变。
    let mut h = &mut g;
    h.push_str("hi");
    h = &mut "foo".to_string();

}

// 含义：传参的时候，实参d绑定的资源D的所有权转移给c
fn do1(c: String) {}

// 含义：传参的时候，实参d将绑定的资源D借给c使用，c对资源D只读
fn do2(c: &String) {}

// 含义：传参的时候，实参d将绑定的资源D借给c使用，c对资源D可读写
fn do3(c: &mut String) {}

// 含义：传参的时候，实参d将绑定的资源D借给c使用，c对资源D可读写。同时，c可绑定到新的资源上面去（更新绑定的能力）
fn do4(mut c: &mut String) {}

// 函数参数里面，冒号左边的部分，mut c，这个mut是对函数体内部有效；冒号右边的部分，&mut String，这个 &mut 是针对外部实参传入时的形式化（类型）说明。
```