# API設計案

画面を開くときは必要な情報をまとめて返し、記録するときは一つのプレーを一度に保存する。以下は自前のAPIの契約案で、稼働中のエンドポイントはまだない。IDは例示用の値。

## 画面とエンドポイント

| 画面操作 | メソッド・パス | 扱う内容 |
| --- | --- | --- |
| 試合を選ぶ・作る | `GET /games`、`POST /games` | 日付、大会、両チーム、球場。同日・同カードの複数試合を識別する |
| スタメンを入力する | `PUT /games/{id}/lineups` | 両チームの打順、守備位置、先発投手 |
| 入力を再開する | `GET /games/{id}/scoring-state` | スコア、回・表裏、アウト、走者、現在の打席ID、打者・投手、判明しているカウント、試合の `revision` |
| 投球・打席・走塁・交代を記録する | `POST /games/{id}/plays` | 同時に起きた動きを1件として保存し、更新後の状態を返す |
| プレーを確認・訂正・取り消す | `GET /games/{id}/plays`、`PATCH`・`DELETE /games/{id}/plays/{playId}` | 順序付き履歴。訂正による走者や後続プレーへの影響も検査する |
| 試合成績を確認・転記する | `GET /games/{id}/box-score`、`PATCH /games/{id}/box-score` | 最終得点、各回の得点、両軍の選手別打撃表・投手表。判明した項目のみ更新する |
| 試合結果を確定する | `POST /games/{id}/finalize` | 試合終了と結果を確定。個人成績の記録範囲は別に返す |
| 過去の年度合計を登録する | `PUT /seasons/{seasonId}/players/{playerId}/summaries/{scopeId}/{category}` | 大会・公式戦等の区分・所属範囲を指定し、打撃または投球の合計を保存 |
| 年度集計に使う記録を選ぶ | `PUT /players/{playerId}/season-stat-source` | シーズン、対象範囲、部門、採用元（試合の合算／年度資料）を指定 |
| 選手・チームの成績を見る | `GET /players/{id}/stats`、`GET /teams/{id}/stats` | 数値とともに、採用元・対象範囲・判明した試合数・不明項目を返す |

### 持ち球の取得と追加

1. 投手が決まったら `GET /pitchers/{pitcherId}/repertoire` で持ち球を取得する。
2. 「持ち球を追加」で共通の球種一覧（`GET /pitch-types`）から選び、`PUT /pitchers/{pitcherId}/repertoire/{pitchTypeId}` でその投手に紐づける。同じ組み合わせを再送しても1件に保つ。
3. 追加した球種を入力中の球に選択する。持ち球の追加ではカウントを進めず、投球の保存は「この球を記録」で行う。

球種名自体が共通一覧に存在しない場合に限り、`POST /pitch-types` で名称を登録してから紐づける。投球の球種が不明なら `pitchTypeId: null` を使う。

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

座標は捕手から見たゾーンの左上を `(0, 0)`、右下を `(1, 1)` とする案。ゾーン外の値も保存する。サーバーは入力時点の実投手・投球腕・実打者・打席左右を記録し、見逃しストライクから `swingDecision = TAKE` を設定する。スイングの有無だけが判明している過去資料では `swingDecision` を指定し、投球結果を不明にできる。

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
