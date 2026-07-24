# dse-vocab
import React, { useState, useEffect, useRef, useMemo } from 'react';
import { Check, RotateCcw, Trash2, Trophy, Volume2, List, Upload, FileText, Loader2, Sparkles, ChevronLeft, ChevronRight, Edit3, Cloud, LogIn, LogOut, X, Save, AlertTriangle } from 'lucide-react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInWithPopup, GoogleAuthProvider, signOut, onAuthStateChanged, signInAnonymously } from 'firebase/auth';
import { getFirestore, doc, setDoc, getDoc, deleteDoc } from 'firebase/firestore';

let app, auth, db, appId = 'default-app-id';
try {
  if (typeof __firebase_config !== 'undefined' && __firebase_config) {
    const firebaseConfig = JSON.parse(__firebase_config);
    app = initializeApp(firebaseConfig);
    auth = getAuth(app);
    db = getFirestore(app);
    appId = typeof __app_id !== 'undefined' ? __app_id : 'dse-vocab-app';
  }
} catch (e) {
  console.log("離線/沙盒模式運行中");
}

export default function App() {
  const [words, setWords] = useState([]);
  const [view, setView] = useState('import'); // 'import', 'study', 'stats', 'list'
  
  const [studyMode, setStudyMode] = useState('all'); 
  const [studyIndex, setStudyIndex] = useState(0);
  const [isFlipped, setIsFlipped] = useState(false);
  
  const [definition, setDefinition] = useState(null);
  const [isLoadingDef, setIsLoadingDef] = useState(false);
  const [isParsing, setIsParsing] = useState(false);
  const [parseStatus, setParseStatus] = useState('');
  
  const [listPage, setListPage] = useState(1);
  const [searchQuery, setSearchQuery] = useState('');
  const [listFilter, setListFilter] = useState('all');
  const [sortBy, setSortBy] = useState('alpha'); 
  const [customWordInput, setCustomWordInput] = useState('');
  
  const [wordToDelete, setWordToDelete] = useState(null);
  const [showClearConfirmModal, setShowClearConfirmModal] = useState(false); // 新增清空確認彈窗狀態
  
  const [activeDetailWord, setActiveDetailWord] = useState(null);
  const [detailNoteInput, setDetailNoteInput] = useState('');
  const [detailDef, setDetailDef] = useState(null);
  const [isLoadingDetailDef, setIsLoadingDetailDef] = useState(false);
  const [detailStatus, setDetailStatus] = useState('new');

  const [user, setUser] = useState(null);
  const [isCloudSyncing, setIsCloudSyncing] = useState(false);
  
  const LIST_PAGE_SIZE = 20;
  const fileInputRef = useRef(null);
  const LOCAL_STORAGE_KEY = 'dse_vocab_react_final_v17';

  useEffect(() => {
    if (auth) {
      const initAuth = async () => {
        try {
          if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
            // Token 登入
          } else {
            await signInAnonymously(auth);
          }
        } catch(e) {
          console.error(e);
        }
      };
      initAuth();
      const unsubscribe = onAuthStateChanged(auth, (currentUser) => {
        setUser(currentUser);
        if (currentUser && db && !currentUser.isAnonymous) {
          loadCloudData(currentUser.uid);
        }
      });
      return () => unsubscribe();
    } else {
      const saved = localStorage.getItem(LOCAL_STORAGE_KEY);
      if (saved) {
        try {
          const parsed = JSON.parse(saved);
          if (parsed.length > 0) {
            setWords(parsed);
            setView('study');
          }
        } catch (e) {
          console.error("載入失敗", e);
        }
      }
    }
  }, []);

  useEffect(() => {
    if (words.length > 0) {
      const safeWords = words.length > 6000 ? words.slice(0, 6000) : words;
      localStorage.setItem(LOCAL_STORAGE_KEY, JSON.stringify(safeWords));
      
      if (user && db && !user.isAnonymous) {
        saveCloudData(user.uid, safeWords);
      }
    }
  }, [words, user]);

  const saveCloudData = async (uid, data) => {
    try {
      setIsCloudSyncing(true);
      const docRef = doc(db, 'artifacts', appId, 'users', uid, 'vocabData', 'main');
      await setDoc(docRef, { words: data, updatedAt: new Date().toISOString() });
    } catch (e) {
      console.error("雲端儲存失敗", e);
    } finally {
      setIsCloudSyncing(false);
    }
  };

  const loadCloudData = async (uid) => {
    try {
      setIsCloudSyncing(true);
      const docRef = doc(db, 'artifacts', appId, 'users', uid, 'vocabData', 'main');
      const docSnap = await getDoc(docRef);
      if (docSnap.exists() && docSnap.data().words) {
        const cloudWords = docSnap.data().words;
        if (cloudWords.length > 0) {
          setWords(cloudWords);
          setView('study');
        }
      }
    } catch (e) {
      console.error("雲端讀取失敗", e);
    } finally {
      setIsCloudSyncing(false);
    }
  };

  const handleGoogleLogin = async () => {
    if (!auth) {
      alert("提示：目前在預覽環境中無法彈出 Google 授權視窗。\n請將程式碼部署至 Netlify 或 GitHub Pages 即可正常綁定！");
      return;
    }
    try {
      const provider = new GoogleAuthProvider();
      await signInWithPopup(auth, provider);
      alert("Google 帳號綁定成功！");
    } catch (e) {
      alert("登入失敗或視窗被瀏覽器攔截。");
    }
  };

  const handleLogout = async () => {
    if (auth) {
      await signOut(auth);
      alert("已登出帳號。");
    }
  };

  const studyList = useMemo(() => {
    if (studyMode === 'all') return words;
    return words.filter(w => w.status === studyMode);
  }, [words, studyMode]);

  useEffect(() => {
    if (studyList.length > 0 && studyIndex >= studyList.length) {
      setStudyIndex(Math.max(0, studyList.length - 1));
    }
  }, [studyList.length, studyIndex]);

  const loadPdfJs = () => {
    return new Promise((resolve, reject) => {
      if (window.pdfjsLib) return resolve(window.pdfjsLib);
      const script = document.createElement('script');
      script.src = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.16.105/pdf.min.js';
      script.onload = () => {
        window.pdfjsLib.GlobalWorkerOptions.workerSrc = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.16.105/pdf.worker.min.js';
        resolve(window.pdfjsLib);
      };
      script.onerror = () => reject(new Error("PDF套件載入失敗"));
      document.head.appendChild(script);
    });
  };

  const handleFileUpload = async (e) => {
    const file = e.target.files[0];
    if (!file) return;
    setIsParsing(true);
    setParseStatus('初始化解析引擎...');

    try {
      const buffer = await file.arrayBuffer();
      const pdfjs = await loadPdfJs();
      const pdf = await pdfjs.getDocument({ data: buffer }).promise;
      let fullText = '';

      for (let i = 1; i <= pdf.numPages; i++) {
        if (i % 5 === 0) {
          setParseStatus(`解析頁面 (${i}/${pdf.numPages})...`);
          await new Promise(r => setTimeout(r, 10));
        }
        const page = await pdf.getPage(i);
        const content = await page.getTextContent();
        fullText += content.items.map(item => item.str).join(' ') + ' ';
      }

      setParseStatus('過濾雜訊與重複單字...');
      await new Promise(r => setTimeout(r, 10));

      const extracted = fullText.match(/\b[a-zA-Z-]+\b/g);
      if (!extracted) throw new Error("找不到英文單字");

      const ignoreList = new Set(['page', 'vocabularylist', 'words', 'p']);
      const uniqueMap = new Map();

      extracted.forEach(w => {
        let clean = w.toLowerCase().replace(/^[-]+|[-]+$/g, '');
        if ((clean.length > 1 || clean === 'a' || clean === 'i') && !ignoreList.has(clean) && !clean.startsWith('vocabularylist')) {
          uniqueMap.set(clean, true);
        }
      });

      const newWords = Array.from(uniqueMap.keys()).map(w => ({ word: w, status: 'new', note: '' }));
      
      setWords(newWords);
      setStudyMode('all');
      setStudyIndex(0);
      setView('study');
    } catch (err) {
      alert("解析失敗，請確認檔案格式正確。");
    } finally {
      setIsParsing(false);
      setParseStatus('');
      if (fileInputRef.current) fileInputRef.current.value = '';
    }
  };

  const speakWord = (wordStr, e) => {
    if (e) e.stopPropagation();
    if ('speechSynthesis' in window) {
      window.speechSynthesis.cancel();
      const utter = new SpeechSynthesisUtterance(wordStr);
      utter.lang = 'en-US';
      utter.rate = 0.9;
      window.speechSynthesis.speak(utter);
    }
  };

  const fetchDefinition = async (wordStr) => {
    setIsLoadingDef(true);
    setDefinition(null);
    try {
      const res = await fetch(`https://api.dictionaryapi.dev/api/v2/entries/en/${wordStr}`);
      if (res.ok) {
        setDefinition((await res.json())[0]);
      } else {
        setDefinition({ notFound: true });
      }
    } catch (e) {
      setDefinition({ error: true });
    } finally {
      setIsLoadingDef(false);
    }
  };

  const openWordDetail = async (item) => {
    setActiveDetailWord(item);
    setDetailNoteInput(item.note || '');
    setDetailStatus(item.status);
    setIsLoadingDetailDef(true);
    setDetailDef(null);

    try {
      const res = await fetch(`https://api.dictionaryapi.dev/api/v2/entries/en/${item.word}`);
      if (res.ok) {
        setDetailDef((await res.json())[0]);
      } else {
        setDetailDef({ notFound: true });
      }
    } catch (e) {
      setDetailDef({ error: true });
    } finally {
      setIsLoadingDetailDef(false);
    }
  };

  const saveWordDetail = () => {
    if (!activeDetailWord) return;
    setWords(prev => prev.map(w => {
      if (w.word === activeDetailWord.word) {
        return { ...w, note: detailNoteInput, status: detailStatus };
      }
      return w;
    }));
    setActiveDetailWord(null);
    alert("備註與狀態已成功儲存！");
  };

  const handleFlip = () => {
    if (!isFlipped && studyList.length > 0) {
      fetchDefinition(studyList[studyIndex].word);
    }
    setIsFlipped(!isFlipped);
  };

  const changeWordStatus = (wordStr, newStatus) => {
    setWords(prev => prev.map(w => w.word === wordStr ? { ...w, status: newStatus } : w));
  };

  const handleMarkCard = (status, e) => {
    if (e) e.stopPropagation();
    if (studyList.length === 0) return;
    
    changeWordStatus(studyList[studyIndex].word, status);
    setIsFlipped(false);

    if (studyIndex < studyList.length - 1) {
      setStudyIndex(prev => prev + 1);
    } else {
      alert('本階段已溫習完畢！');
      setView('stats');
    }
  };

  const handleUpdateNote = (wordStr, noteStr) => {
    setWords(prev => prev.map(w => w.word === wordStr ? { ...w, note: noteStr } : w));
  };

  const triggerDelete = (wordStr, e) => {
    if (e) {
      e.stopPropagation();
      e.preventDefault();
    }
    setWordToDelete(wordStr);
  };

  const confirmDelete = () => {
    if (!wordToDelete) return;
    setWords(prev => prev.filter(w => w.word !== wordToDelete));
    setIsFlipped(false);
    if (view === 'study' && studyIndex >= studyList.length - 1) {
      setStudyIndex(Math.max(0, studyList.length - 2));
    }
    setWordToDelete(null);
  };

  // 實際執行清空所有記錄的邏輯
  const executeClearAll = async () => {
    setWords([]);
    localStorage.removeItem(LOCAL_STORAGE_KEY);
    localStorage.removeItem('dse_vocab_react_final_v11');
    localStorage.removeItem('dse_vocab_react_final_v12');
    localStorage.removeItem('dse_vocab_react_final_v13');
    localStorage.removeItem('dse_vocab_react_final_v14');
    localStorage.removeItem('dse_vocab_react_final_v15');
    localStorage.removeItem('dse_vocab_react_final_v16');

    if (user && db && !user.isAnonymous) {
      try {
        const docRef = doc(db, 'artifacts', appId, 'users', user.uid, 'vocabData', 'main');
        await deleteDoc(docRef);
      } catch (e) {
        console.error("雲端資料清除失敗", e);
      }
    }
    
    setShowClearConfirmModal(false);
    setView('import');
    alert("已徹底清空所有記錄！");
  };

  const handleAddCustomWords = () => {
    if (!customWordInput.trim()) return;
    const extracted = customWordInput.match(/\b[a-zA-Z-]+\b/g);
    if (!extracted) { alert("無效的單字"); return; }

    const currentSet = new Set(words.map(w => w.word));
    const newAdditions = [];
    extracted.forEach(w => {
      const lower = w.toLowerCase();
      if (!currentSet.has(lower)) {
        newAdditions.push({ word: lower, status: 'new', note: '' });
        currentSet.add(lower);
      }
    });

    if (newAdditions.length > 0) {
      setWords(prev => [...prev, ...newAdditions]);
      alert(`新增了 ${newAdditions.length} 個單字！`);
      setCustomWordInput('');
    } else {
      alert("單字皆已存在。");
    }
  };

  // --- 畫面渲染組件 ---

  const renderImport = () => (
    <div className="flex flex-col items-center justify-center min-h-[75vh] py-8">
      <div className="text-center mb-6">
        <h2 className="text-2xl font-extrabold text-gray-800 mb-2">上傳你的單字文件</h2>
        <p className="text-gray-500 text-xs px-4">支援大型 PDF，系統將自動萃取並保護裝置記憶體</p>
      </div>

      <div className="w-full bg-white rounded-3xl shadow-xl p-6 border border-gray-100 flex flex-col items-center">
        <div className="w-16 h-16 bg-indigo-50 rounded-full flex items-center justify-center text-indigo-600 mb-4">
          <FileText className="w-8 h-8" />
        </div>
        <input type="file" ref={fileInputRef} onChange={handleFileUpload} accept=".pdf,.txt" className="hidden" />
        <button 
          onClick={() => fileInputRef.current.click()}
          disabled={isParsing}
          className="w-full py-3 px-6 bg-indigo-600 text-white font-bold rounded-xl shadow hover:bg-indigo-700 transition flex justify-center text-sm disabled:opacity-70 cursor-pointer"
        >
          {isParsing ? <><Loader2 className="w-5 h-5 mr-2 animate-spin" /> {parseStatus}</> : <><Upload className="w-5 h-5 mr-2" /> 選擇 PDF 檔案</>}
        </button>
      </div>
    </div>
  );

  const renderStudy = () => {
    if (words.length === 0) return null;
    if (studyList.length === 0) {
      return (
        <div className="flex flex-col items-center justify-center py-20 text-center">
          <div className="text-4xl mb-4">🎉</div>
          <h2 className="text-xl font-bold text-gray-800">此範圍內沒有單字了！</h2>
          <button onClick={() => setView('stats')} className="mt-4 px-6 py-2 bg-indigo-600 text-white font-bold rounded-lg shadow cursor-pointer">返回統計</button>
        </div>
      );
    }

    const currentWordObj = studyList[studyIndex];
    const progress = Math.round((studyIndex / studyList.length) * 100);
    const modeLabels = { 'all': '全部', 'new': '未學', 'review': '錯題複習', 'known': '已掌握' };

    return (
      <div className="flex flex-col items-center w-full py-4 min-h-[85vh]">
        <div className="w-full flex justify-between items-center mb-3">
          <button onClick={() => setView('stats')} className="text-indigo-600 font-bold px-3 py-1 bg-indigo-50 rounded-lg text-xs hover:bg-indigo-100 cursor-pointer">
            統計 & 設定
          </button>
          <div className="flex items-center gap-2">
            <span className={`text-[10px] font-bold px-2 py-0.5 rounded-full ${studyMode === 'review' ? 'bg-orange-100 text-orange-700' : 'bg-gray-100 text-gray-600'}`}>
              範圍: {modeLabels[studyMode]}
            </span>
            <div className="text-gray-500 text-xs font-mono font-bold bg-white px-2 py-1 rounded-lg border border-gray-200">
              {studyIndex + 1} / {studyList.length}
            </div>
          </div>
        </div>
        
        <div className="w-full bg-gray-200 rounded-full h-1.5 mb-5 overflow-hidden">
          <div className="bg-indigo-600 h-full transition-all duration-300" style={{ width: `${progress}%` }}></div>
        </div>

        <div className="w-full bg-white rounded-[2rem] shadow-xl border border-gray-100 min-h-[360px] flex flex-col p-6 relative overflow-hidden">
          <div className="absolute top-0 right-0 w-24 h-24 bg-indigo-50 rounded-bl-full -mr-12 -mt-12 z-0"></div>

          {!isFlipped ? (
            <div className="text-center w-full break-words relative z-10 flex flex-col items-center justify-center h-full cursor-pointer" onClick={handleFlip}>
              <button 
                type="button"
                onClick={(e) => triggerDelete(currentWordObj.word, e)} 
                className="absolute top-0 left-0 p-3 text-red-400 hover:text-red-600 hover:bg-red-50 rounded-full transition z-20 cursor-pointer"
                title="刪除此單字"
              >
                <Trash2 className="w-5 h-5 pointer-events-none" />
              </button>

              <span className="text-[10px] uppercase tracking-widest text-indigo-400 font-bold mb-3 block bg-indigo-50 px-2 py-0.5 rounded-full mt-2">點擊卡片看解釋</span>
              <h1 className="text-4xl font-black text-gray-900 tracking-tight mb-4">{currentWordObj.word}</h1>
              <button onClick={(e) => speakWord(currentWordObj.word, e)} className="w-14 h-14 bg-indigo-100 text-indigo-600 rounded-full flex items-center justify-center hover:bg-indigo-200 shadow-sm mt-2 cursor-pointer">
                <Volume2 className="w-6 h-6" />
              </button>
              
              {currentWordObj.note && (
                <div className="mt-5 text-xs text-amber-600 font-semibold flex items-center bg-amber-50 px-3 py-1 rounded-full">
                  <Edit3 className="w-3 h-3 mr-1" /> 包含個人備註
                </div>
              )}
            </div>
          ) : (
            <div className="w-full h-full flex flex-col text-left relative z-10 max-h-[320px]">
              <div className="flex justify-between items-start border-b pb-2 mb-2 sticky top-0 bg-white">
                <h1 className="text-2xl font-black text-gray-900">{currentWordObj.word}</h1>
                <button onClick={(e) => speakWord(currentWordObj.word, e)} className="p-1.5 bg-indigo-50 text-indigo-600 rounded-full hover:bg-indigo-100 cursor-pointer">
                  <Volume2 className="w-4 h-4" />
                </button>
              </div>
              
              <div className="overflow-y-auto pr-1 flex-grow mb-2">
                {isLoadingDef ? (
                  <div className="flex justify-center py-6"><Loader2 className="w-6 h-6 animate-spin text-indigo-500" /></div>
                ) : definition?.notFound ? (
                  <div className="text-gray-400 text-xs italic py-4">此為自訂單字，無線上字典資料。</div>
                ) : definition?.meanings ? (
                  <div className="text-xs space-y-3 pb-2">
                    {definition.phonetic && <span className="font-mono text-indigo-600 bg-indigo-50 px-1.5 py-0.5 rounded font-bold">{definition.phonetic}</span>}
                    {definition.meanings.map((m, i) => (
                      <div key={i}>
                        <span className="inline-block px-1.5 py-0.5 bg-gray-800 text-white rounded text-[10px] font-bold mb-1">{m.partOfSpeech}</span>
                        <ul className="list-disc pl-4 space-y-1 text-gray-700">
                          {m.definitions.slice(0, 2).map((def, j) => (
                            <li key={j}>
                              <span className="font-medium">{def.definition}</span>
                              {def.example && <p className="italic text-gray-500 mt-0.5 text-[10px]">"{def.example}"</p>}
                            </li>
                          ))}
                        </ul>
                      </div>
                    ))}
                  </div>
                ) : <div className="text-red-400 text-xs">字典載入失敗</div>}
              </div>

              <div className="border-t border-gray-100 pt-3 mt-auto">
                <label className="text-[10px] font-bold text-gray-500 uppercase flex items-center mb-1">
                  📝 個人備註
                </label>
                <textarea 
                  value={currentWordObj.note}
                  onChange={(e) => handleUpdateNote(currentWordObj.word, e.target.value)}
                  placeholder="輸入專屬筆記、例句或中文翻譯..." 
                  className="w-full text-xs p-2 bg-amber-50/50 border border-amber-100 rounded-lg focus:outline-none focus:ring-1 focus:ring-amber-300 resize-none h-16 text-gray-800 placeholder-gray-400"
                />
              </div>
            </div>
          )}
        </div>

        <div className="w-full flex justify-between mt-5 gap-2">
          <button onClick={(e) => handleMarkCard('review', e)} className="flex-1 bg-white border-2 border-orange-200 text-orange-600 font-extrabold py-3 rounded-xl flex flex-col items-center justify-center shadow-sm text-xs hover:bg-orange-50 active:scale-95 transition cursor-pointer">
            <RotateCcw className="mb-0.5 w-4 h-4" /><span>忘記了</span>
          </button>
          <button onClick={(e) => handleMarkCard('new', e)} className="flex-1 bg-white border-2 border-gray-200 text-gray-500 font-extrabold py-3 rounded-xl flex flex-col items-center justify-center shadow-sm text-xs hover:bg-gray-50 active:scale-95 transition cursor-pointer">
            <span className="text-base mb-0.5 h-4 leading-none block">⏭</span><span>跳過</span>
          </button>
          <button onClick={(e) => handleMarkCard('known', e)} className="flex-1 bg-white border-2 border-green-200 text-green-600 font-extrabold py-3 rounded-xl flex flex-col items-center justify-center shadow-sm text-xs hover:bg-green-50 active:scale-95 transition cursor-pointer">
            <Check className="mb-0.5 w-4 h-4 stroke-[3]" /><span>記得</span>
          </button>
        </div>
      </div>
    );
  };

  const renderStats = () => {
    const known = words.filter(w => w.status === 'known').length;
    const review = words.filter(w => w.status === 'review').length;
    const newWords = words.filter(w => w.status === 'new').length;

    return (
      <div className="flex flex-col items-center py-6 w-full">
        <Trophy className="w-16 h-16 text-yellow-500 mb-3" />
        <h2 className="text-xl font-bold text-gray-800 mb-5">學習進度總覽</h2>
        
        <div className="grid grid-cols-3 gap-3 w-full mb-6">
          <div className="bg-white p-3 rounded-xl shadow-sm text-center border-t-2 border-green-500">
            <p className="text-[10px] text-gray-400 uppercase">已掌握</p>
            <p className="text-xl font-bold text-green-600 mt-1">{known}</p>
          </div>
          <div className="bg-white p-3 rounded-xl shadow-sm text-center border-t-2 border-orange-500">
            <p className="text-[10px] text-gray-400 uppercase">需複習</p>
            <p className="text-xl font-bold text-orange-600 mt-1">{review}</p>
          </div>
          <div className="bg-white p-3 rounded-xl shadow-sm text-center border-t-2 border-gray-400">
            <p className="text-[10px] text-gray-400 uppercase">未學</p>
            <p className="text-xl font-bold text-gray-600 mt-1">{newWords}</p>
          </div>
        </div>

        <div className="w-full flex flex-col gap-3">
          <div className="bg-indigo-50/70 p-4 rounded-xl shadow-sm border border-indigo-100 flex items-center justify-between">
            <div className="flex items-center space-x-3">
              <Cloud className="w-6 h-6 text-indigo-600" />
              <div>
                <p className="text-xs font-bold text-gray-800">雲端帳號同步</p>
                <p className="text-[10px] text-gray-500">
                  {user && !user.isAnonymous ? `已綁定: ${user.email || 'Google 帳號'}` : '尚未綁定 Google 帳號'}
                </p>
              </div>
            </div>
            {user && !user.isAnonymous ? (
              <button onClick={handleLogout} className="px-3 py-1.5 bg-white text-gray-700 border border-gray-300 rounded-lg text-xs font-bold shadow-xs hover:bg-gray-50 cursor-pointer">
                登出
              </button>
            ) : (
              <button type="button" onClick={handleGoogleLogin} className="px-3 py-1.5 bg-indigo-600 text-white rounded-lg text-xs font-bold shadow-xs hover:bg-indigo-700 flex items-center cursor-pointer">
                <LogIn className="w-3.5 h-3.5 mr-1" /> 綁定 Google
              </button>
            )}
          </div>

          <div className="bg-white p-4 rounded-xl shadow-sm border border-gray-100">
            <label className="text-xs font-bold text-gray-700 block mb-2">🎯 自訂溫習範圍：</label>
            <select 
              value={studyMode}
              onChange={(e) => { setStudyMode(e.target.value); setStudyIndex(0); }}
              className="w-full p-2.5 text-sm border border-indigo-200 rounded-lg outline-none bg-indigo-50/50 text-indigo-900 font-semibold cursor-pointer mb-3"
            >
              <option value="all">全部單字</option>
              <option value="new">🆕 只溫習「未學習」的單字</option>
              <option value="review">🔄 只溫習「需複習」的錯題</option>
              <option value="known">✅ 只溫習「已掌握」的單字</option>
            </select>
            <button 
              onClick={() => { setStudyIndex(0); setView('study'); }}
              className="w-full py-3 bg-indigo-600 text-white font-bold rounded-lg text-sm shadow hover:bg-indigo-700 transition cursor-pointer"
            >
              開始溫習
            </button>
          </div>
          
          <button onClick={() => setView('list')} className="w-full py-2.5 bg-white border border-gray-300 text-gray-700 font-bold rounded-xl text-xs hover:bg-gray-50 cursor-pointer">
            進入字庫管理 (點擊單字可看解釋與加備註)
          </button>
          
          {/* 點擊後開啟確認彈窗 */}
          <button onClick={() => setShowClearConfirmModal(true)} className="w-full mt-2 py-2 text-red-500 font-bold text-xs underline decoration-red-200 cursor-pointer hover:text-red-700 transition">
            清空所有記錄並重新上傳
          </button>
        </div>
      </div>
    );
  };

  const renderList = () => {
    const filteredList = words.filter(w => {
      const matchSearch = w.word.includes(searchQuery.toLowerCase()) || (w.note && w.note.toLowerCase().includes(searchQuery.toLowerCase()));
      let matchStatus = true;
      if (listFilter === 'has_note') matchStatus = w.note && w.note.trim() !== '';
      else if (listFilter !== 'all') matchStatus = w.status === listFilter;
      return matchSearch && matchStatus;
    });

    const sortedList = [...filteredList].sort((a, b) => {
      if (sortBy === 'alpha') {
        return a.word.localeCompare(b.word);
      }
      return 0;
    });

    const totalPages = Math.ceil(sortedList.length / LIST_PAGE_SIZE) || 1;
    const validPage = Math.min(listPage, totalPages);
    const paginatedList = sortedList.slice((validPage - 1) * LIST_PAGE_SIZE, validPage * LIST_PAGE_SIZE);

    return (
      <div className="flex flex-col w-full h-[85vh] py-4">
        <div className="flex justify-between items-center mb-3">
          <h2 className="text-lg font-bold text-gray-800">字庫管理 <span className="text-sm font-normal text-gray-500">({sortedList.length})</span></h2>
          <button onClick={() => setView('study')} className="px-3 py-1.5 bg-indigo-600 text-white font-bold rounded-lg text-xs shadow cursor-pointer">返回溫習</button>
        </div>

        <div className="bg-white p-3 rounded-xl shadow-sm border border-gray-100 mb-3 flex flex-col gap-2">
          <div className="flex gap-2">
            <input 
              type="text" 
              placeholder="搜尋單字/備註..." 
              value={searchQuery}
              onChange={(e) => { setSearchQuery(e.target.value); setListPage(1); }}
              className="flex-1 p-2 text-xs border border-gray-300 rounded-lg outline-none focus:border-indigo-500" 
            />
            <select 
              value={listFilter}
              onChange={(e) => { setListFilter(e.target.value); setListPage(1); }}
              className="p-2 text-xs border border-gray-300 rounded-lg outline-none bg-gray-50 cursor-pointer"
            >
              <option value="all">全部狀態</option>
              <option value="review">僅需複習</option>
              <option value="new">未學習</option>
              <option value="known">已掌握</option>
              <option value="has_note">📝 有備註</option>
            </select>
          </div>

          <div className="flex gap-2">
            <select 
              value={sortBy}
              onChange={(e) => setSortBy(e.target.value)}
              className="p-2 text-xs border border-gray-300 rounded-lg outline-none bg-indigo-50 font-semibold text-indigo-900 flex-1 cursor-pointer"
            >
              <option value="alpha">🔤 依照英文字母順序 (A-Z)</option>
              <option value="default">📄 依照匯入/加入順序</option>
            </select>
          </div>

          <div className="flex gap-2">
            <input 
              type="text" 
              value={customWordInput}
              onChange={(e) => setCustomWordInput(e.target.value)}
              placeholder="新增單字 (空白分隔)" 
              className="flex-1 p-2 text-xs border border-gray-300 rounded-lg outline-none focus:border-indigo-500" 
            />
            <button onClick={handleAddCustomWords} className="px-3 py-2 bg-gray-800 text-white font-bold rounded-lg text-xs hover:bg-gray-900 cursor-pointer">
              新增
            </button>
          </div>
        </div>

        <div className="flex-grow overflow-y-auto bg-white rounded-xl shadow-sm border border-gray-100 p-2 mb-3 no-scrollbar">
          {paginatedList.length === 0 ? (
            <div className="text-center py-10 text-xs text-gray-400">找不到單字。</div>
          ) : (
            <div className="divide-y divide-gray-100">
              {paginatedList.map(item => (
                <div key={item.word} className="py-2.5 px-3 flex items-center justify-between hover:bg-indigo-50 transition rounded-lg text-sm">
                  <div className="flex items-center space-x-2 w-[55%] overflow-hidden cursor-pointer" onClick={() => openWordDetail(item)}>
                    <button type="button" onClick={(e) => speakWord(item.word, e)} className="text-gray-400 hover:text-indigo-600 focus:outline-none shrink-0 cursor-pointer">
                      <Volume2 className="w-4 h-4" />
                    </button>
                    <span className="font-bold text-gray-900 truncate hover:text-indigo-600 transition" title={item.word}>{item.word}</span>
                    {item.note && <Edit3 className="w-3 h-3 text-amber-500 shrink-0" title="含備註" />}
                  </div>
                  
                  <div className="flex items-center space-x-2 shrink-0">
                    <select 
                      value={item.status}
                      onChange={(e) => changeWordStatus(item.word, e.target.value)}
                      className={`text-[10px] px-1.5 py-1 rounded-md font-semibold outline-none cursor-pointer border-0 ${
                        item.status === 'known' ? 'bg-green-100 text-green-700' : item.status === 'review' ? 'bg-orange-100 text-orange-700' : 'bg-gray-100 text-gray-600'
                      }`}
                    >
                      <option value="new">未學</option>
                      <option value="review">需複習</option>
                      <option value="known">已掌握</option>
                    </select>
                    <button 
                      type="button" 
                      onClick={(e) => triggerDelete(item.word, e)} 
                      className="p-1.5 text-gray-400 hover:text-red-500 hover:bg-red-50 rounded-md transition cursor-pointer"
                      title="刪除"
                    >
                      <Trash2 className="w-4 h-4 pointer-events-none" />
                    </button>
                  </div>
                </div>
              ))}
            </div>
          )}
        </div>

        <div className="bg-gray-50 px-3 py-2 rounded-xl flex items-center justify-between text-xs border border-gray-200">
          <div className="flex items-center space-x-2 text-gray-500">
            <span>頁數:</span>
            <input 
              type="number" 
              min="1" 
              max={totalPages} 
              value={validPage}
              onChange={(e) => {
                const val = parseInt(e.target.value);
                if (!isNaN(val)) {
                  setListPage(Math.max(1, Math.min(totalPages, val)));
                }
              }}
              className="w-12 text-center p-1 bg-white border border-gray-300 rounded font-bold text-indigo-600 outline-none"
            />
            <span>/ {totalPages}</span>
          </div>

          <div className="flex items-center space-x-1">
            <button onClick={() => setListPage(p => Math.max(1, p - 1))} disabled={validPage === 1} className="p-1 rounded bg-white border border-gray-300 disabled:opacity-40 cursor-pointer">
              <ChevronLeft className="w-4 h-4" />
            </button>
            <button onClick={() => setListPage(p => Math.min(totalPages, p + 1))} disabled={validPage === totalPages} className="p-1 rounded bg-white border border-gray-300 disabled:opacity-40 cursor-pointer">
              <ChevronRight className="w-4 h-4" />
            </button>
          </div>
        </div>
      </div>
    );
  };

  return (
    <div className="min-h-screen bg-slate-50 font-sans pb-16">
      <header className="bg-white shadow-xs sticky top-0 z-20">
        <div className="max-w-md mx-auto px-4 h-14 flex items-center justify-between">
          <div className="flex items-center text-indigo-600 cursor-pointer" onClick={() => words.length > 0 && setView('study')}>
            <Sparkles className="w-5 h-5 mr-1.5" />
            <h1 className="font-bold text-sm tracking-tight">DSE 5000+ 單字神器</h1>
          </div>
          {words.length > 0 && (
            <div className="flex gap-1.5">
              <button onClick={() => setView('list')} className="px-2.5 py-1.5 bg-indigo-50 text-indigo-700 font-semibold rounded-md text-xs hover:bg-indigo-100 flex items-center cursor-pointer">
                <List className="w-4 h-4 mr-1" /> 管理
              </button>
            </div>
          )}
        </div>
      </header>

      <main className="max-w-md mx-auto px-4">
        {view === 'import' && renderImport()}
        {view === 'study' && renderStudy()}
        {view === 'stats' && renderStats()}
        {view === 'list' && renderList()}
      </main>

      {/* 單字詳細資訊與備註編輯彈窗 */}
      {activeDetailWord && (
        <div className="fixed inset-0 bg-black/50 z-50 flex items-center justify-center p-4 backdrop-blur-xs">
          <div className="bg-white rounded-3xl p-6 w-full max-w-sm shadow-2xl flex flex-col max-h-[85vh]">
            <div className="flex justify-between items-center border-b pb-3 mb-3">
              <div className="flex items-center space-x-2">
                <h3 className="text-2xl font-black text-gray-900">{activeDetailWord.word}</h3>
                <button onClick={(e) => speakWord(activeDetailWord.word, e)} className="p-1.5 bg-indigo-50 text-indigo-600 rounded-full cursor-pointer hover:bg-indigo-100">
                  <Volume2 className="w-4 h-4" />
                </button>
              </div>
              <button onClick={() => setActiveDetailWord(null)} className="text-gray-400 hover:text-gray-600 p-1 cursor-pointer">
                <X className="w-5 h-5" />
              </button>
            </div>

            <div className="flex items-center justify-between bg-gray-50 p-2.5 rounded-xl mb-3 text-xs">
              <span className="font-bold text-gray-600">學習狀態：</span>
              <select 
                value={detailStatus}
                onChange={(e) => setDetailStatus(e.target.value)}
                className="rounded-lg bg-white border border-gray-200 text-xs font-bold p-1.5 outline-none cursor-pointer"
              >
                <option value="new">未學習</option>
                <option value="review">需複習</option>
                <option value="known">已掌握</option>
              </select>
            </div>

            <div className="flex-grow overflow-y-auto pr-1 text-xs space-y-2 mb-3 max-h-[180px] no-scrollbar">
              {isLoadingDetailDef ? (
                <div className="flex justify-center py-6"><Loader2 className="w-6 h-6 animate-spin text-indigo-500" /></div>
              ) : detailDef?.notFound ? (
                <div className="text-center py-6 text-gray-400 italic">自訂單字，無線上字典解釋。</div>
              ) : detailDef?.meanings ? (
                <div>
                  {detailDef.phonetic && <p className="text-indigo-600 font-mono bg-indigo-50 inline-block px-2 py-0.5 rounded font-bold mb-2">{detailDef.phonetic}</p>}
                  {detailDef.meanings.map((m, i) => (
                    <div key={i} className="mb-2">
                      <span className="inline-block px-1.5 py-0.5 bg-gray-800 text-white rounded text-[10px] font-bold mb-1">{m.partOfSpeech}</span>
                      <ul className="list-disc pl-4 space-y-1 text-gray-700">
                        {m.definitions.slice(0, 2).map((def, j) => (
                          <li key={j}>
                            <span className="font-medium">{def.definition}</span>
                            {def.example && <p className="italic text-gray-500 mt-0.5 text-[10px]">"{def.example}"</p>}
                          </li>
                        ))}
                      </ul>
                    </div>
                  ))}
                </div>
              ) : <div className="text-center py-6 text-red-400">載入解釋失敗</div>}
            </div>

            <div className="border-t pt-3 mb-4">
              <label className="text-[10px] font-bold text-gray-500 uppercase flex items-center mb-1">
                📝 個人備註
              </label>
              <textarea 
                value={detailNoteInput}
                onChange={(e) => setDetailNoteInput(e.target.value)}
                placeholder="輸入專屬筆記、例句或中文翻譯..." 
                className="w-full text-xs p-2.5 bg-amber-50/50 border border-amber-200 rounded-xl focus:outline-none focus:ring-2 focus:ring-amber-300 resize-none h-20 text-gray-800 placeholder-gray-400"
              />
            </div>

            <div className="flex gap-2">
              <button 
                onClick={() => setActiveDetailWord(null)}
                className="flex-1 py-2.5 bg-gray-100 text-gray-700 font-bold rounded-xl text-xs hover:bg-gray-200 transition cursor-pointer"
              >
                取消
              </button>
              <button 
                onClick={saveWordDetail}
                className="flex-1 py-2.5 bg-indigo-600 text-white font-bold rounded-xl text-xs shadow hover:bg-indigo-700 transition flex items-center justify-center cursor-pointer"
              >
                <Save className="w-4 h-4 mr-1" /> 確認儲存備註
              </button>
            </div>
          </div>
        </div>
      )}

      {/* 清空所有記錄的二次確認彈窗 */}
      {showClearConfirmModal && (
        <div className="fixed inset-0 bg-black/50 z-50 flex items-center justify-center p-4 backdrop-blur-xs">
          <div className="bg-white rounded-3xl p-6 w-full max-w-xs shadow-2xl flex flex-col animate-in fade-in zoom-in duration-150">
            <div className="w-12 h-12 bg-red-100 text-red-500 rounded-full flex items-center justify-center mx-auto mb-3">
              <AlertTriangle className="w-6 h-6" />
            </div>
            <h3 className="text-lg font-black text-gray-900 text-center mb-1">確定要清空所有記錄？</h3>
            <p className="text-xs text-gray-500 text-center mb-6">
              這將會永久刪除你所有的單字進度、自訂單字與個人備註，且無法復原！
            </p>
            <div className="flex gap-2">
              <button 
                onClick={() => setShowClearConfirmModal(false)}
                className="flex-1 py-2.5 bg-gray-100 text-gray-700 font-bold rounded-xl text-xs hover:bg-gray-200 transition cursor-pointer"
              >
                取消
              </button>
              <button 
                onClick={executeClearAll}
                className="flex-1 py-2.5 bg-red-500 text-white font-bold rounded-xl text-xs shadow hover:bg-red-600 transition cursor-pointer"
              >
                確定清空
              </button>
            </div>
          </div>
        </div>
      )}

      {/* 單字刪除確認彈窗 */}
      {wordToDelete && (
        <div className="fixed inset-0 bg-black/50 z-50 flex items-center justify-center p-4">
          <div className="bg-white rounded-2xl p-6 w-full max-w-xs shadow-xl transition-all">
            <h3 className="text-lg font-bold text-gray-900 mb-2">確認刪除</h3>
            <p className="text-sm text-gray-500 mb-6">
              確定要將 <span className="font-bold text-red-500">"{wordToDelete}"</span> 從字庫中永久刪除嗎？
            </p>
            <div className="flex justify-end gap-3">
              <button 
                onClick={() => setWordToDelete(null)}
                className="px-4 py-2 text-sm font-bold text-gray-600 bg-gray-100 rounded-lg hover:bg-gray-200 transition cursor-pointer"
              >
                取消
              </button>
              <button 
                onClick={confirmDelete}
                className="px-4 py-2 text-sm font-bold text-white bg-red-500 rounded-lg hover:bg-red-600 transition cursor-pointer"
              >
                確認刪除
              </button>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}