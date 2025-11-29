---
title: なめらかな連続するTweenアニメーションを簡単に作るクラスをつくった
tags:
  - C++
  - Siv3D
  - OpenSiv3D
private: false
updated_at: '2021-04-29T18:20:41+09:00'
id: 5444a76440894e394b16
organization_url_name: null
slide: false
ignorePublish: false
---
YoutubeAPIを叩く記事にしようと思っていたのですが、間に合わなかったので代替です。

# 動機
[こんなもの](https://github.com/hideakitai/Tween)を見つけました。
いろいろ読んでいくと面白そうな機能がありました。そして、
> これを参考にして、Siv3D版につくったら結構便利なのでは...？

と思ったので作ってみました。

**プロC++erではないので拙いコードですがご容赦ください**

# できたもの
![tween-demo.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/464356/57d33e3a-4c86-11aa-0b57-d31566d0d4bf.gif)
<font color="#00EAEA">水色</font>の線は直線(`Linear`)で、<font color=orange>オレンジ</font>の曲線はイージングでなめらかに遷移します。

# ソースコード
[リファレンス](https://siv3d.github.io/ja-jp/)に合わせて波括弧はオールマンスタイルを使っています

とりあえず`Main.cpp`はこんな感じで書けます。

```cpp:Main.cpp
# include <Siv3D.hpp> // OpenSiv3D v0.4.3
# include "Tween.hpp"

void Main()
{
    Tween::Timeline timeline(0); // 初期値は0

    timeline
        .then(200, 5.0s)     // 200まで5秒で動く
        .then(400, 5.0s)     // そこから400まで5秒で動く
        .wait(1.0s)          // 1秒待ち
        .then(600, 5.0s);    // そこから600まで5秒で動く
 
    timeline.start();
	
    while (System::Update())
    {
        // 更新する
        timeline.update();

        ClearPrint();

        Print << timeline.elapsed();
        // .getValueか()演算子で値を取得
        Print << timeline();
    }
}

```

イージングを使ってなめらかに動かすには`.then`の引数にSiv3Dのイージングの関数を書いてやります。
具体的には

```cpp
timeline
    .then(200, 5.0s, EaseInOutSine)
    .then(400, 5.0s, EaseInOutElastic)
    .wait(1.0s)
    .then(600, 5.0s, EaseOutBounce);
```

こんな感じです。

## ヘッダー部分

```cpp:Tween.hpp
# pragma once
# include <Siv3D.hpp>

namespace Tween
{
    struct trans_t
    {
        double to;
        Duration begin, end;
        std::function<double(double)> easingFunction = EaseInLinear;
    };

    // template <typename T>
    class Timeline
    {
    public:

        explicit Timeline(const double initialValue = 0.0, bool startImmediately = false)
            :m_initialValue(initialValue), m_value(initialValue), m_isRunning(false)
        {
            if (startImmediately)
            {
                m_isRunning = true;
                m_stopwatch.start();
            }
        }

        ~Timeline()
        {
        }

        Timeline& then(const double to, const Duration dur = 0.0s, double easingFunction(double) = EaseInLinear)
        {
            addTransition(to, dur, easingFunction);
            return *this;
        }

        Timeline& wait(const Duration dur)
        {
            const double prevTarget = (m_transitions.isEmpty() ? 0 : m_transitions.back().to);
            addTransition(prevTarget, dur, EaseInLinear);
            return *this;
        }

        bool update()
        {
            if (m_transitions.isEmpty()) return false;

            if (m_transitions.size() <= m_currentIndex)
            {
                m_value = m_transitions.back().to;
                m_isRunning = false;
                return false;
            }

            if (m_transitions[m_currentIndex].end < m_stopwatch.elapsed()) {
                m_currentIndex++;
                if (m_transitions.size() <= m_currentIndex)
                {
                    m_value = m_transitions.back().to;
                    m_isRunning = false;
                    return false;
                }
            }
            
            const double t = (m_transitions[m_currentIndex].end - m_transitions[m_currentIndex].begin).count();
            const double c = m_transitions[m_currentIndex].to - (m_currentIndex > 0 ? m_transitions[m_currentIndex - 1].to : m_initialValue);
            double e = 1;
            if (t > 0) e = m_transitions[m_currentIndex].easingFunction(Min((m_stopwatch.sF() - m_transitions[m_currentIndex].begin.count()), t) / t);
            
            m_value = c*e + (m_currentIndex > 0 ? m_transitions[m_currentIndex - 1].to : m_initialValue);

            return true;
        }

        void start() { m_isRunning = true; m_stopwatch.start(); }
        void clear()
        {
            m_currentIndex = 0;
            m_initialValue = 0.0;
            m_transitions.clear();
            m_isRunning = false;
            m_value = 0.0;
            m_stopwatch.reset();
        }
        void restart()
        {
            m_currentIndex = 0;
            m_value = m_initialValue;
            m_stopwatch.restart();
        }
        bool isEmpty() { return m_transitions.isEmpty(); }
        bool isRunning() { return m_isRunning; }
        size_t size() { return m_transitions.size(); }
        double getValue() { return m_value; }
        double operator()() { return getValue(); }
        Duration elapsed() { return m_stopwatch.elapsed(); }

    private:
        void addTransition(const double to, const Duration dur, double easingFunction(double))
        {
            const Duration prevDur = (m_transitions.isEmpty() ? 0.0s : m_transitions.back().end);
            m_transitions.emplace_back(trans_t{ to, prevDur, prevDur + dur, easingFunction });
        }

        Stopwatch m_stopwatch;
        Array<trans_t> m_transitions;
        int32 m_currentIndex = 0;
        double m_initialValue = 0.0;
        double m_value = 0.0;
        bool m_isRunning;
    };
}

```

殴り書きなのでちょっとコードが汚いかもです...

## 冒頭のやつのコード
<details>
<summary>開くとあります。ちょっと長いです</summary>
<div>

```cpp:Demo.cpp

# include <Siv3D.hpp> // OpenSiv3D v0.4.3
# include "Tween.hpp"

void Main()
{

    Scene::SetBackground(Palette::White);

    Tween::Timeline timelineEase(600), timelineLinear(600);

    timelineEase
        .then(200, 5.0s, EaseInOutSine)
        .then(400, 5.0s, EaseInOutElastic)
        .wait(1.0s)
        .then(600, 5.0s, EaseOutBounce);
    timelineLinear
        .then(200, 5.0s)
        .then(400, 5.0s)
        .wait(1.0s)
        .then(600, 5.0s);

    constexpr Size canvasSize(800, 600);
    constexpr int32 thickness = 4;
    Image image(canvasSize, Palette::White);
    DynamicTexture texture(image);

    timelineEase.start();
    timelineLinear.start();

    Point posEase((int32)(timelineEase.elapsed().count() * 50), (int32)timelineEase());
    Point posLinear = posEase;
    
    while (System::Update())
    {
        ClearPrint();

        timelineEase.update();
        timelineLinear.update();

        const Point fromL = posLinear;
        posLinear = { (int32)(timelineLinear.elapsed().count() * 50), (int32)timelineLinear() };
        const Point toL = posLinear;

        Line(fromL, toL).overwrite(image, thickness, Palette::Cyan);

        const Point fromE = posEase;
        posEase = { (int32)(timelineEase.elapsed().count() * 50), (int32)timelineEase() };
        const Point toE = posEase;

        Line(fromE, toE).overwrite(image, thickness, Palette::Orange);

        texture.fill(image);

        if (KeyC.down())
        {
            timelineEase.restart();
            timelineLinear.restart();

            posLinear = { (int32)(timelineLinear.elapsed().count() * 50), (int32)timelineLinear() };

            posEase = { (int32)(timelineEase.elapsed().count() * 50), (int32)timelineEase() };

            image.fill(Palette::White);

            texture.fill(image);
        }

        texture.draw();

        Print << timelineEase.elapsed();
    }
}
```
</div>
</details>

# 参考にしたもの
[Tween library for Arduino (hideakitai)](https://github.com/hideakitai/Tween)

# あとがき
初投稿なのであまり文章が良くないかもしれませんがいかがでしたでしょうか。
Siv3Dは最近になって初めて触れたのでこれから活用していこうと思います！

来年はもっといい記事・内容を書けるようにしたいところ。
