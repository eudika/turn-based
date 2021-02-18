# Tic-tac-toe API

勉強会にてゲームAIを扱うことになったので、各プレイヤーを仲介するサーバーを実装する。

参加者の希望により、通信はHTTPを双方向に飛ばすことで行う。


## フロー

### マッチングフェーズ

1. クライアント1は、サーバーの[マッチングAPI](#マッチングAPI)にリクエストを送る。(R1)
2. サーバーは、クライアント1の[テストAPI](#テストAPI)にリクエストを送る。(R2)
3. クライアント1は、R2にレスポンスを返す。
4. クライアント2は、サーバーの[マッチングAPI](#マッチングAPI)にリクエストを送る。(R3)
5. サーバーは、クライアント2の[テストAPI](#テストAPI)にリクエストを送る。(R4)
6. クライアント2は、R4にレスポンスを返す。
3. サーバーはR1, R3にレスポンスを返す。
8. ゲームフェーズに移行する。

### ゲームフェーズ

1. サーバーは、手番のクライアントの[次手API](#次手API)にリクエストを送る。(R5)
2. クライアントは、タイムアウト以内に、R5にレスポンスを返す。
3. サーバーは、ゲーム状態を更新し、決着が着いているなら結果フェーズに移行する。
   着いていないなら、手番を入れ替えて1.に戻る。

### 結果フェーズ

1. サーバーは、クライアント1, 2の[結果API](#結果API)にリクエストを送る。(R6)
2. クライアント1, 2は、R6にレスポンスを返す。

## API 一覧

### マッチングAPI

#### リクエスト

* URI: `{サーバー側URIルート}/lobby`
* メソッド: POST

| キー                     | 型                  | 値                                 |
| ------------------------ | ------------------- | ---------------------------------- |
| `player`                 | [`Player`](#player) | プレイヤー情報（ただし`id`は不要） |
| `communication.type`     | `string`            | `"webhook"`固定                    |
| `communication.uri_root` | `string`            | クライアント側APIのURIルート       |
| `filter`                 | `object`            | 空（ネゴシエーション用）           |

例
```json
{
    "player": {
        "name": "My AI",
        "version": "1.0.0",
        "author": "Me"
    },
    "communication": {
        "type": "webhook",
        "uri_root": "http://192.168.1.100:8080/ttt"
    },
    "filter": {
    }
}
```

#### レスポンス

| ステータスコード | 説明                                                       |
| ---------------- | ---------------------------------------------------------- |
| 200              | OK. マッチ開始。                                           |
| 400              | Bad Request. リクエストの形式違反。あるいはAPIが通らない。 |
| 408              | Timeout. 一定時間以内に対戦相手が現れなかった。            |

##### 成功時

| キー         | 型                    | 値                         |
| ------------ | --------------------- | -------------------------- |
| `players[i]` | [`Player`](#player)   | プレイヤー情報             |
| `match.id`   | `string`              | この対戦のID               |
| `match.rule` | [`TttRule`](#tttRule) | ゲームルール               |
| `you.id`     | `string`              | クライアントのプレイヤーID |

例
```json
{
    "players": [
        {
            "id": "#1",
            "name": "My AI",
            "version": "1.0.0",
            "author": "Me"
        },
        {
            "id": "#2",
            "name": "Another AI",
            "version": "2.3.4",
            "author": "Other"
        }
    ],
    "match": {
        "id": "#m001",
        "rule": {
            "game": "ttt",
            "timeout": 30,
            "first": "#1",
            "marks": {
                "blank": " ",
                "#1": "o",
                "#2": "x"
            }
        }
    },
    "you": {
        "id": "#1"
    }
}
```

##### 失敗時

[`Error`](#error)


### テストAPI

#### リクエスト

* URI: `{クライアント側URIルート}/test`
* メソッド: GET

#### レスポンス

| ステータスコード | 説明 |
| ---------------- | ---- |
| 200              | OK.  |


### 次手API

#### リクエスト

* URI: `{クライアント側URIルート}/next`
* メソッド: GET

| キー                | 型                      | 値                                |
| ------------------- | ----------------------- | --------------------------------- |
| `state`             | [`TttState`](#tttState) | ゲーム状態                        |
| `hint.your_mark`    | `string`                | 自身の記号                        |
| `hint.available[i]` | `number`                | 可能な手（`state.table`の添え字） |

例

```json
{
    "state": {
        "phase": 4,
        "in_turn": "#1",
        "table": [
            "o", " ", " ",
            " ", "o", " ",
            "x", " ", "x"
        ]
    },
    "hint": {
        "your_mark": "o",
        "available": [
            1, 2, 3, 5, 7
        ]
    }
}
```

#### レスポンス

| ステータスコード | 説明                 |
| ---------------- | -------------------- |
| 200              | OK.                  |
| 500              | エラー。降参。棄権。 |

##### 成功時

[`NextMove`](#nextMove)

例

```
{
    "next": 7
}
```

##### エラー時

[`Error`](#error)


### 結果API

#### リクエスト

* URI: `{クライアント側URIルート}/result`
* メソッド: POST

| キー     | 型                      | 値                 |
| -------- | ----------------------- | ------------------ |
| `state`  | [`TttState`](#tttState) | 最終的なゲーム状態 |
| `result` | [`Result`](#result)     | 結果               |

例
```json
{
    "state": {
        "phase": 7,
        "in_turn": "#2",
        "table": [
            "o", "o", "x",
            " ", "o", " ",
            "x", "o", "x"
        ]
    },
    "result": {
        "winners": ["#1"],
        "losers": ["#2"],
        "you": {
            "abstract": "1",
            "code": "100",
            "message": "You win"
        }
    }
}
```

#### レスポンス

| ステータスコード | 説明                 |
| ---------------- | -------------------- |
| 200              | OK.                  |


## オブジェクト一覧

### `Player`

| キー      | 型       | 値                         |
| --------- | -------- | -------------------------- |
| `id`      | `string` | 対戦内で有効なプレイヤーID |
| `author`  | `string` | 作者名                     |
| `name`    | `string` | プレイヤー名               |
| `version` | `string` | バージョン                 |

### `TttRule`

| キー          | 型       | 値                      |
| ------------- | -------- | ----------------------- |
| `type`        | `string` | `"Tic-tac-toe"` 固定    |
| `first`       | `string` | 先手のプレイヤーID      |
| `timeout`     | `number` | 次手APIのタイムアウト秒 |
| `marks[pid]`  | `string` | プレイヤーIDごとの記号  |
| `marks.blank` | `string` | 空白記号                |

### `TttState`

| キー      | 型         | 値                         |
| --------- | ---------- | -------------------------- |
| `phase`   | `number`   | 何ターン目か               |
| `in_turn` | `string`   | 手番のプレイヤーID         |
| `table`   | `string[]` | 盤面状態（記号）           |

### `Result`

| キー           | 型         | 値                             |
| -------------- | ---------- | ------------------------------ |
| `winners`      | `string[]` | 勝者のプレイヤーID一覧         |
| `losers`       | `string[]` | 敗者のプレイヤーID一覧         |
| `you.abstract` | `string`   | `you.code`の1文字目            |
| `you.code`     | `string`   | 結果コード（下表参照）         |
| `you.message`  | `string`   | `you.code`に対応するメッセージ |

| `code`  | `message`                                         |
| ------- | ------------------------------------------------- |
| `"000"` | `"Draw"`                                          |
| `"100"` | `"You win"`                                       |
| `"110"` | `"You win because of some fault of the opponent"` |
| `"111"` | `"You win because of connection"`                 |
| `"112"` | `"You win because of timeout"`                    |
| `"113"` | `"You win because of bad response"`               |
| `"200"` | `"You lose"`                                      |
| `"210"` | `"You lose because of some fault"`                |
| `"211"` | `"You lose because of connection"`                |
| `"212"` | `"You lose because of timeout"`                   |
| `"213"` | `"You lose because of bad reponse"`               |
| `"3**"` | `"Error"`                                         |


### `NextMove`

| キー      | 値                                  |
| --------- | ----------------------------------- |
| `next`    | 次手（`state.table`の添え字で表現） |

### `Error`

| キー      | 値                         |
| --------- | -------------------------- |
| `message` | エラーメッセージ           |


## 拡張

* 簡単のためTic-tac-toeから始めたが、オセロや将棋といった、
  ターン制のゲームをある程度包括できる仕組みを目指す。
* `communication.type`は以下の値を選択可能にする。
    - `"commet"`
    - `"poll"`
    - `"webhook"`
* WebSocket版も用意したい。
    - こちらは`communication.type`ではなくマッチングAPIのURLで区別。
* 認証の仕組み。
