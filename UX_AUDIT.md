# slk UX監査 — 改善優先度リスト

調査日: 2026-07-31
対象: `gammons/slk`（fork: `rrrrnmtsu/slk`）main branch
方法: README/wiki(Features/Keybindings/Configuration/Tradeoffs)通読 → `internal/ui/sidebar`,
`internal/ui/newmessagepicker`, `internal/config` のコードリーディング → upstream GitHub Issues 全件確認。
コード変更なし。読み取り専用調査。

**実装ステータス（2026-07-31追記）**: High優先度3件のうち、項目1・3を実装済み（下記参照、
`go build ./... && go test ./...` 全PASS確認済み）。項目2（Slack-nativeモードでの折りたたみ初期状態）は未着手。

---

## 1. 【最優先】DMがサイドバーから消える／選べない (`hide_inactive_after_days` の暗黙仕様)

**症状**: ユーザーの実体験どおり、DM一覧がサイドバーに出てこない、または一部しか出ない。
GitHub Issue [#88](https://github.com/gammons/slk/issues/88)
「Direct Messages doesn't show in sidebar at all」と一致（メンテナ自身が
"unable to replicate" とコメントしており未解決＝根本原因が特定されていない状態）。

**根本原因**:
- `internal/config/config.go:213` で `Sidebar.HideInactiveAfterDays` の**デフォルト値が30**に設定されている。
  ```go
  Sidebar: Sidebar{
      HideInactiveAfterDays: 30,
  },
  ```
- `internal/ui/sidebar/staleness.go:42-56` の `IsStale()` に致命的な設計コメントがある:
  ```go
  // For "dm" and "group_dm", empty IS stale. Slack's client.counts
  // endpoint only returns these types when the conversation is
  // currently "open"; absence in counts ... is Slack's canonical
  // "this conversation is closed" signal. Roughly half of the
  // user's DMs and 98% of mpdms in real workspaces fall into this
  // bucket.
  ```
  つまり `lastReadTS` が空（Slack側で「クローズ」扱いのDM）は **経過日数に関係なく無条件でstale判定＝非表示**。
  コード自身のコメントで「実運用ではDMの約半分・グループDMの98%がこのバケツに入る」と明言されている。
- さらに `internal/ui/sidebar/model.go` の `buildCache()` は `len(m.items) == 0` のときだけ
  "No channels" プレースホルダーを出す (line 1409)。**アイテムは存在するが staleness フィルタで
  全部消えた場合は何の表示も出ない**（該当セクションヘッダーごと消える。`modelOrderedSections` は
  `m.filtered` から構築するため）。ユーザーから見るとDMセクション自体が存在しないように見える。
- **この機能(`[sidebar] hide_inactive_after_days`) は Wiki の `Configuration.md` に一切記載がない。**
  READMEにもない。デフォルトONで大量のDM/グループDMを黙って消す挙動が、設定リファレンスのどこにも
  出てこない状態。

**改善案**:
1. `HideInactiveAfterDays` のデフォルトを `0`（無効）に変更する、または少なくとも
   dm/group_dm の「lastReadTSが空＝無条件stale」ルールをデフォルトでは適用しない
   （経過日数ベースの判定に統一する）。
2. `Configuration.md` に `[sidebar]` セクション（`hide_inactive_after_days`, `width`）を追記する。
3. 全セクションが空になった場合でも「N件の会話が非表示中 (30日超・未読なし) — Ctrl+T で検索可」
   のようなヒントメッセージをサイドバー下部に出す（現状は完全な沈黙失敗）。
4. Ctrl+T ファジーファインダーは非表示DMも検索対象に含んでいる可能性が高い
   (`internal/ui/reducer_workspace.go:198,298` で `channelFinder.SetItems` は staleness を通さない
   フルリストを渡している）が、これがフォールバック手段であることはドキュメント上どこにも明示されていない。

**優先度: High**（影響度=大：新規ユーザーのDMが軒並み消える／実装コスト=小：デフォルト値変更+ドキュメント追記で大部分は解決）

**実装済み**: `HideInactiveAfterDays` デフォルトを`0`(無効)に変更(改善案1)。加えて`hiddenStaleCount`を
`rebuildFilter`で計測し、staleness適用時(`hide_inactive_after_days>0`を明示設定した場合)に
セクション全消滅時の完全な沈黙を解消——`m.items`ではなく`m.filtered`の空チェックに変え、
「No channels」の代わりに「N hidden (inactive) — Ctrl+T to find」を表示するようにした(改善案3相当)。
改善案2(Configuration.md追記)は未着手。

---

## 2. Slack-native sections 有効時、DM/Appsセクションが「初期状態で展開済み」になる矛盾

**症状**: config-glob モード（`use_slack_sections = false`）では `Channels` と `Apps` セクションが
デフォルトで折りたたみ済み（`internal/ui/sidebar/model.go:502-505`, `New()`）。
しかしデフォルトの Slack-native sections モード（`use_slack_sections = true`、config.tomlの既定値）では
`collapseByID` が空マップから始まるため（`IsCollapsed()` line 553-564）、**折りたたみ初期状態は一切適用されない**。
Wiki Features.md には「デフォルトの Channels セクションは折りたたみ済みで開く」と書かれているが、
これはデフォルト設定（Slack-native sections）では実際には成立しない。

**根本原因**: `Model.New()` は config モード用の `collapsed` マップだけを初期化し、
Slack モード用の `collapseByID` を初期化するロジックがどこにも無い
(`internal/ui/model.go` 全文検索でも `collapseByID` への書き込みは `ToggleCollapse` からのみ)。

**改善案**: Slack-native sections モードでも、type が `channels`/`recent_apps` のセクションを
初回ロード時に `collapseByID` へ `true` としてシードする。

**優先度: Med**（ドキュメントと実装の乖離。DM/未読が埋もれる二次被害もある）

**実装済み**: `SetSectionsProvider`から`seedDefaultCollapse()`を呼び出し、Slack-nativeモードで
`channels`/`recent_apps`型のセクションIDが`collapseByID`に未登録（＝初見）の場合のみ`true`をシード。
既にキーが存在する場合（ユーザーが手動トグル済み、またはワークスペース再訪問）は上書きしない。
`go build ./... && go test ./...`全PASS確認済み。

---

## 3. 新規メッセージダイアログ (Ctrl+N) で既存グループDMを「名前で」再オープンできない

**症状**: upstream Issue [#44](https://github.com/gammons/slk/issues/44)。
メンテナ自身のコメント: 「新規メッセージのユーザー選択欄にグループが出てこない」。

**根本原因**: `internal/ui/newmessagepicker/model.go` の `Model.users []User` はフラットな
ユーザーリストのみで、既存の group_dm/mpim をエンティティとして持たない。
`filter.go` の `matchTier` もユーザー名／ハンドルにしかマッチしない。
つまり「あのプロジェクトの3人グループ」を名前で検索して再度開く手段が picker 内に無く、
サイドバーから消えた（項目1のstaleness影響を受けやすい：group_dm は98%が該当）group DM を
探すには、そのメンバー全員のユーザー名を思い出して選び直すか、Ctrl+T で覚えている表示名を
検索するしかない。

**改善案**: Ctrl+N のリストに「既存の会話（DM/Group DM）」も候補として混在させ、
選択したら新規作成ではなく既存チャンネルを開く（Slackデスクトップの挙動と同じ）。

**優先度: High**（項目1のstaleness問題と組み合わさると、消えたグループDMを見つける手段が実質ない）

**実装済み**: `newmessagepicker.User`に`IsExisting`/`ConvType`を追加し、`seedNewMessagePicker`で
`a.sidebar.AllItems()`(staleness適用前の全件)からDM/group_dm行を合成してピッカーに混在させた。
既存行は名前でファジー検索でき、Enterで`conversations.open`を経由せず直接そのチャンネルへスイッチする
(`mode_new_message.go`の`ExistingChannelID`分岐)。Space/Tabでのピル追加は既存行では無効化(組み合わせ不可)。

---

## 4. `keep_focus_on_list` が無く、チャンネル選択のたびにフォーカスがメッセージペインへ強制移動

**症状**: upstream Issue [#83](https://github.com/gammons/slk/issues/83)。
複数のDM/チャンネルを素早く見比べたいユーザーが、1つ選ぶたびにサイドバーからフォーカスが
奪われ、j/k で次のDMへ移動する前に Tab か h で毎回サイドバーへ戻る必要がある。

**根本原因**: `internal/ui/reducer_channels.go:326` `reduceChannelSelected()` 内で
`a.focusedPanel = PanelMessages` が常に無条件実行される。ユーザーが未読を素早くさばく
ワークフロー（=ユーザーの実体験そのもの）と相性が悪い。設計コメントには
「選択後すぐ j/k でメッセージを読めるように」という意図で書かれており、単一チャンネル
巡回には妥当だが、複数DMのトリアージには逆効果。

**改善案**: config に `keep_focus_on_list = false`（既定値）を追加し、trueならサイドバー
フォーカスを維持する。

**優先度: Med**（ユーザー自身のDM操作の詰まり方と直接関係。実装コストは小〜中）

---

## 5. 折りたたみセクションの集約未読バッジが「消えたDM」を勘定に入れず、未読が実在しないように見える

**症状**: `aggregateUnreadForSection()` (`internal/ui/sidebar/model.go:1063-1079`) は
`m.filtered`（=staleness後）だけを走査する。つまり未読があるのに30日ルールで非表示になった
DMは、折りたたみヘッダーの `•N` バッジにもカウントされない。
「未読があるはずなのにどこにも出ない」という体感的な不整合を生む
（項目1と組み合わせて発生：ただし `hasUnread` は staleness の免除条件でもあるので、
未読がある限りは基本的に filtered に残る。空のセクション丸ごと消失時のみ発生する境界ケース）。

**改善案**: 項目1の是正（デフォルトOFF化）で大部分は解消するが、機能を維持するなら
非表示件数を集約バッジ横に小さく出す。

**優先度: Low**（項目1が直れば実害はほぼ消える）

---

## 6. `Configuration.md` に載っていない設定キーが多数存在（ドキュメントとの乖離）

**症状**: `internal/config/config.go` の struct タグを全数確認したところ、Wiki
`Configuration.md` の例には出てこないキーが複数ある:
`[sidebar] hide_inactive_after_days` / `width`、`[appearance] show_avatars` /
`max_image_cols` / `mouse_wheel_lines` / `emoji_images` / `emoji_cells`、
`[animations] toast_transitions` / `message_fade_in`、`[notifications] notify_command`、
`[general] status_command`（README側には $SLK_TITLE 等の説明はあるが Configuration.md
には無い）。

**根本原因**: ドキュメントの更新がコードの機能追加ペースに追いついていない。
特に `hide_inactive_after_days` は項目1のとおり挙動への影響が大きいのに完全に欠落している。

**改善案**: `config.Default()` の struct タグを唯一の真実源として、Configuration.md の
フルサンプルを機械的に再生成 or 手動で追記する。

**優先度: Med**

---

## 7. Slackセクションが11件以上のチャンネルを含む場合、初回ロードで一部DMが「行方不明」になる

**症状**: `Configuration.md` 自身に書かれている既知の制約
（v1 limitations セクション）: 「10件を超えるチャンネルを持つセクションは初回ロードで
部分的にしか返らず、残りは catch-all バケツに一時的に落ちる」。

**根本原因**: `internal/service/sectionstore.go:52-68` の `Bootstrap()` に
`ChannelsCount > len(ChannelIDs)` の警告ログのみで、UIには一切通知が出ない
（SLK_DEBUGログにしか出ない = 一般ユーザーは気づけない）。DMが多いユーザーほど
発生しやすく、項目1のstaleness問題と誤診断が混同されやすい
（「DMが消えた」原因の切り分けが困難）。

**改善案**: 部分読み込みが発生したセクションについて、サイドバーのヘッダーに
一時的な「読み込み中」インジケータを出す（デバッグログではなくUI側で）。

**優先度: Med**（発生条件はDM数が多いユーザーに限られるが、まさに今回のユーザーの
利用実態=業務でSlackを日常的に使う=DM/チャンネル数が多いプロファイルに刺さりやすい）

---

## 8. カスタムキーバインド不可（README/Tradeoffsに明記済みだが再確認）

**症状**: `?`キーバインド一覧はハードコード。vim風モーダルに慣れていないユーザーが
既存の別ツールの筋肉記憶と衝突しても変更できない。

**根本原因**: `Tradeoffs-and-Non-Goals.md` に「on the roadmap」として明記されている
既知の未実装機能（コード側にも設定パーサ相当が無いことを確認済み）。

**改善案**: 対応はロードマップ通りでよいが、Configuration.md にも
「キーバインドは現状カスタマイズ不可」と明示し、README同様に期待値をユーザー側に揃える。

**優先度: Low**（既知・計画済みのため監査上の驚きは小さい）

---

## 9. Threads行の「未読」と個別チャンネルの「未読」でロジックが分岐しており、目視で気づきにくい二重管理

**症状**: `SetThreadsUnreadCount` はApp側から都度計算して渡される別経路
（`internal/ui/sidebar/model.go:601-615`）で、通常チャンネルの `IsVisiblyUnread`
とは独立した値。ミュート設定などの反映漏れがあると「Threadsは未読ゼロなのに
実際は返信がある」といった不整合が起きやすい構造（実際のバグ報告は今回未確認だが、
構造的リスクとして監査に含める）。

**改善案**: Threads行の未読カウントも `IsVisiblyUnread` 系のヘルパーと同じ入力
（read-state DB）から導出する単一責務にする。

**優先度: Low**（現状report済みバグではないため、リファクタ余地としての記録）

---

## 10. Enterprise Grid / トークン失効時のリカバリ導線がUI上で分かりにくい

**症状**: upstream Issue [#111](https://github.com/gammons/slk/issues/111)
「Failed to mint token: mint token: status 403」（Open）。ブラウザCookie認証の
仕様上、期限切れ時は `--add-workspace` を再実行する必要があるが
（`Tradeoffs-and-Non-Goals.md` の Auth caveat に記載）、アプリ内エラーメッセージから
その復旧手順への導線がない（README/Wikiを読みに行く前提）。

**改善案**: mint失敗時のエラートーストに「`slk --add-workspace` を再実行してください」
の一文を追加する。

**優先度: Med**（発生頻度は低いが、詰まった際の自己解決コストが高い＝今回のユーザー
プロファイル（オペレーション用途で日常的に使うが低レベルデバッグは苦手）に直撃しやすい）

---

## 11. サイドバー内の「セクションが空でヘッダーごと消える」設計が視覚的な急な変化を生む

**症状**: 項目1・5と関連。折りたたみ中のセクションが最後の1件まで staleness/未読変化で
消えると、ヘッダーごと消滅し、レイアウトが詰める（`modelOrderedSections` が
`m.filtered` の中身だけを見て動的にヘッダーリストを作るため）。ユーザー体感としては
「さっきまであったDirect Messagesセクションが急に無くなった」という驚きに繋がる
（今回の実体験「選べなくて詰まった」の一因として、"消える"体験自体が学習性の低いUIになっている）。

**改善案**: 直近開いていたセクション（`Direct Messages` など既定3種）は中身が0件でも
ヘッダーだけ残す（"No conversations" のグレー表示）。

**優先度: Low**（項目1が直れば発生頻度は大幅に下がるが、根本設計の脆さとして記録）

---

## 総括

- 検出項目数: **11件**（High 3 / Med 5 / Low 3）
- **DM選択問題の根本原因**: config のデフォルト値 `hide_inactive_after_days = 30`
  ＋ `internal/ui/sidebar/staleness.go` の「DM/group_dmはlastReadTSが空なら経過日数を問わず
  無条件stale」という設計（コード内コメントで「実運用DMの半分・グループDMの98%が該当」と
  明言）が組み合わさり、多くのDM/グループDMがサイドバーから**デフォルトで**消える。
  この挙動は Configuration.md に一切文書化されておらず、全消滅時にプレースホルダーも
  出ないため、ユーザーからは「DMが選べない/存在しない」ようにしか見えない。
  upstream Issue #88 と症状が一致するが、メンテナ自身が再現できず放置されている
  ＝コード上のデフォルト値のクセに起因する典型的な「再現条件が分からず塩漬け」パターン。
- 副次的に、Ctrl+N新規メッセージダイアログが既存グループDMの再オープンに対応しておらず
  (Issue #44)、消えたDMを見つける唯一の実用的な手段はCtrl+Tファジーファインダー
  （ただし未文書化）のみ、という悪い組み合わせになっている。
