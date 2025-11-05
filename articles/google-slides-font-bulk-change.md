---
title: Googleスライドのフォントを一括変更する方法
emoji: '📝'
type: 'tech' # tech: 技術記事 / idea: アイデア記事
topics: ['GoogleSlide', 'フォント', '業務効率化']
published: true
---

## はじめに

Google スライドで大量のスライドのフォントを 1 つずつ変更するのは大変ですよね。Apps Script を使えば、全スライドのフォントを一括で変更できます。

## 手順

1. フォントを変更したい Google スライドを開く
2. メニューから「拡張機能 → Apps Script」を開く
3. 下記スクリプトを貼り付け

```js
function changeAllFontsToBizUDPGothic() {
  const TARGET_FONT = 'BIZ UDPGothic';
  const pres = SlidesApp.getActivePresentation();

  pres.getSlides().forEach((slide) => {
    slide
      .getPageElements()
      .forEach((pe) => processPageElementSafe_(pe, TARGET_FONT));
  });

  pres.getMasters().forEach((master) => {
    master
      .getPageElements()
      .forEach((pe) => processPageElementSafe_(pe, TARGET_FONT));
    master.getLayouts().forEach((layout) => {
      layout
        .getPageElements()
        .forEach((pe) => processPageElementSafe_(pe, TARGET_FONT));
    });
  });

  SlidesApp.getUi().alert(`全テキストを「${TARGET_FONT}」に変更しました`);
}

function processPageElementSafe_(pe, targetFont) {
  const type = pe.getPageElementType();

  try {
    if (type === SlidesApp.PageElementType.SHAPE && pe.asShape().getText) {
      pe.asShape().getText().getTextStyle().setFontFamily(targetFont);
    } else if (type === SlidesApp.PageElementType.TABLE) {
      const table = pe.asTable();
      for (let r = 0; r < table.getNumRows(); r++) {
        for (let c = 0; c < table.getNumColumns(); c++) {
          const cell = table.getCell(r, c);
          if (!cell) continue;
          const text = cell.getText();
          if (text) text.getTextStyle().setFontFamily(targetFont);
        }
      }
    } else if (type === SlidesApp.PageElementType.GROUP) {
      pe.asGroup()
        .getChildren()
        .forEach((child) => processPageElementSafe_(child, targetFont));
    }
    // 画像・線・チャートなどテキストを持たない要素はスキップ
  } catch (err) {
    console.log(`Skipped element (${type}): ${err.message}`);
  }
}
```

4. 保存して ▶️ Run ボタンで実行(初回のみ権限許可が必要)
5. 処理が完了するとポップアップが表示される

## スクリプトの説明

このスクリプトは以下の要素のフォントを変更します

- 通常のスライド内のテキスト(図形、テーブル内のテキストを含む)
- マスタースライドとレイアウトのテキスト
- グループ化された要素内のテキスト

## カスタマイズ方法

フォントを変更したい場合は、スクリプト内の `TARGET_FONT` の値を変更してください。

```js
const TARGET_FONT = 'BIZ UDPGothic'; // ← 好きなフォント名に変更
```

利用可能なフォント名は、Google スライドのフォント選択メニューで確認できます。

## まとめ

Apps Script を使うことで、手作業では時間がかかるフォントの一括変更を簡単に実行できます。プレゼンテーションの統一感を保つのに役立ててください。
