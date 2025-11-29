---
title: 【C++/OpenSiv3D】YouTube Data API を使ってライブのチャットを取得・表示する
tags:
  - C++
  - YouTubeAPI
  - Siv3D
  - YoutubeDataAPIv3
  - OpenSiv3D
private: false
updated_at: '2021-12-18T12:17:58+09:00'
id: 6257c2b144dcb58e1245
organization_url_name: null
slide: false
ignorePublish: false
---
# 動機

v0.6 から HTTP クライアント機能である `SimpleHTTP` が実装されました。
これのおかげで HTTP 通信関係の処理がかなり簡単に扱えるようになったわけです。

これを聞いて、ふとこう思いました。
> 💭これと YouTube の API を使えば、配信のチャットを取得して画面に流せるのでは...？

ということでいろいろと試行錯誤して実装したので今回はこれについて書こうと思います。

::: note info
誤字や改善点等があればコメントや編集リクエストからお願いします！
:::

# YouTube Data API って？
https://developers.google.com/youtube/v3/docs

YouTube が提供している API で、チャンネルや動画の情報を取得することができます。

:::note warn
Queries という一日あたりの API 使用量の上限があり、無料枠だと上限が10,000なので注意してください。
使った量などの詳細はダッシュボードから確認出来ます。
:::


# 準備

まず、API を利用するために準備が必要です。

:::note warn
2021年12月現在時点でのやり方です
:::

## Google Cloud Platform で プロジェクトを作成

[Google Cloud Platform](https://console.cloud.google.com/) にアクセスします。
初めて利用する場合は利用規約に同意し、チェックボックスにチェックを入れて同意して続行をクリックします。

その後、「プロジェクトの選択」から「新しいプロジェクト」を押し、プロジェクトを作成します。プロジェクト名を適当に決めたら「作成」をクリックして完了です。

![HowTo1.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/464356/617e60f0-8282-beb8-a16c-f4c7766f0b9c.gif)


## YouTube Data API v3 の有効化

プロジェクトが作成できたら、左上のメニューから「API とサービス」、「ライブラリ」の順に進みます。その後、「YouTube Data API v3」を選択し、「有効にする」をクリックします。
![HowTo2.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/464356/1e557356-ac14-1631-e1fc-af0b083c8752.gif)

## APIキーの作成

「認証情報」タブから「+認証情報を追加」、「API キー」の順にクリックします。

![HowTo3.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/464356/3a4a9f5a-debd-e1e0-5203-a802d50a3ce3.gif)

これで準備は完了です！

# チャットを取得して描画してみる

YouTube Data API を使ってライブのチャットを扱うためのクラス `YouTubeLiveChat` を作りました。
これを使えば簡単にライブチャットを取得し、処理することができます。

```cpp:Main.cpp
# include <Siv3D.hpp> // OpenSiv3D v0.6.3
# include "YouTubeLiveChat.hpp"

# define VIDEO_ID U"xxxxxxxxxxx"  // URL の watch?v= 以降の文字列
# define API_KEY U"xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"  // 取得した API キー

struct ChatEffect : IEffect
{
    int32 m_y;
    String m_message;

    explicit ChatEffect(const String& message)
        : m_message{ message }
        , m_y{ Random(Scene::Height()) } {}

    bool update(double t) override
    {
        constexpr double speed = 3.5;

        const auto width = FontAsset(U"Chat.message")(m_message).region().w;
        const auto sceneWidth = Scene::Width();

        FontAsset(U"Chat.message")(m_message).draw(sceneWidth - (sceneWidth + width) * t / speed, m_y);

        return (t < speed);
    }
};

void Main()
{

    YouTubeLiveChat ytchat(VIDEO_ID, API_KEY);
    if (!ytchat.getActiveLiveChatId())
    {
        return;
    }

    Effect chatText;
    FontAsset::Register(U"Chat.message", 30);

    double lastGetTime = Scene::Time() - 5.0;

    while (System::Update())
    {

        if (lastGetTime + 5.0 < Scene::Time())  // 5秒おきに更新
        {
            Array<ChatItem> items;
            ytchat.getNewItems(items);

            for (auto elem : items)
            {
                chatText.add<ChatEffect>(elem.messageText);
            }

            lastGetTime = Scene::Time();
        }

        chatText.update();
    }
}

```

追記 (2021/12/18):
> このままだと文字列リテラルが直接実行ファイルに埋め込まれ、APIキーなどが簡単に抜き取られてしまうのですが、Siv3D の`SIV3D_OBFUSCATE`という機能を使えば多少難読化できるそうです！
> 
> @Reputeless さん[コメント](https://qiita.com/Ryoga-exe/items/6257c2b144dcb58e1245#comment-9b1bd4d5b81dfcc3ca85)ありがとうございます。

実行するとこんな感じになります。

![Test1.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/464356/8c7cc7a0-dad3-ccef-5765-d05b300719c3.gif)


こんな感じでチャットを画面上に流せます！
Effect はエフェクト以外にもこんな感じの描画に使えるので便利ですね！

<details>
<summary>ちなみに、文字を流すエフェクトだけを使っても面白い演出ができそうです</summary>
<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">それっぽいコメントを流すだけのやつを作った <a href="https://t.co/7Gz6r6dZEc">pic.twitter.com/7Gz6r6dZEc</a></p>&mdash; Ryoga.exe (@Ryoga_exe) <a href="https://twitter.com/Ryoga_exe/status/1471511815138217986?ref_src=twsrc%5Etfw">December 16, 2021</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
</summary>
</details>

# 実装

`YouTubeLiveChat` の実装を説明します。

## コード (全文)
```cpp:YouTubeLiveChat.hpp
#pragma once

struct ChatItem
{
    String userName;
    String messageText;
};

class YouTubeLiveChat
{
public:
    YouTubeLiveChat(const String& videoID, const String& apiKey)
        : m_videoId(videoID), m_apikey(apiKey)
    {

    }
    ~YouTubeLiveChat()
    {

    }

    bool getActiveLiveChatId()
    {
        const URL url = U"https://youtube.googleapis.com/youtube/v3/videos?part=liveStreamingDetails&id=" + m_videoId + U"&key=" + m_apikey;
        const HashTable<String, String> headers = { { U"Content-Type", U"application/json" } };

        String result;
        if (HTTPGet(url, headers, result))
        {
            JSON json = JSON::Parse(result);

            m_activeLiveChatId = json[U"items"][0][U"liveStreamingDetails"][U"activeLiveChatId"].getString();

            return true;
        }
        else
        {
            return false;
        }
    }

    bool getNewItems(Array<ChatItem>& items)
    {
        if (m_activeLiveChatId.empty())
        {
            return false;
        }

        URL url = U"https://youtube.googleapis.com/youtube/v3/liveChat/messages?liveChatId=" + m_activeLiveChatId + U"&part=authorDetails%2Csnippet&key=" + m_apikey;
        const HashTable<String, String> headers = { { U"Content-Type", U"application/json" } };

        if (!m_nextPageToken.empty())
        {
            url += U"&pageToken=" + m_nextPageToken;
        }

        String result;
        if (HTTPGet(url, headers, result))
        {
            JSON json = JSON::Parse(result);

            m_nextPageToken = json[U"nextPageToken"].getString();

            Array<ChatItem> res;
            for (const auto& object : json[U"items"].arrayView())
            {
                ChatItem item;
                item.userName = object[U"authorDetails"][U"displayName"].getString();
                item.messageText = object[U"snippet"][U"displayMessage"].getString();
                res << item;
            }

            items = res;

            return true;
        }
        else
        {
            return false;
        }

    }

private:

    bool HTTPGet(const URL& url, const HashTable<String, String>& headers, String& result)
    {

        MemoryWriter writer;

        if (auto response = SimpleHTTP::Get(url, headers, writer))
        {
            if (response.isOK())
            {
                auto res = writer.getBlob().asArray();
                std::string s;
                for (auto elem : res)
                {
                    s += (char)elem;
                }
                result = Unicode::FromUTF8(s);
                return true;
            }
        }
        else
        {
            return false;
        }

        return false;
    }

private:
    String m_apikey;
    String m_videoId;
    String m_nextPageToken;
    String m_activeLiveChatId;
};

```

実装する際の注意点として GET リクエストで返ってくる文字列は UTF-8 なので変換してやる必要があります。

## 流れ

- [このAPI](https://developers.google.com/youtube/v3/docs/videos/list?hl=ja) によって videoID と API キーから chatID を取得
- [このAPI](https://developers.google.com/youtube/v3/live/docs/liveChatMessages/list) でチャット欄を繰り返し取得。
    - 初回はそのまま取る。そしてレスポンスに含まれる `nextPageToken` の値を覚えておきます。
    - 二回目以降は `pageToken` に前回の `nextPageToken` の値を指定すれば差分を取ることができます。

## 仕様
### `YouTubeLiveChat::getActiveLiveChatId()`

YouTube Data API v3 を使って activeLiveChatId を取得します。これをしないと、チャットの内容が取れません。
そのため、最初に一度だけ実行する必要があります。
失敗したら false を返します。

### `YouTubeLiveChat::getNewItems()`

チャットを取得します。引数に `Array<ChatItem>` 型の変数を入れます。取得したチャットの内容はここに代入されます。
二回以降は `pageToken` によって前回取得したときからの差分のみ取得します。
失敗したら false を返します。

# 参考にしたもの

https://developers.google.com/youtube/v3/live/docs/liveChatMessages

https://hawksnowlog.blogspot.com/2020/12/getting-started-youtube-data-api-with-curl.html

# あとがき

今回は新たに v0.6 から追加された `SimpleHTTP` を使っていろいろと遊んでみました。
YouTube のチャットと連動するゲームなどが作れそうです！
スーパーチャットなどの扱いは未実装なのでいつか実装したいところです...

## 余談
Queries の存在が悩みどころです... すぐに上限に達しそう...
実は API 無しでチャットを取得することもでき、それをすれば上限を気にしなくて済みますが、[利用規約](https://www.youtube.com/static?template=terms&hl=ja&gl=JP)を見る感じグレーです。
