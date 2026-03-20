---
title: "goverterのススメ CQRSのクエリ結果→APIレスポンス変換を自動生成する"
emoji: "🔄"
type: "tech"
topics: ["Go", "goverter", "CQRS", "oapicodegen"]
published: true
---

そろそろ暖かくなってまいりました。春といえばコード生成の季節ですね。
世は大LLM時代ですが、確率的に吐かれるコードをなるべく減らしたいということでスキーマ駆動のコード生成の機運が高まっているかと思います。

Goでは oapi-codegen などのライブラリでOpenAPIスキーマからAPIの型やハンドラを生成し、CQRSパターンでクエリ層を分離する構成がよく見られます。しかしこの構成では、クエリ層の出力型とAPIの型が微妙に食い違い、間をつなぐ変換コードが必要になります。手書きだとフィールド追加時に漏れやすく、リフレクションベースのライブラリでは実行時まで問題に気づけません。

本記事では、goverterによるコード生成でこの変換を安全に自動化する方法を紹介します。

## 課題: クエリ型とAPI型のギャップ

oapi-codegenが生成するAPI型と、クエリ層で定義する出力型を並べてみます。

```go:query/model.go
type TaskResult struct {
	ID          string
	Title       string
	Description string
	Status      string
	CreatedAt   time.Time
}
```

```go:api/gen/api.gen.go
type Task struct {
	CreatedAt   *time.Time `json:"createdAt,omitempty"`
	Description *string    `json:"description,omitempty"`
	Id          string     `json:"id"`
	Status      TaskStatus `json:"status"`
	Title       string     `json:"title"`
}
```

同じ「タスク」を表していますが、以下の差異があります。

- **フィールド名**: `ID` vs `Id`
- **ポインタ（string）**: `Description` が `string` vs `*string`
- **ポインタ（time）**: `CreatedAt` が `time.Time` vs `*time.Time`
- **named type**: `Status` が `string` vs `TaskStatus`

これらの差異を吸収する変換コードが必要です。

## 手書きだとこうなる

愚直に書くとこうなります。

```go
func ConvertTask(src query.TaskResult) gen.Task {
	return gen.Task{
		Id:          src.ID,
		Title:       src.Title,
		Description: &src.Description,
		Status:      gen.TaskStatus(src.Status),
		CreatedAt:   &src.CreatedAt,
	}
}

func ConvertTaskList(src []query.TaskResult) []gen.Task {
	tasks := make([]gen.Task, len(src))
	for i, s := range src {
		tasks[i] = ConvertTask(s)
	}
	return tasks
}
```

動くには動きますが、問題はフィールドが増えたときです。`TaskResult`に新しいフィールドを追加してもこの関数を更新し忘れると、**コンパイルは通るのに値が欠落する**という静かなバグになります。

## copierだとこうなる

リフレクションベースのコピーライブラリを使う手もあります。

@[card](https://github.com/jinzhu/copier)

```go
import "github.com/jinzhu/copier"

func ConvertTask(src query.TaskResult) gen.Task {
	var dest gen.Task
	copier.Copy(&dest, &src)
	return dest
}
```

実際に動かしてみると、今回のケースではすべてのフィールドが正しくコピーされます。

```text
=== copier変換結果 ===
Id:          "task-001"
Title:       "サンプルタスク"
Description: "タスクの説明文"
Status:      "in_progress"
CreatedAt:   2025-01-01 00:00:00 +0000 UTC
```

`ID` → `Id` のフィールド名不一致もcopierが大文字・小文字を無視してマッチしてくれます。`string` → `*string` やnamed typeへの変換も自動です。

ただし、**リフレクションベースゆえの根本的な問題**があります。

たとえばOpenAPIスキーマに `dueDate` フィールドが追加され、oapi-codegenで `gen.Task` を再生成したとします。

```diff go
 type Task struct {
 	CreatedAt   *time.Time `json:"createdAt,omitempty"`
 	Description *string    `json:"description,omitempty"`
+	DueDate     *time.Time `json:"dueDate,omitempty"`
 	Id          string     `json:"id"`
 	Status      TaskStatus `json:"status"`
 	Title       string     `json:"title"`
 }
```

`query.TaskResult` にはまだ `DueDate` がない状態で `copier.Copy` を実行すると、**エラーは一切発生せず**、 `DueDate` は黙って `nil` になります。

```text
=== copier変換結果 ===
Id:          "task-001"
Title:       "サンプルタスク"
Description: "タスクの説明文"
Status:      "in_progress"
CreatedAt:   2025-01-01 00:00:00 +0000 UTC
DueDate:     nil
```

テストもコンパイルも通るので、APIレスポンスから `dueDate` が抜け落ちていることに気づけるのは、フロントエンドが「値が来ない」と報告してきたときかもしれません。

手軽ですが、安全ではありません。

## goverterによる解決

goverterはインターフェース定義からフィールドマッピングコードを**コード生成**するツールです。

@[card](https://github.com/jmattheis/goverter)

```go:converter/converter.go
// goverter:converter
// goverter:output:file ./gen/converter.gen.go
// goverter:output:package github.com/imsugeno/query-generator-sample/converter/gen
// goverter:extend TimeToTimePtr
// goverter:useZeroValueOnPointerInconsistency
type Converter interface {
	// goverter:map ID Id
	ConvertTask(source query.TaskResult) gen.Task
	ConvertTaskList(source []query.TaskResult) []gen.Task
}

// TimeToTimePtr は time.Time → *time.Time の変換ヘルパー。
func TimeToTimePtr(t time.Time) *time.Time {
	return &t
}
```

各ディレクティブの役割は以下の通りです。

- `goverter:converter` — このインターフェースがgoverterの変換定義であることを示す
- `goverter:output:file` / `goverter:output:package` — 生成先のファイルパスとパッケージ名
- `goverter:extend TimeToTimePtr` — `time.Time` → `*time.Time` の変換に使うヘルパー関数を登録
- `goverter:useZeroValueOnPointerInconsistency` — `string` → `*string` のようなポインタの差異をゼロ値ベースで自動変換
- `goverter:map ID Id` — フィールド名の対応を明示的に指定

先ほどのcopierの例と同じく `gen.Task` に `DueDate` が追加された状態でコード生成を実行すると、goverterは以下のエラーを出します。

```text
Error while creating converter method:
    converter/converter.go:17
    func Converter.ConvertTask(source query.TaskResult) gen.Task

source.???
target.DueDate
       |
       | *time.Time

Cannot match the target field with the source entry: "DueDate" does not exist.
```

対応するソースフィールドが存在しないことを**コード生成時に検出**してくれるので、実行時まで問題が残りません。

## 生成結果

以下のコマンドで変換コードを生成します。

```bash
go tool goverter gen ./converter/
```

生成されるコードはこのようになります。

```go:converter/gen/converter.gen.go
type ConverterImpl struct{}

func (c *ConverterImpl) ConvertTask(source query.TaskResult) gen.Task {
	var genTask gen.Task
	genTask.CreatedAt = converter.TimeToTimePtr(source.CreatedAt)
	pString := source.Description
	genTask.Description = &pString
	genTask.Id = source.ID
	genTask.Status = gen.TaskStatus(source.Status)
	genTask.Title = source.Title
	return genTask
}

func (c *ConverterImpl) ConvertTaskList(source []query.TaskResult) []gen.Task {
	var genTaskList []gen.Task
	if source != nil {
		genTaskList = make([]gen.Task, len(source))
		for i := 0; i < len(source); i++ {
			genTaskList[i] = c.ConvertTask(source[i])
		}
	}
	return genTaskList
}
```

スライス変換も自動で生成され、ポインタ変換の細部（`&pString` で変数を介す、`TimeToTimePtr` の呼び出し）も気にする必要はありません。

## ハンドラでの利用

ハンドラではインターフェースを通じて変換を呼び出します。

```go:handler/handler.go
type Handler struct {
	query     query.TaskQuery
	converter converter.Converter
}

func (h *Handler) ListTasks(w http.ResponseWriter, r *http.Request, params gen.ListTasksParams) {
	// ... パラメータ処理 ...

	results, err := h.query.ListTasks(input)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	resp := gen.ListTasksResponse{
		Tasks: h.converter.ConvertTaskList(results),
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(resp)
}
```

`h.converter.ConvertTaskList(results)` の1行で、クエリ結果のスライスからAPIレスポンス型のスライスへの変換が完結します。

## まとめ

手書きやリフレクションベースの変換はフィールド追加時に黙って壊れるリスクがあります。goverterを使えばインターフェース定義からマッピングコードを自動生成でき、**フィールドの追加漏れをコード生成時に検出**できます。

デメリットとしては面倒なディレクティブを書かないといけないことが挙げられますが、皆さん既にほとんどのコードをLLMで生成しているかと思いますので、コード量よりも安全性に振り切る方が良いのかなと個人的には考えています。

サンプルリポジトリはこちらです。

@[card](https://github.com/imsugeno/goverter-sample)
