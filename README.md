<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>look - 瞬間プレビュー＆ファイル管理</title>
    <script src="https://unpkg.com/react@18/umd/react.development.js" crossorigin></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js" crossorigin></script>
    <script src="https://unpkg.com/babel-standalone@6/babel.min.js"></script>
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://cdn.jsdelivr.net/npm/lucide-static@0.320.0/font/lucide.css" rel="stylesheet">
    <style>
        body { font-family: 'Helvetica Neue', Arial, sans-serif; background-color: #f8fafc; }
        .app-icon { box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06); }
        /* プレビュー領域のスタイル */
        .preview-container {
            width: 100%;
            height: 200px;
            display: flex;
            justify-content: center;
            align-items: center;
            background-color: #f1f5f9;
            border-radius: 0.75rem;
            overflow: hidden;
            margin-bottom: 1rem;
            border: 2px dashed #cbd5e1;
        }
        .preview-media {
            max-width: 100%;
            max-height: 100%;
            object-fit: contain;
        }
        /* アニメーション */
        .fade-in-up { animation: fadeInUp 0.4s cubic-bezier(0.16, 1, 0.3, 1); }
        @keyframes fadeInUp { from { opacity: 0; transform: translateY(20px); } to { opacity: 1; transform: translateY(0); } }
    </style>
</head>
<body>
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useRef } = React;

        // --- モックデータベース ---
        let mockUsersDB = [];

        // --- アプリアイコンコンポーネント ---
        // look.jpg があれば表示し、なければデフォルトのアイコンを表示
        const AppIcon = ({ size = 'w-24 h-24', className = '' }) => {
            const [imgError, setImgError] = useState(false);
            return (
                <div className={`${size} bg-gradient-to-br from-yellow-400 to-orange-500 rounded-2xl flex items-center justify-center overflow-hidden ${className} app-icon`}>
                    {!imgError ? (
                        <img src="look.jpg" alt="look" className="w-full h-full object-cover" onError={() => setImgError(true)} />
                    ) : (
                        <span className="text-white font-black text-4xl tracking-tighter">look</span>
                    )}
                </div>
            );
        };

        function App() {
            // --- 状態管理 ---
            const [isLoggedIn, setIsLoggedIn] = useState(false);
            const [currentUser, setCurrentUser] = useState(null);
            // 認証関連
            const [authStep, setAuthStep] = useState(0); // 0:LP, 1:モード選択, 2:ID入力, 3:パス入力
            const [authMode, setAuthMode] = useState(null); // 'login' or 'signup'
            const [username, setUsername] = useState('');
            const [password, setPassword] = useState('');
            const [errorMsg, setErrorMsg] = useState('');
            // アプリ機能関連
            const [showProfile, setShowProfile] = useState(false);
            const [uploadedFiles, setUploadedFiles] = useState([]); // プレビューデータを含むファイルリスト
            const fileInputRef = useRef(null);

            // --- バリデーション ---
            const validateUsername = (name) => name.length >= 5;
            const validatePassword = (pass) => /[a-zA-Z]/.test(pass) && /[0-9]/.test(pass) && pass.length >= 5;

            // --- 認証フロー ---
            const handleAuthAction = (action) => {
                setErrorMsg('');
                switch (action) {
                    case 'START_LOGIN': setAuthMode('login'); setAuthStep(2); break;
                    case 'START_SIGNUP': setAuthMode('signup'); setAuthStep(2); break;
                    case 'CHECK_USER':
                        if (!validateUsername(username)) return setErrorMsg('アカウント名は5文字以上必要です。');
                        if (authMode === 'signup' && mockUsersDB.some(u => u.username === username)) return setErrorMsg('その名前は既に使われています。');
                        if (authMode === 'login' && !mockUsersDB.some(u => u.username === username)) return setErrorMsg('アカウントが見つかりません。');
                        setAuthStep(3);
                        break;
                    case 'COMPLETE_AUTH':
                        if (authMode === 'signup') {
                            if (!validatePassword(password)) return setErrorMsg('パスワードは英字と数字を含む5文字以上で設定してください。');
                            const newUser = { username, password };
                            mockUsersDB.push(newUser);
                            setCurrentUser(newUser);
                            setIsLoggedIn(true);
                        } else {
                            const user = mockUsersDB.find(u => u.username === username && u.password === password);
                            if (!user) return setErrorMsg('パスワードが違います。');
                            setCurrentUser(user);
                            setIsLoggedIn(true);
                        }
                        break;
                    case 'BACK': setAuthStep(Math.max(1, authStep - 1)); setErrorMsg(''); break;
                    case 'LOGOUT':
                        setIsLoggedIn(false); setCurrentUser(null); setAuthStep(0);
                        setUsername(''); setPassword(''); setShowProfile(false); setUploadedFiles([]);
                        break;
                }
            };

            // --- ファイルアップロードとプレビュー処理 ---
            const handleFileUpload = (e) => {
                const file = e.target.files[0];
                if (!file) return;

                // ファイル読み込み用のリーダーを作成
                const reader = new FileReader();
                
                // 読み込み完了時の処理
                reader.onloadend = () => {
                    const newFile = {
                        id: Date.now(),
                        name: file.name,
                        type: file.type,
                        size: (file.size / 1024 / 1024).toFixed(2) + ' MB',
                        previewData: reader.result // 読み込んだデータ(Data URL)を保存
                    };
                    // 新しいファイルをリストの先頭に追加
                    setUploadedFiles([newFile, ...uploadedFiles]);
                };

                // ファイルの種類に応じて読み込みを開始
                if (file.type.startsWith('image/') || file.type.startsWith('video/')) {
                    reader.readAsDataURL(file); // 画像や動画はData URLとして読み込む
                } else {
                    // その他のファイルはプレビューデータなしで登録
                    reader.onloadend({ target: { result: null } });
                }
            };
            
            const triggerFileInput = () => fileInputRef.current.click();

            // ==========================================
            // ログイン前画面 (認証フロー)
            // ==========================================
            if (!isLoggedIn) {
                return (
                    <div className="min-h-screen flex flex-col items-center justify-center p-6 bg-gray-100">
                        <div className="w-full max-w-md fade-in-up">
                            {/* ステップ0: ランディングページ */}
                            {authStep === 0 && (
                                <div className="text-center space-y-8">
                                    <AppIcon size="w-32 h-32" className="mx-auto shadow-xl" />
                                    <h1 className="text-5xl font-black text-gray-800 tracking-tighter">look</h1>
                                    <p className="text-xl text-gray-600 font-medium">あらゆるファイルを、その場で。</p>
                                    <button onClick={() => setAuthStep(1)} className="w-full bg-gray-900 hover:bg-black text-white text-lg font-bold py-4 px-8 rounded-2xl transition-all transform hover:scale-[1.02] shadow-lg">
                                        ゆいきちアカウントで始める
                                    </button>
                                </div>
                            )}

                            {/* 認証フォームコンテナ */}
                            {authStep > 0 && (
                                <div className="bg-white p-8 rounded-3xl shadow-xl space-y-6">
                                    <div className="text-center mb-6">
                                        <AppIcon size="w-16 h-16" className="mx-auto mb-2" />
                                        <h2 className="text-2xl font-bold text-gray-800">
                                            {authStep === 1 ? 'ようこそ' : authMode === 'login' ? 'ログイン' : '新規登録'}
                                        </h2>
                                    </div>
                                    
                                    {errorMsg && <div className="bg-red-50 text-red-600 p-3 rounded-xl text-sm font-bold flex items-center gap-2"><i className="lucide-alert-circle"></i>{errorMsg}</div>}

                                    {/* ステップ1: モード選択 */}
                                    {authStep === 1 && (
                                        <div className="space-y-4">
                                            <button onClick={() => handleAuthAction('START_LOGIN')} className="w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 rounded-xl transition">ログイン</button>
                                            <button onClick={() => handleAuthAction('START_SIGNUP')} className="w-full bg-gray-100 hover:bg-gray-200 text-gray-800 font-bold py-3 rounded-xl transition">アカウントを作成</button>
                                            <button onClick={() => setAuthStep(0)} className="w-full text-gray-500 font-bold py-2 transition">トップへ戻る</button>
                                        </div>
                                    )}

                                    {/* ステップ2: アカウント名入力 */}
                                    {authStep === 2 && (
                                        <div className="space-y-4">
                                            <input type="text" placeholder="アカウント名 (5文字以上)" value={username} onChange={(e) => setUsername(e.target.value)} className="w-full p-4 bg-gray-50 border-2 border-gray-200 rounded-xl font-bold focus:outline-none focus:border-blue-500 transition" autoFocus />
                                            <div className="flex gap-4">
                                                <button onClick={() => handleAuthAction('BACK')} className="flex-1 bg-gray-100 hover:bg-gray-200 text-gray-700 font-bold py-3 rounded-xl transition">戻る</button>
                                                <button onClick={() => handleAuthAction('CHECK_USER')} className="flex-1 bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 rounded-xl transition">次へ</button>
                                            </div>
                                        </div>
                                    )}

                                    {/* ステップ3: パスワード入力 */}
                                    {authStep === 3 && (
                                        <div className="space-y-4">
                                            <p className="text-center text-gray-500 font-bold">{username} さん</p>
                                            <input type="password" placeholder="パスワード (英数混在5桁以上)" value={password} onChange={(e) => setPassword(e.target.value)} className="w-full p-4 bg-gray-50 border-2 border-gray-200 rounded-xl font-bold focus:outline-none focus:border-blue-500 transition" autoFocus />
                                            <div className="flex gap-4">
                                                <button onClick={() => handleAuthAction('BACK')} className="flex-1 bg-gray-100 hover:bg-gray-200 text-gray-700 font-bold py-3 rounded-xl transition">戻る</button>
                                                <button onClick={() => handleAuthAction('COMPLETE_AUTH')} className="flex-1 bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 rounded-xl transition">
                                                    {authMode === 'login' ? 'ログイン' : '登録を完了'}
                                                </button>
                                            </div>
                                        </div>
                                    )}
                                </div>
                            )}
                        </div>
                    </div>
                );
            }

            // ==========================================
            // ログイン後 メイン画面
            // ==========================================
            return (
                <div className="min-h-screen pb-20">
                    {/* ヘッダー */}
                    <header className="bg-white/80 backdrop-blur-md shadow-sm p-4 sticky top-0 z-20 flex justify-between items-center">
                        <div className="flex items-center gap-3">
                            <AppIcon size="w-10 h-10" className="rounded-lg shadow-sm" />
                            <span className="text-2xl font-black text-gray-900 tracking-tight">look</span>
                        </div>
                        <button onClick={() => setShowProfile(true)} className="bg-gray-100 hover:bg-gray-200 rounded-full p-1 pr-4 flex items-center gap-2 transition border-2 border-transparent hover:border-gray-300">
                            <div className="w-8 h-8 bg-gradient-to-tr from-blue-500 to-purple-500 rounded-full flex items-center justify-center text-white font-bold">
                                {currentUser.username.charAt(0).toUpperCase()}
                            </div>
                            <span className="font-bold text-gray-700 hidden sm:block">{currentUser.username}</span>
                        </button>
                    </header>

                    {/* メインコンテンツ */}
                    <main className="max-w-3xl mx-auto p-4 space-y-8">
                        {/* アップロードエリア */}
                        <div className="bg-white rounded-3xl shadow-sm border-2 border-dashed border-gray-300 p-8 text-center transition hover:border-blue-400 hover:bg-blue-50 group cursor-pointer" onClick={triggerFileInput}>
                            <input type="file" ref={fileInputRef} onChange={handleFileUpload} className="hidden" accept="*/*" />
                            <div className="w-20 h-20 bg-blue-100 text-blue-600 rounded-full flex items-center justify-center mx-auto mb-4 group-hover:scale-110 transition">
                                <i className="lucide-upload-cloud text-4xl"></i>
                            </div>
                            <h2 className="text-2xl font-bold text-gray-800 mb-2 group-hover:text-blue-700">ファイルをここへドロップ</h2>
                            <p className="text-gray-500 font-medium">またはクリックして選択。画像・動画は即プレビュー！</p>
                        </div>

                        {/* ファイルリスト */}
                        <div className="space-y-6">
                            <h3 className="text-xl font-bold text-gray-800 flex items-center gap-2">
                                <i className="lucide-files"></i>最近のファイル ({uploadedFiles.length})
                            </h3>
                            {uploadedFiles.length === 0 ? (
                                <div className="text-center py-12 text-gray-400 bg-white rounded-3xl border-2 border-gray-100">
                                    <i className="lucide-ghost text-6xl mb-4 opacity-50"></i>
                                    <p className="font-bold">まだファイルがありません</p>
                                    <p className="text-sm">アップロードして同期を開始しましょう</p>
                                </div>
                            ) : (
                                <div className="grid gap-6 md:grid-cols-2">
                                    {uploadedFiles.map(file => (
                                        <div key={file.id} className="bg-white rounded-3xl shadow-sm border border-gray-200 overflow-hidden fade-in-up hover:shadow-md transition">
                                            {/* プレビューエリア */}
                                            <div className="preview-container m-0 rounded-none border-0 border-b-2 border-gray-100 bg-gray-50 h-48">
                                                {file.previewData ? (
                                                    file.type.startsWith('image/') ? (
                                                        <img src={file.previewData} alt={file.name} className="preview-media" />
                                                    ) : file.type.startsWith('video/') ? (
                                                        <video src={file.previewData} controls className="preview-media"></video>
                                                    ) : null
                                                ) : (
                                                    <div className="text-center text-gray-400 flex flex-col items-center">
                                                        <i className="lucide-file-type text-5xl mb-2 opacity-50"></i>
                                                        <span className="font-bold text-sm px-3 py-1 bg-gray-200 rounded-full">{file.name.split('.').pop().toUpperCase()}</span>
                                                        <span className="text-xs mt-2">この形式はプレビューできません</span>
                                                    </div>
                                                )}
                                            </div>
                                            {/* ファイル情報とアクション */}
                                            <div className="p-4">
                                                <div className="mb-4">
                                                    <h4 className="font-bold text-lg text-gray-800 truncate">{file.name}</h4>
                                                    <p className="text-sm text-gray-500 font-medium">{file.size}</p>
                                                </div>
                                                <div className="flex gap-2">
                                                    <button className="flex-1 bg-blue-50 hover:bg-blue-100 text-blue-600 font-bold py-2 rounded-xl text-sm transition flex items-center justify-center gap-1">
                                                        <i className="lucide-download"></i>DL
                                                    </button>
                                                    <button className="flex-1 bg-gray-100 hover:bg-gray-200 text-gray-700 font-bold py-2 rounded-xl text-sm transition flex items-center justify-center gap-1">
                                                        <i className="lucide-share-2"></i>共有
                                                    </button>
                                                    <button className="flex-1 bg-gray-100 hover:bg-gray-200 text-gray-700 font-bold py-2 rounded-xl text-sm transition flex items-center justify-center gap-1">
                                                        <i className="lucide-more-horizontal"></i>
                                                    </button>
                                                </div>
                                            </div>
                                        </div>
                                    ))}
                                </div>
                            )}
                        </div>
                    </main>

                    {/* アカウント情報モーダル */}
                    {showProfile && (
                        <div className="fixed inset-0 bg-black/60 backdrop-blur-sm flex items-center justify-center p-4 z-50 fade-in-up" onClick={() => setShowProfile(false)}>
                            <div className="bg-white rounded-3xl p-8 w-full max-w-sm shadow-2xl space-y-6" onClick={e => e.stopPropagation()}>
                                <div className="text-center border-b pb-4">
                                    <div className="w-20 h-20 bg-gradient-to-tr from-blue-500 to-purple-500 rounded-full flex items-center justify-center text-white text-3xl font-bold mx-auto mb-4 shadow-lg">
                                        {currentUser.username.charAt(0).toUpperCase()}
                                    </div>
                                    <h2 className="text-2xl font-bold text-gray-800">{currentUser.username}</h2>
                                    <p className="text-blue-600 text-sm font-bold flex items-center justify-center gap-1">
                                        <i className="lucide-check-circle"></i>ゆいきちアカウント同期中
                                    </p>
                                </div>
                                <div className="bg-gray-50 p-4 rounded-2xl border border-gray-200">
                                    <label className="block text-xs font-bold text-gray-500 mb-1">パスワード (タップして表示)</label>
                                    <p className="font-mono font-bold text-lg text-gray-800 break-all cursor-pointer select-all" onClick={e => e.currentTarget.classList.toggle('text-transparent') || e.currentTarget.classList.toggle('bg-clip-text') || e.currentTarget.classList.toggle('bg-gradient-to-r') || e.currentTarget.classList.toggle('from-gray-800') || e.currentTarget.classList.toggle('to-gray-600')}>
                                        {currentUser.password}
                                    </p>
                                </div>
                                <button onClick={() => handleAuthAction('LOGOUT')} className="w-full bg-red-50 hover:bg-red-100 text-red-600 font-bold py-3 rounded-2xl transition flex items-center justify-center gap-2">
                                    <i className="lucide-log-out"></i>ログアウト
                                </button>
                            </div>
                        </div>
                    )}
                </div>
            );
        }

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>
