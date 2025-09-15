# C/C++ メモリ管理

## 1. メモリ領域の全体像
 <figure style="text-align:center;">
<img width="500" src="./images/memory_using.drawio.png">
<figcaption>図１　メモリ領域の全体像</figcaption>
</figure>

各領域の役割は以下の通り。

- **テキスト領域**：プログラム命令（機械語）が格納される。 CPU がこの命令順に実行してプログラムが動作する。
- **静的領域**：グローバル変数などの静的変数が置かれる。プログラム開始から終了まで存続。
- **ヒープ領域**：メモリの動的管理 (C 言語の malloc 関数や C++ の new 演算子でメモリを確保すること) で用いられる。手動管理のため解放操作が必要。
- **スタック領域**：ローカル変数（自動変数）や呼び出し関数の戻り値、引数が置かれる。自動管理のためスコープ終了時に自動的に解放される。

## 2. C のメモリ管理
### 2.1 スタック変数
- 自動的にスコープ終了時に解放される
```c
int func(int a int* b){
    int x = 10;
    int arr[5];

    return x
}
```

### 2.2 ヒープ変数
- `malloc`/`free` を組み合わせる必要がある
- 初期化やエラー処理は自分で行う
```c
int* p = (int*)malloc(5 * sizeof(int));
if (p == NULL) {
    fprintf(stderr, "メモリ確保に失敗しました\n");
    return 1; // 異常終了
}
free(p);
```

## 3. C++ のメモリ管理
### 3.1 スタック変数
- 自動的にスコープ終了時に解放される
```cpp
Foo foo; // ctor/dtor 自動呼び出し
```
### 3.2 ヒープ変数
- `new`/`delete` を組み合わせる必要がある
- 初期化やエラー処理は自分で行う
- 配列を確保するときは `new[]` / `delete[]` を使う。
```cpp
Foo* p = new(std::nothrow) Foo();
if (p == nullptr) {
    std::cerr << "メモリ確保に失敗しました" << std::endl;
} else {
    // ...利用...
    delete p;
}

Foo* arr = new Foo[10];   // Fooの配列を10個分確保
delete[] arr;             // 配列解放
```

### 3.3 スマートポインタ
#### 3.3.1 shared_ptr
- ヒープ領域に確保したメモリを自動的に解放するクラス
```cpp
std::shared_ptr<Foo> sp = std::make_shared<Foo>();
```
#### 3.3.2 参照カウンタ
##### 参照カウンタが増えるとき
1. 新しい shared_ptr が同じオブジェクトを所有するとき
   ```cpp
    auto sp1 = std::make_shared<int>(42);
    auto sp2 = sp1;  // 参照カウント +1

    // use_count() == 2
   ```

2. 関数に値渡しされたとき
   ```cpp
   void foo(std::shared_ptr<int> p) {
       // use_count() == 2
       std::cout << p.use_count() << std::endl;
   }
   auto sp = std::make_shared<int>(42);
   foo(sp); // 引数でカウンタ +1
   ```

3. 関数の戻り値で返されたとき
   ```cpp
   std::shared_ptr<int> makePtr() {
       return std::make_shared<int>(99); // 呼び出し元に所有権が移る
   }
   auto sp = makePtr(); // use_count == 1
   ```

4. コンテナに格納したとき
   ```cpp
    auto sp1 = std::make_shared<int>(42);
    std::vector<std::shared_ptr<Foo>> v;
    v.push_back(sp1);  // コピーされるので +1
   ```

##### 参照カウンタが減るとき
1. shared_ptr がスコープを抜けたとき
   ```cpp
   {
       auto sp = std::make_shared<int>(42);
   } // スコープ終了 → use_count -1
   ```

2. 代入で別のポインタを持つようになったとき
   ```cpp
   auto sp1 = std::make_shared<int>(42);
   auto sp2 = std::make_shared<int>(100);
   sp1 = sp2; // sp1 が 42 を解放、100 を共有 (カウント増減)
   ```

3. 明示的にリセットしたとき
    ```cpp
    auto sp = std::make_shared<int>(42);
    sp.reset(); // use_count -1 → 0 なら解放
    ```

4. コンテナから削除されたとき
    ```cpp
    v.pop_back();  // その要素の shared_ptr が
    ```

### 3.4 配列や複数オブジェクトの管理
#### 3.4.1 shared_ptr<vector<Foo>>
```cpp
std::shared_ptr<std::vector<Foo>> spVec = std::make_shared<std::vector<Foo>>(10);
```

#### 3.4.2 vector<shared_ptr<Foo>>
```cpp
std::vector<std::shared_ptr<Foo>> vec;
for(int i=0;i<10;i++)
    vec.push_back(std::make_shared<Foo>());
```
## 4. 典型的なバグ
| バグの種類 | 原因 | 対策・防止策 | コンパイル時 | 実行時の動作 |
|------------|------|--------------|--------------------|--------------|
| **メモリリーク** | 確保したメモリを解放し忘れる | - `free` / `delete` を必ず呼ぶ<br>- `スマートポインタ` を利用（C++）<br>- 静的解析ツールや Valgrind で検出 | ❌ エラーにならない | 長時間実行でメモリ不足、リソース枯渇 |
| **ダングリングポインタ** | 解放後のメモリを再度参照する | - 解放後はポインタを `NULL` / `nullptr` に設定<br>- スマートポインタを利用 | ❌ エラーにならない | 未定義動作：クラッシュ、予測不能な値の参照 |
| **二重解放** | 同じポインタを `free` / `delete` する | - 解放後に `nullptr` を代入<br>- スマートポインタは参照カウントで防止 | ❌ エラーにならない | 実装依存：クラッシュ、ヒープ破壊 |
| **未確保領域アクセス** | - 未初期化ポインタを使う<br>- 配列の範囲外をアクセス | - ポインタは必ず初期化<br>- 配列は範囲チェック<br>- `std::vector::at()` を利用 | ❌ エラーにならない（範囲外でもコンパイル可） | 未定義動作：セグフォ、データ破壊、例外発生（C++ STL） |


### 4.1 メモリリーク
長時間動作するプログラムでは、確保したメモリが解放されず、最終的にシステムがメモリ不足に陥る
- 原因
   確保したメモリを `free` / `delete` し忘れる。
- 予防策
  -`free` / `delete`を必ず呼ぶ
  - スマートポインタを使う(C++)

```c
void leak_example() {
    int* p = (int*)malloc(10 * sizeof(int));
    // free(p); を忘れる
} // p はスコープを抜けてもメモリは解放されない
```
### 4.2 タングリングポインタ
解放したメモリにアクセスする。

- NG
    ```c
    int* p = (int*)malloc(sizeof(int));
    *p = 42;
    free(p);

    printf("%d\n", *p);  //  解放済みの領域にアクセス → 未定義動作
    ```
- OK
    ```c
    int* p = (int*)malloc(sizeof(int));
    *p = 42;
    free(p);
    p = NULL; // 解放後は必ずNULL

    if(p){ //  NULLチェックが可能になる
        printf("%d\n", *p);
    }
    ```
### 4.3 二重解放
解放済みのメモリを再度解放する
- OK
    ```c
    int* p = (int*)malloc(10 * sizeof(int));
    free(p);
    free(p); // 二重解放
    ```
- NG
    ```c
    int* p = (int*)malloc(sizeof(int));
    free(p);
    p = NULL;
    if (p) { //  NULLチェックをする
        free(p); // 実行されない
    }
    ```

### 4.4 未確保領域アクセス
未確保または確保範囲外のメモリにアクセスする
- OK
    ```c
    int* p;        // 初期化されていない
    *p = 10;       // 未定義動作

    int* arr = (int*)malloc(3 * sizeof(int));
    arr[3] = 42;   // 範囲外アクセス (0,1,2 までしか使えない)
    free(arr);
    ```
- NG
    ```c
    int* p=NULL;   // NULLで初期化
    if(p){      
        *p = 10;   // 実行されない
    }
    ```