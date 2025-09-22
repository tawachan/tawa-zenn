---
title: "lexical（+ React）でEnterキーの挙動をカスタマイズする方法"
emoji: "⌨️"
type: "tech"
topics:
  - "react"
  - "lexical"
  - "javascript"
  - "typescript"
published: true
published_at: "2024-02-21 15:26"
---

![](/images/lexical-enter-hero.png)

[lexical](https://lexical.dev/) を使って Enter を押したときの挙動をカスタマイズしたくなったのでその手順のメモ。

Zenn のスクラップに書いたものを整理している。

https://zenn.dev/tawachan/scraps/ee34d1b8ce4261

## Enter キーのデフォルト挙動

lexical では、Enter キーを押すと選択中の行の要素に応じて、よしなにいい感じに要素を追加してくれる。たとえば、リストアイテム内で Enter キーを押すと、新しいリストアイテムが追加される。

しかし、リストアイテムのときの改行の挙動が個人的には気になった。文字が入力されていないときに Enter キーが押されても新しいリストアイテムが追加される。

![](/images/lexical-default-behavior.gif)

しかし、新しいリストアイテムを追加するのではなく、新しい段落を追加したいので、Enter キーの挙動をカスタマイズすることにした。

## カスタマイズの実装

カスタマイズのために、それ用のプラグインを作成する。プラグインを作成することで、lexical のエディターの挙動をカスタマイズできる。

### ステップ 1: プラグインの作成

まず、Enter キーの挙動をカスタマイズするためのプラグインを作成します。このプラグイン内で、lexical の`registerCommand`メソッドを使用して Enter キーのイベントを捕捉します。

```tsx
export const EnterPlugin: FC = () => {
  const [editor] = useLexicalComposerContext();

  useEffect(() => {
    editor.registerCommand(
      KEY_ENTER_COMMAND,
      (payload) => {
        // ここに条件とカスタム挙動を実装
      },
      COMMAND_PRIORITY_EDITOR
    );
  }, [editor]);

  return null;
};
```

### ステップ 2: 条件の設定

特定の条件下でのみカスタム挙動を実行するように、条件を設定する。

まずはカーソルがあたっている選択中の要素を取得し、それがリストアイテムであるかどうかを判定するのはこんな感じでできた。

```tsx
const selection = $getSelection();
if (!selection) return false;
if (!$isRangeSelection(selection)) return false;

const anchorNode = selection.anchor.getNode();
if (!$isListItemNode(anchorNode)) return false;
// ...略
```

文字が入力されているかは以下で確認できそうだった。

```tsx
anchorNode.getTextContentSize();
```

最後のアイテムかどうかは以下で確認できそうだった。

```tsx
anchorNode.isLastChild();
```

### ステップ 3: カスタム挙動の実装

条件に合致する場合に実行されるカスタム挙動を実装する。ここでは、`event.preventDefault()`を呼び出してデフォルトの挙動をキャンセルし、代わりにカスタムの挙動を実行する。

実装はこんな感じ。ただのパラグラフを追加するように矯正している。

```tsx
if (anchorNode.getTextContentSize() === 0 && anchorNode.isLastChild()) {
  event.preventDefault();
  editor.update(() => {
    selection.insertNodes([$createParagraphNode()]);
  });
}
```

### ステップ 4: プラグインの登録

最後に、作成したプラグインを lexical のエディターに登録すればよい。

## 最終的なプラグインのコード

```tsx
import { useLexicalComposerContext } from "@lexical/react/LexicalComposerContext";
import { $getSelection, $isRangeSelection, COMMAND_PRIORITY_EDITOR, KEY_ENTER_COMMAND, $createParagraphNode, RangeSelection } from "lexical";
import { FC, useEffect } from "react";
import { $isListItemNode } from "@lexical/list";

// 条件をチェックする関数を定義
const shouldPreventDefaultEnter = (selection: RangeSelection) => {
  const anchorNode = selection.anchor.getNode();
  if (!$isListItemNode(anchorNode)) return false;
  // リストアイテムノードで、テキストが0以上で最後の子ではない場合にtrueを返す
  return anchorNode.getTextContentSize() === 0 && anchorNode.isLastChild();
};

export const EnterPlugin: FC = () => {
  const [editor] = useLexicalComposerContext();

  useEffect(() => {
    editor.registerCommand(
      KEY_ENTER_COMMAND,
      (payload) => {
        if (!payload) return true;
        const event: KeyboardEvent = payload;

        const selection = $getSelection();
        if (!selection) return false;
        if (!$isRangeSelection(selection)) return false;

        if (shouldPreventDefaultEnter(selection)) {
          event.preventDefault();
          editor.update(() => {
            selection.insertNodes([$createParagraphNode()]);
          });
        }
        return true;
      },
      COMMAND_PRIORITY_EDITOR
    );
  }, [editor]);

  return null;
};
```

## まとめ

lexical のエディターの挙動をカスタマイズするために、プラグインを作成する方法を紹介した。この方法を使えば、lexical のエディターの挙動を自由にカスタマイズできる。

↓ これがカスタム挙動を実装した後の挙動。

![](/images/lexical-after-custom.gif)