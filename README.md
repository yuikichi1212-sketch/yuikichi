<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ゆいきちアプリ</title>
    <style>
        /* ベースのスタイル（Apple風のフォントと余白） */
        :root {
            --bg-color: #fbfbfd;
            --text-color: #1d1d1f;
            --secondary-text: #86868b;
            --accent-color: #0071e3;
        }

        body {
            margin: 0;
            padding: 0;
            font-family: -apple-system, BlinkMacSystemFont, "Helvetica Neue", "Segoe UI", "Hiragino Kaku Gothic ProN", "Hiragino Sans", sans-serif;
            background-color: var(--bg-color);
            color: var(--text-color);
            line-height: 1.5;
            -webkit-font-smoothing: antialiased;
        }

        /* ヘッダー */
        header {
            background-color: rgba(251, 251, 253, 0.8);
            backdrop-filter: blur(10px);
            position: fixed;
            top: 0;
            width: 100%;
            height: 48px;
            display: flex;
            align-items: center;
            justify-content: center;
            z-index: 100;
            border-bottom: 1px solid rgba(0,0,0,0.1);
        }

        header h1 {
            font-size: 16px;
            font-weight: 500;
            margin: 0;
        }

        /* セクション共通 */
        section {
            padding: 120px 20px;
            display: flex;
            flex-direction: column;
            align-items: center;
            text-align: center;
            min-height: 80vh;
        }

        /* ヒーローセクション（一番上） */
        .hero {
            padding-top: 180px;
        }

        .hero h2 {
            font-size: 56px;
            font-weight: 700;
            letter-spacing: -0.005em;
            margin: 0 0 10px 0;
        }

        .hero p {
            font-size: 28px;
            color: var(--secondary-text);
            margin: 0 0 40px 0;
            font-weight: 500;
        }

        /* アプリ紹介セクション */
        .app-section {
            background-color: #fff;
            margin: 20px;
            border-radius: 30px;
            box-shadow: 0 10px 30px rgba(0,0,0,0.05);
            max-width: 980px;
            width: 100%;
            overflow: hidden;
            box-sizing: border-box;
        }

        .app-title {
            font-size: 48px;
            font-weight: 700;
            margin: 0 0 10px 0;
        }

        .app-headline {
            font-size: 24px;
            margin: 0 0 20px 0;
        }

        .app-description {
            font-size: 17px;
            color: var(--secondary-text);
            max-width: 600px;
            margin: 0 auto 30px auto;
        }

        /* メディアエリア（動画や画像を入れる枠） */
        .media-container {
            width: 100%;
            max-width: 800px;
            height: 450px;
            background-color: #f5f5f7;
            border-radius: 20px;
            margin: 0 auto 40px auto;
            display: flex;
            align-items: center;
            justify-content: center;
            color: #ccc;
            /* 動画を入れた際にはみ出さないように */
            overflow: hidden; 
        }

        /* 動画要素用のスタイル */
        .media-container video {
            width: 100%;
            height: 100%;
            object-fit: cover;
        }

        /* ボタン */
        .btn {
            display: inline-block;
            background-color: var(--accent-color);
            color: white;
            padding: 12px 24px;
            border-radius: 980px;
            text-decoration: none;
            font-size: 17px;
            font-weight: 400;
            transition: background-color 0.3s;
        }

        .btn:hover {
            background-color: #0051a3;
        }

        /* フッター */
        footer {
            padding: 40px;
            text-align: center;
            color: var(--secondary-text);
            font-size: 12px;
        }

        /* ふわっと表示させるアニメーション用のクラス */
        .fade-in {
            opacity: 0;
            transform: translateY(30px);
            transition: opacity 1s ease-out, transform 1s ease-out;
        }

        .fade-in.visible {
            opacity: 1;
            transform: translateY(0);
        }
    </style>
</head>
<body>

    <header>
        <h1>ゆいきちアプリ</h1>
    </header>

    <section class="hero fade-in">
        <h2>ゆいきちの世界へようこそ。</h2>
        <p>驚くほど便利で、ちょっと楽しいアプリたち。</p>
    </section>

    <section class="app-section fade-in">
        <h2 class="app-title">アプリ名 1</h2>
        <p class="app-headline">すべてを、よりシンプルに。</p>
        
        <p class="app-description">
            ここにアプリの詳しい説明を書きます。どんな機能があるのか、どんな人に使ってほしいのか、熱い思いを自由に書いてください。
        </p>

        <div class="media-container">
            <p>ここに動画や画像を挿入</p>
        </div>

        <a href="#" class="btn">ダウンロード</a>
    </section>

    <section class="app-section fade-in">
        <h2 class="app-title">アプリ名 2</h2>
        <p class="app-headline">想像を超える体験を。</p>
        
        <p class="app-description">
            2つ目のアプリの説明をここに書きます。余白をたっぷりとっているので、長めの文章でもスッキリ見えます。
        </p>

        <div class="media-container">
            <p>ここに動画や画像を挿入</p>
        </div>

        <a href="#" class="btn">詳細を見る</a>
    </section>

    <footer>
        <p>&copy; 2026 ゆいきち. All rights reserved.</p>
    </footer>

    <script>
        document.addEventListener("DOMContentLoaded", function() {
            const observerOptions = {
                root: null,
                rootMargin: '0px',
                threshold: 0.15
            };

            const observer = new IntersectionObserver((entries, observer) => {
                entries.forEach(entry => {
                    if (entry.isIntersecting) {
                        entry.target.classList.add('visible');
                        observer.unobserve(entry.target);
                    }
                });
            }, observerOptions);

            const elements = document.querySelectorAll('.fade-in');
            elements.forEach(el => observer.observe(el));
        });
    </script>
</body>
</html>
