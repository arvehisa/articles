---
title: "Wordの検索と置換をマクロで自動化する方法"
emoji: "🐡"
type: "tech"
topics:
  - "macro"
  - "vba"
  - "マクロ"
  - "word"
published: true
published_at: "2024-03-10 13:32"
---

# はじめに
Wordの文書を特定の形式で分割し、分割した内容をデータベースに取り込む処理を行う場合、手動で行うと非常に手間がかかります。特に、Wordの文書が頻繁に更新される場合、繰り返し同じ処理を行う必要があります。そこで、マクロを使ってワンクリックで同じ処理ができるようにすることで、作業効率を大幅に向上させることができます。

本記事では、Wordのマクロを使って、特定形式の文字を削除や検索する方法、検索と置換で特殊文字を使う方法、置換後の文字のサイズやフォント、色を変更する方法について解説します。

# 基本的なマクロで特定形式の文字を削除や検索する方法
Wordのマクロを使うと、特定の形式を持つ文字を簡単に削除したり、検索したりすることができます。例えば、以下のマクロでは、16pxのフォントサイズを持つテキストを削除しています。

```vb
With ActiveDocument.Content.Find
    .ClearFormatting
    .Font.Size = 16
    .Execute FindText:="", Format:=True, Replace:=wdReplaceAll, ReplaceWith:=""
End With

```

# 検索と置換で特殊文字を使う
Wordの検索と置換では、特殊文字を使うことができます。例えば、改行は `^p`、改ページは `^m` で表されます。以下のマクロでは、改ページを削除しています。

```vb
With ActiveDocument.Content.Find
    .Text = "^m"
    .Replacement.Text = ""
    .Execute Replace:=wdReplaceAll
End With
```

# 置換後の文字のサイズやフォント、色を変更する
検索と置換を行いながら、置換後の文字のサイズやフォント、色を変更することができます。
以下のマクロでは、2回の改行 `^p^p` を `##PAGEBREAK##^m`、 つまり `##PAGEBREAK##` + 改ページに置換し、置換後の文字の色を灰色に変更しています。
（最初は ChatGPT から置換後のフォントの色だけを変えられず、フォント色を変えるのにまた検索して遍歴しないといけないと言われてかなりパフォーマンスの悪いマクロだったのですが `.Replacement.Font.ColorIndex` で一瞬で終わらせることができた）

```vb
Copy code
With ActiveDocument.Content.Find
    .Text = "^p^p"
    .Replacement.Text = "##PAGEBREAK##^m"
    .Replacement.Font.ColorIndex = wdGray25
    .Execute Replace:=wdReplaceAll
End With
```

# 実際のマクロはこちら
実際筆者が使っているものはこちら。
```vb
Sub FormatDocument()
    Dim rng As Range
    Dim i As Long

    ' 16pxの文字サイズを持つテキストを削除
    With ActiveDocument.Content.Find
        .ClearFormatting
        .Font.Size = 16
        .Execute FindText:="", Format:=True, Replace:=wdReplaceAll, ReplaceWith:=""
    End With

    ' 改ページを削除
    With ActiveDocument.Content.Find
        .Text = "^m"
        .Replacement.Text = ""
        .Execute Replace:=wdReplaceAll
    End With

    ' 2回以上の改行を2回にする
    With ActiveDocument.Content.Find
        .Text = "^p{2,}"
        .Replacement.Text = "^p^p"
        .MatchWildcards = False
        .Execute Replace:=wdReplaceAll
    End With

    ' すべてのフォントをヒラギノ丸ゴ Proに変更し、サイズを10.5にする
    ActiveDocument.Select
    With Selection
        .Font.Name = "ヒラギノ丸ゴ Pro"
        .Font.Size = 10.5
    End With

    ' ^p^pを##PAGEBREAK##^mに変更したうえで、色を変更する
    With ActiveDocument.Content.Find
        .Text = "^p^p"
        .Replacement.Text = "##PAGEBREAK##^m"
        .Replacement.Font.ColorIndex = wdGray25
        .Execute Replace:=wdReplaceAll
    End With
    
End Sub

```

# まとめ
Wordのマクロを使うことで、手動では大変な作業を自動化することができます。特に、頻繁に更新されるWordの文書を処理する場合、マクロを活用することで作業効率を大幅に向上させることができます。本記事で紹介した方法を参考に、ぜひマクロを活用してみてください。

※ 本記事は Claude 3 Opus の文章生成にアシストされています。
※ 筆者環境は Microsoft Word for Mac 16.82 です。