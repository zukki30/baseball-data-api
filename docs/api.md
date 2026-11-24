# API設計案

初期はNPBの終了済み試合や過去資料を本人が手入力し、個人・チーム・リーグごとの閲覧・比較に使う。画面を開くときは必要な情報をまとめて返し、詳細を記録するときは一つのプレーを一度に保存する。以下は自前のAPIの契約案で、稼働中のエンドポイントはまだない。IDは例示用の値。

## 利用者と権限

全登録・更新・削除操作は認証済みの本人に許可する。本人が閲覧を許可した人には、データ取得・検索・比較を提供する。閲覧用の認証や共有方法は今後確定する。API側でも操作ごとの権限を検査し、画面のボタン表示と対応させる。

## 画面とエンドポイント

| 画面操作 | メソッド・パス | 扱う内容 |
| --- | --- | --- |
| 選手・チーム・リーグ・球場を選ぶ | `GET /players`、`GET /teams`、`GET /leagues`、`GET /venues` | 名称や対象年度等で候補を取得する |
| 試合を選ぶ・作る | `GET /games`、`POST /games` | 日付、大会・試合区分、両チームと当時の所属、球場、出典。終了済み試合を作り、同日・同カードの複数試合も識別する |
| スタメンを入力する | `PUT /games/{id}/lineups` | 両チームの打順、守備位置、先発投手 |
| 入力を再開する | `GET /games/{id}/scoring-state` | 入力済みプレーに基づくスコア、回・表裏、アウト、走者、入力中の打席ID、打者・投手、判明しているカウント、試合の `revision`、試合状態、記録状態 |
| 投球・打席・走塁・交代を記録する | `POST /games/{id}/plays` | 同時に起きた動きを1件として保存し、更新後の状態を返す |
| プレーを確認・訂正・取り消す | `GET /games/{id}/plays`、`PATCH`・`DELETE /games/{id}/plays/{playId}` | 順序付き履歴。訂正による走者や後続プレーへの影響も検査する |
| 試合成績を確認・転記する | `GET /games/{id}/box-score`、`PATCH /games/{id}/box-score` | 最終得点、各回の得点、両軍の選手別打撃表・投手表。判明した項目のみ更新する |
| 入力記録を確認済みにする | `POST /games/{id}/finalize` | 資料との照合が済んだ記録を確認済みにする。実際の試合状態と個人成績の記録範囲も返す |
| 過去の年度合計を登録する | `PUT /seasons/{seasonId}/players/{playerId}/summaries/{scopeId}/{category}` | 大会・公式戦等の区分・所属範囲を指定し、打撃または投球の合計と記録状態を保存 |
| 年度集計に使う記録を選ぶ | `PUT /players/{playerId}/season-stat-source` | シーズン、対象範囲、部門、採用元（試合の合算／年度資料）を指定 |
| 選手・チーム・リーグの成績を見る | `GET /players/{id}/stats`、`GET /teams/{id}/stats`、`GET /leagues/{id}/stats` | 数値とともに、採用元・対象範囲・判明した試合数・不明項目を返す |
| 対象を並べて比較する | `GET /stats/batting`、`GET /stats/pitching`、`GET /stats/pitches` | 共通条件で複数の選手・チーム・リーグを取得し、年度ごとの比較も行う |

実際の試合が終了済みでも転記途中の記録を保存できるよう、`gameStatus` と `recordingStatus` を返す案とする。記録状態の候補は `DRAFT` と `CONFIRMED`。本人は確認済みの記録も訂正でき、訂正後の記録を再確認する。プレーの登録・訂正・確認済みへの変更には、現在の `revision` を使って競合を検査する。年度合計にも `recordingStatus` を持たせ、年度合計登録APIで保存する。

## 絞り込みと比較

以下のパラメーターと返却項目は、比較画面に向けた案。個別の成績取得APIと比較APIは、同じ集計処理と採用元の規則を使う。

| 条件 | パラメーター案と意味 |
| --- | --- |
| 対象と年度 | `organization=NPB`、`groupBy=PLAYER\|TEAM\|LEAGUE`、対応するID一覧、`seasonIds`。複数年度では年度ごとの行を返す |
| 期間・大会 | `dateFrom`、`dateTo`、`competitionIds`、`gameTypes`。判明している記録の粒度に応じて適用する |
| 所属 | `teamIds`、`leagueIds`。打撃では打者側、投球では投手側の所属を指す |
| 対戦相手 | `opponentTeamIds`、`opponentLeagueIds`。対象側の所属と併用できる |
| 投球分析 | `perspective=BATTING\|PITCHING`、`pitchTypeIds`、`batterSide`、`throwingHand`、カウント、コース、`isBorderline`、スイング・投球結果 |
| 記録範囲 | `recordingStatus`、`pitchCoverage`。比較対象とした記録状態や投球の網羅性を明示する |

例えば投手2人の左打者へのスライダーを比較する場合、次のように投球分析APIへ指定する。ID一覧はカンマ区切りとする案で、例のIDは説明用。

`GET /stats/pitches?organization=NPB&perspective=PITCHING&groupBy=PLAYER&playerIds=pitcher-9,pitcher-10&seasonIds=season-2025&pitchTypeIds=slider&batterSide=L`

投球分析の `perspective` は、個人・チーム・リーグの集計対象が打者側か投手側かを指定する。球種・コース等の条件は同じ投球に適用する。打撃・投手成績のAPIでは、各指標に適用できる期間や所属等の条件を受け付ける。条件とエンドポイントの組み合わせを検査し、未対応の条件には入力エラーを返す。

返却する各行には対象ID・年度・指標値を持たせ、適用条件、分子・分母、採用元、対象件数、不明件数、記録の範囲も返す。年度合計に日付や対戦相手の内訳がない場合、その条件に使えた詳細記録の範囲と、条件を適用できなかった資料の範囲を明示する。該当する詳細記録がなければ検索に一致した記録件数を0、実績が不明な成績項目と未算出の率を `null` として返す。条件付き分析に使った資料の範囲を明示し、年度全体の表示に使う採用元は既存の設定に従う。

リーグ比較では、打撃側と投球側それぞれの当時の所属リーグを使う。入力済みの一部記録を集計した値には、その範囲が分かる情報を付ける。試合数と率の計算方法は[データモデル](./data-model.md)に従う。

### 持ち球の取得と追加

1. 投手が決まったら `GET /pitchers/{pitcherId}/repertoire` で持ち球を取得する。
2. 「持ち球を追加」で共通の球種一覧（`GET /pitch-types`）から、資料で使用を確認した球種を選び、`PUT /pitchers/{pitcherId}/repertoire/{pitchTypeId}` でその投手に紐づける。同じ組み合わせを再送しても1件に保つ。
3. 追加した球種を入力中の球に選択する。持ち球の追加ではカウントを進めず、投球の保存は「この球を記録」で行う。

球種名自体が共通一覧に存在しない場合に限り、`POST /pitch-types` で名称を登録してから紐づける。投球の球種が不明なら `pitchTypeId: null` を使う。

追加・修正は本人の権限で行う。使用確認の出典と対象年・試合を持ち球に関連付け、取得時に表示できる案とする。元の投球記録とその出典も参照できるようにする。

## 1球を記録する例

3回表、右投手が投じたスライダーを打者が見送った。記録者にはゾーンのわずかに外に見えたが、実際の判定はストライクだった。

`POST /games/game-001/plays`

```json
{
  "type": "PITCH",
  "plateAppearanceId": "pa-024",
  "clientPlayId": "2d6f36ec-8675-4f9c-b936-835af046a363",
  "expectedRevision": 42,
  "pitch": {
    "pitchTypeId": "slider",
    "location": { "source": "MANUAL", "x": 1.02, "y": 0.85 },
    "result": "CALLED_STRIKE"
  }
}
```

座標は捕手から見たゾーンの左上を `(0, 0)`、右下を `(1, 1)` とする案。ゾーン外の値も保存する。サーバーは試合内の該当時点の実投手・投球腕・実打者・打席左右を記録し、見逃しストライクから `swingDecision = TAKE` を設定する。スイングの有無だけが判明している過去資料では `swingDecision` を指定し、投球結果を不明にできる。

`GET /games/game-001/plays?isBorderline=true` で取得する1件の例。上の座標を境界付近とする設定を想定している。

```json
{
  "id": "play-043",
  "type": "PITCH",
  "plateAppearanceId": "pa-024",
  "pitch": {
    "pitcherId": "pitcher-9",
    "throwingHand": "R",
    "batterId": "batter-3",
    "batterSide": "R",
    "pitchTypeId": "slider",
    "location": {
      "source": "MANUAL", "x": 1.02, "y": 0.85,
      "isBorderline": true, "zoneSide": "OUTSIDE"
    },
    "swingDecision": "TAKE",
    "result": "CALLED_STRIKE"
  }
}
```

| 分かっていること | 登録・取得時の扱い |
| --- | --- |
| タップ位置がある | 原座標を保存。サーバーで境界付近かと内外を算出する |
| きわどいことだけ分かる | `x: null, y: null, isBorderline: true, zoneSide: null`。境界付近の検索に含む |
| コースの記録がない | `isBorderline: null`。境界付近にも、境界から離れた球にも数えない |

球種・投球腕・コース・スイングを組み合わせた検索では、一つの投球に全条件を適用する。同じ打席の別々の球が条件を一つずつ満たしたケースは該当しない。

## 打席の確定・訂正・過去資料

- **同じ球で複数のことが起きた場合**は、1件のプレーに投球、任意の打席結果、各走者の移動を含める。最終球と打席結果を結び、得点や打席数を一度だけ反映する。投球明細がない打席の結果も単独で登録できる。
- **打席中の走塁・交代**は既存の打席に紐づける。単独の盗塁や継投で打席数を増やさない。投球と同時の盗塁は、その投球と同じプレーで扱う。
- **二重送信と編集競合**は役割を分ける。同じ試合・同じ `clientPlayId`・同じ内容の再送には前回の保存結果を返す。新しい操作の `expectedRevision` が古ければ、最新状態の再取得を促す。過去の記録を変えた結果、後続の盗塁などが成立しなくなる場合は矛盾のあるプレーを示す。
- **成績表の部分更新**では、省略した項目は現在の値を保持し、明示した `null` は不明、`0` は確認したゼロを意味する。打席から集計する項目と資料から直接採用する項目の扱いは[データモデルと集計](data-model.md)に従う。
