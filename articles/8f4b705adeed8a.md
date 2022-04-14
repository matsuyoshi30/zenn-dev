---
title: "Go の標準ライブラリで最もインポートされているもの"
emoji: "💨"
type: "tech"
topics: ["go"]
published: true
---

Go のパッケージドキュメントサイト [pkg.go.dev](https://pkg.go.dev/) ではパッケージの説明やエクスポートされた定数、変数、関数の説明のほか、該当パッケージを直接インポートしている他のパッケージの数や、リンク先ではパッケージの一覧まで確認できます。

Go を書いていて pkg.go.dev でドキュメントを確認したことがある人は、一度は「標準パッケージで最もインポートされてるのは何なんだろう」と気になったことがあると思います。今回はそれを調べてみました。

![https://pkg.go.dev/net/http](https://storage.googleapis.com/zenn-user-upload/af22940be958-20220414.png)
*net/http*

# api.godoc.org で確認する

pkg.go.dev の話から入ったのにいきなり昔の話になりますが、pkg.go.dev が導入される前まではパッケージのドキュメントは godoc.org で確認することができました。今は [Go Blog の移行記事](https://go.dev/blog/godoc.org-redirect)にもあるとおり、 pkg.go.dev へリダイレクトするようになっています。そして、この移行記事でも言及されていますが、 godoc.org には API によるアクセスが想定された別のURL [api.godoc.org](https://api.godoc.org) があり、これは pkg.go.dev で同様のことができる URL が用意されるまでは提供される予定です。

api.godoc.org の使い方は godoc.org のソースコードリポジトリの [wiki](https://github.com/golang/gddo/wiki/API) に記載されています。 search クエリを用いた検索やパッケージ全列挙、指定したパッケージがインポートしているパッケージを取得することができ、また指定したパッケージをインポートしているパッケージも取得することが可能です。今回は標準パッケージの中で何が一番インポートされているかを確認したいので、最後の「指定したパッケージをインポートしているパッケージを取得」できれば目的が達成されます。

標準パッケージのリストを取得するには、[golang.org/x/tools/go/packages](https://pkg.go.dev/golang.org/x/tools/go/packages) の `Load` を使います。 `Load` のパターンに std を指定することで `go list std` とおなじ実行結果を取得できます。これで標準パッケージの一覧を取得して各パッケージと api.godoc.org を結合して HTTP リクエストを投げます。

```go
results := make(map[string]int)
for _, stdlib := range removeInternalPkg(stdlibs) { // パッケージパスに internal を含んだものを除外
	stdlib := stdlib
	go func() {
		resp, err := http.Get(apiURL + stdlib)
		if err != nil {
			log.Printf("[ERR ] package %s: %q\n", stdlib, err)
			return
		}
		defer resp.Body.Close()

		body, err := io.ReadAll(resp.Body)
		if err != nil {
			log.Printf("[ERR ] package %s: %q\n", stdlib, err)
			return
		}

		var apiResp APIResp
		err = json.Unmarshal(body, &apiResp)
		if err != nil {
			log.Printf("[ERR ] package %s: %q\n", stdlib, err)
			return
		}
		results[stdlib] = len(apiResp.Results)
		return nil
	}()
}
```


一応これでも取得したいものは取得できますが、実際に実行してみるとめちゃくちゃ遅いです。多分 api.godoc.org の裏にはキャッシュが用意されてなくて、 `api.godoc.org/{packagePath}` の呼び出しでは都度データベースにアクセスしていると思われます。標準パッケージをインポートしているパッケージの件数になると、それはもう結果が非常に大きいのでレスポンスを返すのがとても遅いです。また、実際に返ってくる結果を見ると pkg.go.dev の Imported By の結果とは異なる数値になっていて、情報が更新されていないことも伺えます。

余談ですが pkg.go.dev のインフラ構成は [ここ](https://github.com/golang/pkgsite/blob/a3a009e12ea183c9e6e1abd028f5e091c9fb2601/doc/design.md) で確認できます。また、特定のパッケージをインポートしているパッケージ群の取得は [ここ](https://github.com/golang/pkgsite/blob/a3a009e12ea183c9e6e1abd028f5e091c9fb2601/internal/postgres/details.go#L79-L100) で行われています。

# pkg.go.dev の HTML をパースする

上述の通り api.godoc.org でも一応それっぽい結果は取得できますが、いかんせん遅かったりそもそもの数値があやしかったり、試している最中に api.godoc.org がダウンしている時もありました。同様の機能( API 用の URL )が pkg.go.dev に来てくれるとありがたいんですが、[issue](https://github.com/golang/go/issues/36785) を見てもまだ来ていないようなので、別の策として pkg.go.dev にアクセスしてそのレスポンスの HTML をパースしようと思います。

HTML のパースには https://pkg.go.dev/golang.org/x/net/html の `Parse` を使います。

```go
results := make(map[string]int)
for _, stdlib := range removeInternalPkg(stdlibs) { // パッケージパスに internal を含んだものを除外
	stdlib := stdlib
	go func() {
		resp, err := http.Get(apiURL + stdlib)
		if err != nil {
			log.Printf("[ERR ] package %s: %q\n", stdlib, err)
			return
		}
		defer resp.Body.Close()

		body, err := io.ReadAll(resp.Body)
		if err != nil {
			log.Printf("[ERR ] package %s: %q\n", stdlib, err)
			return
		}

		ret := extractImportedBy(doc)
		if ret == "" {
			err := fmt.Errorf("failed to extract imported-by value about %s", stdlib)
			log.Printf("[ERR ] package %s: %q\n", stdlib, err)
			return
		}

		importedBy, err := strconv.Atoi(strings.ReplaceAll(strings.TrimSpace(ret), ",", ""))
		if err != nil {
			log.Printf("[ERR ] package %s: %q\n", stdlib, err)
			return
		}

		results[stdlib] = importedBy
		return nil
	}()
}
```

`extractImpotedBy` は pkg.go.dev の HTML を確認して Imported By のノードを探索してその数字を返す関数です。

# 結果

さて、気になる結果ですが2022/04/14時点だと以下のとおりになりました。

まずは上位10個。

| Package       | Imported By |
| ------------- | ----------- |
| fmt           | 1672745     |
| time          | 987350      |
| strings       | 956597      |
| os            | 806548      |
| errors        | 595251      |
| io            | 586116      |
| net/http      | 577690      |
| context       | 570418      |
| encoding/json | 524932      |
| strconv       | 517514      |

`fmt` パッケージが群を抜いて多いですね。 `fmt` より Imported By の件数が多い非標準のパッケージってあるんでしょうか。

逆に下位10個。

| Package             | Imported By |
| ------------------- | ----------- |
| testing/quick       | 103         |
| time/tzdata         | 81          |
| net/netip           | 80          |
| testing/iotest      | 38          |
| testing/fstest      | 35          |
| go/build/constraint | 27          |
| debug/plan9obj      | 18          |
| runtime/metrics     | 16          |
| debug/buildinfo     | 10          |
| runtime/race        | 1           |

先日リリースされた Go 1.18 から導入された `net/netip` や `debug/buildinfo` が下位にいますがこれから増えていきそうです。あとはテスト関連や runtime 関連、 build タグのパーズや評価をするための `go/build/constraint` でした。 `testing/quick` とか正直知りませんでした。

コードは https://github.com/matsuyoshi30/count-importedby においてあるので気になる方はご確認ください。改善点などあれば PR もお待ちしてます。
