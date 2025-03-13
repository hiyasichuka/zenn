---
title: "(初心者向け)Pythonで学ぶオブジェクト指向プログラミング OOP"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python","oop","begineer"]
published: true
---

# オブジェクト指向プログラミング（OOP）とは？

オブジェクト指向プログラミング（OOP）は、ソフトウェア開発における設計思想の一つで、データとそれに関連する操作を「オブジェクト」としてまとめ、プログラムを作る手法です。​これにより、**コードの再利用性や保守性が向上し、複雑なシステムでも効率的に開発・管理することが可能**となります。

# なぜオブジェクト指向が難しいのか

概念としては知っていても、その**具体的な活用方法やメリットを理解できていないパターン**が挙げられます。
特に、従来の手続き型プログラミング（上から順番に実行して構築する方法）に慣れている人にとって、オブジェクト指向の考え方は抽象的で難解に感じると思います。​
また概念を理解していても、実際のプログラミングにおいて**どのように適用すれば良いか分からない、または適切に設計できない**場合があると思います。
本記事は、こうした**難しく感じるプログラミング初心者を救うきっかけ**になればと思って書きます。

# オブジェクト指向を学ぶステップ

オブジェクト指向を効果的に習得するためには、以下の順序で学習を進めることが良さそうと思います。
オブジェクト指向と密接に関わる**ドメイン駆動の実践は、挫折する恐れがある**ため、一番最後でいいと思います。
~~異論はガチで認めます。~~

1. **基本的なコーディングスキルの習得**：​条件分岐やループ、配列処理、標準入出力など、プログラミングの基礎を習得する。​
2. **オブジェクト指向の学習**：​クラスやオブジェクトの概念、カプセル化、継承、ポリモーフィズムなどの基本原則を学ぶ。​
3. **デザインパターンの学習**：​過去の偉人たちが編み出したベストプラクティス（デザインパターン）を学び、再利用性の高いコードを書く技術を身につける。​
4. **テスト駆動開発（TDD）の実践**：​テストを先に書き、そのテストをパスするコードを書くことで、バグの少ない堅牢なソフトウェアを開発する手法を習得する。​
5. **ドメイン駆動開発（DDD）の学習**：​ビジネスドメイン（業務領域）の知識をソフトウェア設計に反映させることで、現実の問題に即した柔軟なシステムを構築する技術を学ぶ。


# オブジェクト指向プログラミングを理解するための3つの重要な概念

オブジェクト指向プログラミングを実践する上で、次の3つの重要な概念が存在します。
以下の3つは特に重要なので、ぜひ空で言えるレベルで覚えておきましょう。

1. カプセル化（Encapsulation）
1. ポリモーフィズム（多様性）（Polymorphism）
1. 継承（Inheritance）

それぞれの概念を簡単に解説します。

## カプセル化（Encapsulation）
データとそれに関連する操作を一つの単位（クラス）にまとめ、**外部から直接アクセスできないようにする**ことで、データを安全に保護します。

Pythonにおいて完全に外部からのアクセスを防ぐ方法は存在せず、ダブルアンダースコア（__変数名）を使った名前マングリング（外部からアクセスしにくくする仕組み）を利用します。

## ポリモーフィズム（Polymorphism）
ポリモーフィズムとは、「同じメソッドを持った複数の異なるクラスが、それぞれ異なる動作をする」という特性です。
これを実現するための具体的な方法として「インターフェース（Pythonでは抽象基底クラス）」を使用します。

インターフェースは、クラスが実装すべきメソッド群を事前に定義し、異なるクラスでも共通の操作を持つことを保証します。

## 継承（Inheritance）
既存のクラスの変数やメソッドを新しいクラスに引き継ぎ、再利用性を高めます。ただし、継承の多用はコードの複雑化を招く可能性があるため、本当に必要な場合のみ利用しましょう。

以下では、カプセル化、インターフェース、継承について詳しく解説し、それぞれのPythonの実装例を紹介します。

---

## カプセル化（Encapsulation）

### カプセル化とは？

改めて一言でいうと、
カプセル化とは、データ（属性）を外部から直接変更できないようにし、特定のメソッド（操作）を通じてのみアクセスできるようにする仕組みです。

- **メリット**
  - データの誤操作を防ぐ（適切なメソッドを通してのみ変更可能）
  - クラスの内部構造を隠蔽し、変更の影響を最小限に抑える

### Pythonでのカプセル化の実装

```python
class BankAccount:
    def __init__(self, balance):
        self.__balance = balance  # ダブルアンダースコアで名前マングリング（外部からのアクセスを抑制）

    def deposit(self, amount):
        if amount > 0:
            self.__balance += amount
            return f"Deposited {amount}. New balance: {self.__balance}"
        return "Invalid amount"

    def withdraw(self, amount):
        if 0 < amount <= self.__balance:
            self.__balance -= amount
            return f"Withdrew {amount}. Remaining balance: {self.__balance}"
        return "Insufficient funds"

    def get_balance(self):
        return self.__balance

# インスタンス化とテスト
account = BankAccount(1000)
print(account.deposit(500))        # Deposited 500. New balance: 1500
print(account.withdraw(200))       # Withdrew 200. Remaining balance: 1300
print(account.get_balance())       # 1300

# 名前マングリングでアクセス可能（非推奨）
print(account._BankAccount__balance)  # 1300（直接アクセスは避けること）

```

**ポイント**:
- `__balance` はPythonの名前マングリングにより `account._BankAccount__balance` のように変換される。
- 直接アクセスは推奨されず、`get_balance()` などのメソッド経由でアクセスするのが適切。

---

## インターフェース（Interface）

### インターフェースとは？

インターフェースとは、クラスが実装すべきメソッド群を定義するものです。
Pythonには明示的なインターフェースはありませんが、抽象クラス（`ABC`）を使って実装を強制することができます。

### Pythonでのインターフェース（抽象基底クラス）

Pythonでは、インターフェースという専用の仕組みはなく、代わりに抽象基底クラス（ABC: Abstract Base Class）を使って実装を強制します。

```python
from abc import ABC, abstractmethod

class Animal(ABC):
    @abstractmethod
    def speak(self):
        pass

class Dog(Animal):
    def speak(self):
        return "Woof!"

class Cat(Animal):
    def speak(self):
        return "Meow!"

# ポリモーフィズムの実践例
animals = [Dog(), Cat()]
for animal in animals:
    print(animal.speak())
    # Woof!
    # Meow!

```

**ポイント**:
- `ABC` クラスを継承し、`@abstractmethod` を使うことで、
  具象クラス（`Dog`, `Cat`）が `speak()` メソッドを必ず実装することを強制できる。

---

## 継承（Inheritance）

### 継承とは？

継承とは、あるクラス（親クラス）の機能を別のクラス（子クラス）が引き継ぐ仕組みです。

- **メリット**
  - 共通の機能を親クラスにまとめることで、コードの重複を減らせる。
  - 子クラスで追加の機能を拡張できる。

### Pythonでの継承の実装

```python
class Vehicle:
    def __init__(self, brand, model):
        self.brand = brand
        self.model = model

    def drive(self):
        return f"{self.brand} {self.model} is driving."

class Car(Vehicle):
    def __init__(self, brand, model, doors):
        super().__init__(brand, model)
        self.doors = doors

    def honk(self):
        return "Beep beep!"

# インスタンス化とテスト
car = Car("Toyota", "Corolla", 4)
print(car.drive())  # → Toyota Corolla is driving.
print(car.honk())   # → Beep beep!
```

**ポイント**:
- `Vehicle` クラスの `drive()` メソッドを `Car` クラスが継承。
- `Car` クラスは `honk()` という独自のメソッドを追加。
- `super().__init__()` を使って親クラスのコンストラクタを呼び出し。
- 継承の多用は避け、まずはインタフェースを活用することを推奨します。
---

# まとめ

| 概念 | 説明 | Pythonでの実装例 |
|------|------|----------------|
| **カプセル化** | データを外部から直接変更できないように保護する仕組み（名前マングリングを利用） | `__変数名` を使用し、メソッドでアクセス |
| **インターフェース（抽象クラス）** | 同一メソッドを複数クラスが異なる動作で実装（抽象基底クラスで実現） | `ABC` クラスと `@abstractmethod` を使用 |
| **継承** |親クラスの機能を子クラスが引き継ぎ、コードを再利 | `super().__init__()` で親クラスを呼び出す |


オブジェクト指向プログラミング（OOP）は、複雑なアプリケーションを整理しやすくする重要な設計手法です。
OOPの導入部分だけ本記事では詳しく説明させていただきました。

# 参考

[Python公式ドキュメント：抽象基底クラス](https://docs.python.org/ja/3/library/abc.html)
[Python公式FAQ: 名前マングリング](https://docs.python.org/ja/3/tutorial/classes.html#private-variables)