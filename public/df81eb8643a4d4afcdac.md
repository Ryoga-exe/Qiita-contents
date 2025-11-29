---
title: Siv3D でDiscord の Rich Presence を使ってみた
tags:
  - C++
  - Siv3D
  - discord
  - OpenSiv3D
  - discord-game-sdk
private: false
updated_at: '2022-12-09T12:27:13+09:00'
id: df81eb8643a4d4afcdac
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

この記事は [Siv3D Advent Calendar 2022](https://qiita.com/advent-calendar/2022/siv3d) 9 日目の記事です。

突然ですが、Discord には [Rich Presence](https://discord.com/rich-presence) という機能があります。
簡単にいえば現在プレイしているゲームがプロフィールに表示されるあれです。

例えば Minecraft をプレイしているとこんな感じに表示されます。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/464356/482a46d4-5893-ce89-9e3b-7efd59a5bf2c.png)
このように他のユーザにリアルタイムに情報が伝わるようになります。この例の場合はタイトルと画像とプレイ時間のみが表示されていますが、複数のテキストを入れてより詳細なことを伝えたり、ゲームへの参加・観戦を受け付けるボタンを設置することもできます。

この機能を Siv3D 製のアプリにも対応させたいなと感じたためいろいろやってみたところ、うまくいったので今回はこれについて書きたいと思います。

# 準備

Rich Presence を導入するためにはいくつか準備が必要です。

## Discord アカウントの作成

もちろん Discord アカウントが必要です。アカウントがなければ[ここ](https://discord.com)から作成しましょう。

## Discord Developer Portal からアプリケーションを作成する

Rich Presence を動かすには、Discord の BOT を作るときと同様に [Discord Developer Portal](https://discordapp.com/developers) からアプリケーションを作成する必要があります。

[Discord Developer Portal](discordapp.com/developers) にアクセスし、右上にある `New Application` と書かれたボタンをクリックします。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/464356/8939f869-84f9-efcf-ac36-28c6e60e7c2a.png)

ポップアップが現れ、アプリケーションの名前を入力するように求められるので適当な名前を入力します。 (今回は `Siv3D App` とします)
ここで入力する名前は、「〇〇をプレイ中」のようにプロフィール上で表示されるゲーム名になります。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/464356/5dbc1534-9869-4a07-93e5-ff280519d3c6.png)

利用規約に同意し、`Create` ボタンを押して作成します。

作成すると画面が遷移します。今回必要となるのは `APPLICATION ID` (CLIENT ID) です。 `APPLICATION ID` という項目があるのでそこの ID をコピーして控えておきます。

<details>
<summary>この ID は公開しても大丈夫？</summary>
<div>

> 公開しても大丈夫です。
> 例として [VSCode 向けの Rich Presence の拡張機能のソースコード](https://github.com/leonardssh/vscord/blob/main/src/helpers/getApplicationId.ts)を見てみると普通に公開されています。
> 
> もちろんですが、`CLIENT SECRET` やら BOT 用の Token などはまずいです。
</div>
</details>

## Discord Game SDK をダウンロード・導入する

今回 Rich Presence を導入するために [Discord Game SDK](https://discord.com/developers/docs/game-sdk/sdk-starter-guide) という Discord が公式に公開・配布している SDK を使用します。

:::note warn
[Discord-RPC](https://github.com/discord/discord-rpc) というライブラリを使用することでも可能ですが、現在 deprecated なようです。
:::

[こちらのサイト](https://discord.com/developers/docs/game-sdk/sdk-starter-guide#step-1-get-the-thing) から Discord GameSDK v3.2.1 をダウンロードし、適当な場所に解凍します。

その後、[Code Primer - No Engine (Cpp)](https://discord.com/developers/docs/game-sdk/sdk-starter-guide#code-primer-no-engine-cpp) を参考に導入します。
使いたいプロジェクトに `/cpp` の中身を入れ、`/lib` 内にある lib ファイルをリンクします。

<details>
<summary>Visual Studio (Windows 環境) への導入・設定方法</summary>
<div>

> ここでは、参考として筆者が行った Visual Studio への導入方法について書きます。
> 
> - ソリューションのディレクトリにダウンロードし解凍した Game SDK 内の `/cpp` をフォルダごと移します。その後、わかりやすいように `/discord-game-sdk` のような名前に変更します。`/lib` フォルダについてもソリューションのディレクトリに移します。
> 
> - その後、Visual Studio でさきほど移した .cpp ファイルと .hpp ファイルを追加します。
>
>    - ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/464356/20d1a0db-ed16-d225-312d-b3f558cef871.png)
>    - 追加するとこのような感じになります。この画像では `discord-game-sdk` というフィルターを追加した上でその中に追加しています。
>
> - 次にメニューの『プロジェクト(P)』→『(プロジェクト名) のプロパティ(E)...』を選んで、プロジェクトのプロパティダイアログを開きます。
> 
> - ダイアログの左上にある『構成(C):』と書かれている項目を『すべての構成』に変更し、左側のリストから『構成プロパティ』→『リンカー』→『全般』を選びます。
> 
> - 『追加のライブラリ ディレクトリ』を編集し、`$(ProjectDir)lib\x86_64\` を追加します。
> 
> - 『リンカー』→『入力』を選び、『追加の依存ファイル』の項目に `discord_game_sdk.dll.lib` を追加します。
> 
> - 最後に、`/lib/x86_64` 内にある `discord_game_sdk.dll` を `/App` ディレクトリに移します。
> 
> **これで準備完了です！**

</div>
</details>

# 動かしてみる

実際に表示されるか確かめてみます。

以下のシンプルなコードを実行してみます。Discord が起動している状態で実行してください。

::: note info
ClientID にはさきほど控えておいた APPLICATION ID を入れてください。
また、コードは Game SDK のインクルードするファイルを /discord-game-sdk に入れた際のものです。適宜読み替えてください。
:::

```cpp
# include <Siv3D.hpp> // OpenSiv3D v0.6.6
# include "discord-game-sdk/discord.h"

constexpr discord::ClientId ClientID = /* APPLICATION ID (CLIENT ID) をここに入れる */;

void Main()
{
	discord::Core* core_raw{};
	auto result = discord::Core::Create(ClientID, DiscordCreateFlags_Default, &core_raw);
	std::unique_ptr<discord::Core> core(core_raw);
	if (!core)
	{
		throw Error{ U"Failed to instantiate discord core (error code: {})"_fmt(static_cast<int>(result)) };
	}
    
	discord::Activity activity{};
	activity.SetState(Unicode::ToUTF8(U"Playing Siv3D App!").c_str());
	activity.SetType(discord::ActivityType::Playing);
    
    core->ActivityManager().UpdateActivity(activity, nullptr);
    
	while (System::Update())
	{
		core->RunCallbacks();
	}
}
```

`discord::Activity` で表示させたい情報を構築し、`core->ActivityManager().UpdateActivity()` を使って適用、`core->RunCallbacks()` で Discord に情報を伝えます。

::: note warn
Discord Game SDK 内で `strncpy` 関数が使われているため、このままでは警告 C4996 が出てコンパイルできないことがあります。その場合は、[プリプロセッサの定義] プロパティで、`_CRT_SECURE_NO_WARNINGS` を追加します。
:::

::: note warn
Discord Game SDK で文字列を渡すときの文字コードは UTF-8 です。また、`activity.SetState()` の引数の型は `const char*` 型なので単純に `Unicode::ToUTF8()` で包むだけでなく、`.c_str()` を使って変換してやります。
:::

これを実行すると...

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/464356/157ad316-7bbe-e1ed-de13-37368228b713.png)

いい感じに表示されました！

## もうちょっといろいろやってみる

### 画像を表示する

これだけでは味気無いので画像を表示したりしてみます。

Discord Developer Portal を開き、作成したアプリケーションを選択します。
左の SETTINGS の項目の Rich Presence > Art Assets から表示したい画像をアセットに追加します。
Rich Presence Assets の下にある `Add Image(s)` と書かれたボタンを押して画像を追加します。ここで追加できる画像は 512x512 が最小です。アセットの名前を適当なものに変更して `Save Changes` を押して変更を保存します。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/464356/2d8ade2a-2e8d-4a3e-e701-368d2bb0fb6c.png)
追加するとこのような見た目になります。(今回は `siv3d` という名前で追加しました。)

これで画像を使用する準備は整いました！

さきほどのサンプルプログラムを少しいじります。
```diff_cpp
void Main()
{
    // ...略
    activity.SetState(Unicode::ToUTF8(U"Playing Siv3D App!").c_str());
+   activity.GetAssets().SetLargeImage(Unicode::ToUTF8(U"siv3d").c_str()); // 追加
    // ...
}
```

`activity.GetAssets().SetLargeImage(Unicode::ToUTF8(U"追加したアセット名の文字列").c_str());` とすると画像を表示することができます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/464356/89d665ae-ea73-0416-2b4a-d9f270ca1de4.png)

> うまく表示された！！！

アセットは最大で 300 個追加できるので、プレイ中のステージの画像を表示させたり、音ゲーならプレイ中の楽曲のジャケット画像の表示など結構いろいろなことができそうです...！

また、正しく追加したのに表示されないことがありますが、恐らくアセットの反映に時間が多少かかるためだと思われます。時間をおいて再度実行してみてください。

`activity.GetAssets().SetSmallImage()` なるものもありますが、これを使うと画像の右下に小さな画像を表示させることができます。さらに、`activity.GetAssets().SetLargeText()` や `activity.GetAssets().SetLargeText()`を使うと、ホバーされた際に表示されるテキストを設定することができます。

## 経過時間を表示する

`activity.GetTimestamps().SetStart()` を使います。ここで引数には現在時刻 (Unix時間 [秒]) を入れてやります。Siv3D だと `Time::GetSecSinceEpoch()` があるので簡単に UNIX 時間を取得できますね！

```diff_cpp
void Main()
{
    // ...略
    activity.SetState(Unicode::ToUTF8(U"Playing Siv3D App!").c_str());
+   activity.GetTimestamps().SetStart(Time::GetSecSinceEpoch()); // 追加
    // ...
}
```

実行してみるとアクティビティの欄に経過時間が表示されるのが確認できます。

### これだと開始時刻をこっちで指定できるから嘘のプレイ時間を表示することができる...？

A. できます
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/464356/2e6c8b45-e345-cb30-abda-9b7b95779fcb.png)
例えばこの画像のように 22 時間プレイしている人を簡単に演じることができます。 (需要...)

### `SetStart` があるなら `SetEnd` もある...？

A. あります

実際に

```diff_cpp
void Main()
{
    // ...略
    activity.GetTimestamps().SetStart(Time::GetSecSinceEpoch());
+   activity.GetTimestamps().SetEnd(Time::GetSecSinceEpoch() + 3600); // 追加
    // ...
}
```
このようにしてみると

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/464356/adee8e88-e34c-50cc-ae69-2e597a096e95.png)

残り時間として表示されます！

# 情報をリアルタイムに変更する

もちろん、これまで紹介したものはリアルタイムに更新することが可能です。
これらを踏まえて DiscordRPCMaker っぽいものを作ってみます。コードの全文は以下に示します。

```cpp
# include <Siv3D.hpp> // OpenSiv3D v0.6.6
# include "discord-game-sdk/discord.h"

constexpr discord::ClientId ClientID = /* APPLICATION ID (CLIENT ID) をここに入れる */;

void Main()
{
	discord::Core* core_raw{};
	auto result = discord::Core::Create(ClientID, DiscordCreateFlags_Default, &core_raw);
	std::unique_ptr<discord::Core> core(core_raw);
	if (!core)
	{
		throw Error{ U"Failed to instantiate discord core (error code: {})"_fmt(static_cast<int>(result)) };
	}
	discord::Activity activity{};
	activity.GetTimestamps().SetStart(Time::GetSecSinceEpoch());
	activity.SetType(discord::ActivityType::Playing);
	
	Font font(15, Typeface::Medium);

	TextEditState state(U"Playing Siv3D App!");
	TextEditState details(U"ゲームを開発中");
	TextEditState largeImage(U"siv3d");
	TextEditState largeImageText(U"Siv3D");
	TextEditState smallImage;
	TextEditState smallImageText;
	
	core->ActivityManager().UpdateActivity(activity, nullptr);
	while (System::Update())
	{
		font(U"STATE").draw(20, 55);
		SimpleGUI::TextBox(state, Vec2{ 20, 75 });

		font(U"DETAILS").draw(20, 115);
		SimpleGUI::TextBox(details, Vec2{ 20, 135 });

		font(U"LARGE IMAGE KEY").draw(20, 175);
		SimpleGUI::TextBox(largeImage,     Vec2{ 20, 195 });
		font(U"LARGE IMAGE TEXT").draw(250, 175);
		SimpleGUI::TextBox(largeImageText, Vec2{ 250, 195 });

		font(U"SMALL IMAGE KEY").draw(20, 235);
		SimpleGUI::TextBox(smallImage, Vec2{ 20, 255 });
		font(U"LARGE IMAGE TEXT").draw(250, 235);
		SimpleGUI::TextBox(smallImageText, Vec2{ 250, 255 });
		
		if (SimpleGUI::Button(U"更新", Vec2{ 20, 10 }))
		{
			activity.SetState(Unicode::ToUTF8(state.text).c_str());
			activity.SetDetails(Unicode::ToUTF8(details.text).c_str());
			activity.GetAssets().SetLargeImage(Unicode::ToUTF8(largeImage.text).c_str());
			activity.GetAssets().SetLargeText(Unicode::ToUTF8(largeImageText.text).c_str());
			activity.GetAssets().SetSmallImage(Unicode::ToUTF8(smallImage.text).c_str());
			activity.GetAssets().SetSmallText(Unicode::ToUTF8(smallImageText.text).c_str());

			core->ActivityManager().UpdateActivity(activity, nullptr);
		}

		core->RunCallbacks();
	}
}
```

これを実行すると、複数のテキストボックスが出てきます。更新と書かれたボタンを押すことで Discord に反映することができます！

# おわりに

今回は Discord Game SDK を使って Siv3D 製のアプリに Rich Presence を導入してみました。[ドキュメント](https://discord.com/developers/docs/game-sdk/activities)を見ると、今回紹介した機能以外にも様々な機能があります。SDK をダウンロードしたものに入っている `examples` を見ると実装例があり、とても参考になります。また、公式の [Rich Presence Best Practices](https://discord.com/developers/docs/rich-presence/best-practices) というドキュメントも参考になります。

余談ですが、この SDK では任意のボタンを表示できなかったり (例えば[これ](https://www.npmjs.com/package/discord-rpc)ならできる)、あまりモダンな C++ での実装がされていないのでラップしたものやらを作ってみたいところ...

みなさんも導入していい感じに Discord にアクティビティを表示させてみてはいかがでしょうか。
