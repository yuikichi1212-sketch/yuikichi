<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>look - 究極のフル画面ビューア</title>
    <script src="https://unpkg.com/react@18/umd/react.development.js" crossorigin></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js" crossorigin></script>
    <script src="https://unpkg.com/babel-standalone@6/babel.min.js"></script>
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://cdn.jsdelivr.net/npm/lucide-static@0.320.0/font/lucide.css" rel="stylesheet">
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;700;900&display=swap');
        body { font-family: 'Inter', sans-serif; background: #0f172a; color: white; overflow-x: hidden; }
        .glass { background: rgba(255, 255, 255, 0.05); backdrop-filter: blur(12px); border: 1px solid rgba(255, 255, 255, 0.1); }
        .app-gradient { background: linear-gradient(135deg, #3b82f6 0%, #8b5cf6 100%); }
        .full-screen-preview { position: fixed; inset: 0; z-index: 100; background: rgba(0,0,0,0.95); display: flex; align-items: center; justify-content: center; animation: zoomIn 0.3s ease-out; }
        @keyframes zoomIn { from { opacity: 0; transform: scale(0.9); } to { opacity: 1; transform: scale(1); } }
        .custom-scrollbar::-webkit-scrollbar { width: 6px; }
        .custom-scrollbar::-webkit-scrollbar-thumb { background: rgba(255,255,255,0.2); border-radius: 10px; }
    </style>
</head>
<body>
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useRef, useEffect } = React;

        const AppIcon = ({ size = 'w-24 h-24' }) => (
            <img src="look.jpg" alt="look" className={`${size} rounded-[2rem] shadow-2xl object-cover border-2 border-white/20`} 
                 onError={(e) => { e.target.src = 'https://via.placeholder.com/200?text=look'; }} />
        );

        function App() {
            // --- 状態管理 ---
            const [isLoggedIn, setIsLoggedIn] = useState(false);
            const [currentUser, setCurrentUser] = useState({ username: '', password: '', profilePic: 'look.jpg' });
            const [authStep, setAuthStep] = useState(0); 
            const [authMode, setAuthMode] = useState(null);
            const [inputName, setInputName] = useState('');
            const [inputPass, setInputPass] = useState('');
            const [error, setError] = useState('');

            const [files, setFiles] = useState([]);
            const [fullscreenFile, setFullscreenFile] = useState(null);
            const [showSettings, setShowSettings] = useState(false);

            // --- 認証ロジック ---
            const handleNext = () => {
                setError('');
                if (authStep === 1) {
                    if (inputName.length < 5) return setError('アカウント名は5文字以上必要です。');
                    setAuthStep(2);
                } else if (authStep === 2) {
                    if (inputPass.length < 5 || !/[a-zA-Z]/.test(inputPass) || !/[0-9]/.test(inputPass)) {
                        return setError('数字と文字を混ぜて5桁以上で入力してください！');
                    }
                    setCurrentUser({ ...currentUser, username: inputName, password: inputPass });
                    setIsLoggedIn(true);
                }
            };

            // --- ファイル・プロフ画像処理 ---
            const onFileUpload = (e) => {
                const file = e.target.files[0];
                if (!file) return;
                const reader = new FileReader();
                reader.onloadend = () => {
                    setFiles([{ id: Date.now(), name: file.name, type: file.type, data: reader.result }, ...files]);
                };
                reader.readAsDataURL(file);
            };

            const changeProfilePic = (e) => {
                const file = e.target.files[0];
                if (!file) return;
                const reader = new FileReader();
                reader.onloadend = () => {
                    setCurrentUser({ ...currentUser, profilePic: reader.result });
                };
                reader.readAsDataURL(file);
            };

            // ==========================================
            // ログイン画面
            // ==========================================
            if (!isLoggedIn) {
                return (
                    <div className="min-h-screen flex items-center justify-center p-4 bg-[radial-gradient(ellipse_at_top,_var(--tw-gradient-stops))] from-blue-900 via-slate-900 to-black">
                        <div className="glass p-10 rounded-[3rem] w-full max-w-md text-center shadow-2xl">
                            <div className="flex justify-center mb-6"><AppIcon /></div>
                            <h1 className="text-4xl font-black mb-8 tracking-tighter">look</h1>
                            
                            {error && <div className="mb-4 text-red-400 font-bold bg-red-400/10 p-3 rounded-xl text-sm">{error}</div>}

                            {authStep === 0 ? (
                                <button onClick={() => setAuthStep(1)} className="w-full app-gradient py-4 rounded-2xl font-black text-xl hover:scale-105 transition shadow-lg">
                                    ゆいきちアカウントを作成
                                </button>
                            ) : authStep === 1 ? (
                                <div className="space-y-4">
                                    <input type="text" placeholder="アカウント名を決めてください" value={inputName} onChange={e=>setInputName(e.target.value)} 
                                           className="w-full bg-white/10 p-4 rounded-2xl border border-white/20 focus:outline-none focus:border-blue-500 font-bold" />
                                    <button onClick={handleNext} className="w-full bg-white text-black py-4 rounded-2xl font-black">次へ進む</button>
                                </div>
                            ) : (
                                <div className="space-y-4">
                                    <p className="text-blue-400 font-bold">@{inputName} さんのパスワード</p>
                                    <input type="password" placeholder="5桁以上 (英数字混合)" value={inputPass} onChange={e=>setInputPass(e.target.value)} 
                                           className="w-full bg-white/10 p-4 rounded-2xl border border-white/20 focus:outline-none focus:border-blue-500 font-bold" />
                                    <button onClick={handleNext} className="w-full app-gradient py-4 rounded-2xl font-black">アプリを開始！</button>
                                </div>
                            )}
                        </div>
                    </div>
                );
            }

            // ==========================================
            // メイン画面
            // ==========================================
            return (
                <div className="min-h-screen flex flex-col">
                    {/* 全画面プレビュー */}
                    {fullscreenFile && (
                        <div className="full-screen-preview" onClick={() => setFullscreenFile(null)}>
                            <button className="absolute top-8 right-8 text-white text-4xl">&times;</button>
                            {fullscreenFile.type.startsWith('image/') ? (
                                <img src={fullscreenFile.data} className="max-w-[90%] max-h-[90%] object-contain" />
                            ) : fullscreenFile.type.startsWith('video/') ? (
                                <video src={fullscreenFile.data} controls autoPlay className="max-w-[90%] max-h-[90%]" />
                            ) : (
                                <div className="text-center">
                                    <div className="text-8xl mb-4">📄</div>
                                    <p className="text-2xl font-bold">{fullscreenFile.name}</p>
                                    <p className="text-gray-400">プレビュー非対応形式</p>
                                </div>
                            )}
                        </div>
                    )}

                    <header className="glass m-4 rounded-3xl p-4 flex justify-between items-center sticky top-0 z-50">
                        <div className="flex items-center gap-3">
                            <AppIcon size="w-12 h-12" />
                            <span className="text-2xl font-black tracking-tighter">look</span>
                        </div>
                        <div className="flex gap-4">
                            <label className="bg-blue-600 hover:bg-blue-500 p-3 rounded-2xl cursor-pointer transition flex items-center gap-2">
                                <span className="font-bold hidden sm:block">アップロード</span>
                                <span className="text-xl">+</span>
                                <input type="file" className="hidden" onChange={onFileUpload} />
                            </label>
                            <img src={currentUser.profilePic} onClick={() => setShowSettings(true)} 
                                 className="w-12 h-12 rounded-2xl object-cover border-2 border-blue-500 cursor-pointer hover:scale-110 transition" />
                        </div>
                    </header>

                    <main className="flex-1 p-6 max-w-6xl mx-auto w-full">
                        <h2 className="text-3xl font-black mb-8 flex items-center gap-3">
                            <span className="w-2 h-8 app-gradient rounded-full"></span>
                            マイ・ドライブ
                        </h2>

                        {files.length === 0 ? (
                            <div className="glass rounded-[3rem] p-20 text-center border-dashed border-2 border-white/10">
                                <div className="text-6xl mb-4 opacity-20">📁</div>
                                <p className="text-gray-400 font-bold">まだファイルがありません。<br/>上の「+」ボタンから追加してください！</p>
                            </div>
                        ) : (
                            <div className="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-6">
                                {files.map(file => (
                                    <div key={file.id} onClick={() => setFullscreenFile(file)}
                                         className="glass rounded-3xl overflow-hidden group cursor-pointer hover:border-blue-500/50 transition-all transform hover:-translate-y-2">
                                        <div className="h-40 bg-black/40 flex items-center justify-center overflow-hidden">
                                            {file.type.startsWith('image/') ? (
                                                <img src={file.data} className="w-full h-full object-cover group-hover:scale-110 transition duration-500" />
                                            ) : (
                                                <span className="text-4xl">📄</span>
                                            )}
                                        </div>
                                        <div className="p-4 bg-white/5">
                                            <p className="font-bold truncate text-sm">{file.name}</p>
                                            <p className="text-[10px] text-gray-500 mt-1 uppercase tracking-widest font-black">{file.type.split('/')[1]}</p>
                                        </div>
                                    </div>
                                ))}
                            </div>
                        )}
                    </main>

                    {/* 設定モーダル */}
                    {showSettings && (
                        <div className="fixed inset-0 bg-black/80 backdrop-blur-md z-[60] flex items-center justify-center p-4">
                            <div className="glass p-10 rounded-[3rem] w-full max-w-md relative">
                                <button onClick={()=>setShowSettings(false)} className="absolute top-6 right-6 text-2xl">&times;</button>
                                <h2 className="text-2xl font-black mb-8">ゆいきちアカウント設定</h2>
                                
                                <div className="flex flex-col items-center gap-4 mb-8">
                                    <img src={currentUser.profilePic} className="w-32 h-32 rounded-3xl object-cover border-4 border-blue-500 shadow-2xl" />
                                    <label className="text-blue-400 font-bold cursor-pointer hover:underline text-sm">
                                        プロフィール画像を変更
                                        <input type="file" className="hidden" accept="image/*" onChange={changeProfilePic} />
                                    </label>
                                </div>

                                <div className="space-y-4">
                                    <div className="bg-white/5 p-4 rounded-2xl">
                                        <p className="text-xs text-gray-400 font-bold">アカウント名</p>
                                        <p className="text-lg font-black">{currentUser.username}</p>
                                    </div>
                                    <div className="bg-white/5 p-4 rounded-2xl">
                                        <p className="text-xs text-gray-400 font-bold">現在のパスワード</p>
                                        <p className="text-lg font-black">{currentUser.password}</p>
                                    </div>
                                    <button onClick={() => location.reload()} className="w-full bg-red-500/20 text-red-500 py-4 rounded-2xl font-black hover:bg-red-500 hover:text-white transition">
                                        ログアウト
                                    </button>
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
