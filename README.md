# SRT を活用したリモート配信システムレシピ「ビューイング配信版」

## 0. 前提

新型コロナの影響で、リモートワークが増えており、配信もリモートで行う機会が増えていると思います。
本レシピでは、ビューイング配信を行う上で、リモートから全出演者が参加する場合の配信システムを構築したいと思います。
SRT という聞きなれないプロトコルを駆使することで、低遅延かつ高品質で信頼性の高いシステムを目指します。

尚、DISCORD、OBS Studio、vMix、その他配信に関する基本的な知識、PCの知識があることを前提とします。

## 1. 要件定義

1. 出演者はクリーンフィードを視聴しながらコメントする。
2. 出演者が 2 名。それぞれリモートから配信に参加し、お互いにボイスコミュニケーションが可能とする。
3. 配信オペレーターは 1 名、出演者とは別の場所（スタジオ）で配信を実施する。
4. 配信オペレーターからトークバックで出演者に指示を出せる。トークバックは配信には乗せないこと。
5. 出演者 2 名は顔出し。それぞれから Webcam 映像（＋マイク音声）を受け取って、スタジオで合成して配信する。
6. 出演者 2 名に、クリーンフィードの絵音を遅延なく返す。
7. 出演者 2 名は自分の配信機材を使用する。運営から特別な機材を用意しない。

## 2. 用意するもの

### 2.1. スタジオ側

* [DISCORD（サーバーも）](https://discord.com/)
* [vMix（HD エディション以上）](https://www.vmix.com/)
* [PARSEC](https://parsec.app/) と、PARSEC を動かす PC （出来ればキャプボ付き、無ければ [NDI tools](https://www.ndi.tv/tools/)導入）
* ポートフォーワーディング可能なポート番号 2 つ
* パスワード（10 文字以上 79 文字以下）
* アップロード・ダウンロード共に　100 Mbps 以上出る光インターネット回線

### 2.2. 出演者側

* [DISCORD](https://discord.com/)
* [OBS Studio](https://obsproject.com/)
* [PARSEC](https://parsec.app/)
* Webcam
* その他、出演者がいつも使ってる配信機材
* アップロード・ダウンロード共に　50 Mbps 以上出る光インターネット回線

## 3. システム図

![Picture.2](images/remotework002.jpg?raw=true)

## 4. SRT とは

まず、本レシピの中核をなす「SRT」というプロトコルについて説明します。

SRT とは、Secure Reliable Transport の略で、セキュアで信頼性のある伝送路を目的としたストリーミングプロトコルです。
TCP伝送（信頼性）やUDP伝送（低遅延）の良いとこ取りを目指し、エラーリカバリやパスワード保護・暗号化等の機能を追加した物で、
更に低遅延であるという特徴を有しています。

SRT は、RTMP と違ってまだまだマイナーで、対応している配信プラットフォームは **未だありません**。
もう一度言います **未だ1つもありません**。

「そんなもん何に使えるんや」という疑問はもっともですが、実は今回みたいな要件には結構使えるんです。
今回は、システム図で示した通り、出演者2名の自宅からスタジオまで絵音を伝送するのに使います。

SRT を使う理由は、以下の通りです。

* ストリーミング（中継）サーバーがいらない。P2P で直接 vMix へ送信できる（ポートは開ける必要あり）
* パスワード設定と暗号化が出来て安全。
* 遅延が少ないか、予測可能である。通常は 1 秒以下に収まる。
* 安定かつ伝送品質が（TS や RTMP に比べて）マシ。
* OBS Studio が対応している（重要）

SRT には、3つのモード(mode というパラメータで指定）があります。

| mode           | 動作                                        |
| -------------- | ----------------------------------------- |
| **caller**     | ストリームを送信する                                |
| **listener**   | ストリームを受信する                                |
| **rendezvous** | ハンドシェイクによって caller となるか listener となるかが決まる |

デフォルトは caller モードになるため、送信側はパラメータを省略可能です。
今回は OBS Studio を送信側エンコーダーとして使用します。
他には下記のパラメータがあります。

| パラメータ          | 内容                                                                                                                                                                                                                                                |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **passphrase** | 暗号化パスワード。10 文字以上 79 文字以下。（任意）                                                                                                                                                                                                                                      |
| **latency**    | 遅延をマイクロ秒で指定。大きいほどパケロスに対する強度が増し、安定した伝送が可能になります。送信側、受信側双方で同じ時間を設定します。諸説有りますが vMix では少なくとも ping 値の 4 倍以上が必要とされています（例： ping 値が 30ms なら 120ms 以上にする）。デフォルトは 120ms（120000μs）です。全体の遅延は、エンコーダーとデコーダーのオーバーヘッドに依存しますが、通常は両端の latency 値を合計したよりも短い時間で収まります。 |
| **pbkeylen**   | 暗号化キーの長さ。0,16,24,32 のいずれかを指定できます。0 にすると暗号化無しになります（デフォルト）。送信側のみの設定で受信側は必要がありません。また、passphrase を使用する場合は 0 以外にしなければなりません。                                                                                                                            |

[SRT の詳細はこちらのページに分かりやすくまとまっています](https://obsproject.com/wiki/Streaming-With-SRT-Protocol)

## 5. PARSEC とは

さて、ビューイング配信では出演者が遅延なくクリーンフィードを視聴出来る環境が重要です。
その為、本レシピでは PARSEC という画面共有ツールを使用します。

PARSEC は本来、オフライン対戦（1 PC対戦）しか出来ないゲームを、インターネットを介してマルチプレイするためのソフトで、クラウドゲーミングの様に少ない遅延で画面共有しつつ、リモートからゲーミングコントローラーで操作できるという物です。今回は「少ない遅延で画面共有」という部分を、出演者がクリーンフィードを視聴するために活用します。

[PARSEC はこちらで入手できます（無料）](https://parsec.app/)

> #### PARSEC がうまく機能しなかった場合は？
>
> 代用策としては、DISCORD の画面共有やその他のビデオ通話アプリがありますが、画質が悪いデメリットがあります。
> DISCORD は課金してサーバーブーストを行えばある程度品質を上げられます。
>
> その他にも代用策は有りますが、長くなるのでここでは紹介を割愛します。

## 6. スタジオ側の仕込み

### 6.1. DISCORD の設定

お互いに音声通話しながら仕込みを行うでしょうから、まず DISCORD から準備してください。

[こちらから DISCORD をダウンロード](https://discord.com/)してインストールし、アカウントを作成してログインしてください。
また、専用のサーバーを作成して接続し、招待リンクを生成して出演者側に知らせてください（出演者にサーバーへ接続してもらう）
最後に、音声通話できる様、音声デバイスを設定してボイスチャンネルに参加してください。

尚、配信用の出演者音声は、SRT で送られてきますので、DISCORD の音声は vMix に取り込む必要はありません。

### 6.2. ポートフォーワーディングの設定

次に、SRT を受けるポート番号を2つ決めてください。
ここでは `10001`, `10002` とします。

ポート番号を決めたら、SRT を受信する PC に対して UDP でポートフォーワーディングを設定してください。
ポートフォーワーディング設定の仕方は、お使いのブロードバンドルーターのマニュアルを参照するか、ネットワーク担当者に問い合わせてください。

### 6.3. vMix の設定

vMix に SRT を受信する Input を作成します。
「Add Input」→「Stream / SRT」を開いてください

| 項目                   | 設定値                           |
| -------------------- | ----------------------------- |
| Stream Type          | SRT (Listener)                |
| Port                 | 10001                         |
| Latency (ms)         | 120 ※ここだけ単位がミリ秒なので注意          |
| Decoder Delay (ms)   | 0                             |
| Passphrase           | 10文字以上79文字以下のパスワード。出演者側の OBS Studio に同じものを設定 |
| Key Length           | 変更不要                          |
| Stream ID            | 空欄のまま                         |
| Use Hardware Decoder | チェック入れる                       |

![Picture.2](images/remotework003.jpg?raw=true)

同じ手順で、もう1つポート番号を「10002」に変えて作成してください。

### 6.4. PARSEC の設定

PARSEC は必ず vMix 用とは別の PC で動作させてください。
その場合、何らかの方法でクリーンフィードを PARSEC PC に映し出す必要があります。オプションとしては

1. PARSEC PC にキャプボを刺して、vMix から出力したクリーンフィードを OBS Studio で取り込んでプレビュー<br />
   **メリット:** クリーンフィード以外も出演者に返せる（例: 配信 Output）<br />
   **デメリット:** システムが複雑で必要機材が増える<br />
2. NDI を使用して、vMix から出力したクリーンフィードを取り込む。NDI のプレビューには [NDI Tools（無料）](https://www.ndi.tv/tools/) を使用すれば OK<br />
   **メリット:** クリーンフィード以外も出演者に返せる（例: 配信 Output）。システムがシンプルで準備が二番目に楽<br />
   **デメリット:** vMix PC のリソースを食う。wifi 経由では帯域がきついので有線LAN必須。<br />
3. PARSEC PC でクリーンフィードを再生して絵音を、vMix に取り込む<br />
   **メリット:** システムがシンプルで準備が一番楽<br />
   **デメリット:** クリーンフィード以外を出演者に返せない<br />

の3つが挙げられますので、お好きな方法を選択してください。

[PARSEC をこちらからダウンロードしてインストールしてください。](https://parsec.app/)
また、ホストの ID（`username#1234` という形式）を出演者側に伝えて、フレンド登録を行ってください。

次に「⚙Settings」→「Host」で以下の設定を行ってください。

| 項目                   | 設定値                  |
| -------------------- | -------------------- |
| Hosting Enabled      | Enabled              |
| Resolution           | Keep Host Resolution |
| Bandwidth            | 10 Mbps              |
| FPS                  | 60                   |
| Exclusive Input Mode | 変更不要                 |
| Display              | 取り込むディスプレイを選択        |
| Echo Cancelling      | 変更不要                 |
| Virtual Gamepad Type | 変更不要                 |
| Machine Level User   | 変更不要                 |

![Picture.3](images/remotework004.jpg?raw=true)

![Picture.4](images/remotework005.jpg?raw=true)

## 7. 出演者側の仕込み

### 7.1. DISCORD の設定

お互いに音声通話しながら仕込みを行うでしょうから、まず DISCORD から準備してください。

[こちらから DISCORD をダウンロード](https://discord.com/)してインストールし、アカウントを作成してログインしてください。
DISCORD のインストール先は OBS Studio と同じ PC で構いません。

次に、スタジオ側から送られてきた招待リンクを開いてサーバーに接続し、ボイスチャンネルに参加してください。
また、音声通話できるように音声デバイスを設定しましょう。

DISCORD は出演者間のボイスコミュニケーションと、スタジオからのトークバックにのみ使用します。原則として DISCORD の音声を配信にミックスしたりはしません。理由としては音質の問題です。

出演者の音声は前述の SRT で、Webcam 映像と共にスタジオまで送信します。このことで、品質を確保し、映像と音声の同期（リップシンク）を取る手間を省きます。

### 7.2. OBS Studio の設定

使用する OBS Studio のインスタンスは 1 つです。
今回は顔出し配信を想定していますから、Webcam の映像とマイク音声をスタジオに送信するために、OBS Studio に Webcam と マイク入力を追加してください。

![Picture.5](images/remotework007.jpg?raw=true)

次に配信の設定です。RTMP の代わりに SRT というプロトコルを使用します。

1. OBS Studio の「設定」→「配信」→「サービス」を「カスタム...」に変更
2. 「サーバー」に以下のフォーマットで送信先 URL を記入<br />
   srt://`アドレス`:`port番号`?mode=caller&passphrase=`パスワード`&latency=`レイテンシ(μsec)`&pbkeylen=`暗号化キーの長さ`<br />
   例: `srt://192.168.0.10:10001?mode=caller&passphrase=enjoysrt&latency=120000&pbkeylen=16`<br />

![Picture.6](images/remotework008.jpg?raw=true)

3. OBS Studio の「設定」→「出力」→「配信」を以下の様に設定

   | 項目                | 設定値                        |
   | ----------------- | -------------------------- |
   | 映像ビットレート          | 妥当なビットレート。例: `6000 Kbps`   |
   | エンコーダ             | 「ハードウェア」推奨                 |
   | 音声ビットレート          | 妥当なビットレートを指定。例： `160 Kbps` |
   | 高度なエンコーダの設定を有効にする | チェック入れる                    |
   | エンコーダのプリセット       | 「Low-Latency」推奨            |
   | エンコーダのカスタム設定      | 空欄のまま                      |

![Picture.7](images/remotework009.jpg?raw=true)

4. 「映像」→「出力（スケーリング）解像度を、適切な解像度に変更。例： `1920x1080`<br />
   「FPS共通値」とし、適切なフレームレートに変更。例： `60`

![Picture.8](images/remotework022.jpg?raw=true)

最後に、「音声ミキサー」で「マイク入力」以外をミュートしてください。
**「デスクトップ音声」を取り込まない様に注意してください**。
そうしないと、DISCORD の音声が配信に流れてしまいます。トークバックが配信に乗るのと、出演者の声が2重になる問題があるので、**必ずOBS Studio上でデスクトップ音声をミュートしてください**。

![Picture.9](images/remotework006.jpg?raw=true)

### 7.3. 疎通確認

OBS Studio の設定が済んだら、「配信開始」を押して、映像と音声がスタジオ側の vMix に送信されることを確認してください。

![Picture.10](images/remotework010.jpg?raw=true)

絵音が来ない場合は OBS Studio かスタジオ側の vMix のどちらかの設定がおかしい可能性があります。

送信されていても映像が乱れる場合は、出演者宅とスタジオ間の通信状況が良くないので、以下の対策を講じてください。

1. 映像ビットレートを下げる（トレードオフとして画質が悪くなります）
2. latency の値を増やす（トレードオフとして遅延が増えます）<br />
   latency を増やす場合は、全出演者側の OBS Studio とスタジオ側の vMix とで同一の値に設定してください。

通信の状況は vMix ウインドウ右下の「📊Statistics」→「SRT」で確認できます。
以下の RTT が ping 値になりますので、これをもとに Latency の値を見直してください。Packet Loss(%) が　1 % を超える場合は、Latency 値を更に増やす必要があります。

Latency 値は RTT（Ping）値の倍数で設定してください。

![Picture.11](images/remotework024.jpg?raw=true)

![Picture.12](images/remotework023.jpg?raw=true)

### 7.4. PARSEC の設定

[PARSEC をこちらからダウンロードしてインストールしてください。](https://parsec.app/)

PARSEC の使用には、アカウント作成とログインが必要になりますので、予め出演者全員に作成してもらい、ログインしてホスト用アカウントとフレンド登録を行ってください。
「👥Friends」→「Add Friend」で、ホスト用アカウントのID（ `username#1234` という形式）を入力すればフレンド登録申請が飛びますので、ホスト用アカウントの方で「👥Friends」→「View Friend Requests」から「✔Accept」を行ってください。

また、「⚙Settings」で設定を以下の様に変更します。

| 項目                    | 設定値                                                    |
| --------------------- | ------------------------------------------------------ |
| Overlay               | On ・・・ PARSECオーバーレイを表示する                               |
| Window Mode           | Windowed ・・・ ウィンドウモードにする                               |
| Decoder               | ハードウェア（NVIDIA, INTEL or AMD）を推奨。動作がおかしければ Software に設定 |
| Renderer              | Direct3D 11 のまま                                        |
| VSync                 | On 推奨                                                  |
| H.265                 | On 推奨。動作がおかしければ Off に設定                                |
| Decoder Compatibility | Off 推奨。動作がおかしければ On に設定                                |
| Immersive Mode        | Off のまま                                                |
| Discord Status        | On or Off どちらでも                                        |

![Picture.13](images/remotework011.jpg?raw=true)

![Picture.14](images/remotework012.jpg?raw=true)

> #### リモートでセットアップする場合
>
> 出演者の環境を配信オペレーターがリモートでセットアップを行う場合も、PARSEC が使えます。
> ただし、PARSEC の無料版は 1 画面しか取り込めないので、マルチモニター環境では「⚙Settings」→「Host」→「Display」で共有するモニターを選択する必要があります。
> 多くの出演者（ストリーマー）はマルチモニターでしょうから、「ウインドウがどっか行ってしまう問題」が発生して不便なら Chrome Remote Desktop や VNC と言った他のリモートデスクトップアプリを使用しましょう。Windows 標準のリモートアシスタンスも遠隔操作に使えますが、PC の所有者が在席している前提の設計なので注意してください。
>
> 尚、PARSEC も有料版の「PARSEC Warp」なら 2 画面まで共有できます。

設定が完了したら、出演者側の PARSEC からホストに接続してください。
「💻Computers」にホスト PC が表示されるので、「Connect」で接続申請します。

![Picture.15](images/remotework013.jpg?raw=true)

ホスト側では接続を申請してきたクライアントが、画面下に表示されますので、メニューから Accept を行ってください。

![Picture.16](images/remotework014.jpg?raw=true)

画質が十分でない場合は、ホスト側の設定で Bandwidth を増やしてください。
逆にカク付いたり止まったりする場合は、出演者宅とスタジオ間の通信状況が良くないので、Bandwidth を減らしてください。最低設定は 5 Mbps です。

## 8. 注意点

SRT の latency は全ての OBS Studio で同一の値を使ってください。
また、vMix 側でも同一の時間に設定してください。

つまり、1か所の OBS Studio で latency を増やした場合は、他の OBS Studio も同様に増やさなければならず、vMix 側も同様の時間に設定します。

そうすることにより、絵音のタイミングが大体揃います。
