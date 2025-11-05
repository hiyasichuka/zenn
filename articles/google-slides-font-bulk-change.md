---
title: Googleスライドのフォントを一括変更する方法
emoji: '📝'
type: 'tech' # tech: 技術記事 / idea: アイデア記事
topics: ['GoogleSlide', 'フォント', '業務効率化']
published: true
---

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

4. 保存して ▶️ Run ボタンで実行（初回のみ権限許可が必要）
5. 処理が完了するとポップアップが表示される
