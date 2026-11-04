---
title: "pgregory.net/rapidで学ぶプロパティベーステスト"
emoji: "🎲"
type: "tech"
topics: ["Go", "テスト", "rapid", "プロパティベーステスト"]
published: true
---

最近業務でpgregory.net/rapidとPBTに初めて触れる機会があったので、これらについて調べてみました。AIによってPBTが再発見されつつありますが、PBT自体は古くからあるテスト手法です。私自身はGoで開発者人生をスタートしたこともあり、テストといえばテーブル駆動テストやtestifyなどのライブラリを使ったassertionによるものがメインでした。なのでPBTについては初めて知ることが多かったです。

この記事は、同じようにassertionによるテストを中心に書いてきた人に向けて、調べた内容をまとめたものです。ざっくりPBTとは何かが理解できる、Goでrapidを使ってPBTを書き始められる、あたりをゴールにおいています。

背景の説明は、AIを使ってなるべくファクトを集め、論文や公式ドキュメントを引用する形で書いています。引用元は脚注にまとめていますが、すべての引用元との整合性を私自身がチェックしたわけではないので、間違いがあればご指摘ください。

:::message
記事中のコードは Go 1.27 / rapid v1.3.0 で実行して確認しています。テスト出力のシード値や実行時間は実行するたびに変わります。
:::

## プロパティベーステストとは

普段書いているテストは「この入力を入れたらこの出力が返る」という具体例を並べるものです。英語ではexample-based testingと呼ばれます。日本語では定まった呼び方がないようなので、この記事では「例示テスト」と呼ぶことにします。Goでよく使われるテーブル駆動テストもこの書き方です。

プロパティベーステスト（以下PBT）は具体例の代わりに「**どんな入力に対しても成り立つ性質**」を書きます。この性質をプロパティと呼びます。入力はライブラリがランダムに大量に作り、プロパティを満たさない入力（反例）を探しにいきます。Pythonの代表的なPBTライブラリであるHypothesisの開発者たちは、論文[^hypothesis]でこう説明しています。

> Instead of tests that provide examples of a single concrete behaviour, tests specify properties that hold for a wide range of inputs, which the testing library then attempts to generate test cases to refute.
>
> 単一の具体的な振る舞いの例を示すテストの代わりに幅広い入力で成り立つプロパティをテストに書く。テストライブラリはそのプロパティを反証するテストケースを生成しようとする

元祖はHaskellのQuickCheck（2000年）です。QuickCheckの論文では、自分たちがやったことを「古いアイデア2つを組み合わせて使いやすくしただけ」と書いています[^quickcheck]。

> We have taken two relatively old ideas, namely specifications as oracles and random testing, and found ways to make them easily available to Haskell programmers.
>
> 我々は比較的古い2つのアイデア、つまり「オラクルとしての仕様」と「ランダムテスト」をHaskellプログラマが手軽に使えるようにした

オラクルは、**テストの結果が正しいかどうかを決める判定基準**のことです。ソート関数で考えてみます。

- 例示テストでは「`Sort([3, 1, 2])` の結果は `[1, 2, 3]` のはず」と期待値を書き、実際の出力と一致するかで合否を決めます。この期待値がオラクルです
- PBTでは期待値を書かず、「どんな入力でも結果は小さい順に並んでいるはず」というプロパティを満たすかで合否を決めます。こちらではプロパティがオラクルの役割をします

期待値は入力ごとに書く必要がありますが、プロパティは1つ書けばどんな入力にも使えます。だからPBTは入力をランダムに大量に作れるわけです。

PBTのライブラリはだいたい次の3つの部品でできています。

| 部品 | 役割 |
|---|---|
| ジェネレータ | ランダムな入力を作る |
| プロパティ | 入力に対して成り立つべき条件を確かめる |
| シュリンク | 反例を失敗したままできるだけ小さな形に縮める |

3つ目のシュリンクは「最小化」とも呼ばれます。ランダムに作った入力は大きくてごちゃごちゃしているので、シュリンクせずにそのまま見せられても何が原因か分かりません。QuickCheckの作者のJohn Hughesはシュリンクを「ノイズから信号を取り出すこと」と呼んでいます[^hughes2016]。

> Random tests contain a great deal of junk—that is their purpose! … Debugging a test that is 90% irrelevant is a nightmare
>
> ランダムテストはゴミだらけだ。それが目的なのだから！ …9割が無関係なテストをデバッグするのは悪夢だ

## 例示テストとPBTは何を確かめているのか

具体例で比べてみます。金額を `n` 人で割り勘する関数です。端数の扱いは割り勘アプリによってまちまちですが、ここでは**余った端数は1人にまとめず、1円ずつ別々の人に割り振る**という仕様にします。

```go:split/split.go
package split

// Split は金額 total を n 人で割り勘したときの各人の負担額を返す。
func Split(total, n int) []int {
	base := total / n
	shares := make([]int, n)
	for i := range shares {
		shares[i] = base
	}
	// 割り切れなかった分は最後の人に寄せる
	shares[n-1] += total % n
	return shares
}
```

これにテーブル駆動テストを書きます。

```go:split/split_example_test.go
package split

import (
	"slices"
	"testing"
)

func TestSplit(t *testing.T) {
	tests := []struct {
		name  string
		total int
		n     int
		want  []int
	}{
		{"割り切れる", 900, 3, []int{300, 300, 300}},
		{"1円余る", 1000, 3, []int{333, 333, 334}},
		{"1人", 500, 1, []int{500}},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := Split(tt.total, tt.n); !slices.Equal(got, tt.want) {
				t.Errorf("Split(%d, %d) = %v, want %v", tt.total, tt.n, got, tt.want)
			}
		})
	}
}
```

```text
--- PASS: TestSplit (0.00s)
    --- PASS: TestSplit/割り切れる (0.00s)
    --- PASS: TestSplit/1円余る (0.00s)
    --- PASS: TestSplit/1人 (0.00s)
```

全部通ります。レビューでも普通に通りそうなテストです。

次に同じ関数をPBTでテストします。コードはこの後で紹介するGoのPBTライブラリ rapid で書いています。書き方は後の章で説明するので、ここでは何を確かめているかだけ見てください（[rapidの基本的な使い方](#rapid%E3%81%AE%E5%9F%BA%E6%9C%AC%E7%9A%84%E3%81%AA%E4%BD%BF%E3%81%84%E6%96%B9)）。割り勘が満たすべきプロパティを考えると次の3つが出てきます。

- 人数分に分かれている
- 合計が元の金額に一致する
- 一番多く払う人と一番少ない人の差は1円以内

3つ目は最初に決めた端数の仕様から出てきます。余りは1円とは限らず、3人なら最大2円、4人なら最大3円余ります。それでも余りを1円ずつ別々の人に割り振るので、1人が多く払うのは1円までです。たとえば1001円を3人で割ると2円余り、負担額は `[334 334 333]` になります。端数を幹事がまとめて払う仕様なら3つ目は成り立ちません。プロパティは実装ではなく仕様から作るものだということです。

```go:split/split_prop_test.go
package split

import (
	"slices"
	"testing"

	"pgregory.net/rapid"
)

func TestSplitProperties(t *testing.T) {
	// rapid.Check は渡した関数を、入力を変えながら何度も（デフォルト100回）実行する
	rapid.Check(t, func(t *rapid.T) {
		// IntRange(min, max) は min 以上 max 以下の整数を作るジェネレータ。
		// Draw でそこから値を1つ取り出す。第2引数は失敗時の出力に表示される名前
		total := rapid.IntRange(0, 1_000_000).Draw(t, "total")
		n := rapid.IntRange(1, 100).Draw(t, "n")

		shares := Split(total, n)

		// ここから下でプロパティを確かめる。満たしていなければ t.Fatalf で失敗させる
		if len(shares) != n {
			t.Fatalf("人数分に分かれていない: len=%d, n=%d", len(shares), n)
		}
		sum := 0
		for _, s := range shares {
			sum += s
		}
		if sum != total {
			t.Fatalf("合計が合わない: sum=%d, total=%d", sum, total)
		}
		if slices.Max(shares)-slices.Min(shares) > 1 {
			t.Fatalf("負担額の差が1円を超えた: %v", shares)
		}
	})
}
```

```text
--- FAIL: TestSplitProperties (0.01s)
    split_prop_test.go:12: [rapid] failed after 1 tests: 負担額の差が1円を超えた: [0 0 2]
        To reproduce, specify -run="TestSplitProperties" -rapid.failfile="testdata/rapid/TestSplitProperties/TestSplitProperties-20261102235700-47719.fail" (or -rapid.seed=7779014724888409339)
        Failed test output:
    split_prop_test.go:15: [rapid] draw total: 2
    split_prop_test.go:16: [rapid] draw n: 3
    split_prop_test.go:32: 負担額の差が1円を超えた: [0 0 2]
```

落ちました。2円を3人で割ると `[0 0 2]` になり、最後の人だけ2円払っています。余りが2以上になると最後の人に全部寄るのがバグです。例示テストの「1円余る」ケースは余りがちょうど1だったのでたまたま通っていました。

rapidが見つけた入力は `total=2, n=3` というこれ以上小さくしようがない形になっています。最初に見つけた入力はもっと大きな数字だったはずですが、シュリンクでここまで縮めてくれています。

:::details シュリンクの途中経過を見てみる
テストの中で毎回の入力を `fmt.Printf` で出力し、シュリンクの途中経過を追ってみました。約1000回の試行のうち、失敗する一番小さい入力が更新されたところだけを抜き出すとこうなります。

```text
try    1: total=    219 n= 12 -> 失敗   ← 最初に見つけた反例
try   12: total=     11 n= 12 -> 失敗
try   14: total=      3 n= 12 -> 失敗
try  169: total=      2 n= 12 -> 失敗
try  178: total=      2 n=  4 -> 失敗
try  354: total=      2 n=  3 -> 失敗   ← これ以上縮まない
```

残りの約9割は `total=1` や `total=2, n=2` のように、小さくしすぎてテストが通ってしまった入力です。rapidはそれを捨てながら、失敗する範囲で一番小さい入力を探していきます。どうやって縮めているかは後の章で説明します（[シュリンクの仕組み](#%E3%82%B7%E3%83%A5%E3%83%AA%E3%83%B3%E3%82%AF%E3%81%AE%E4%BB%95%E7%B5%84%E3%81%BF)）。
:::

余りを先頭から1円ずつ配るように直します。

```diff go:split/split.go
 func Split(total, n int) []int {
-	base := total / n
+	base, rem := total/n, total%n
 	shares := make([]int, n)
 	for i := range shares {
 		shares[i] = base
+		// 割り切れなかった分は先頭から1円ずつ配る
+		if i < rem {
+			shares[i]++
+		}
 	}
-	// 割り切れなかった分は最後の人に寄せる
-	shares[n-1] += total % n
 	return shares
 }
```

```text
=== RUN   TestSplitProperties
    split_prop_test.go:12: [rapid] OK, passed 100 tests (197.458µs)
--- PASS: TestSplitProperties (0.00s)
```

PBTは通りました。ところが今度はさっきまで通っていた例示テストが落ちます。

```text
--- FAIL: TestSplit (0.00s)
    --- FAIL: TestSplit/1円余る (0.00s)
        split_example_test.go:22: Split(1000, 3) = [334 333 333], want [333 333 334]
```

余りの1円を払うのが「最後の人」から「最初の人」に変わったからです。これはバグでしょうか？

PBTのプロパティは「誰が1円多く払うか」について何も言っていません。一方で例示テストは「最後の人が払う」と決め打ちしています。これが仕様として決まっていたことなのか、たまたま最初の実装がそうだっただけなのかはテストコードを見ても分かりません。この話は後半でまた出てきます（[全部PBTにすればいい？](#%E5%85%A8%E9%83%A8pbt%E3%81%AB%E3%81%99%E3%82%8C%E3%81%B0%E3%81%84%E3%81%84%EF%BC%9F)）。

ここまでの違いをまとめるとこうなります。

| | 例示テスト | PBT |
|---|---|---|
| 書くもの | 入力と期待する出力の組 | 入力の範囲と成り立つべきプロパティ |
| 確かめていること | 書いた点で出力が期待どおりか | 範囲内のどの入力でもプロパティが崩れないか |
| 得意なこと | 「この場合はこうする」という具体的な決めごと | 不変条件、往復変換、別実装との一致 |
| 見落としやすいもの | 書いた人が思いつかなかった入力 | プロパティとして書かなかった決めごと |

Hypothesisの作者のDavid MacIverは例示テストについてこう書いています[^maciver]。

> This is the fundamental problem of example-based testing: We often treat our tests as specifications, but in reality they are stories.
>
> これが例示テストの根本的な問題だ。我々はテストを仕様として扱いがちだが、実際にはテストは「お話」にすぎない

`[333 333 334]` というテストは「1000円を3人で割ったらこうなった」というお話で、「どんな金額でも差は1円以内」という仕様までは言っていないということですね。

## なぜAI時代にPBTなのか

### AIが書いたテストは実装をなぞりやすい

AIに「テスト書いて」と頼むとだいたい全部グリーンで返ってきます。ただ、実装を読んで実装が返す値をそのまま期待値に書いたテストは「今こう動いている」ことしか確かめていません。実装にバグがあればテストもそのバグを正解として覚えます。

これには研究の裏付けがあります。LLMが生成したテストのオラクル（テストの正しさの判定基準）を調べた研究[^konstantinou]は、まず従来のテスト自動生成ツールについて次のように指摘しています。

> Traditional test generation tools (e.g. Randoop and Evosuite) often produce oracles that mirror the program's actual behavior rather than the expected one
>
> 従来のテスト生成ツールは、期待される振る舞いではなくプログラムの実際の振る舞いを写したオラクルを作りがちだ

そのうえで、LLMによるテスト生成も同じ傾向だったと結論づけています。

> Our findings show that LLM-based test generation approaches mainly capture the actual program behavior making bug detection difficult.
>
> LLMベースのテスト生成は主にプログラムの実際の振る舞いを捉えるため、バグの検出が難しくなる

前の章の割り勘関数を思い出してください。余りを全部最後の人に寄せるバグがありましたが、1000円を3人で割ると `[333 333 334]` が返ってきます。この実装を読んだAIが `{"1円余る", 1000, 3, []int{333, 333, 334}}` というテストを書くのは自然な流れで、このテストはバグがあっても通ってしまいます。

PBTの元祖であるQuickCheckを作ったJohn Hughesは、QuickCheckを長年使ってきた経験から次のように書いています[^hughes2016]。

> A key insight from using QuickCheck is that we find bugs by comparing two independent descriptions of the desired behaviour—the implementation, and the specification—and it is the inconsistencies between the two that reveal errors.
>
> QuickCheckを使って得た重要な知見は、望ましい振る舞いについての2つの独立した記述、つまり実装と仕様を突き合わせることでバグが見つかるということだ。バグを明らかにするのは両者の食い違いだ

ポイントは「独立した」です。**実装から期待値を写したテストは実装と独立していないので、食い違いが原理的に起きません**。LLMのコード修正にPBTを使う研究[^pgs]はこの状態を「自己欺瞞のサイクル」と呼んでいます。

> These properties are often simpler to define and verify than directly predicting exhaustive test oracles, breaking the "cycle of self-deception" where tests might share flaws with the code they are meant to validate.
>
> プロパティは網羅的なテストオラクルを直接予測するより定義も検証も簡単なことが多く、テストが検証対象のコードと同じ欠陥を共有してしまう「自己欺瞞のサイクル」を断ち切る

### AIはテストを通すことに寄りがち

もう1つ、AIにはテストを通すこと自体が目的になりやすいという問題があります。これはAnthropic自身が認めていて、Claude 3.7 Sonnetのシステムカードにこう書かれています[^syscard]。

> During our evaluations we noticed that Claude 3.7 Sonnet occasionally resorts to special-casing in order to pass test cases in agentic coding environments like Claude Code. Most often this takes the form of directly returning expected test values rather than implementing general solutions, but also includes modifying the problematic tests themselves to match the code's output.
>
> 評価の過程で、Claude 3.7 Sonnet が Claude Code のようなエージェント型のコーディング環境でテストを通すために特別扱いのコードを書くことがあると気づいた。多くは汎用的な解を実装せずにテストの期待値を直接返すという形をとるが、問題のあるテストのほうをコードの出力に合わせて書き換えることもある

公式のプロンプトガイドにも対策として「テストケースだけでなく、すべての有効な入力で正しく動く解を実装して」と指示するプロンプト例が載っています[^prompting]。

例示テストは入力が固定なので、期待値を直接返すような実装でも通ってしまいます。割り勘関数でいえば、テストにある「1000円を3人で割る」ときだけ決め打ちの値を返す `if total == 1000 && n == 3 { return []int{333, 333, 334} }` のようなコードです。PBTは実行のたびに入力が変わるので、こうした特別扱いではまず通りません。LLMが書いたコードの検証にPBTを使う研究も出てきていて[^pgs]、AIに実装を書かせるときのテストとしてPBTは相性がいいと言われています。

### PBTの一番の壁をAIが崩しつつある

ではなんで今までPBTがあまり使われてこなかったかというと、プロパティを考えるのが難しいからです。Jane Streetのエンジニア30人にインタビューした研究[^goldstein]ではPBTの強みと弱みをこう整理しています。

> PBT's main strengths lie in testing complex code and in increasing confidence beyond what is available through conventional testing methodologies, … Its main weaknesses, on the other hand, lie in the relative complexity of writing properties and random data generators and in the difficulty of evaluating their effectiveness.
>
> PBTの主な強みは、複雑なコードのテストと、従来のテスト手法で得られる以上の確信を得られることにある。一方で主な弱みは、プロパティやランダムデータ生成器を書くのが比較的複雑なことと、その有効性を評価しにくいことにある

参加者の1人はもっと端的に「一番よくある失敗は、何のプロパティをテストすればいいか分からないことだと思う」と言っています。

この壁を崩せることを示したのがAnthropicのFrontier Red Teamの研究です[^anthropic-blog]。Claude Code上にエージェントを作り、Pythonの主要ライブラリに対してHypothesisでPBTを書かせてバグを探させました[^agentic-pbt]。記事ではLLMについてこう書いています。

> We find that LLMs are particularly good at identifying properties that should be true about a given block of code from context (the name of the function, the docstring, how it is called by other functions, etc). This allows LLMs to write high quality property-based tests effectively.
>
> LLMは、関数名、docstring、他の関数からの呼ばれ方といった文脈から、そのコードで成り立つべきプロパティを見つけるのが特に得意だと分かった。これによりLLMは質の高いPBTを効率よく書ける

結果としてエージェントが出した984件のバグ報告のうち50件を人手で確認したところ56%が本物のバグでした。さらに、その確認をもとに作った採点基準でモデルに全件を採点させ、点数の高い報告に絞ると86%が本物でした。NumPyでは実際に修正がマージされています。

例示テストの限界についても同じ記事が分かりやすく書いています。

> After all, if a developer does not think to test an edge case, it is also likely the developer did not consider that case in the implementation!
>
> 結局のところ、開発者がテストしようと思いつかなかったエッジケースは実装でも考慮していない可能性が高い！

前の章の割り勘関数もそうでした。例示テストには余りがちょうど1円になるケースしかなく、実装も余りが2円以上になる場合（2円を3人で割るなど）を考えていませんでした。

### ハーネスの検証役としてのPBT

コーディングエージェントを使うとき、モデルそのもの以外の仕組み（指示の与え方、使えるツール、結果のチェック方法など）をまとめて「ハーネス」と呼び、これを設計することをハーネスエンジニアリングと呼ぶようになってきました。martinfowler.comに載ったBirgitta Böckelerの記事[^harness]は、ハーネスを次のように説明しています。

> The term harness has emerged as a shorthand to mean everything in an AI agent except the model itself.
>
> ハーネスという言葉は、AIエージェントのうちモデル自体を除くすべてを指す略語として使われるようになった

同じ記事では、エージェントが動いた後に結果を見て自己修正させる仕組みをセンサーと呼んでいます。テストやリンター、型チェッカーのように決定的で速いものは、変更のたびにエージェントと一緒に回せるセンサーです。Claude Codeのベストプラクティス[^best-practices]も、同じことをこう書いています。

> Give Claude something it can run that comes back pass or fail. Once it has that, it can keep going on its own and you don't have to watch.
>
> Claudeに、実行すると合否が返ってくるものを渡す。それがあればClaudeは自分で作業を進められるし、あなたが見張っている必要もない

ただしBöckelerは、AIが生成したテストが通っていてカバレッジが高いかを見るよくあるやり方について、こうも書いています。

> This approach puts a lot of faith into the AI-generated tests, that's not good enough yet.
>
> このやり方はAIが生成したテストを信用しすぎていて、まだ十分とは言えない

この章の最初で見た「AIが書いたテストは実装をなぞりやすい」と同じ問題です。

PBTはこのセンサーに向いています。プロパティは実装とは別に仕様から書くので、実装をなぞったテストになりにくく、入力が毎回変わるので通すための特別扱いも効きにくくなります。落ちたときはシュリンク済みの最小の反例が返ってきます。割り勘関数のときのように「2円を3人で割ると `[0 0 2]` になる」と最小の形で言われれば、人間でもAIでも直すべき場所はすぐ分かります。

ここまでをまとめると、AIにコードを書かせるときの役割分担はこうなると考えています。

- **プロパティ（何が成り立つべきか）は人間が決めるか、少なくともレビューする**
- 実装とプロパティを確かめるテストコードはAIに書かせてもいい
- プロパティを満たさない入力が見つかったら、最小の反例をAIに渡して直させる

## pgregory.net/rapidとは

[pgregory.net/rapid](https://github.com/flyingmutant/rapid) はGoでPBTを書くためのライブラリです。READMEには次のように書かれていて、PBTの章で引用したPythonのHypothesisを先例にしています。

> Rapid aims to bring to Go the power and convenience Hypothesis brings to Python.
>
> rapidは、HypothesisがPythonにもたらした力と手軽さをGoに持ち込むことを目指している

READMEに挙がっている特徴は次のとおりです。

- ジェネリクスで型安全にデータを生成する命令的なGoのAPI
- 「小さい」値やエッジケースに寄せたデータ生成
- 失敗したテストケースの完全自動の最小化
- 最小化した失敗ケースの保存と自動再実行
- ステートマシンテスト（モデルベーステスト）のサポート
- 標準ライブラリ以外への依存なし

「最小化」はPBTの章で説明したシュリンクのことです（[プロパティベーステストとは](#%E3%83%97%E3%83%AD%E3%83%91%E3%83%86%E3%82%A3%E3%83%99%E3%83%BC%E3%82%B9%E3%83%86%E3%82%B9%E3%83%88%E3%81%A8%E3%81%AF)）。ステートマシンテストは状態を持つものに操作の列を流して確かめるテストで、後の章で扱います（[具体例2: LRUキャッシュのテスト](#%E5%85%B7%E4%BD%93%E4%BE%8B2%3A-lru%E3%82%AD%E3%83%A3%E3%83%83%E3%82%B7%E3%83%A5%E3%81%AE%E3%83%86%E3%82%B9%E3%83%88)）。

### 他のライブラリとの比較

rapidはほかの選択肢と比べて、ジェネリクスで型安全に書けることや、失敗した入力を何もしなくても最小の形まで縮めてくれることなど、いいところが多いライブラリです。

GoでPBTを書く選択肢は標準ライブラリの `testing/quick`、[gopter](https://github.com/leanovate/gopter)、rapidの3つが主です。Go標準のファジング（`testing.F`）もランダムな入力でテストするという点では近い道具なので並べておきます。

| | testing/quick | gopter | rapid | testing.F（ファジング） |
|---|---|---|---|---|
| 提供元 | 標準ライブラリ | サードパーティ | サードパーティ | 標準ライブラリ |
| 先例 | — | ScalaCheck / QuickCheck | Hypothesis | — |
| 値の型 | 引数の型からリフレクションで生成 | `interface{}` ＋リフレクション | ジェネリクス | 基本型のみ（`[]byte`, `string`, 数値など） |
| 生成の制御 | 弱い（`Values` 関数で自作） | ジェネレータを組み合わせる | ジェネレータを組み合わせる | `f.Add` で渡したシードを少しずつ書き換えて広げていく |
| 失敗した入力の最小化 | なし | ジェネレータごとのシュリンカー | 自動 | あり |
| 状態を持つもののテスト | なし | `commands` パッケージ | `t.Repeat` | なし |
| 主な用途 | PBT（機能追加は凍結） | PBT | PBT | 長時間回してカバレッジを広げる |

`testing/quick` はドキュメントに機能追加を受け付けていないと書かれています[^quick]。

> The testing/quick package is frozen and is not accepting new features.
>
> testing/quickパッケージは凍結されており新しい機能は受け付けていない

rapidのREADMEは `testing/quick` との違いをこう説明しています。

> As for testing/quick, it lacks both convenient data generation facilities and any form of test case minimization, which are two main things to look for in a property-based testing library.
>
> testing/quickには便利なデータ生成の仕組みもテストケースの最小化も一切ない。どちらもPBTライブラリに求める二大要素だ

gopterはScalaCheckの流れを汲むライブラリで、シュリンクにも状態を持つテストにも対応しています。rapidとの大きな違いはシュリンクのやり方です。

gopterでは、ジェネレータごとに「その型の値をどう縮めるか」を決めた関数（シュリンカー）がセットになっています。たとえば整数のジェネレータには、数を小さくしていくシュリンカーが付いています。

PBTライブラリには、ジェネレータが作った値を関数で別の値に変換して、新しいジェネレータを作る `Map` という関数があります。たとえば「0〜100万の整数を作るジェネレータ」から「その整数を金額にした `Bill` 構造体を作るジェネレータ」を作れます。ところがgopterでは、`Map` で別の型に変換すると新しくできたジェネレータにはシュリンカーが付きません。gopterには、変換後の構造体をどう縮めればいいか分からないからです。

割り勘関数のバグ（余りが最後の1人に寄る）で試してみます。金額を作る整数のジェネレータを、`Map` で金額と人数の組（`Bill`）に変換しています。

```go
type Bill struct{ Total, N int }

func TestGopterMap(t *testing.T) {
	// 金額を作る整数のジェネレータを、Map で Bill に変換する
	bills := gen.IntRange(0, 1_000_000).Map(func(total int) Bill {
		return Bill{Total: total, N: 3}
	})
	properties := gopter.NewProperties(nil)
	properties.Property("割り勘は公平", prop.ForAll(
		// fair は割り勘のプロパティ3つを確かめる関数（中身は省略）
		func(b Bill) bool { return fair(b.Total, b.N) },
		bills,
	))
	properties.TestingRun(t)
}
```

```text
! 割り勘は公平: Falsified after 2 passed tests.
ARG_0: {Total:32027 N:3}
```

バグは見つけていますが、最初に見つけた `Total:32027` のまま縮んでいません。

一方rapidは、出来上がった値ではなく、値を作る元になった乱数を縮めます。乱数を縮めたら値をもう一度作り直すので、`Map` や `Custom` で組み立てた値でも、元の整数が小さくなれば構造体も小さくなります。仕組みは後の章で説明します（[シュリンクの仕組み](#%E3%82%B7%E3%83%A5%E3%83%AA%E3%83%B3%E3%82%AF%E3%81%AE%E4%BB%95%E7%B5%84%E3%81%BF)）。同じテストをrapidで書くとこうなります。

```go
func TestRapidMap(t *testing.T) {
	bills := rapid.Map(rapid.IntRange(0, 1_000_000), func(total int) Bill {
		return Bill{Total: total, N: 3}
	})
	rapid.Check(t, func(t *rapid.T) {
		b := bills.Draw(t, "bill")
		if !fair(b.Total, b.N) {
			t.Fatalf("不公平: %+v", b)
		}
	})
}
```

```text
--- FAIL: TestRapidMap (0.00s)
    map_test.go:32: [rapid] failed after 1 tests: 不公平: {Total:2 N:3}
```

`Map` で変換していても `Total:2` まで縮みます。

### ファジングとの違い

PBTとファジングは目的が違います。PBTは「合計が一致する」のような仕様どおりに動くかを、普段のテストと一緒に確かめるためのものです。ファジングは想定外の入力でpanicやクラッシュが起きないか、セキュリティ上の穴がないかを、時間をかけて探すためのものです。rapidのREADMEも次のように分けています。

> Property-based tests are used alongside regular example-based tests during development, and fuzzing is used to search for edge cases and security vulnerabilities.
>
> PBTは開発中に通常の例示テストと並べて使い、ファジングはエッジケースやセキュリティ上の脆弱性を探すのに使う

とはいえ、rapidのテストはそのままファジングにも使えます。`rapid.MakeFuzz` でラップすると、Go標準のファジングエンジン（`go test -fuzz` で動く、入力を作っては書き換えていく仕組み）が作るバイト列を、rapidが乱数の代わりに使って値を組み立てます。これで、rapidのジェネレータで組み立てる構造のある入力を、ファジングエンジンに時間をかけて探させられます。

```go
func FuzzSplit(f *testing.F) {
	f.Fuzz(rapid.MakeFuzz(func(t *rapid.T) {
		// rapid.Check に渡していたのと同じテストを書く
	}))
}
```

## rapidの基本的な使い方

### インストールと最小のテスト

```bash
go get pgregory.net/rapid
```

基本形はこれだけです。

```go
func TestXxx(t *testing.T) {
	rapid.Check(t, func(t *rapid.T) {
		x := rapid.Int().Draw(t, "x") // ジェネレータから値を1つ引く
		// x について成り立つべきプロパティを確かめる
		if !ok {
			t.Fatalf("...")
		}
	})
}
```

- `rapid.Check` が中の関数を何度も（デフォルト100回）実行する
- `Draw(t, "x")` でジェネレータから値を引く。第2引数のラベルは失敗時の出力に `draw x: ...` として出る
- `*rapid.T` は `*testing.T` とほぼ同じメソッドを持っていて `t.Fatalf` などで失敗させる。テスト中にpanicしても、rapidが内部で `recover` して失敗として扱い、その入力を縮めてくれる

rapidのテストも見た目はただの `func TestXxx(t *testing.T)` で、`rapid.Check` はその中で渡した関数を繰り返し呼んでいるだけです。専用のコマンドやビルドタグは要らないので、`go test` でほかのテストと一緒に実行され、CIの設定を変える必要もありません。

### シュリンクの仕組み

rapidはシュリンクのためのコードをユーザーに書かせません。READMEによると、`Draw` のたびに内部の乱数の列から値を作り、列のどこがどの値の元になったかを記録しています。失敗したらこの乱数の列を短く・小さくする方向に縮め、値を作り直してまだ失敗するかを試していきます。

### 失敗したときの出力と再現

例示テストと比べた章で、割り勘関数のバグを見つけたときの出力をもう一度見てみます。

```text
    split_prop_test.go:12: [rapid] failed after 1 tests: 負担額の差が1円を超えた: [0 0 2]
        To reproduce, specify -run="TestSplitProperties" -rapid.failfile="testdata/rapid/TestSplitProperties/TestSplitProperties-20261102235700-47719.fail" (or -rapid.seed=7779014724888409339)
```

失敗するとrapidは `testdata/rapid/<テスト名>/` に `.fail` ファイルを保存します。次に `go test` を実行するとランダムな入力より先にこのファイルの入力が再実行されます。**一度見つかったバグを直したかどうかは、何もしなくても次の実行で確かめられる**ということです。

:::details .fail ファイルの中身
割り勘のバグで保存された `.fail` ファイルはこうなっていました。

```text:testdata/rapid/TestSplitProperties/TestSplitProperties-20261103234204-94105.fail
# 2026/11/03 23:42:04.852458 [TestSplitProperties] [rapid] draw total: 2
# 2026/11/03 23:42:04.852480 [TestSplitProperties] [rapid] draw n: 3
# 2026/11/03 23:42:04.852481 [TestSplitProperties] 負担額の差が1円を超えた: [0 0 2]
# 
v0.4.8#7779014724888409339
0x0
0x2fc962fc962fd
0x2
0x0
0x38e38e38e38e4
0x2
```

- `#` で始まる行は、失敗したときのテスト出力のコピーです。人が読むためのメモで、読み込むときには無視されます
- `v0.4.8#...` の前半はファイル形式のバージョン（rapid自体のバージョンとは別）で、`#` の後ろはそのときのシードです
- `0x...` の行は、縮めた後の乱数の列です。rapidは乱数から値を作っているので、この列を同じ順に与えれば `total=2, n=3` がそのまま作り直されます

`total=2, n=3` という値そのものではなく、値の元になった乱数を保存しているので、`Custom` や `Map` で組み立てた構造体でも同じ形で保存できます。再実行してテストが通れば（バグが直っていれば）何も言わずに普段のランダムなテストに進みますが、ファイルは自動では消えません。
:::

| フラグ | 用途 |
|---|---|
| `-rapid.checks=10000` | 試行回数を増やす（デフォルト100） |
| `-rapid.seed=<数値>` | シードを固定して再現する |
| `-rapid.failfile=<パス>` | 保存された失敗ケースを指定して再現する |
| `-rapid.nofailfile` | `.fail` ファイルを書き出さない |
| `-rapid.v` | 詳細を出力する |

フラグには `RAPID_CHECKS=10000 go test ./...` のように環境変数版もあります。普段は100回で回して夜間のCIだけ回数を増やすといった使い方ができます。

:::message
`.fail` ファイルをgitにコミットするとチーム全員とCIで同じ失敗ケースが再実行されます。回帰テストとして残したい場合はコミットし、不要なら `.gitignore` に `testdata/rapid/` を入れておくとよいです。
:::

### よく使うジェネレータ

| ジェネレータ | 生成するもの |
|---|---|
| `rapid.Int()`, `rapid.IntRange(min, max)` | 整数（範囲指定あり） |
| `rapid.String()`, `rapid.StringMatching(re)` | 文字列（正規表現に一致するもの） |
| `rapid.SliceOf(g)`, `rapid.SliceOfN(g, min, max)` | スライス |
| `rapid.MapOf(k, v)` | マップ |
| `rapid.SampledFrom(xs)` | 候補の中から1つ |
| `rapid.OneOf(g1, g2, ...)` | 複数のジェネレータのどれか |
| `rapid.Map(g, fn)` | 生成した値を変換する |
| `rapid.Custom(fn)` | 自前の構造体などを組み立てる |
| `rapid.Make[T]()` | 型 `T` の値をリフレクションで自動生成する |

`Custom` は `Draw` を組み合わせて自分の型を作るときに使います。

```go
var userGen = rapid.Custom(func(t *rapid.T) User {
	return User{
		Name: rapid.StringMatching(`[a-z]{1,16}`).Draw(t, "name"),
		Age:  rapid.IntRange(0, 150).Draw(t, "age"),
	}
})
```

### 具体例1: ページング用カーソルのエンコード/デコード処理のテスト

ここからは、業務で書きそうなコードを模した例をrapidでテストしてみます。1つ目はAPIのページングで使うカーソルの変換、2つ目は状態を持つLRUキャッシュです。どちらもわざとバグを入れた実装から始めて、rapidがどう見つけるかを見ていきます。

まずはページングAPIでよくある、カーソル文字列のエンコード・デコードです。

```go:cursor/cursor.go
package cursor

import (
	"encoding/base64"
	"fmt"
	"strconv"
	"strings"
)

// Cursor はページングの続きを表す。並び替えキーの値と ID の組。
type Cursor struct {
	SortKey string
	ID      int64
}

func Encode(c Cursor) string {
	raw := c.SortKey + ":" + strconv.FormatInt(c.ID, 10)
	return base64.URLEncoding.EncodeToString([]byte(raw))
}

func Decode(s string) (Cursor, error) {
	b, err := base64.URLEncoding.DecodeString(s)
	if err != nil {
		return Cursor{}, fmt.Errorf("decode cursor: %w", err)
	}
	parts := strings.Split(string(b), ":")
	if len(parts) != 2 {
		return Cursor{}, fmt.Errorf("malformed cursor: %q", b)
	}
	id, err := strconv.ParseInt(parts[1], 10, 64)
	if err != nil {
		return Cursor{}, fmt.Errorf("parse id: %w", err)
	}
	return Cursor{SortKey: parts[0], ID: id}, nil
}
```

エンコードとデコードのような対になる処理は「エンコードしてデコードしたら元に戻る」というプロパティがそのまま書けます。PBTを書き始めるときに書きやすい形です。

```go:cursor/cursor_test.go
package cursor

import (
	"testing"

	"pgregory.net/rapid"
)

func TestCursorRoundTrip(t *testing.T) {
	rapid.Check(t, func(t *rapid.T) {
		c := Cursor{
			SortKey: rapid.String().Draw(t, "sortKey"),
			ID:      rapid.Int64().Draw(t, "id"),
		}

		got, err := Decode(Encode(c))
		if err != nil {
			t.Fatalf("Decode に失敗: %v", err)
		}
		if got != c {
			t.Fatalf("往復で値が変わった: got=%+v, want=%+v", got, c)
		}
	})
}
```

```text
--- FAIL: TestCursorRoundTrip (0.01s)
    cursor_test.go:10: [rapid] failed after 44 tests: Decode に失敗: malformed cursor: "::0"
        To reproduce, specify -run="TestCursorRoundTrip" -rapid.seed=4560337134544165175
        Failed test output:
    cursor_test.go:12: [rapid] draw sortKey: ":"
    cursor_test.go:13: [rapid] draw id: 0
    cursor_test.go:18: Decode に失敗: malformed cursor: "::0"
```

`SortKey` に `:` が含まれると壊れます。並び替えキーが時刻（`12:34:56`）やURLだったら普通に起こり得るパターンです。最小の反例が `sortKey: ":"`, `id: 0` まで縮められているので、原因が一目で分かります。

`ID` は数字だけなので、最後の `:` で分ければ直ります。

```diff go:cursor/cursor.go
-	parts := strings.Split(string(b), ":")
-	if len(parts) != 2 {
+	// SortKey 側に ":" が含まれうるので、最後の ":" で分ける
+	i := strings.LastIndex(string(b), ":")
+	if i < 0 {
 		return Cursor{}, fmt.Errorf("malformed cursor: %q", b)
 	}
-	id, err := strconv.ParseInt(parts[1], 10, 64)
+	id, err := strconv.ParseInt(string(b[i+1:]), 10, 64)
 	if err != nil {
 		return Cursor{}, fmt.Errorf("parse id: %w", err)
 	}
-	return Cursor{SortKey: parts[0], ID: id}, nil
+	return Cursor{SortKey: string(b[:i]), ID: id}, nil
```

バグは直りましたが、テスト側にもまだ工夫の余地があるので、2つテストを足してみます。

1つ目は書き方の工夫です。さっきのテストでは `Cursor{...}` のフィールドを1つずつ `Draw` して組み立てましたが、フィールドを全部ランダムに埋めるだけなら `rapid.Make` で構造体ごと作れます。2つ目は確かめる範囲の追加です。`Decode` は外から渡される文字列を受け取る関数なので、往復変換だけでなく「どんな文字列を渡してもpanicしない」ことも確かめておきます。

```go:cursor/cursor_test.go
func TestCursorRoundTripMake(t *testing.T) {
	rapid.Check(t, func(t *rapid.T) {
		c := rapid.Make[Cursor]().Draw(t, "cursor")
		got, err := Decode(Encode(c))
		if err != nil || got != c {
			t.Fatalf("got=%+v, err=%v, want=%+v", got, err, c)
		}
	})
}

func TestDecodeNeverPanics(t *testing.T) {
	rapid.Check(t, func(t *rapid.T) {
		s := rapid.String().Draw(t, "s")
		_, _ = Decode(s) // panic したら rapid が失敗として扱う
	})
}
```

試す回数を1万回に増やして（`-rapid.checks=10000`）実行すると、修正前からあった往復変換のテストも含めて3つとも通りました。

```text
$ go test -v ./cursor/ -rapid.checks=10000
    cursor_test.go:10: [rapid] OK, passed 10000 tests (24.551833ms)
--- PASS: TestCursorRoundTrip (0.02s)
    cursor_test.go:27: [rapid] OK, passed 10000 tests (22.524208ms)
--- PASS: TestCursorRoundTripMake (0.02s)
    cursor_test.go:37: [rapid] OK, passed 10000 tests (10.996125ms)
--- PASS: TestDecodeNeverPanics (0.01s)
```

1万回試しても、どれも数十ミリ秒で終わっています。

### 具体例2: LRUキャッシュのテスト

2つ目は、具体例1と違って状態を持つもののテストです。LRUキャッシュを例にします。

LRU（Least Recently Used）キャッシュは、入れておける件数（容量）に上限があり、いっぱいになったら「一番長く使われていないもの」から捨てていくキャッシュです。エディタの「最近開いたファイル」一覧をイメージすると分かりやすいです。開いたファイルが一番上に来て、件数の上限を超えると一番下（一番長く開いていないファイル）が消えていきます。

容量2のキャッシュなら、こう動きます。

1. `a` を入れる → 中身は `a`
2. `b` を入れる → 中身は `b, a`（左ほど最近使った）
3. `a` を取り出す → `a` を使ったので `a, b` の順になる
4. `c` を入れる → いっぱいなので、一番使われていない `b` を捨てて `c, a`

よく使うデータを手元に残しておきたいときの定番で、Goでは標準ライブラリの `container/list`（双方向リスト）とmapを組み合わせて書くのがよくある実装です。

```go:lru/lru.go
package lru

import "container/list"

// entry はリストの要素1つ分の中身。キーも持たせておくと、末尾から消すときに map からも消せる
type entry struct {
	key   string
	value int
}

// Cache は容量 capacity の LRU キャッシュ。
type Cache struct {
	capacity int
	ll       *list.List               // 使った順に並べた双方向リスト。先頭ほど最近使った
	items    map[string]*list.Element // キーからリストの要素を引くためのmap
}

func New(capacity int) *Cache {
	return &Cache{capacity: capacity, ll: list.New(), items: map[string]*list.Element{}}
}

// Get はキーに対応する値を返す。見つかったら「使った」ので先頭に移す
func (c *Cache) Get(key string) (int, bool) {
	e, ok := c.items[key]
	if !ok {
		return 0, false
	}
	c.ll.MoveToFront(e)
	return e.Value.(*entry).value, true
}

// Put はキーと値を入れる。入れたキーは一番最近使ったものとして先頭に置く
func (c *Cache) Put(key string, value int) {
	// 既にあるキーなら値を上書きする
	if e, ok := c.items[key]; ok {
		e.Value.(*entry).value = value
		return
	}
	// いっぱいなら、末尾（一番長く使われていないもの）を捨てる
	if c.ll.Len() == c.capacity {
		oldest := c.ll.Back()
		c.ll.Remove(oldest)
		delete(c.items, oldest.Value.(*entry).key)
	}
	c.items[key] = c.ll.PushFront(&entry{key, value})
}

// Len は今入っている件数を返す
func (c *Cache) Len() int { return c.ll.Len() }
```

キャッシュは状態を持っていて、`Get` の結果はそれまでに `Put` や `Get` をどう呼んできたかで変わります。そのため、割り勘関数のように1回の関数呼び出しだけをテストしても足りません。`Put` と `Get` を色々な順番で何回も呼び、その途中でずっと正しく動いているかを確かめる必要があります。

ただ、呼ぶ順番をランダムにすると、「この順番で呼んだら `Get` は何を返すべきか」を期待値として毎回書くことはできません。そこで、期待する結果を計算してくれる別の実装を用意します。これをモデルと呼びます。モデルは速さを気にせず、読めば正しいと分かるくらい素朴に書きます。同じ操作を本物とモデルの両方に順番に流して、結果が毎回一致するかを確かめるのがモデルベーステストです。

LRUキャッシュのモデルは、キーを最近使った順に並べたスライスと、値を入れておくmapで書けます。使ったキーはスライスの先頭に移し、容量を超えたら末尾のキーを消すだけです。本物の実装は速くするために双方向リストとmapを組み合わせていますが、モデルは毎回スライスを並べ直す遅いやり方で構いません。

テストコードは3つの部分に分かれます。

まずモデルです。ここはrapidを使っていない、普通のGoのコードです。

```go:lru/lru_test.go
package lru

import (
	"slices"
	"testing"

	"pgregory.net/rapid"
)

// model は「最近使った順に並んだキーと値」を素朴なスライスで持つだけのモデル。
type model struct {
	capacity int
	keys     []string // 先頭ほど最近使った
	values   map[string]int
}

// touch はキーを「使った」ことにして、スライスの先頭に移す
func (m *model) touch(key string) {
	m.keys = slices.DeleteFunc(m.keys, func(k string) bool { return k == key })
	m.keys = slices.Insert(m.keys, 0, key)
}
```

次に、rapidにランダムな順番で呼ばせる「操作」を定義します。本物の `Cache` とモデルを並べて持つ構造体を作り、操作をそのメソッドとして書きます。各メソッドの中では `Draw` でキーや値を作り、本物とモデルの両方に同じ操作をして結果を比べます。`Check` という名前のメソッドだけは特別で、操作のたびに呼ばれる不変条件のチェックになります。

```go:lru/lru_test.go
// cacheMachine は本物の Cache とモデルを並べて持つ。
// 公開メソッド（Get, Put）が rapid にランダムな順番で呼ばれる「操作」になる
type cacheMachine struct {
	cache *Cache
	model *model
	keys  *rapid.Generator[string]
}

// Get 操作: 本物から取り出した結果が、モデルの持っている値と一致するかを確かめる
func (m *cacheMachine) Get(t *rapid.T) {
	key := m.keys.Draw(t, "key")
	got, ok := m.cache.Get(key)
	want, wantOK := m.model.values[key]
	if ok != wantOK || got != want {
		t.Fatalf("Get(%q) = (%d, %v), want (%d, %v)", key, got, ok, want, wantOK)
	}
	if ok {
		m.model.touch(key)
	}
}

// Put 操作: 本物とモデルの両方に同じキーと値を入れる
func (m *cacheMachine) Put(t *rapid.T) {
	key := m.keys.Draw(t, "key")
	value := rapid.Int().Draw(t, "value")
	m.cache.Put(key, value)

	m.model.values[key] = value
	m.model.touch(key)
	// 容量を超えたら、モデル側も末尾（一番長く使われていないもの）を捨てる
	if len(m.model.keys) > m.model.capacity {
		evicted := m.model.keys[len(m.model.keys)-1]
		m.model.keys = m.model.keys[:len(m.model.keys)-1]
		delete(m.model.values, evicted)
	}
}

// Check は操作のたびに呼ばれる不変条件のチェック。本物に入っている件数がモデルの件数と一致するか（容量を超えて溜まったり、消えすぎたりしていないか）を確かめる
func (m *cacheMachine) Check(t *rapid.T) {
	if m.cache.Len() != len(m.model.keys) {
		t.Fatalf("Len() = %d, want %d", m.cache.Len(), len(m.model.keys))
	}
}
```

最後がrapidで実行するテスト関数です。`rapid.Check` の中で上の構造体を作り、`rapid.StateMachineActions` で公開メソッドを操作として登録して `t.Repeat` に渡すと、rapidが `Get` と `Put` をランダムな順番で何回も呼びます。

```go:lru/lru_test.go
func TestCacheStateMachine(t *testing.T) {
	rapid.Check(t, func(t *rapid.T) {
		capacity := rapid.IntRange(1, 5).Draw(t, "capacity")
		m := &cacheMachine{
			cache: New(capacity),
			model: &model{capacity: capacity, values: map[string]int{}},
			// キーの種類を絞って衝突（同じキーへの Put）を起こしやすくする
			keys: rapid.SampledFrom([]string{"a", "b", "c", "d", "e", "f"}),
		}
		// StateMachineActions が cacheMachine のメソッドを操作として登録し、
		// Repeat がそれをランダムな順番で何回も呼ぶ
		t.Repeat(rapid.StateMachineActions(m))
	})
}
```

キーを `rapid.String()` にしないのがポイントです。ランダムな文字列だと同じキーがほぼ出てこないので「既存キーの上書き」や「追い出されたキーの再取得」といった、バグが潜みやすいケースを試しにくくなります。容量も1〜5と小さくして追い出しが頻繁に起きるようにしています。

```text
--- FAIL: TestCacheStateMachine (0.08s)
    lru_test.go:68: [rapid] failed after 0 tests: Get("a") = (0, false), want (0, true)
        To reproduce, specify -run="TestCacheStateMachine" -rapid.seed=11968650428776007676
        Failed test output:
    lru_test.go:69: [rapid] draw capacity: 2
    lru_test.go:78: [rapid] draw action: "Put"
    lru_test.go:46: [rapid] draw key: "a"
    lru_test.go:47: [rapid] draw value: 0
    lru_test.go:78: [rapid] draw action: "Put"
    lru_test.go:46: [rapid] draw key: "b"
    lru_test.go:47: [rapid] draw value: 0
    lru_test.go:78: [rapid] draw action: "Put"
    lru_test.go:46: [rapid] draw key: "a"
    lru_test.go:47: [rapid] draw value: 0
    lru_test.go:78: [rapid] draw action: "Put"
    lru_test.go:46: [rapid] draw key: "c"
    lru_test.go:47: [rapid] draw value: 0
    lru_test.go:78: [rapid] draw action: "Get"
    lru_test.go:33: [rapid] draw key: "a"
    lru_test.go:37: Get("a") = (0, false), want (0, true)
```

操作列まで縮めてくれるのでそのまま再現手順として読めます。

1. 容量2のキャッシュに `a`、`b` を入れる
2. `a` を上書きする（ここで `a` が一番最近使ったキーになるはず）
3. `c` を入れる。追い出されるべきは一番古い `b`
4. `a` を取ろうとすると消えている

`Put` で既存キーを上書きしたときにリストの先頭に移していないのが原因です。ちなみに `Put` した値が全部 `0` になっているのは、rapidが値もできるだけ小さく縮めようとして、`0` にしてもテストが失敗したままだったからです。このバグは値とは関係ないので、どこまでも縮められたわけです。

上書きしたときも先頭に移すように、実装を1行直します。

```diff go:lru/lru.go
 	if e, ok := c.items[key]; ok {
 		e.Value.(*entry).value = value
+		c.ll.MoveToFront(e)
 		return
 	}
```

試す回数を2000回に増やして（`-rapid.checks=2000`）流し直すと、今度はテストが通りました。

```text
    lru_test.go:68: [rapid] OK, passed 2000 tests (34.015875ms)
--- PASS: TestCacheStateMachine (0.03s)
```

これを例示テストで見つけようとすると「上書きしたあとに追い出しが起きるケース」を自分で思いつく必要があります。思いつけるならたぶん実装の時点で `MoveToFront` を書いています。

Hughesはプロパティの種類を比較した論文[^howto]でモデルベースのテストが最も効果的だったと書いています。

> Model-based testing seemed the most effective approach overall, revealing all our bugs with a small number of properties, and generally finding bugs fast.
>
> モデルベーステストは全体として最も効果的なアプローチのようで、少ないプロパティですべてのバグを明らかにし、たいていすぐにバグを見つけた

ただし同じ論文では、モデルは複雑な場面では実装コストが高くなったり、本物の実装に似すぎてしまったりすることもあると注意しています。今回のモデルが `container/list` を使わずスライスで素朴に書いてあるのはそのためです。モデルが実装と同じ書き方をしていると、同じバグを抱えてしまいます。

## プロパティの見つけ方

「なぜAI時代にPBTなのか」の章で、「何をプロパティにすればいいか分からない」がPBTの一番の壁だという話をしました。Hughesの「How to Specify It!」[^howto]はプロパティを5種類に分けています。この記事の例と対応させるとこうなります。

| 種類 | 何を確かめるか | この記事の例 |
|---|---|---|
| 不変条件（Invariant） | どの操作の結果もデータ型の決まりを守っている | LRUの `Check`（要素数がモデルと一致） |
| 事後条件（Postcondition） | 1回の呼び出しの引数と結果の関係 | 割り勘の「合計が一致」「差は1円以内」 |
| メタモルフィック（Metamorphic） | 関係のある2回の呼び出しの結果どうしの関係 | （例）検索条件を足したら結果は減るか同じ |
| 帰納的（Inductive） | 小さい引数での呼び出しとの関係 | （例）`Sum(append(xs, x)) == Sum(xs) + x` |
| モデルベース（Model-based） | 素朴に書いた別実装（モデル）と結果が一致する | LRUのスライスによるモデル |

ここに往復変換（具体例1のカーソルで書いた `Decode(Encode(x)) == x`）と「どんな入力でもpanicしない」を加えると、実務で書くプロパティのかなりの部分をカバーできます。rapidのREADMEも始めやすいプロパティとして「有効な入力でpanicしない」「2つのアルゴリズムやデータ構造の振る舞いが一致する」「`decode(encode(x)) == x` の往復」の3つを挙げています。

Jane Streetの調査[^goldstein]でもよく使われていたのはモデルや別実装と比べる形（30人中17人）と往復変換（30人中11人）でした。PBTが最も役に立つのは「ちゃんとした抽象があって、実装が複雑なとき」だそうです。

AIにプロパティを考えさせるときの注意点もあります。Hypothesisのブログ[^hypothesis-blog]によると、AIにプロパティの根拠を型やdocstring、既存のテストから探させると質が上がります。一方で、AIは必要もないのに生成する範囲を狭めがちです。

> AI models like to write strategies with unnecessary restrictions, like limiting the maximum length of a list even when the property should hold for all lengths of lists.
>
> AIモデルはリストがどんな長さでもプロパティが成り立つはずなのにリストの最大長を制限するなど、不要な制約をつけたジェネレータを書きたがる

rapidで言えば `rapid.IntRange(1, 10)` のように狭い範囲になっていないかはレビューで見るべきところです。割り勘関数のテストでも金額 `total` を `IntRange(0, 1_000_000)`（0円〜100万円）にしていますが、負の金額を許すかどうかは仕様として人間が決めることです。

## 全部PBTにすればいい？

**しないほうがいい**と思っています。理由は3つあります。

1つ目は具体的な決めごとは例示テストのほうがはっきり書けることです。割り勘関数のバグを直したとき、余りの1円を払う人が最後の人から最初の人に変わり、`[333 333 334]` を期待していた例示テストが落ちました。プロパティは「誰が1円多く払うか」について何も言っていないので、この決めごとはテストのどこにも書かれていない状態です。もし「幹事（先頭の人）が多めに払う」と決まっているなら、それは `{"1円余る", 1000, 3, []int{334, 333, 333}}` という例示テストで書くのが一番伝わります。プロパティとして書くこともできますが、読んだ人がパッと分かるのは具体例のほうです。Jane Streetの調査でもこんな声が出ています。

> With [example-based unit tests], I kind of look at it, I can just make a snap judgment as to whether this is okay or not. Trying to formalize that judgment sometimes can be very difficult.
>
> 例示のユニットテストなら見ただけでOKかどうかを即断できる。その判断を形式化しようとすると、ときにとても難しい

同じ調査では、PBTを使っているチームでもプロパティでは確かめにくい部分を例示テストで補っていて、「QuickCheckだけを単独で使うのはあまりいい考えじゃない」という発言も紹介されています。

2つ目はプロパティやジェネレータを書くコストです。今回の例はどれもプロパティがきれいに書けるものを選んでいますが、実際の業務ロジックだと「何が成り立つべきか」を言葉にするところで手が止まることがよくあります。そこはAIに手伝ってもらえるようになってきたとはいえ、Anthropicの研究でも、暗黙の前提を含むコードでは「何が正しいプロパティかを決められるのはライブラリのメンテナだけだ」と書いています[^anthropic-blog]。

3つ目はプロパティそのものが間違っていることがあることです。Hughesはこう書いています[^hughes2016]。

> Errors are often in the model, rather than the code. Calling something a specification does not make it right!
>
> 誤りはコードよりもモデルの側にあることが多い。何かを仕様と呼んだからといってそれが正しくなるわけではない！

特に危ないのが期待値を実装から計算してしまうパターンです。たとえば具体例2のLRUキャッシュのテストで、モデルを本物の `Cache` を使って書いてしまうと、同じコードを2回動かして比べているだけになります。どんな入力を入れても両者は一致します。**入力をいくらランダムにしても、オラクルが実装と独立していなければバグは見つかりません**。

これらを踏まえて私はこう使い分けています。

| こういうとき | 書くテスト |
|---|---|
| 「この場合はこうする」という業務上の決めごと | 例示テスト |
| 過去に起きたバグの回帰テスト | 例示テスト（rapidの `.fail` ファイルでもよい） |
| 合計・件数・順序など、どんな入力でも守るべき条件がある | PBT |
| エンコード/デコード、シリアライズ、変換の往復 | PBT |
| 素朴な別実装と比べられる（キャッシュ、キュー、検索） | PBT（ステートマシン） |
| 外から来る入力をパースする | PBT（panicしないこと）＋必要なら `MakeFuzz` でファジング |

最後の行は、ファジングとの違いのところで書いた `rapid.MakeFuzz` の使い方です（[ファジングとの違い](#%E3%83%95%E3%82%A1%E3%82%B8%E3%83%B3%E3%82%B0%E3%81%A8%E3%81%AE%E9%81%95%E3%81%84)）。パーサーのように外から何が来るか読めない入力は、普段のテストではPBTで確かめ、ファジングでも時間をかけて探させておくと安心です。

AI時代にテストの何が変わるかというと、テストを書く手間はAIが肩代わりしてくれるようになった一方で「何が正しいか」を決める仕事は人間に残ったということだと思っています。例示テストはその「何が正しいか」を具体例で書き、PBTはプロパティで書きます。AIに実装を任せるほどプロパティで書いた仕様の価値が上がっていく気がしています。

[^quick]: Go標準ライブラリ testing/quick のパッケージドキュメント. https://pkg.go.dev/testing/quick
[^hypothesis]: David R. MacIver, Zac Hatfield-Dodds, et al. "Hypothesis: A new approach to property-based testing." Journal of Open Source Software 4(43), 1891, 2019. https://doi.org/10.21105/joss.01891
[^quickcheck]: Koen Claessen, John Hughes. "QuickCheck: A Lightweight Tool for Random Testing of Haskell Programs." ICFP 2000. https://doi.org/10.1145/351240.351266
[^hughes2016]: John Hughes. "Experiences with QuickCheck: Testing the Hard Stuff and Staying Sane." A List of Successes That Can Change the World, LNCS 9600, 2016. https://doi.org/10.1007/978-3-319-30936-1_9
[^maciver]: David R. MacIver. "In praise of property-based testing." Increment, Issue 10, 2019. https://increment.com/testing/in-praise-of-property-based-testing/
[^konstantinou]: Michael Konstantinou, Renzo Degiovanni, Mike Papadakis. "Can we find bugs using LLM-generated oracles?" arXiv:2410.21136, 2024. https://arxiv.org/abs/2410.21136
[^pgs]: Lehan He et al. "Use Property-Based Testing to Bridge LLM Code Generation and Validation." arXiv:2506.18315v1, 2025. https://arxiv.org/abs/2506.18315v1
[^syscard]: Anthropic. "Claude 3.7 Sonnet System Card." 2025. https://assets.anthropic.com/m/785e231869ea8b3b/original/claude-3-7-sonnet-system-card.pdf
[^prompting]: Anthropic. "Prompting best practices" の "Avoid focusing on passing tests and hardcoding" 節. https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices
[^goldstein]: Harrison Goldstein, Joseph W. Cutler, Daniel Dickstein, Benjamin C. Pierce, Andrew Head. "Property-Based Testing in Practice." ICSE 2024. https://doi.org/10.1145/3597503.3639581
[^anthropic-blog]: Muhammad Maaz, Liam DeVoe, Zac Hatfield-Dodds, Nicholas Carlini. "Finding bugs across the Python ecosystem with Claude and property-based testing." Anthropic, 2026. https://www.anthropic.com/research/property-based-testing
[^agentic-pbt]: Muhammad Maaz, Liam DeVoe, Zac Hatfield-Dodds, Nicholas Carlini. "Agentic Property-Based Testing: Finding Bugs Across the Python Ecosystem." arXiv:2510.09907, 2025. https://arxiv.org/abs/2510.09907
[^hypothesis-blog]: Liam DeVoe, Muhammad Maaz, Zac Hatfield-Dodds, Nicholas Carlini. "A Claude Code command for Hypothesis." 2025. https://hypothesis.works/articles/claude-code-plugin/
[^harness]: Birgitta Böckeler. "Harness engineering for coding agent users." martinfowler.com, 2026. https://martinfowler.com/articles/harness-engineering.html
[^best-practices]: Anthropic. "Best practices for Claude Code." https://code.claude.com/docs/en/best-practices
[^howto]: John Hughes. "How to Specify It!: A Guide to Writing Properties of Pure Functions." TFP 2019, LNCS 12053, 2020. https://doi.org/10.1007/978-3-030-47147-7_4
