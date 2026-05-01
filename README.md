<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>歐德智慧丈量 - 智揚性能旗艦版</title>
    
    <!-- 核心套件：確保絕對不空白 -->
    <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/lucide@latest"></script>

    <style>
        @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+TC:wght@400;700;900&display=swap');
        body { font-family: 'Noto Sans TC', sans-serif; background-color: #f1f5f9; -webkit-tap-highlight-color: transparent; }
        .record-pulse { animation: pulse 1.5s infinite; }
        @keyframes pulse {
            0% { transform: scale(1); box-shadow: 0 0 0 0 rgba(37, 99, 235, 0.4); }
            70% { transform: scale(1.05); box-shadow: 0 0 0 15px rgba(37, 99, 235, 0); }
            100% { transform: scale(1); box-shadow: 0 0 0 0 rgba(37, 99, 235, 0); }
        }
        .virtual-list { max-height: 60vh; overflow-y: auto; }
        .shorthand-badge { background: #e0f2fe; color: #0369a1; font-size: 10px; padding: 2px 6px; border-radius: 4px; font-weight: 900; }
        input:focus { border-color: #2563eb !important; outline: none; box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.1); }
    </style>
</head>
<body>
    <div id="root">
        <div class="min-h-screen bg-slate-900 flex flex-col items-center justify-center text-white">
            <div class="w-12 h-12 border-4 border-blue-500 border-t-transparent rounded-full animate-spin mb-4"></div>
            <h2 class="text-xl font-black tracking-widest italic uppercase">智揚官方系統 啟動中</h2>
            <p class="text-slate-500 text-xs mt-2">正在與雲端後台建立安全連線...</p>
        </div>
    </div>

    <!-- Firebase SDK (穩定版) -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, onAuthStateChanged, signInWithCustomToken } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, setDoc, getDoc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        window.FB = { initializeApp, getAuth, signInAnonymously, onAuthStateChanged, signInWithCustomToken, getFirestore, doc, setDoc, getDoc };
    </script>

    <script type="text/babel">
        const { useState, useEffect, useRef, useMemo } = React;

        const App = () => {
            const [isReady, setIsReady] = useState(false);
            const [db, setDb] = useState(null);
            const [user, setUser] = useState(null);
            const [appId, setAppId] = useState('od-official-app');
            const [measurements, setMeasurements] = useState([]);
            const [isRecording, setIsRecording] = useState(false);
            const [isLoading, setIsLoading] = useState(false);
            const [timer, setTimer] = useState(0);
            const [videoUrl, setVideoUrl] = useState(null);
            const [status, setStatus] = useState({ type: 'info', msg: '系統就緒' });
            const [searchTerm, setSearchTerm] = useState("");

            const mediaRecorder = useRef(null);
            const timerRef = useRef(null);
            const fileInputRef = useRef(null);

            const [header, setHeader] = useState({
                company: "歐德",
                contact: "王廷羽",
                address: "桃園市中壢區青商路419號14樓",
                orderNo: "20260412",
                surveyor: "教父",
                phone: "0923390718",
                date: new Date().toISOString().split('T')[0]
            });

            useEffect(() => {
                const init = async () => {
                    try {
                        const configStr = typeof __firebase_config !== 'undefined' ? __firebase_config : '{}';
                        const config = JSON.parse(configStr);
                        const app = window.FB.initializeApp(config);
                        const auth = window.FB.getAuth(app);
                        const firestore = window.FB.getFirestore(app);
                        const aid = typeof __app_id !== 'undefined' ? __app_id : 'od-survey-system';
                        
                        setDb(firestore);
                        setAppId(aid);

                        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                            await window.FB.signInWithCustomToken(auth, __initial_auth_token);
                        } else {
                            await window.FB.signInAnonymously(auth);
                        }

                        window.FB.onAuthStateChanged(auth, (u) => {
                            setUser(u);
                            setIsReady(true);
                            if (u && header.orderNo) loadCloudData(firestore, aid, header.orderNo);
                        });
                    } catch (e) {
                        setIsReady(true);
                        setStatus({ type: 'error', msg: '雲端模式異常，目前為本地運作' });
                    }
                };
                setTimeout(init, 500);
            }, []);

            const calc = (v, offset) => {
                const n = parseInt(v) || 0;
                return n > 0 ? n - offset : 0;
            };

            const loadCloudData = async (fdb, aid, oNo) => {
                if (!fdb || !oNo) return;
                setIsLoading(true);
                try {
                    const docRef = window.FB.doc(fdb, 'artifacts', aid, 'public', 'data', 'cases', oNo);
                    const snap = await window.FB.getDoc(docRef);
                    if (snap.exists()) {
                        const d = snap.data();
                        setHeader(prev => ({ ...prev, ...d, orderNo: oNo }));
                        setMeasurements(d.items || []);
                        setStatus({ type: 'success', msg: `已同步載入 ${d.items?.length || 0} 筆案場數據` });
                    }
                } catch (e) { console.error(e); } finally { setIsLoading(false); }
            };

            const syncToCloud = async (itemsToSave = measurements) => {
                if (!db || !user || !header.orderNo) return;
                try {
                    const docRef = window.FB.doc(db, 'artifacts', appId, 'public', 'data', 'cases', header.orderNo);
                    await window.FB.setDoc(docRef, {
                        ...header,
                        items: itemsToSave,
                        lastUpdate: new Date().toISOString()
                    });
                    setStatus({ type: 'success', msg: '官方後台同步成功' });
                } catch (e) { setStatus({ type: 'error', msg: '同步失敗' }); }
            };

            const handleRecording = async () => {
                if (isRecording) {
                    mediaRecorder.current.stop();
                    mediaRecorder.current.stream.getTracks().forEach(t => t.stop());
                    setIsRecording(false);
                    clearInterval(timerRef.current);
                } else {
                    try {
                        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
                        mediaRecorder.current = new MediaRecorder(stream);
                        const chunks = [];
                        mediaRecorder.current.ondataavailable = e => chunks.push(e.data);
                        mediaRecorder.current.onstop = () => processVoice(new Blob(chunks, { type: 'audio/webm' }));
                        mediaRecorder.current.start();
                        setIsRecording(true);
                        setTimer(0);
                        timerRef.current = setInterval(() => setTimer(p => p + 1), 1000);
                        setStatus({ type: 'info', msg: '聆聽中 (支援斜線快速報法)...' });
                    } catch (e) { alert("麥克風被拒絕"); }
                }
            };

            const processVoice = async (blob) => {
                setIsLoading(true);
                const reader = new FileReader();
                reader.readAsDataURL(blob);
                reader.onloadend = async () => {
                    const base64 = reader.result.split(',')[1];
                    const apiKey = ""; 
                    const prompt = `你是一位專業裝潢秘書。提取大量數據並換算。
                    支援斜線格式：編號 洞寬/洞高/牆厚/樣式/門止/備註。
                    例如：1號 898/2190/125/房/隱/門下大陸。
                    規則：門寬=洞寬-48, 門高=洞高-30。
                    回傳 JSON: {"items":[{"no":"1.","w":898,"h":2190,"t":125,"style":"房","stop":"隱","note":"門下大陸"}]}`;
                    try {
                        const res = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
                            method: 'POST',
                            headers: { 'Content-Type': 'application/json' },
                            body: JSON.stringify({
                                contents: [{ parts: [{ text: "解析錄音數據：" }, { inlineData: { mimeType: "audio/webm", data: base64 } }] }],
                                systemInstruction: { parts: [{ text: prompt }] },
                                generationConfig: { responseMimeType: "application/json" }
                            })
                        });
                        const data = await res.json();
                        const p = JSON.parse(data.candidates[0].content.parts[0].text);
                        if (p.items) {
                            const newItems = p.items.map(i => ({
                                id: Date.now() + Math.random(),
                                no: i.no || (measurements.length + 1) + ".",
                                w: i.w, h: i.h,
                                dw: calc(i.w, 48), dh: calc(i.h, 30),
                                t: i.t, body: i.t, trim: 50,
                                style: i.style || "房", stop: i.stop || "隱", note: i.note || ""
                            }));
                            const updated = [...measurements, ...newItems];
                            setMeasurements(updated);
                            syncToCloud(updated);
                        }
                    } catch (e) { setStatus({ type: 'error', msg: '解析超時' }); }
                    finally { setIsLoading(false); }
                };
            };

            const addManualRow = () => {
                const newRow = { id: Date.now(), no: (measurements.length + 1) + ".", w: 0, h: 0, dw: 0, dh: 0, t: 0, body: 0, trim: 50, style: "房", stop: "隱", note: "" };
                setMeasurements(prev => [...prev, newRow]);
            };

            const updateItem = (id, field, val) => {
                setMeasurements(prev => prev.map(m => {
                    if (m.id === id) {
                        let u = { ...m, [field]: val };
                        if (field === 'w') u.dw = calc(val, 48);
                        if (field === 'h') u.dh = calc(val, 30);
                        if (field === 't') u.body = val;
                        return u;
                    }
                    return m;
                }));
            };

            const backupToSynology = () => {
                const data = { header, items: measurements, count: measurements.length, backupTime: new Date().toLocaleString() };
                const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' });
                const link = document.createElement("a");
                link.href = URL.createObjectURL(blob);
                link.download = `智揚備份_${header.orderNo}_${measurements.length}筆.json`;
                link.click();
                setStatus({ type: 'success', msg: '備份檔已生成，請上傳至 Synology Drive' });
            };

            // 核心：Excel 指定格式導出邏輯
            const exportSpecificExcel = () => {
                if (measurements.length === 0) return alert("尚無丈量數據");
                
                // 依照範本格位精確建構 CSV 陣列
                const rows = [
                    [,,,, "下料單",,,, "訂單編號：",, header.orderNo, "", "", "", "", "", ""],
                    [,,,,,,,, "丈量人員：",, header.surveyor, "", "", "", "", "", ""],
                    ["公        司：        ", "", "", header.company, "", "", "", "", "丈量日期：",, header.date, "", "", "", "", "房", "崁", "高-32"],
                    ["聯  絡  人：        ", "", "", header.contact, "", "電話:", header.phone, "", "", "", "", "", "", "", "", "房", "隱", ""],
                    ["案場地址：", "", "", header.address, "", "", "", "", "", "", "", "", "", "", "", "房", "吸", ""],
                    [,,,,,,,,,,,,,,,, "浴", "一", ""],
                    [`"門框\n樣式:"`, "", "同門片", "", "", "", `"門片\n樣式:  "`, "平烤正白", "", "", "", "", "", "", "", "", ""],
                    ["編號", "", "洞寬", "洞高", "門寬", "門高", "牆厚", "主體", "線板", "備註", "", "", "樣式", "門止", "底"]
                ];

                measurements.forEach(m => {
                    rows.push([
                        m.no, "", m.w, m.h, m.dw, m.dh, m.t, m.body, m.trim, m.note, "", "", m.style, m.stop, ""
                    ]);
                });

                const csvContent = "\ufeff" + rows.map(r => r.join(",")).join("\n");
                const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' });
                const link = document.createElement("a");
                link.href = URL.createObjectURL(blob);
                link.download = `尺寸單_${header.contact}_${header.orderNo}.csv`;
                link.click();
                setStatus({ type: 'success', msg: '已按官方格式導出尺寸單' });
            };

            const filteredItems = useMemo(() => {
                if (!searchTerm) return measurements;
                return measurements.filter(m => m.no.includes(searchTerm) || m.note.includes(searchTerm));
            }, [measurements, searchTerm]);

            useEffect(() => { 
                if (isReady) setTimeout(() => lucide.createIcons(), 100); 
            }, [measurements, isRecording, videoUrl, isReady, status]);

            if (!isReady) return null;

            return (
                <div className="min-h-screen flex flex-col">
                    {/* Top Nav */}
                    <nav className="bg-slate-900 text-white p-4 shadow-xl sticky top-0 z-50 flex justify-between items-center border-b border-white/10">
                        <div className="flex items-center gap-3">
                            <div className="bg-blue-600 p-2 rounded-lg shadow-lg"><i data-lucide="shield-check" className="w-5 h-5"></i></div>
                            <div>
                                <h1 className="text-lg font-black italic tracking-tighter uppercase">Zhiyang <span className="text-blue-400">Pro</span></h1>
                                <div className="flex items-center gap-1">
                                    <div className={`w-1.5 h-1.5 rounded-full ${user ? 'bg-emerald-500' : 'bg-red-500'}`}></div>
                                    <span className="text-[8px] font-bold text-slate-500 uppercase tracking-widest">{user ? 'Online' : 'Local'}</span>
                                </div>
                            </div>
                        </div>
                        <div className="flex gap-2">
                            <button onClick={backupToSynology} className="bg-white/10 hover:bg-white/20 border border-white/10 px-3 py-1.5 rounded-xl text-[10px] font-black transition flex items-center gap-1">
                                <i data-lucide="database" className="w-3 h-3 text-amber-400"></i> NAS 備份
                            </button>
                            <button onClick={exportSpecificExcel} className="bg-emerald-600 hover:bg-emerald-700 px-4 py-1.5 rounded-xl text-[10px] font-black shadow-lg transition flex items-center gap-1">
                                <i data-lucide="file-spreadsheet" className="w-3 h-3"></i> 導出 EXCEL
                            </button>
                        </div>
                    </nav>

                    <main className="p-4 lg:p-8 max-w-7xl mx-auto w-full space-y-6">
                        {/* Header Info */}
                        <div className="bg-white rounded-[2.5rem] p-8 shadow-sm border border-slate-200 grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6 relative">
                            <div className="space-y-1">
                                <label className="text-[9px] font-black text-slate-400 uppercase tracking-[0.2em]">訂單編號 / {measurements.length} 樘</label>
                                <div className="flex gap-1">
                                    <input value={header.orderNo} onBlur={() => loadCloudData(db, appId, header.orderNo)} onChange={e=>setHeader({...header, orderNo: e.target.value})} className="w-full border-b-2 border-slate-100 p-1 text-sm font-black text-blue-600 outline-none" />
                                    <button onClick={() => loadCloudData(db, appId, header.orderNo)} className="p-1 text-blue-600"><i data-lucide="refresh-cw" className="w-4 h-4"></i></button>
                                </div>
                            </div>
                            <div className="space-y-1 lg:col-span-2">
                                <label className="text-[9px] font-black text-slate-400 uppercase tracking-[0.2em]">案場地址</label>
                                <input value={header.address} onChange={e=>setHeader({...header, address: e.target.value})} className="w-full border-b-2 border-slate-100 p-1 text-sm font-bold outline-none" />
                            </div>
                            <div className="space-y-1">
                                <label className="text-[9px] font-black text-slate-400 uppercase tracking-[0.2em]">手機影片參考</label>
                                <button onClick={()=>fileInputRef.current.click()} className="w-full bg-slate-50 hover:bg-slate-100 py-2 rounded-xl text-[10px] font-black border border-slate-100 flex items-center justify-center gap-2 transition-all">
                                    <i data-lucide="video" className="w-3 h-3 text-red-500"></i> {videoUrl ? "預覽中" : "選取案場影片"}
                                </button>
                                <input type="file" ref={fileInputRef} hidden accept="video/*" onChange={e=>setVideoUrl(URL.createObjectURL(e.target.files[0]))} />
                            </div>
                        </div>

                        {videoUrl && (
                            <div className="bg-black rounded-[2rem] overflow-hidden shadow-2xl aspect-video border-8 border-white relative max-w-2xl mx-auto group">
                                <video src={videoUrl} controls className="w-full h-full object-cover"></video>
                                <button onClick={()=>setVideoUrl(null)} className="absolute top-4 right-4 bg-white/20 hover:bg-red-500 text-white p-2 rounded-full backdrop-blur-md opacity-0 group-hover:opacity-100 transition-all">
                                    <i data-lucide="trash-2" className="w-4 h-4"></i>
                                </button>
                            </div>
                        )}

                        {/* Recorder Area */}
                        <div className="bg-slate-900 rounded-[3rem] p-12 text-center relative overflow-hidden shadow-2xl border-8 border-white">
                            {isLoading && (
                                <div className="absolute inset-0 bg-slate-900/95 z-30 flex flex-col items-center justify-center">
                                    <div className="w-12 h-12 border-4 border-white/20 border-t-blue-500 rounded-full animate-spin mb-4"></div>
                                    <p className="text-white text-[10px] font-black tracking-[0.3em] animate-pulse uppercase">AI 正在優化智揚下料數據...</p>
                                </div>
                            )}

                            <div className="absolute top-6 left-1/2 -translate-x-1/2">
                                <div className="bg-blue-600 text-white text-[10px] font-black px-4 py-1.5 rounded-full uppercase tracking-widest shadow-lg flex items-center gap-2">
                                    <div className="w-1.5 h-1.5 bg-white rounded-full animate-ping"></div>
                                    智揚標準：寬-48 / 高-30
                                </div>
                            </div>

                            <div className="text-white/10 font-mono text-7xl mb-10 tracking-tighter">
                                {Math.floor(timer/60).toString().padStart(2,'0')}:{(timer%60).toString().padStart(2,'0')}
                            </div>
                            
                            <button onClick={handleRecording} className={`w-32 h-32 rounded-full flex items-center justify-center mx-auto transition-all border-[12px] border-slate-800 shadow-2xl ${isRecording ? 'bg-red-500 border-red-50 record-pulse' : 'bg-white hover:scale-105'}`}>
                                {isRecording ? <div className="w-10 h-10 bg-white rounded-lg"></div> : <i data-lucide="mic" className="w-12 h-12 text-slate-900"></i>}
                            </button>
                            
                            <p className="text-slate-500 mt-10 text-[10px] font-black uppercase tracking-[0.5em]">{status.msg}</p>
                        </div>

                        {/* Search & List */}
                        <div className="bg-white rounded-[2.5rem] shadow-sm border border-slate-200 overflow-hidden">
                            <div className="p-6 bg-slate-50/80 border-b border-slate-100 flex flex-col md:flex-row justify-between items-center gap-4">
                                <div className="relative w-full md:w-64">
                                    <i data-lucide="search" className="absolute left-3 top-1/2 -translate-y-1/2 w-4 h-4 text-slate-400"></i>
                                    <input placeholder="搜索編號或備註..." value={searchTerm} onChange={e=>setSearchTerm(e.target.value)} className="w-full pl-10 pr-4 py-2.5 bg-white border border-slate-200 rounded-xl text-xs font-bold outline-none focus:border-blue-500" />
                                </div>
                                <button onClick={addManualRow} className="bg-indigo-600 text-white px-6 py-2.5 rounded-2xl text-xs font-black hover:bg-indigo-700 transition shadow-lg shadow-indigo-200">+ 手動新增項目</button>
                            </div>
                            
                            <div className="overflow-x-auto virtual-list">
                                <table className="w-full text-left min-w-[1200px] border-separate border-spacing-0">
                                    <thead>
                                        <tr className="bg-slate-50 text-[10px] text-slate-400 font-black uppercase tracking-widest border-b border-slate-100">
                                            <th className="px-6 py-4 w-20 text-center">編號</th>
                                            <th className="px-4 py-4 text-blue-700 font-black">洞寬 (W)</th>
                                            <th className="px-4 py-4 text-blue-700 font-black">洞高 (H)</th>
                                            <th className="px-4 py-4 text-emerald-600 bg-emerald-50/30 font-black text-center border-x border-slate-100">門寬(-48)</th>
                                            <th className="px-4 py-4 text-emerald-600 bg-emerald-50/30 font-black text-center">門高(-30)</th>
                                            <th className="px-4 py-4 font-black">牆厚 (T)</th>
                                            <th className="px-4 py-4 text-slate-400 font-black text-center">主體</th>
                                            <th className="px-4 py-4 text-slate-400 font-black text-center">線板</th>
                                            <th className="px-6 py-4 font-black">樣式 / 門止 / 備註</th>
                                            <th className="px-4 py-4"></th>
                                        </tr>
                                    </thead>
                                    <tbody className="divide-y divide-slate-100">
                                        {filteredItems.map(m => (
                                            <tr key={m.id} className="group hover:bg-blue-50/30 transition-all">
                                                <td className="px-6 py-4 font-bold text-slate-300 text-xs text-center">{m.no}</td>
                                                <td className="px-4 py-4"><input type="number" value={m.w} onChange={e=>updateItem(m.id, 'w', e.target.value)} className="w-full bg-transparent border-none p-0 text-sm font-black text-blue-800 focus:ring-0" /></td>
                                                <td className="px-4 py-4"><input type="number" value={m.h} onChange={e=>updateItem(m.id, 'h', e.target.value)} className="w-full bg-transparent border-none p-0 text-sm font-black text-blue-800 focus:ring-0" /></td>
                                                <td className="px-4 py-4 bg-emerald-50/10 font-black text-emerald-600 text-sm text-center border-x border-emerald-50/20">{m.dw}</td>
                                                <td className="px-4 py-4 bg-emerald-50/10 font-black text-emerald-600 text-sm text-center">{m.dh}</td>
                                                <td className="px-4 py-4"><input type="number" value={m.t} onChange={e=>updateItem(m.id, 't', e.target.value)} className="w-full bg-transparent border-none p-0 text-sm text-slate-600 font-bold focus:ring-0" /></td>
                                                <td className="px-4 py-4 text-sm text-slate-400 text-center">{m.body}</td>
                                                <td className="px-4 py-4 text-sm text-slate-400 text-center">{m.trim}</td>
                                                <td className="px-6 py-4 flex flex-col gap-1">
                                                    <div className="flex gap-2">
                                                        <span className="shorthand-badge">{m.style}</span>
                                                        <span className="shorthand-badge">{m.stop}</span>
                                                    </div>
                                                    <input value={m.note} onChange={e=>updateItem(m.id, 'note', e.target.value)} className="w-full bg-transparent border-none focus:ring-0 p-0 text-[10px] text-slate-400 italic" placeholder="點此輸入備註細節..." />
                                                </td>
                                                <td className="px-4 py-4 text-center">
                                                    <button onClick={()=>setMeasurements(measurements.filter(x=>x.id!==m.id))} className="text-slate-200 hover:text-red-500 opacity-0 group-hover:opacity-100 transition-all">
                                                        <i data-lucide="trash-2" className="w-4 h-4"></i>
                                                    </button>
                                                </td>
                                            </tr>
                                        ))}
                                    </tbody>
                                </table>
                            </div>
                        </div>
                    </main>

                    <footer className="text-center py-10 opacity-30 text-[10px] font-black uppercase tracking-[0.5em] mt-auto">
                        Official Synology Linked Engine v13.5
                    </footer>
                </div>
            );
        };

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>
