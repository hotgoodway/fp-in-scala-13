# fp-in-scala-13

外部作用とI/O

---

### 概要-1

#### IOモナドの利点

> 参照透過性を維持しつつ、I/O作用を持つ命令型プログラミングを純粋なプラグラムに埋め込むための直接的な方法を提供する

---

### 概要-2

#### 純粋関数を使ってエフェクトフルな処理の記述を生成するテクニック

> 生成された記述は、外部作用を実際に適用するために別のインタープリタによって実行される。
> 実質的には、命令型プログラミングのためのEDSLを組み立てて行く。

※ EDSL: Embedded Domain-Specific Language

#### 本章のGOAL

> EDSLの作成に必要なスキルを身につける

---

## 13.1 作用のリファクタリング

まずは、副作用を持つ単純な例から考えてみよう

---

#### 副作用を持つプログラム

```
case class Player(name: String, score: Int)

def contest(p1: Player, p2: Player): Unit =
  if (p1.score > p2.score)
    println(s"${p1.name} is the winner!")
  else if (p2.score > p1.score)
    println(s"${p2.name} is the winner!")
  else
    println("It's a draw.")
```

---

#### 勝者を割り出すロジックを抜き出すリファクタリング

```
def winner(p1: Player, p2: Player): Option[Player] =
  if (p1.score > p2.score) Some(p1)
  else if (p2.score > p1.score) Some(p2)
  else None

def contest(p1: Player, p2: Player): Unit = winner(p1, p2) match {
  case Some(Player(name, _)) => println(s"${name} is the winner!")
  case None => println("It's a draw.")
}
```

1. 純粋関数 winner が切り出さられた
2. contest関数の役割がまだ2つある

---

#### さらにリファクタリング

```
// どちらのメッセージが妥当であるかを判断する役割
def winnerMsg(p: Option[Player]): String = p map {
  case Play(name, _) => s"$name is the winner!"
} getOrElse "It's a draw."

// メッセージをコンソールに出力する役割
def contest(p1: Player, p2: Player): Unit =
  println(winnerMsg(winner(p1, p2)))
```

1. 副作用である println がプログラムの最も外側のレイヤにのみある
2. println の呼び出しの内側にあるのが純粋な式

---

#### 大規模で複雑なプログラムにも同じ原理が当てはまる

> 副作用を持つ関数の中には必ず純粋関数として抽出部分がある

```
// 非純粋関数 f
A => B

// 分割後
A => D // 純粋関数
D => B // 非純粋関数

```

非純粋関数をプログラムの純粋なコアの「命令殻」と呼ぶ

---

## 13.2 単純なIO型

IOという名前の新しいデータ型を導入すれば、同じようにリファクタリングができる

#### 導入後

```
trait IO { def run: Unit }

def PrintLine(msg: String): IO =
  new IO { def run = println(msg) }

def contest(p1: Player, p2: Player): IO =
  PrintLine(winnerMsg(winner(p1, p2)))
```

1. contest が純粋関数になった。IO型の値を返しますが、実行はしない。
2. 副作用を発生させるのは IOのインタープリタ、runメソッドだけ。
3. contest の役割はただ1つ、プログラムの各部分を合成すること。

---

#### 参照透過性以外の価値

```
trait IO { self =>
  def run: Unit
  def ++(io: IO): IO = new IO {
    def run = { self.run; io.run }
  }
  object IO {
    def empty: IO = new IO { def run = () }
  }
}
```

Monoidを形成すること
1. 副作用の発生タイミングを遅らせる
2. 各部分が自由に変えられる(言語設計の醍醐味)

---

## 13.2.1 入力作用の処理

IO型が表現できるのは「出力」作用だけで、入力が必要な場合はどうするの

#### 華氏気温から摂氏気温へ変換のプログラム

```
def fahrenheitToCelsius(f: Double): Double =
  (f - 32) * 5.0 / 9.0

def converter: Unit = {
  println("Enter a temperature in degrees Fahrenheit: ")
  val d = readLine.toDouble
  println(fahrenheitToCelsius(d))
}
```

---

#### IOを返す純粋関数を書いてみると

```
def fahrenheitToCelsius(f: Double): Double =
  (f - 32) * 5.0 / 9.0

def converter: IO = {
  val prompt: IO = PrintLine("Enter a temperature in degrees Fahrenheit: ")
  // さて、どうしたものか?
  // readLine の呼び出しをIOで包み込むことができたが、結果を取っておく場所はない。
}
```

問題点: 有効な型の値が得られる計算を表現できない

--- 

#### 入力を許可するために型パラメータを導入

```
sealed trait IO[A] { self =>
  def run: A
  def map[B](f: A => B): IO[B] =
    new IO[B] { def run = f(self.run) }
  def flatMap[B](f: A => IO[B]): IO[B] =
    new IO[B] { def run = f(self.run).run }
}
```

map 関数と flatMap 関数を追加することで、
IO を for 内包表記で使用できるようになった。

---

#### IOモナド

IO はモナドを形成するにようになる

```
object IO extends Monad[IO] {
  def unit[A](a: => A): IO[A] = new IO[A] { def run = a }
  def flatMap[A, B](fa: IO[A])(f: A => IO[B]) = fa flatMap f
  def apply[A](a: => A): IO[A] = unit(a)
}
```

---

#### 再び converter の例を書いてみると

```
def ReadLine: IO[String] = IO { readLine }
def PrintLine(msg: String): IO[Unit] = IO { println(msg) }

def converter: IO[Unit] = for {
  _ <- PrintLine("Enter a temperature in degrees Fahrenheit: ")
  d <- ReadLine.map(_.toDouble)
  _ <- PrintLine(fahrenheitToCelsius(d).toString)
} yield ()
```

1. converter の定義から副作用がなくなって、参照透過な記述である
2. converter.run は作用を実際に実行するインタープリタ
3. モナドコンビネータのどれでも利用できる
例: sequence, traverse, replicateM

---

#### 他のユースケース

```
// コンソールから1行を読み取り、それを表示するIO[Unit]
val echo = ReadLine.flatMap(PrintLine)

// コンソールから1行を読み取ることでIntを解析するIO[Int]
val readInt = ReadLine.map(_.toInt)

// コンソールから2行を読み取ることで(Int, Int)を解析するIO[(Int, Int)]
// a ** b は map2(a, b)((_, _))と同じである
val readInts = readInt ** readInt

// コンソールから10行を読み取り、結果をリストで返すIO[List[String]]
// replicateM(3)(fa) が sequence(List(fa, fa, fa))と同じである
replicateM(10)(ReadLine)
```

---

#### 対話形式のプログラム例

ループの中でユーザに入力を求め、入力された値の階乗を計算する。

```
The Amazing Factorial REPL, v2.0
q -quit
<number> - compute the factorial of the given number
<anything else> - crash spectacularly
3
factorial: 6
7
factorial: 5040
q 
```

---

#### IOを使った結果

```
def factorial(n: Int): IO[Int] = for {
  acc <- ref(1)
  _ <- foreachM(1 to n toStream) (i => acc.modify(_ * i).skip)
  result <- acc.get
} yield result

val factorialREPL: IO[Unit] = sequence_(
  IO { println(helpstring) },
  doWhile { IO { readLine } } { line =>
    val ok = line != "q"
    when (ok) { for {
      n <- factorial(line.toInt)
      _ <- IO { println("factorial: " + n) }
    } yield () }
  }
)
```

---

#### 新たなモナドコンビネータ-1

```
// cond関数が trueを返す限り、1つ目の引数の作用を繰り返す。
def doWhile[A](a: F[A])(cond: A => F[Boolean]): F[Unit] = for {
  a1 <- a
  ok <- cond(a1)
  _ <- if (ok) doWhile(a)(conf) else unit(())
}

// 引数の作用を無限に繰り返す。
def forever[A, B](a: F[A]): F[B] = {
  lazy val t: F[B] = forever(a)
  a flatMap (_ => t)
}
```

---

#### 新たなモナドコンビネータ-2

```
// ストリームを関数fで畳み込み、作用を結合し、その結果を返す。
def foldM[A, B](l: Stream[A])(z: B)(f: (B, A) => F[B]): F[B] =
  l match {
    case h #:: t => f(z, h) flatMap (z2 => foldM(t)(z2)(f))
    case _ => unit(z)
  }

// foldM関数と同じだが、結果を無視する
def foldM_[A, B](l: Stream[A])(z: B)(f: (B, A) => F[B]): F[Unit] =
  skip { foldM(l)(z)(f) }

// ストリームの要素ごとにf関数を呼び出し、作用を結合する。
def foreachM[A](l: Stream[A])(f: A => F[Unit]): F[Unit] =
  foldM_(1)(())((u, a) => skip(f(a)))
```

命令型プログラムをそのままIOモナドに埋め込んだとしても、全てのプログラムを純粋関数方式で表現することは可能。

---

## 13.2.2 単純なIO型の長所と短所

#### 長所(メリット)

1. IOの計算は通常の値であり、リストに格納したり、関数に渡したり、動的に作成したりできる。
共通のパターンは関数にまとめて再利用できる。
2. IOの計算を値として具体化すると、IO型に組み込まれた単純なrunメソッドよりも興味深いインタープリタを作成できる。
つまり、IOの表現は完全にIOインタープリタの実装上の詳細であり、プログラマの目に触れることはない。

#### 短所(問題点)

1. 多くのIOプログラムは、ランタイムコールスタックをオーバーフローさせ、StackOverflowErrorを発生させる。
factorialREPLプログラムに数字を入力し続けると、最終的にスタックオーバーフローが発生する。
2. IO[A]型の値は完全に不透明である。一般的にすぎる。
3. 今の単純なIO型には、並列処理や非同期処理のことは何も定義されていないので実用性が低い。
