# ［C++］モジュールにおける所有権モデル

モジュールにおける所有権とは、名前付きモジュールに属しているエンティティの名前に関する所有権の事で、次のようなコードがどう振舞うかを決定するものです。

```cpp
/// moduleA.ixx
export module moduleA;

export int munge(int a, int b) {
    return a + b;
}
```

```cpp
/// moduleB.ixx
export module moduleB;

export int munge(int a, int b) {
    return a - b;
}
```

```cpp
/// libA.h

int libm_munge(int a, int b);
```

```cpp
/// libA.cpp

#include "libA.h"
import moduleA;

int libm_munge(int a, int b) {
    return munge(a, b);
}
```

```cpp
/// main.cpp

#include "libA.h"
import moduleB;

// ここでは、moduleAはインポートされていない

int main() {
  // それぞれ、どちらが呼ばれる？
  int a = munge(1, 2);      // moduleBのものが呼ばれる？
  int b = libm_munge(1, 2); // moduleAのものが呼ばれる？
}
```

C++の意味論としてはこれは当然明白なことで、コメントにある通りに呼ばれるはずです。しかし、このプログラムはコンパイルされリンクされた結果として一つの実行ファイルになります。C++20の名前付きモジュールはそれそのものが1つの翻訳単位をなすので、そこには2つのモジュール（`moduleA`と`moduleB`）をコンパイルした2つのオブジェクトファイルが含まれます。  
その時、翻訳単位`main.cpp`と`libA.cpp`はそれぞれのコードの意味論から期待されるモジュールからの関数を呼び出すでしょうか？

これはつまりODR違反が起きている状態です。C++17まではこれは未定義動作であり、リンカがオブジェクトファイルをリンクする順番に依存して結果が変わります。そして、モジュールファイルでもこれは未定義動作とされています。

モジュールの所有権とはこの未定義動作をどう実装するか？という問題の解です。そこには、2つの選択肢があります。

なお、同じ場所で2つの異なるモジュールからの同じ宣言が見えている場合は確実にコンパイルエラーとなります、今回のケースはリンカからしかそれが見えていません。

```cpp
import moduleA;
import moduleB;

int main() {
  int a = munge(1, 2);  // コンパイルエラー！
}
```

### 弱い所有権（*weak ownership*）モデル

弱い所有権（*weak ownership*）はこの問題を解決しないモデルです。すなわち、C++17以前と同じくこれは未定義動作であり、リンカがリンクする順番に依存します。

```cpp
/// main.cpp

#include "libA.h"
import moduleB;

// ここでは、moduleAはインポートされていない

int main() {
  // 弱い所有権モデルの下では未定義動作！！
  int a = munge(1, 2);
  int b = libm_munge(1, 2);
}
```

### 強い所有権（*strong ownership*）モデル

強い所有権（*strong ownership*）はこれを解決するモデルです。すなわち、異なるモジュールからエクスポートされているものは、その宣言が完全に同一であったとしても、リンク時にもモジュール名によって別のものとして扱われます。

```cpp
/// main.cpp

#include "libA.h"
import moduleB;

// ここでは、moduleAはインポートされていない

int main() {
  // 強い所有権モデルの下では期待通りに呼ばれる
  int a = munge(1, 2);      // -1（moduleBのものが呼ばれる）
  int b = libm_munge(1, 2); // 3（moduleAのものが呼ばれる）
}
```

### モジュールリンケージ

C++20モジュールにおいてはリンケージ区分にもう一つモジュールリンケージが追加されました。モジュールリンケージは名前付きモジュール内で外部リンケージを持ち得る名前のうち、エクスポートされていないものが持つリンケージです。モジュールリンケージをもつ名前は同じモジュール内からしか参照できません。つまりは、内部リンケージを同じモジュール内部の範囲まで拡張したものです。

```cpp
export module moduleA;

// モジュールリンケージ
int func1(int a, int b) {
  return a + b;
}

// 外部リンケージ
export int func2(int a, int b) {
  return a + b;
}

// 内部リンケージ
static int func3(int a, int b) {
  return a + b;
}
```

リンケージの観点からは、2つの所有権モデルの差異は外部リンケージを持つ名前をモジュールが所有するかどうか？という問題とみることができます。

弱い所有権モデルでは、内部リンケージとモジュールリンケージを持つ名前だけをモジュールが所有し、エクスポートされている名前をモジュールは所有しません。

強い所有権モデルでは、外部リンケージを持つ名前を含めたモジュールに属するすべての名前がモジュールによって所有されます。

### 所有権と名前マングリング

この2つの所有権モデルのどちらを選択するか？という問題は、実装から見ると名前マングリングの問題となるでしょう（他の実装もあり得ますが）。

1つのモジュールは複数の翻訳単位から構成されうるので、リンカは少なくともモジュールリンケージを識別する必要があります。そのため、モジュールに所有されている名前のマングル方法にはいくつかの選択肢が生まれます。そして、現在の実装では次の2つが存在しています。

1. モジュールリンケージを持つ名前のマングルを変更して、それだけをリンカが特別扱いする
2. エクスポートされている（外部リンケージをもつ）名前のマングルを変更したうえで、リンカはモジュールを認識することでリンケージを処理する

1つ目の方法では、エクスポートされている名前は既存の外部リンケージを持つ名前と同じマングルとすることができ、それ以外はこれまで通りです。また、モジュールリンケージの処理に関してもモジュールを構成する時だけ考慮したうえで、モジュールと他の翻訳単位をリンクするときは内部リンケージと同様に扱うことができ、リンカにかかる負担はかなり軽くなります。また、リンク時の扱いもこれまでと大きく変わらなさそうです。

2つ目の方法では、まずリンカはオブジェクトファイルがモジュールファイルなのかどうかを認識し、そこにある名前を見てエクスポートされている名前かどうかを識別します。この方法ではモジュール毎に名前を識別することができますが、コンパイラだけでなくリンカにもそれなりの変更が必要です。

そして、1つ目の方法はまさに弱い所有権モデルの実装であり、2つ目の方法は強い所有権モデルの実装です（他にも方法はあるでしょうが、現状はこの2つに1つです）。外部リンケージを持つ名前に対してマングル名で区別をつけないということは、リンク時に別々のモジュール（翻訳単位）からの同じ名前を区別できません。

名前マングリングの観点から見ると、所有権の強弱はモジュールリンケージを持つ名前とエクスポートされている名前のどちらのマングリング方法を変更するのか？という問題とみることもできます。そしておそらく、そのマングル方法とは単にマングル名にモジュール名とモジュールであることを示す文字列が入るだけでしょう。

### 実装の選択

現在のC++処理系の主要3実装は既にある程度モジュールを実装しています。そこでは、GCC\Clangは弱い所有権モデルを採用しており、MSVCは強い所有権モデルを採用しています。

例えばGCC/Clangで次のようなモジュールコードをコンパイルすると、外部リンケージを持つ名前はこれまで通りのマングルであるのに対して、内部リンケージとモジュールリンケージを持つ名前のマングル名にはモジュール名が含まれるようになっていることが分かります。また、モジュールリンケージと内部リンケージはマングル名レベルでは同じ扱いをされていることも分かります。

```cpp
export module moduleA;

// モジュールリンケージ
int func1(int a, int b) {
  return a + b;
}

// 外部リンケージ
export int func2(int a, int b) {
  return a + b;
}

// 内部リンケージ
static int func3(int a, int b) {
  return a + b;
}
```

- [モジュールコードとしてコンパイルした結果](https://godbolt.org/z/f9c3YW)
- [非モジュールコードとしてコンパイルした結果](https://godbolt.org/z/4svMEs)

[GCCのモジュール実装に関するWiki](https://gcc.gnu.org/wiki/cxx-modules)によれば、GCCとClangは互いの相互運用性のために連携してモジュールの実装を進めており、その結果として双方共に弱い所有権モデルを採用しているようです。

一方、[MSVCは単独ですでにC++モジュールの実装をいったん完了](https://devblogs.microsoft.com/cppblog/standard-c20-modules-support-with-msvc-in-visual-studio-2019-version-16-8/)しており、そこでは強い所有権モデルを採用していると明言されています。

この実装間の差異は、おそらくリンカの扱いから来ています。MSVCは1つの社内でリンカとコンパイラを開発しており、ターゲットはWindowsのみで、MSVCコンパイラは自社のリンカ（link.exe）以外を考慮する必要はありません。そしてそれはリンカ側から見ても同様です。そのため、リンカとコンパイラをより密に連携することができ、リンカに対して思い切った変更を加えることができます。

一方GCC/ClangはMSVCに比べるとリンカとコンパイラの開発体制には距離があります。また、それぞれ使われうるリンカは特定の一つではありません。例えば、GNU ld、lld、goldなどいくつかの実装があります。また、ClangとGCCはマングル名に関してお互いに強く互換性を意識しており、片方でコンパイルされたバイナリをもう片方でコンパイルされたバイナリとリンクすることができます（標準ライブラリ周りは別として）。そのため、モジュールの大部分を実装するコンパイラとしてはなるべくリンカに影響を与えないアプローチをとらざるを得なかったものと思われます。  
また、既存のABIを大きく変更しないという方針を取ったものでもあるようです。FFIで他言語から呼び出すときにも、モジュール前後で変更が必要ないはずです。

（この辺りのことは完全に私の想像なので正しくないかもしれません）

明確にこの所有権が問題となるコードを書くことはないと思いますが、結果としてこの問題にあたってしまうことはあり得ます。その時に、実装のこの差異はモジュールコードのポータビリティに問題を与えるかもしれません。

なお、GCC/Clangはモジュールを実装中なので、このことは最終的には変化する可能性があります（低いと思わますが）。

### 参考文献

- [P0142R0 A Module System for C++ (Revision 4)](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0142r0.pdf)
- [Standard C++20 Modules support with MSVC in Visual Studio 2019 version 16.8 - C++ Team Blog](https://devblogs.microsoft.com/cppblog/standard-c20-modules-support-with-msvc-in-visual-studio-2019-version-16-8/)
- [C++ Modules - GCC Wiki](https://gcc.gnu.org/wiki/cxx-modules)
- [What are the problems with modules in C++20? - reddit](https://www.reddit.com/r/cpp/comments/iukqju/what_are_the_problems_with_modules_in_c20/)