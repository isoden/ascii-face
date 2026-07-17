# claude-terminal-face

フィクション作品に見られる「機械的なボディ + 記号化された発光顔（ドットマトリクス / ASCII）」というモチーフをターミナルで再現+Claude Codeと連携させるプロトタイプ。



<video src="https://github.com/user-attachments/assets/a902d3a1-a015-4736-8700-56b43732fc9d" />

## 仕組み

```
Claude Code hooks ──▶ claude-face-hook.sh ──▶ OSC 12（カーソル色） ──▶ Ghostty custom shader
```

1. **Ghostty のカスタムシェーダーで顔を描画する**。`claude-terminal-face-status.glsl` を `~/.config/ghostty/config` の `custom-shader` に指定し、ターミナル背景として ASCII 顔をレンダリングする。カスタムシェーダー（GLSL）対応のターミナル＝Ghostty が必要。
2. **Claude Code の hooks で作業状態を判定する**。`UserPromptSubmit` / `PreToolUse` / `PostToolUse` / `Stop` イベントで `claude-face-hook.sh`（bash）が起動し、イベント内容を `thinking`（調査・プランニング中）/ `working`（実装中）/ `done`（応答完了）/ `err`（Bash 失敗）にマッピングして、状態に対応する色を **OSC 12（カーソル色変更）エスケープシーケンス**として端末に書き込む。
3. **シェーダー側はカーソル色から状態を復元する**。Ghostty のカスタムシェーダーはフレーム間で状態を持てず外部入力もほぼないため、カーソル色をサイドチャネルとして使う。シェーダーは `iCurrentCursorColor` を 5 色パレット（`idle` / `thinking` / `working` / `done` / `err`）と色距離で照合し、最も近い状態の表情アニメーションを再生する。どの色からも遠い場合（通常のテーマ既定カーソル色など）は `idle` にフォールバックする。

設計の詳細・制約は [docs/SPEC.md](docs/SPEC.md)（特に §5, §9）を参照。

## 制限事項

- **[herdr](https://herdr.dev/) 経由では動作しない**。herdr は各 pane を内部の仮想端末（`libghostty-vt`）でシミュレートしており、pane 内から送った OSC 12 はその内部状態を変えるだけで外側の本物の Ghostty には伝播しない。また hook スクリプトに制御端末が割り当てられず、無関係な別セッションの TTY へ誤送信する恐れがあるため、`claude-face-hook.sh` は祖先プロセスに herdr を検出すると安全側に倒して何もしない。Ghostty 上で直接 `claude` を実行するセッションでのみ動作する（詳細は SPEC.md §9.8）。


