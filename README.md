<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>look - 究極のファイルビューア</title>
    <script src="https://unpkg.com/react@18/umd/react.development.js" crossorigin></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js" crossorigin></script>
    <script src="https://unpkg.com/babel-standalone@6/babel.min.js"></script>
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://cdn.jsdelivr.net/npm/lucide-static@0.320.0/font/lucide.css" rel="stylesheet">
    <style>
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background-color: #f3f4f6; }
        .fade-in { animation: fadeIn 0.3s ease-in-out; }
        @keyframes fadeIn { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }
    </style>
</head>
<body>
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useEffect } = React;

        // --- モックデータベース（本来はサーバーに保存されます） ---
        let mockUsersDB = [];

        function App() {
            // アプリ全体の状態
            const [isLoggedIn, setIsLoggedIn] = useState(false);
            const [currentUser, setCurrentUser] = useState(null);

            // ログイン/登録ウィザードの状態
            const [authMode, setAuthMode] = useState(null); // 'login' or 'signup'
            const [authStep, setAuthStep] = useState(0); // 0:選択, 1:ID入力, 2:パス入力
            const [username, setUsername] = useState('');
            const [password, setPassword] = useState('');
            const [errorMsg, setErrorMsg] = useState('');

            // アプリ内の状態
            const [showProfile, setShowProfile] = useState(false);
            const [uploadedFiles, setUploadedFiles] = useState([]);

            // --- バリデーション関数 ---
            const validateUsername = (name) => name.length >= 5;
            const validatePassword = (pass) => {
                const hasLetter = /[a-zA-Z]/.test(pass);
                const hasNumber = /[0-9]/.test(pass);
                return pass.length >= 5 && hasLetter && hasNumber;
            };

            // --- 認証フロー処理 ---
            const handleNextStep = () => {
                setErrorMsg('');
                if (authStep === 1) {
                    if (!validateUsername(username)) {
                        setErrorMsg('アカウント名は5文字以上で入力してください。');
                        return;
                    }
                    if (authMode === 'signup' && mockUsersDB.some(u => u.username === username)) {
                        setErrorMsg('このアカウント名は既に使用されています。');
                        return;
                    }
                    if (authMode === 'login' && !mockUsersDB.some(u => u.username === username)) {
                        setErrorMsg('アカウントが見つかりません。');
                        return;
                    }
                    setAuthStep(2);
                } else if (authStep === 2) {
                    if (authMode === 'signup') {
                        if (!validatePassword(password)) {
                            setErrorMsg('パスワードは英字と数字を含めて5文字以上にしてください。');
                            return;
                        }
                        const newUser = { username, password };
                        mockUsersDB.push(newUser);
                        setCurrentUser(newUser);
                        setIsLoggedIn(true);
                    } else if (authMode === 'login') {
                        const user = mockUsersDB.find(u => u.username === username && u.password === password);
                        if (user) {
                            setCurrentUser(user);
                            setIsLoggedIn(true);
                        } else {
                            setErrorMsg('パスワードが間違っています。');
                        }
                    }
                }
            };

            const handleLogout = () => {
                setIsLoggedIn(false);
                setCurrentUser(null);
                setAuthStep(0);
                setUsername('');
                setPassword('');
                setShowProfile(false);
            };

            const handleFileUpload = (e) => {
                const file = e.target.files[0];
                if (file) {
                    setUploadedFiles([...uploadedFiles, { name: file.name, type: file.name.split('.').pop(), size: (file.size / 1024).toFixed(1) + ' KB' }]);
                }
            };

            // ==========================================
            // ログイン前画面（ゆいきちアカウント認証）
            // ==========================================
            if (!isLoggedIn) {
                return (
                    <div className="min-h-screen flex items-center justify-center p-4">
                        <div className="bg-white p-8 rounded-2xl shadow-xl w-full max-w-md text-center fade-in">
                            <img src="look.jpg" alt="look icon" className="w-24 h-24 mx-auto mb-4 rounded-2xl shadow-md" onError={(e) => { e.target.src = 'https://placehold.co/150x150/FFD700/000000?text=look'; }} />
                            <h1 className="text-3xl font-bold text-gray-800 mb-2">look</h1>
                            <p className="text-gray-500 mb-6 text-sm">ゆいきちアカウントでログインして同期</p>

                            {/* ステップ0: モード選択 */}
                            {authStep === 0 && (
                                <div className="space-y-4 fade-in">
                                    <button onClick={() => { setAuthMode('login'); setAuthStep(1); }} className="w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 px-4 rounded-xl transition">
                                        ログイン
                                    </button>
                                    <button onClick={() => { setAuthMode('signup'); setAuthStep(1); }} className="w-full bg-gray-200 hover:bg-gray-300 text-gray-800 font-bold py-3 px-4 rounded-xl transition">
                                        新しいアカウントを作成
                                    </button>
                                </div>
                            )}

                            {/* ステップ1: ユーザー名入力 */}
                            {authStep === 1 && (
                                <div className="space-y-4 fade-in">
                                    <h2 className="text-xl font-bold">{authMode === 'login' ? 'ログイン' : 'アカウント作成'}</h2>
                                    <input type="text" placeholder="アカウント名 (5文字以上)" value={username} onChange={(e) => setUsername(e.target.value)} className="w-full p-3 border border-gray-300 rounded-xl focus:outline-none focus:ring-2 focus:ring-blue-500" />
                                    {errorMsg && <p className="text-red-500 text-sm">{errorMsg}</p>}
                                    <div className="flex justify-between mt-4">
                                        <button onClick={() => setAuthStep(0)} className="text-gray-500 hover:text-gray-700">戻る</button>
                                        <button onClick={handleNextStep} className="bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-6 rounded-xl">次へ</button>
                                    </div>
                                </div>
                            )}

                            {/* ステップ2: パスワード入力 */}
                            {authStep === 2 && (
                                <div className="space-y-4 fade-in">
                                    <h2 className="text-xl font-bold">パスワードを入力</h2>
                                    <p className="text-sm text-gray-500">{username} さん</p>
                                    <input type="password" placeholder="パスワード (英数字混在5桁以上)" value={password} onChange={(e) => setPassword(e.target.value)} className="w-full p-3 border border-gray-300 rounded-xl focus:outline-none focus:ring-2 focus:ring-blue-500" />
                                    {errorMsg && <p className="text-red-500 text-sm text-left">{errorMsg}</p>}
                                    <div className="flex justify-between mt-4">
                                        <button onClick={() => { setAuthStep(1); setErrorMsg(''); }} className="text-gray-500 hover:text-gray-700">戻る</button>
                                        <button onClick={handleNextStep} className="bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-6 rounded-xl">
                                            {authMode === 'login' ? 'ログイン' : '登録して開始'}
                                        </button>
                                    </div>
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
                <div className="min-h-screen bg-gray-50">
                    {/* ヘッダー */}
                    <header className="bg-white shadow-sm p-4 flex justify-between items-center sticky top-0 z-10">
                        <div className="flex items-center gap-3">
                            <img src="look.jpg" alt="logo" className="w-10 h-10 rounded-lg shadow-sm" onError={(e) => { e.target.src = 'https://placehold.co/100x100/FFD700/000000?text=look'; }} />
                            <h1 className="text-2xl font-black text-gray-800 tracking-tight">look</h1>
                        </div>
                        <button onClick={() => setShowProfile(true)} className="flex items-center gap-2 bg-gray-100 hover:bg-gray-200 px-4 py-2 rounded-full transition">
                            <i className="icon-user">👤</i>
                            <span className="font-bold text-gray-700 hidden sm:inline">{currentUser.username}</span>
                        </button>
                    </header>

                    {/* メインコンテンツ */}
                    <main className="max-w-4xl mx-auto p-4 md:p-8">
                        <div className="bg-white rounded-2xl shadow-sm border border-gray-200 p-8 text-center mb-8 fade-in">
                            <h2 className="text-2xl font-bold text-gray-800 mb-4">ファイルをドロップ、または選択</h2>
                            <p className="text-gray-500 mb-6">PDF, DOCX, MP4 から STL などの特殊形式まで何でもOK！</p>
                            <label className="cursor-pointer bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 px-8 rounded-xl inline-block transition shadow-lg shadow-blue-200">
                                ファイルを選択
                                <input type="file" className="hidden" onChange={handleFileUpload} />
                            </label>
                        </div>

                        {/* ファイルリストモック */}
                        <div className="space-y-4">
                            <h3 className="font-bold text-gray-700">最近のファイル (ゆいきちアカウントで同期中)</h3>
                            {uploadedFiles.length === 0 ? (
                                <p className="text-gray-400 text-center py-8">ファイルがありません</p>
                            ) : (
                                uploadedFiles.map((f, i) => (
                                    <div key={i} className="bg-white p-4 rounded-xl shadow-sm border border-gray-100 flex items-center justify-between fade-in flex-wrap gap-4">
                                        <div className="flex items-center gap-4">
                                            <div className="bg-blue-100 text-blue-600 font-black p-3 rounded-lg w-12 h-12 flex items-center justify-center uppercase">{f.type}</div>
                                            <div>
                                                <p className="font-bold text-gray-800 truncate max-w-[150px] sm:max-w-xs">{f.name}</p>
                                                <p className="text-sm text-gray-400">{f.size}</p>
                                            </div>
                                        </div>
                                        <div className="flex gap-2 w-full sm:w-auto mt-2 sm:mt-0">
                                            <button className="flex-1 sm:flex-none bg-gray-100 hover:bg-gray-200 text-gray-700 px-4 py-2 rounded-lg font-bold text-sm transition">プレビュー</button>
                                            <button className="flex-1 sm:flex-none bg-blue-50 hover:bg-blue-100 text-blue-600 px-4 py-2 rounded-lg font-bold text-sm transition">変換/DL</button>
                                            <button className="flex-1 sm:flex-none bg-green-50 hover:bg-green-100 text-green-600 px-4 py-2 rounded-lg font-bold text-sm transition">共有</button>
                                        </div>
                                    </div>
                                ))
                            )}
                        </div>
                    </main>

                    {/* アカウント情報モーダル */}
                    {showProfile && (
                        <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50 fade-in">
                            <div className="bg-white rounded-2xl p-8 w-full max-w-sm shadow-2xl">
                                <h2 className="text-2xl font-bold mb-6 border-b pb-2">ゆいきちアカウント情報</h2>
                                <div className="space-y-4 mb-8">
                                    <div>
                                        <label className="text-sm text-gray-500 font-bold block">アカウント名</label>
                                        <p className="text-lg text-gray-800 font-mono bg-gray-50 p-2 rounded">{currentUser.username}</p>
                                    </div>
                                    <div>
                                        <label className="text-sm text-gray-500 font-bold block">パスワード</label>
                                        <p className="text-lg text-gray-800 font-mono bg-gray-50 p-2 rounded select-all">{currentUser.password}</p>
                                    </div>
                                    <p className="text-xs text-blue-600 text-center bg-blue-50 p-2 rounded-lg">✨ アカウントのデータは同期されています</p>
                                </div>
                                <div className="flex gap-4">
                                    <button onClick={() => setShowProfile(false)} className="flex-1 bg-gray-200 hover:bg-gray-300 text-gray-800 font-bold py-2 rounded-xl">閉じる</button>
                                    <button onClick={handleLogout} className="flex-1 bg-red-600 hover:bg-red-700 text-white font-bold py-2 rounded-xl">ログアウト</button>
                                </div>
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
