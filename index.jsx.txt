// —- asis 한정판매 라이브 버전2 
import React, { useState, useEffect, useRef } from 'react';
import { 
  Heart, Gift, Ticket, Share2, MoreVertical, X, Send, ShoppingBag, 
  RefreshCw, ChevronRight, Plus, Minus, Check, ShoppingCart, 
  AlertCircle, Store, Info, Share, Settings, RotateCw, Smile, 
  Sticker, Image as ImageIcon, Zap, Gamepad2, Info as InfoIcon, 
  FileText, ChevronLeft, ChevronDown, CreditCard, Wallet
} from 'lucide-react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInWithCustomToken, signInAnonymously, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, onSnapshot, updateDoc, arrayUnion } from 'firebase/firestore';

// --- Firebase 초기화 ---
const firebaseConfig = typeof __firebase_config !== 'undefined' 
  ? JSON.parse(__firebase_config) 
  : { apiKey: "", authDomain: "", projectId: "", storageBucket: "", messagingSenderId: "", appId: "" };

const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'live-commerce-proto';

// 공통 레이아웃 컴포넌트: 모바일 프레임
const MobileFrame = ({ title, children, onShare }) => (
  <div className="flex flex-col items-center gap-4 flex-shrink-0">
    <div className="flex items-center justify-between w-full max-w-[375px] px-2">
      <h2 className="text-white font-bold text-lg bg-gray-800 px-4 py-1 rounded-full">{title}</h2>
      <button 
        onClick={onShare}
        className="flex items-center gap-1.5 bg-indigo-600 hover:bg-indigo-500 text-white text-xs px-3 py-1.5 rounded-full font-bold transition-all shadow-lg active:scale-95"
      >
        <Share className="w-3.5 h-3.5" /> 공유하기
      </button>
    </div>
    <div className="relative w-[375px] h-[812px] bg-black overflow-hidden shadow-2xl rounded-[3rem] border-[8px] border-gray-800">
      {children}
      <div className="absolute bottom-1.5 left-1/2 -translate-x-1/2 w-32 h-1 bg-white/30 rounded-full z-[200]" />
    </div>
  </div>
);

const App = () => {
  // 초기 상품 데이터
  const initialProducts = [
    { id: 1, name: "아주 예쁜 가방", fullName: "[판매 1위] 아주 예쁜 가방 오늘 특가 (강력추천)", option: "컬러: 민트 / 사이즈: FREE", price: 23000, priceLabel: "23,000원", img: "https://images.unsplash.com/photo-1548036328-c9fa89d128fa?q=80&w=400", stats: { available: 50, completed: 45, reserved: 0 } },
    { id: 2, name: "독일 명품 땡처리", fullName: "독일 명품 땡처리 12피스 세트", option: "옵션: 12피스 / 구성품 포함", price: 23000, priceLabel: "23,000원", img: "https://images.unsplash.com/photo-1594223274512-ad4803739b7c?q=80&w=400", stats: { available: 50, completed: 45, reserved: 0 } },
    { id: 3, name: "배송비", fullName: "합배송 상품 (기본 배송비)", option: "기본 배송비 결제", price: 4000, priceLabel: "4,000원", img: "https://images.unsplash.com/photo-1586528116311-ad8dd3c8310d?q=80&w=400", stats: { available: 999, completed: 120, reserved: 0 } },
    { id: 4, name: "데일리 면 티셔츠", fullName: "[1+1] 데일리 순면 기본 티셔츠 화이트/블랙", option: "사이즈: FREE / 수량: 1+1", price: 15900, priceLabel: "15,900원", img: "https://images.unsplash.com/photo-1521572163474-6864f9cf17ab?q=80&w=400", stats: { available: 100, completed: 30, reserved: 0 } },
    { id: 5, name: "디자이너 스니커즈", fullName: "한정판 디자이너 런닝화 2026 Edition", option: "사이즈: 240mm / 컬러: 화이트", price: 89000, priceLabel: "89,000원", img: "https://images.unsplash.com/photo-1542291026-7eec264c27ff?q=80&w=400", stats: { available: 20, completed: 15, reserved: 0 } }
  ];

  // 상태 관리
  const [user, setUser] = useState(null);
  const [roomId, setRoomId] = useState(new URLSearchParams(window.location.search).get('roomId') || 'default-room');
  const [inventory, setInventory] = useState(initialProducts);
  const [upProductId, setUpProductId] = useState(null); 
  const [storeStatus, setStoreStatus] = useState(new Array(5).fill(false)); 
  const [productStatuses, setProductStatuses] = useState(new Array(5).fill('판매예정')); 
  const [comments, setComments] = useState([{ id: 'init', text: '반갑습니다! 라이브 방송 시작합니다.', type: 'system' }]);
  
  // UI 레이어 상태
  const [isStoreOpen, setIsStoreOpen] = useState(false);
  const [storeActiveTab, setStoreActiveTab] = useState('판매상품');
  const [isOptionOpen, setIsOptionOpen] = useState(false);
  const [isCheckoutOpen, setIsCheckoutOpen] = useState(false); 
  const [selectingProduct, setSelectingProduct] = useState(null); 
  const [selectQuantity, setSelectQuantity] = useState(1); 
  const [orderSource, setOrderSource] = useState(null); 
  const [orderedItems, setOrderedItems] = useState([]); 
  const [showToast, setShowToast] = useState(false); 
  const [alertMsg, setAlertMsg] = useState(null);
  const [shareSuccess, setShareSuccess] = useState(false);
  const [isSellerListOpen, setIsSellerListOpen] = useState(true);
  
  // 결제 수단 선택 상태
  const [paymentMethod, setPaymentMethod] = useState('card'); // 'card', 'naver', 'kakao', 'toss'
  
  // 셀러 설정 팝업 상태
  const [isSellerSettingOpen, setIsSellerSettingOpen] = useState(false);
  const [sellerTargetProduct, setSellerTargetProduct] = useState(null);
  const [sellerTargetIdx, setSellerTargetIdx] = useState(null);

  const [visibleBanners, setVisibleBanners] = useState(new Array(5).fill(false));

  const manualItems = [
    { label: "화면 설명", text: "한정판매 방송자/고객 화면 입니다." },
    { label: "한정판매 상품 up", text: "리스트의 UP 버튼 혹은 상품 카드를 클릭하여 팝업을 열어 설정하세요.\nUP 버튼을 통해 선착순 판매상품을 노출할 수 있습니다.\n판매예정인경우 가격없이 판매예정 상태로 고객에게 안내됩니다.\n판매시작을 누르면 가격노출과 함께 판매가 시작됩니다." },
    { label: "스토어 상품리스트 노출하기", text: "스토어 ON 버튼 클릭시 상품리스트에 상품이 노출됩니다." },
    { label: "판매가능 재고 확인", text: "주문버튼을 통해 점유된 재고는 점유수량에 카운트되며 판매가능수량에서 차감됩니다." },
    { label: "주문완료", text: "주문완료시 채팅창에 닉네임 + 한정특가판매 주문성공 노출됩니다." },
  ];

  // 1. Firebase 인증 프로세스 (Rule 3)
  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (err) {
        console.error("Auth Error:", err);
      }
    };
    initAuth();
    return onAuthStateChanged(auth, setUser);
  }, []);

  // 2. 실시간 데이터 동기화 (Rule 1 & 2)
  useEffect(() => {
    if (!user) return;
    const roomDocRef = doc(db, 'artifacts', appId, 'public', 'data', 'rooms', roomId);
    const unsubscribe = onSnapshot(roomDocRef, (snapshot) => {
      if (snapshot.exists()) {
        const data = snapshot.data();
        setInventory(data.inventory || initialProducts);
        setUpProductId(data.upProductId !== undefined ? data.upProductId : null);
        setStoreStatus(data.storeStatus || new Array(5).fill(false));
        setProductStatuses(data.productStatuses || new Array(5).fill('판매예정'));
        if (data.comments) setComments(data.comments);
      }
    }, (error) => console.error("Snapshot Error:", error));
    return () => unsubscribe();
  }, [user, roomId]);

  const syncToCloud = async (newData) => {
    if (!user) return;
    try {
      const roomDocRef = doc(db, 'artifacts', appId, 'public', 'data', 'rooms', roomId);
      await setDoc(roomDocRef, { ...newData, updatedAt: new Date().getTime() }, { merge: true });
    } catch (err) { console.error("Save Error:", err); }
  };

  const handleShare = async () => {
    if (!user) return;
    const newRoomId = roomId === 'default-room' ? Math.random().toString(36).substring(2, 8) : roomId;
    const shareUrl = `${window.location.origin}${window.location.pathname}?roomId=${newRoomId}`;
    try {
      await syncToCloud({ roomId: newRoomId, inventory, upProductId, storeStatus, productStatuses, comments });
      const textArea = document.createElement("textarea");
      textArea.value = shareUrl;
      document.body.appendChild(textArea);
      textArea.select();
      document.execCommand('copy');
      document.body.removeChild(textArea);
      if (roomId === 'default-room') {
        setRoomId(newRoomId);
        window.history.replaceState(null, '', shareUrl);
      }
      setShareSuccess(true);
      setTimeout(() => setShareSuccess(false), 3000);
    } catch (err) { setAlertMsg("공유 링크 복사에 실패했습니다."); }
  };

  const startOptionSelection = (product, source) => {
    setSelectingProduct(product);
    setSelectQuantity(1);
    setOrderSource(source);
    setIsOptionOpen(true);
  };

  const confirmOrder = async () => {
    if (!selectingProduct || !user) return;
    const newInventory = inventory.map(p => 
      p.id === selectingProduct.id 
        ? { ...p, stats: { ...p.stats, available: Math.max(0, p.stats.available - selectQuantity), reserved: p.stats.reserved + selectQuantity } } 
        : p
    );
    const newComment = { id: Date.now().toString(), user: '회원님', text: '한정 특가 판매 주문 성공!', type: 'order-success' };
    const updatedComments = [...comments, newComment].slice(-25);
    
    setOrderedItems(prev => {
      const exists = prev.find(item => item.id === selectingProduct.id);
      if (exists) return prev.map(item => item.id === selectingProduct.id ? { ...item, quantity: item.quantity + selectQuantity } : item);
      return [...prev, { ...selectingProduct, quantity: selectQuantity }];
    });
    setInventory(newInventory);
    setComments(updatedComments);
    await syncToCloud({ inventory: newInventory, comments: updatedComments });
    
    setIsOptionOpen(false);
    if (orderSource === 'limited') { setIsStoreOpen(false); }
    
    setShowToast(true);
    setTimeout(() => setShowToast(false), 5000);
  };

  const removeOrderItem = (id) => {
    const removedItem = orderedItems.find(item => item.id === id);
    if (!removedItem) return;
    setOrderedItems(prev => prev.filter(item => item.id !== id));
    const newInventory = inventory.map(p => 
      p.id === id ? { ...p, stats: { ...p.stats, available: p.stats.available + removedItem.quantity, reserved: Math.max(0, p.stats.reserved - removedItem.quantity) } } : p
    );
    setInventory(newInventory);
    syncToCloud({ inventory: newInventory });
  };

  const toggleBanner = (idx) => {
    const next = [...visibleBanners];
    next[idx] = !next[idx];
    setVisibleBanners(next);
  };

  // 셀러 설정 핸들러
  const openSellerSetting = (product, idx) => {
    setSellerTargetProduct(product);
    setSellerTargetIdx(idx);
    setIsSellerSettingOpen(true);
  };

  // 기존 수직 버튼용 핸들러 (이벤트 버블링 방지 포함)
  const handleUpToggleInList = (e, product) => {
    e.stopPropagation();
    const next = upProductId === product.id ? null : product.id;
    setUpProductId(next);
    syncToCloud({ upProductId: next });
  };

  const handleStoreToggleInList = (e, idx) => {
    e.stopPropagation();
    const next = [...storeStatus];
    next[idx] = !next[idx];
    setStoreStatus(next);
    syncToCloud({ storeStatus: next });
  };

  const currentUpProduct = upProductId !== null ? inventory.find(p => p.id === upProductId) : null;
  const currentStatus = upProductId !== null ? productStatuses[inventory.findIndex(p => p.id === upProductId)] : null;
  const onSaleProducts = inventory.filter((_, idx) => storeStatus[idx]);
  const totalOrderAmount = orderedItems.reduce((acc, item) => acc + (item.price * item.quantity), 0);
  const finalPayment = totalOrderAmount + (orderedItems.length > 0 ? 3000 : 0);

  // --- [방송화면] 셀러 송출 UI ---
  const SellerBroadcastUI = () => {
    const upProduct = upProductId !== null ? inventory.find(p => p.id === upProductId) : null;
    const chatRef = useRef(null);
    useEffect(() => { chatRef.current?.scrollIntoView({ behavior: 'smooth' }); }, [comments]);

    return (
      <div className="relative h-full w-full bg-black overflow-hidden text-white text-left">
        <img src="https://images.unsplash.com/photo-1490481651871-ab68de25d43d?q=80&w=600" className="absolute inset-0 w-full h-full object-cover opacity-70" alt="Broadcast" />
        <div className="absolute inset-0 bg-gradient-to-b from-black/40 via-transparent to-black/60 z-10" />
        <div className="relative z-20 flex flex-col p-4 pt-10">
          <div className="flex items-center justify-between">
             <div className="flex items-center gap-2">
               <div className="bg-red-500 text-white text-[10px] font-bold px-1.5 py-0.5 rounded tracking-tighter">LIVE</div>
               <div className="bg-black/30 backdrop-blur-md px-2 py-0.5 rounded text-[10px] font-medium">00:42:15</div>
               <div className="flex items-center gap-1 text-[10px] font-medium"><span className="opacity-80">👁️</span> 16,556</div>
             </div>
             <div className="flex gap-4"><Share2 size={20}/><Settings size={20}/><X size={20}/></div>
          </div>
        </div>
        
        <div className="absolute bottom-44 left-4 right-4 z-20 flex flex-col gap-1.5 max-h-[140px] overflow-y-auto scrollbar-hide text-left">
          {comments.map((msg) => (
            <p key={msg.id} className={`text-[12px] font-bold leading-tight drop-shadow-sm ${msg.type === 'order-success' ? 'text-pink-400' : 'text-white'}`}>
              {msg.user && <span className="mr-1 opacity-90">{msg.user}</span>}{msg.text}
            </p>
          ))}
          <div ref={chatRef} />
        </div>

        {upProduct && (
          <div className="absolute bottom-[176px] left-3 right-3 z-30 animate-in fade-in slide-in-from-bottom-2 duration-300">
             <div className="bg-white rounded-lg flex items-center p-0 overflow-hidden shadow-2xl border border-white/20">
                <div className="relative w-16 h-16 flex-shrink-0">
                   <img src={upProduct.img} className="w-full h-full object-cover" alt="up-p" />
                   <div className="absolute top-1 left-1 w-4 h-4 bg-indigo-600 rounded-sm flex items-center justify-center shadow-md"><Zap size={12} className="text-white fill-white" /></div>
                </div>
                <div className="flex-grow px-3 min-w-0">
                   <h4 className="text-[12px] font-bold text-gray-900 leading-tight truncate">{upProduct.name}</h4>
                   <div className="flex items-center gap-2 mt-0.5">
                      <span className="text-[#D14F5F] font-black text-[13px]">30%</span>
                      <span className="text-gray-900 font-black text-[13px]">{upProduct.price.toLocaleString()}원</span>
                   </div>
                </div>
                <div className="px-3 py-2 border-l bg-gray-50 flex flex-col items-end min-w-[70px]">
                   <div className="flex items-center gap-1"><span className="text-[#D14F5F] text-[9px] font-black uppercase">Ready</span><span className="text-[#D14F5F] text-[14px] font-black">{upProduct.stats.available}</span></div>
                   <div className="flex items-center gap-1 mt-0.5"><span className="text-gray-400 text-[9px] font-bold">STOCK</span><span className="text-gray-900 text-[11px] font-black">{upProduct.stats.available + upProduct.stats.reserved}</span></div>
                </div>
             </div>
          </div>
        )}

        <div className="absolute bottom-0 left-0 right-0 z-40 pb-6 bg-gradient-to-t from-black/80 to-transparent">
          <div className="px-4 flex items-end justify-between mb-4">
            <div onClick={() => setIsSellerListOpen(true)} className="flex flex-col items-center cursor-pointer active:scale-95 transition-transform">
              <div className="relative w-12 h-12 rounded-xl overflow-hidden border-2 border-red-500 mb-1">
                <img src="https://images.unsplash.com/photo-1543163521-1bf539c55dd2?q=80&w=100" className="w-full h-full object-cover" alt="item" />
                <div className="absolute top-0 right-0 bg-red-500 text-white text-[9px] px-1 rounded-bl-md font-bold">12</div>
              </div>
              <span className="text-[10px] font-bold shadow-sm">전체 상품</span>
            </div>
            <div className="flex-grow px-3"><div className="bg-white/20 backdrop-blur-md rounded-full px-4 py-2 text-[12px] border border-white/20 text-white/60 font-medium">메시지 입력</div></div>
            <div className="flex flex-col items-center gap-1"><Heart size={28} className="fill-white" /><span className="text-[10px] font-bold">0</span></div>
          </div>
          <div className="grid grid-cols-6 border-t border-white/10 pt-4 px-2 opacity-90">
            {[
              { icon: Zap, label: '플래시' },
              { icon: Gamepad2, label: '게임' },
              { icon: ImageIcon, label: '첨부' },
              { icon: RotateCw, label: '전환' },
              { icon: Smile, label: '효과' },
              { icon: Sticker, label: '스티커' },
            ].map((item, i) => (
              <div key={i} className="flex flex-col items-center gap-1 active:scale-90 transition-transform cursor-pointer">
                <item.icon size={20}/>
                <span className="text-[9px] font-bold tracking-tight">{item.label}</span>
              </div>
            ))}
          </div>
        </div>
      </div>
    );
  };

  // --- 메인 렌더링 ---
  return (
    <div className="min-h-screen bg-[#111111] p-8 flex flex-nowrap items-start gap-12 font-sans overflow-x-auto relative text-left">
      
      {shareSuccess && (
        <div className="fixed top-10 left-1/2 -translate-x-1/2 z-[1000] bg-indigo-600 text-white px-6 py-3 rounded-full font-bold shadow-2xl animate-in fade-in zoom-in flex items-center gap-2">
          <Check size={18} /> 공유 링크가 클립보드에 복사되었습니다!
        </div>
      )}

      {/* 1. 고객 뷰 */}
      <MobileFrame title="고객 뷰" onShare={handleShare}>
        <div className="relative h-full w-full overflow-hidden text-left">
          <img src="https://images.unsplash.com/photo-1490481651871-ab68de25d43d?q=80&w=600" className="absolute inset-0 w-full h-full object-cover opacity-80" alt="Live" />
          <div className="absolute inset-0 bg-gradient-to-b from-black/40 via-transparent to-black/80 z-10" />
          
          <div className="relative z-20 flex items-center justify-between p-4 pt-10">
            <div className="flex items-center gap-2 bg-black/30 backdrop-blur-md rounded-full pl-1 pr-3 py-1 text-[10px] text-white">
              <img src="https://api.dicebear.com/7.x/avataaars/svg?seed=host1" className="w-8 h-8 rounded-full border border-white/40 bg-pink-500" alt="host" />
              <div><div className="font-bold flex items-center gap-1">특가라이브💎</div><div className="opacity-80 font-medium">❤️ 5만 👁️ 723</div></div>
              <button className="bg-pink-600 px-2 py-0.5 rounded-full font-bold ml-1 active:scale-95 transition-transform">팔로우</button>
            </div>
            <div className="flex gap-2 text-white"><Share2 size={20} className="cursor-pointer active:scale-90" /><X size={20} className="cursor-pointer active:scale-90" /></div>
          </div>

          <div className="absolute bottom-40 left-4 right-4 z-20 flex flex-col gap-1.5 max-h-[140px] overflow-y-auto scrollbar-hide text-left">
            {comments.map((msg) => (
              <div key={msg.id} className="inline-flex items-start">
                <div className="bg-black/20 backdrop-blur-sm px-2.5 py-1 rounded-md text-[12px] font-bold text-white shadow-sm">
                  <span className="text-pink-300 mr-2 opacity-90">{msg.user || '그립봇'}</span>
                  <span className={msg.type === 'order-success' ? 'text-pink-400' : 'text-white'}>{msg.text}</span>
                </div>
              </div>
            ))}
          </div>

          {currentUpProduct && (
            <div className="absolute bottom-24 left-4 right-4 z-30 bg-white rounded-xl p-2 flex items-center gap-3 shadow-2xl animate-in fade-in slide-in-from-bottom-4 bounce-subtle-animation border border-white/20">
              <div className="absolute -top-3 -left-1 bg-indigo-600 text-white text-[8px] px-2 py-0.5 rounded-sm font-bold uppercase tracking-widest shadow-lg">Limited Item</div>
              <img src={currentUpProduct.img} className="w-12 h-12 rounded-lg object-cover" alt="item" />
              <div className="flex-grow min-w-0">
                <div className="text-[10px] font-bold text-gray-800 truncate">{currentUpProduct.name}</div>
                {currentStatus !== '판매예정' && <div className="text-xs font-black text-gray-900 mt-0.5">{currentUpProduct.priceLabel}</div>}
              </div>
              <button 
                onClick={() => currentStatus === '판매시작' && startOptionSelection(currentUpProduct, 'limited')} 
                className={`${currentStatus === '판매시작' ? 'bg-[#D14F5F] active:scale-95 shadow-lg shadow-pink-100' : 'bg-gray-400'} text-white px-4 py-2 rounded-lg text-xs font-black transition-all`}
              >
                {currentStatus === '판매시작' ? '주문' : currentStatus === '판매예정' ? '예정' : '품절'}
              </button>
            </div>
          )}

          {showToast && (
            <>
              <div className="absolute inset-0 bg-black/40 z-[200] animate-in fade-in duration-300" onClick={() => setShowToast(false)} />
              <div className="absolute bottom-24 left-3 right-3 z-[210] bg-white rounded-xl shadow-2xl p-2.5 flex items-center gap-3 border border-gray-100 animate-in slide-in-from-bottom-3">
                 <img src={selectingProduct?.img} className="w-11 h-11 rounded-lg object-cover" alt="toast" />
                 <div className="flex-grow text-[13px] font-black text-gray-900 truncate">주문서에 상품이 담겼습니다.</div>
                 <button onClick={() => { setIsStoreOpen(true); setStoreActiveTab('라이브 주문서'); setShowToast(false); }} className="text-pink-600 font-black text-[13px] whitespace-nowrap active:scale-95">바로가기 {'>'}</button>
              </div>
            </>
          )}

          <div className="absolute bottom-0 left-0 right-0 z-40 bg-gradient-to-t from-black/80 to-transparent p-4 pb-8 flex items-center gap-4">
            <div onClick={() => setIsStoreOpen(true)} className="relative cursor-pointer active:scale-95 group">
              <div className="w-10 h-10 rounded-lg overflow-hidden border border-white/40 bg-black/40 flex items-center justify-center text-white transition-all group-hover:bg-black/60">
                {onSaleProducts.length > 0 ? <img src={onSaleProducts[0].img} className="w-full h-full object-cover" alt="on-sale" /> : <Store size={20} />}
              </div>
              {onSaleProducts.length > 0 && <div className="absolute -top-1 -right-1 text-[9px] text-white w-4 h-4 rounded-full flex items-center justify-center font-black border border-white bg-pink-500 shadow-md animate-bounce-subtle">{onSaleProducts.length}</div>}
            </div>
            <div className="flex-grow"><input type="text" placeholder="메시지 입력" className="w-full bg-white/20 border border-white/20 rounded-full py-2 px-4 text-xs outline-none text-white placeholder:text-white/60 font-medium" /></div>
            <div className="flex gap-3 text-white">
              <div className="flex flex-col items-center gap-0.5 cursor-pointer active:scale-90"><Gift size={20}/><span className="text-[8px] font-bold">선물</span></div>
              <div className="flex flex-col items-center gap-0.5 cursor-pointer active:scale-90"><Heart size={20}/><span className="text-[8px] font-bold">좋아요</span></div>
            </div>
          </div>

          {/* 스토어 레이어 */}
          {isStoreOpen && (
            <div className="absolute inset-0 z-[100] flex flex-col justify-end text-left">
              <div className="absolute inset-0 bg-black/40" onClick={() => setIsStoreOpen(false)} />
              <div className="relative bg-white rounded-t-3xl h-[66%] flex flex-col animate-in slide-in-from-bottom duration-300">
                <div className="flex border-b h-14 items-center justify-around px-8 sticky top-0 bg-white rounded-t-3xl z-10">
                  <button onClick={() => setStoreActiveTab('판매상품')} className={`h-full text-sm font-black transition-colors ${storeActiveTab === '판매상품' ? 'text-pink-500 border-b-2 border-pink-500' : 'text-gray-400'}`}>판매상품</button>
                  <button onClick={() => setStoreActiveTab('라이브 주문서')} className={`h-full text-sm font-black transition-colors ${storeActiveTab === '라이브 주문서' ? 'text-pink-500 border-b-2 border-pink-500' : 'text-gray-400'}`}>라이브 주문서 ({orderedItems.length})</button>
                  <X className="absolute right-4 text-gray-400 cursor-pointer active:scale-90" onClick={() => setIsStoreOpen(false)} />
                </div>
                <div className="flex-grow overflow-y-auto p-4 scrollbar-hide">
                  {storeActiveTab === '판매상품' ? (
                    <div className="space-y-3">
                      {onSaleProducts.map(item => (
                        <div key={item.id} className="flex items-center gap-3 p-2 rounded-xl border border-gray-50 shadow-sm bg-white hover:bg-gray-50 transition-colors">
                          <img src={item.img} className="w-16 h-16 rounded-lg object-cover" alt="item" />
                          <div className="flex-grow min-w-0">
                            <h4 className="text-[12px] font-bold text-gray-800 truncate">{item.fullName}</h4>
                            <div className="flex items-center gap-1.5 mt-0.5"><span className="text-[13px] font-black text-gray-900">{item.priceLabel}</span><span className="bg-pink-100 text-pink-500 text-[9px] px-1 py-0.5 rounded font-black uppercase">LIVE</span></div>
                          </div>
                          <button onClick={() => startOptionSelection(item, 'store')} className="border border-pink-500 text-pink-500 px-3 py-1.5 rounded-full text-[11px] font-black hover:bg-pink-50 active:scale-95 transition-all">구매</button>
                        </div>
                      ))}
                    </div>
                  ) : (
                    <div className="h-full flex flex-col">
                      {orderedItems.length > 0 ? (
                        <>
                          <div className="flex-grow space-y-4">
                            {orderedItems.map(item => (
                              <div key={item.id} className="flex gap-3 border-b border-gray-50 pb-3 relative">
                                <img src={item.img} className="w-14 h-14 rounded-lg object-cover" alt="cart-item" />
                                <div className="flex-grow min-w-0">
                                  <div className="flex justify-between items-start"><h4 className="text-[12px] font-bold truncate pr-4 text-gray-900">{item.name}</h4><X size={14} className="text-gray-300 cursor-pointer hover:text-gray-500" onClick={() => removeOrderItem(item.id)} /></div>
                                  <div className="text-[10px] text-gray-400 font-medium">{item.option}</div>
                                  <div className="flex justify-between items-end mt-1">
                                    <span className="text-sm font-black text-gray-900">{item.price.toLocaleString()}원</span>
                                    <div className="flex items-center border border-gray-100 rounded-md px-1.5 py-0.5 gap-2 bg-gray-50">
                                      <Minus size={12} className="text-gray-400 cursor-pointer hover:text-gray-600" />
                                      <span className="text-xs font-black w-4 text-center text-gray-800">{item.quantity}</span>
                                      <Plus size={12} className="text-gray-400 cursor-pointer hover:text-gray-600" />
                                    </div>
                                  </div>
                                </div>
                              </div>
                            ))}
                          </div>
                          <div className="pt-4 border-t border-gray-100 space-y-3 pb-6">
                            <div className="flex justify-between text-xs text-gray-500 font-medium"><span>총 주문금액</span><span>{totalOrderAmount.toLocaleString()}원</span></div>
                            <div className="flex justify-between items-center"><span className="font-black text-sm text-gray-900">최종 결제 금액</span><span className="font-black text-pink-600 text-xl">{finalPayment.toLocaleString()}원</span></div>
                            <button onClick={() => setIsCheckoutOpen(true)} className="w-full bg-[#D14F5F] text-white font-black py-4 rounded-xl shadow-lg mt-2 active:scale-95 transition-all">결제하기</button>
                          </div>
                        </>
                      ) : <div className="h-full flex flex-col items-center justify-center text-gray-400 text-sm font-medium">주문서가 비어있습니다.</div>}
                    </div>
                  )}
                </div>
              </div>
            </div>
          )}

          {/* 옵션 선택 팝업 */}
          {isOptionOpen && selectingProduct && (
            <div className="absolute inset-0 z-[200] flex flex-col justify-end text-left">
              <div className="absolute inset-0 bg-black/60" onClick={() => setIsOptionOpen(false)} />
              <div className="relative bg-white rounded-t-3xl p-5 animate-in slide-in-from-bottom duration-300 shadow-2xl">
                <div className="flex justify-end mb-2"><X size={24} className="text-gray-400 cursor-pointer active:scale-90" onClick={() => setIsOptionOpen(false)} /></div>
                <div className="flex gap-4 mb-6">
                  <img src={selectingProduct.img} className="w-20 h-20 rounded-xl object-cover border border-gray-100 shadow-sm" alt="sel" />
                  <div className="flex flex-col justify-center min-w-0">
                    <h4 className="text-sm font-bold text-gray-800 leading-tight line-clamp-2">{selectingProduct.fullName}</h4>
                    <div className="text-xl font-black text-gray-900 mt-1">{selectingProduct.priceLabel}</div>
                  </div>
                </div>
                <div className="p-3 bg-gray-50 rounded-xl border border-gray-100 text-[12px] text-gray-500 mb-6 font-medium">{selectingProduct.option}</div>
                <div className="flex justify-between items-center mb-8">
                  <span className="font-black text-sm text-gray-900">수량 선택</span>
                  <div className="flex items-center gap-4 border border-gray-200 rounded-lg p-1.5 px-5 bg-white shadow-inner">
                    <Minus size={18} className="text-gray-400 cursor-pointer active:scale-75" onClick={() => setSelectQuantity(Math.max(1, selectQuantity - 1))} />
                    <span className="font-black text-lg w-6 text-center text-gray-900">{selectQuantity}</span>
                    <Plus size={18} className="text-gray-400 cursor-pointer active:scale-75" onClick={() => setSelectQuantity(selectQuantity + 1)} />
                  </div>
                </div>
                <button onClick={confirmOrder} className="w-full bg-[#D14F5F] text-white font-black py-4 rounded-2xl shadow-xl active:scale-95 transition-all mb-2">구매하기 ({(selectingProduct.price * selectQuantity).toLocaleString()}원)</button>
              </div>
            </div>
          )}

          {/* 결제 화면 */}
          {isCheckoutOpen && (
            <div className="absolute inset-0 bg-white z-[1000] flex flex-col animate-in slide-in-from-bottom duration-300 overflow-y-auto text-left">
               <header className="sticky top-0 bg-white border-b flex items-center justify-between px-4 h-14 z-20">
                  <ChevronLeft size={24} className="cursor-pointer active:scale-90" onClick={() => setIsCheckoutOpen(false)} />
                  <h1 className="text-lg font-black text-gray-900">주문</h1>
                  <X size={24} className="cursor-pointer active:scale-90" onClick={() => setIsCheckoutOpen(false)} />
               </header>
               <div className="flex-grow bg-gray-50 pb-24 space-y-3">
                  {/* 배송지 */}
                  <section className="bg-white p-4 flex justify-between items-center shadow-sm">
                     <div className="min-w-0">
                        <div className="flex items-center gap-1.5 mb-1.5">
                           <h2 className="text-sm font-black text-gray-900">배송지</h2>
                           <span className="text-[10px] bg-indigo-50 text-indigo-600 px-1.5 py-0.5 rounded font-bold">기본배송지</span>
                        </div>
                        <p className="text-xs text-gray-600 leading-relaxed font-medium truncate">홍길동 (010-1234-5678)<br/>서울특별시 강남구 테헤란로 123 (그립타워 11층)</p>
                     </div>
                     <ChevronRight size={18} className="text-gray-300 flex-shrink-0 ml-2" />
                  </section>

                  {/* 주문 상품 */}
                  <section className="bg-white p-4 shadow-sm">
                     <h2 className="text-sm font-black text-gray-900 mb-4">주문 상품 {orderedItems.length}개</h2>
                     <div className="space-y-4">
                        {orderedItems.map(item => (
                           <div key={item.id} className="flex gap-3">
                              <img src={item.img} className="w-16 h-16 rounded-lg object-cover border border-gray-100 shadow-sm" alt="checkout-p" />
                              <div className="flex-grow min-w-0">
                                 <h3 className="text-xs font-bold leading-snug line-clamp-2 text-gray-800">{item.fullName}</h3>
                                 <p className="text-[10px] text-gray-400 mt-0.5 font-medium">{item.option}</p>
                                 <div className="flex justify-between items-end mt-1.5"><span className="text-sm font-black text-gray-900">{item.price.toLocaleString()}원</span><span className="text-[11px] text-gray-500 font-bold">{item.quantity}개</span></div>
                              </div>
                           </div>
                        ))}
                     </div>
                  </section>

                  {/* 결제 수단 추가 섹션 */}
                  <section className="bg-white p-4 shadow-sm">
                     <h2 className="text-sm font-black text-gray-900 mb-4">결제 수단</h2>
                     
                     {/* 간편 결제 */}
                     <div className="mb-4">
                        <p className="text-[11px] font-bold text-gray-400 mb-2 uppercase tracking-tight">간편 결제</p>
                        <div className="grid grid-cols-3 gap-2">
                           {[
                              { id: 'naver', label: 'N Pay', color: 'border-emerald-500 text-emerald-600' },
                              { id: 'kakao', label: 'Kakao Pay', color: 'border-yellow-400 text-gray-800' },
                              { id: 'toss', label: 'Toss Pay', color: 'border-blue-500 text-blue-600' },
                           ].map(method => (
                              <button 
                                 key={method.id}
                                 onClick={() => setPaymentMethod(method.id)}
                                 className={`py-3 rounded-xl border text-xs font-black transition-all ${paymentMethod === method.id ? `${method.color} ring-2 ring-opacity-20` : 'border-gray-100 bg-gray-50 text-gray-400'}`}
                              >
                                 {method.label}
                              </button>
                           ))}
                        </div>
                     </div>

                     {/* 일반 결제 */}
                     <div>
                        <p className="text-[11px] font-bold text-gray-400 mb-2 uppercase tracking-tight">일반 결제</p>
                        <button 
                           onClick={() => setPaymentMethod('card')}
                           className={`w-full py-3.5 rounded-xl border flex items-center justify-center gap-2 text-sm font-black transition-all ${paymentMethod === 'card' ? 'border-gray-900 text-gray-900 bg-gray-50' : 'border-gray-100 bg-gray-50 text-gray-400'}`}
                        >
                           <CreditCard size={18} /> 카드 결제
                        </button>
                     </div>
                  </section>

                  {/* 결제 금액 */}
                  <section className="bg-white p-4 shadow-sm space-y-2 pb-8">
                     <div className="flex justify-between text-xs text-gray-500 font-medium"><span>상품 총 금액</span><span>{totalOrderAmount.toLocaleString()}원</span></div>
                     <div className="flex justify-between text-xs text-gray-500 font-medium"><span>배송비</span><span>3,000원</span></div>
                     <div className="flex justify-between items-center pt-3 border-t font-black"><span className="text-sm text-gray-900">최종 결제 예정 금액</span><span className="text-pink-600 text-xl">{finalPayment.toLocaleString()}원</span></div>
                  </section>
               </div>
               <div className="absolute bottom-0 inset-x-0 p-4 bg-white border-t z-20 shadow-[0_-4px_10px_rgba(0,0,0,0.03)]">
                  <button className="w-full py-4 bg-[#D14F5F] text-white font-black text-lg rounded-xl shadow-lg active:scale-95 transition-all">
                     {finalPayment.toLocaleString()}원 결제하기
                  </button>
               </div>
            </div>
          )}
        </div>
      </MobileFrame>

      {/* 2. 셀러 뷰 */}
      <MobileFrame title="셀러 뷰" onShare={handleShare}>
        <div className="relative h-full w-full bg-white flex flex-col overflow-hidden text-left">
          {isSellerListOpen ? (
            <>
              <div className="p-4 border-b flex items-center justify-between sticky top-0 bg-white z-10 shadow-sm">
                <RefreshCw size={20} className="text-gray-400 cursor-pointer hover:rotate-180 transition-transform duration-500" />
                <h3 className="font-black text-gray-900 uppercase tracking-tight">Product Setting</h3>
                <X size={24} className="text-gray-400 cursor-pointer active:scale-90" onClick={() => setIsSellerListOpen(false)} />
              </div>
              <div className="flex-grow overflow-y-auto pb-20 scrollbar-hide">
                {inventory.map((product, idx) => (
                  <div 
                    key={product.id} 
                    onClick={() => openSellerSetting(product, idx)}
                    className={`p-4 border-b transition-colors cursor-pointer active:bg-gray-50 ${upProductId === product.id ? 'bg-pink-50/40' : 'bg-white'}`}
                  >
                    <div className="flex gap-3">
                      <img src={product.img} className="w-20 h-20 rounded-lg object-cover bg-gray-100 shadow-inner" alt="thumb" />
                      <div className="flex-grow min-w-0 flex flex-col justify-between py-0.5">
                        <div className="text-[13px] font-bold text-gray-900 leading-snug line-clamp-2">{product.name}</div>
                        <div className="text-sm font-black text-gray-900 tracking-tight">{product.priceLabel}</div>
                      </div>
                      
                      {/* 수직 정렬 버튼 - 의도하신 레이아웃 유지 */}
                      <div className="flex flex-col gap-1.5 w-[70px]">
                        <button 
                          onClick={(e) => handleUpToggleInList(e, product)}
                          className={`w-full py-1.5 text-[11px] font-black rounded-md transition-all ${upProductId === product.id ? 'bg-pink-600 text-white shadow-md' : 'bg-gray-400 text-white'}`}
                        >
                          UP
                        </button>
                        <button 
                          onClick={(e) => handleStoreToggleInList(e, idx)}
                          className={`w-full py-1.5 text-[10px] font-black rounded-md transition-all ${storeStatus[idx] ? 'bg-emerald-500 text-white shadow-md' : 'bg-gray-300 text-white'}`}
                        >
                          {storeStatus[idx] ? '스토어ON' : '스토어OFF'}
                        </button>
                      </div>
                    </div>
                    
                    <div className="mt-4 flex justify-between items-center">
                      <div className="flex gap-1.5">
                        {['판매예정', '판매시작', '품절'].map(status => (
                          <button 
                            key={status} 
                            onClick={(e) => {
                              e.stopPropagation();
                              const next = [...productStatuses]; next[idx] = status; 
                              setProductStatuses(next); syncToCloud({ productStatuses: next });
                            }}
                            className={`px-2 py-1 text-[10px] font-black rounded border transition-colors ${productStatuses[idx] === status ? 'bg-pink-50 text-pink-600 border-pink-200 shadow-sm' : 'bg-white text-gray-400 border-gray-100'}`}
                          >
                            {status}
                          </button>
                        ))}
                      </div>
                      <div className="text-right flex gap-3 text-[11px] font-bold text-gray-800">
                         <div>
                           <div className="text-[9px] text-gray-400 text-right uppercase">Ready</div>
                           <div className="font-black text-gray-800">{product.stats.available}</div>
                         </div>
                         <div>
                           <div className="text-[9px] text-gray-400 text-right uppercase">Reserved</div>
                           <div className="font-black text-indigo-600">{product.stats.reserved}</div>
                         </div>
                      </div>
                    </div>
                  </div>
                ))}
              </div>

              {/* 셀러 한정판매 상세 설정 팝업 레이어 */}
              {isSellerSettingOpen && sellerTargetProduct && (
                <div className="absolute inset-0 z-[100] flex flex-col justify-end text-left">
                  <div className="absolute inset-0 bg-black/40 animate-in fade-in" onClick={() => setIsSellerSettingOpen(false)} />
                  <div className="relative bg-white rounded-t-3xl p-5 animate-in slide-in-from-bottom duration-300 border-t border-gray-100 shadow-2xl">
                    <div className="flex justify-between items-center mb-4">
                      <h3 className="font-black text-gray-900 tracking-tight">상품 상세 설정</h3>
                      <X size={24} className="text-gray-400 cursor-pointer active:scale-90" onClick={() => setIsSellerSettingOpen(false)} />
                    </div>
                    
                    <div className="flex gap-4 mb-6 p-3 bg-gray-50 rounded-2xl border border-gray-100">
                      <img src={sellerTargetProduct.img} className="w-16 h-16 rounded-xl object-cover border border-white shadow-sm" alt="target" />
                      <div className="flex flex-col justify-center min-w-0">
                        <div className="text-xs font-bold text-gray-800 truncate mb-1">{sellerTargetProduct.fullName}</div>
                        <div className="text-sm font-black text-indigo-600">{sellerTargetProduct.priceLabel}</div>
                      </div>
                    </div>

                    <div className="space-y-6">
                      <div className="flex gap-3">
                        <div className="flex-1 space-y-2">
                           <p className="text-[11px] font-black text-gray-400 pl-1 uppercase tracking-wider">Flash Display</p>
                           <button 
                            onClick={() => {
                              const next = upProductId === sellerTargetProduct.id ? null : sellerTargetProduct.id;
                              setUpProductId(next); syncToCloud({ upProductId: next });
                            }}
                            className={`w-full py-4 text-[13px] font-black rounded-xl transition-all flex items-center justify-center gap-2 ${upProductId === sellerTargetProduct.id ? 'bg-pink-600 text-white shadow-lg shadow-pink-100' : 'bg-gray-100 text-gray-500'}`}
                           >
                            <Zap size={16} fill={upProductId === sellerTargetProduct.id ? "white" : "none"} />
                            한정판매 UP {upProductId === sellerTargetProduct.id ? '해제' : '설정'}
                           </button>
                        </div>
                        <div className="flex-1 space-y-2">
                           <p className="text-[11px] font-black text-gray-400 pl-1 uppercase tracking-wider">Store Active</p>
                           <button 
                            onClick={() => {
                              const next = [...storeStatus];
                              next[sellerTargetIdx] = !next[sellerTargetIdx];
                              setStoreStatus(next); syncToCloud({ storeStatus: next });
                            }}
                            className={`w-full py-4 text-[13px] font-black rounded-xl transition-all flex items-center justify-center gap-2 ${storeStatus[sellerTargetIdx] ? 'bg-emerald-500 text-white shadow-lg shadow-emerald-100' : 'bg-gray-100 text-gray-500'}`}
                           >
                            <Store size={16} />
                            스토어 {storeStatus[sellerTargetIdx] ? 'OFF' : 'ON'}
                           </button>
                        </div>
                      </div>

                      <div className="space-y-2">
                        <p className="text-[11px] font-black text-gray-400 pl-1 uppercase tracking-wider">Status Control</p>
                        <div className="grid grid-cols-3 gap-2">
                          {['판매예정', '판매시작', '품절'].map(status => (
                            <button 
                              key={status} 
                              onClick={() => {
                                const next = [...productStatuses];
                                next[sellerTargetIdx] = status;
                                setProductStatuses(next); syncToCloud({ productStatuses: next });
                              }}
                              className={`py-3.5 text-[12px] font-black rounded-xl border transition-all ${productStatuses[sellerTargetIdx] === status ? 'bg-indigo-600 text-white border-indigo-600 shadow-lg shadow-indigo-100' : 'bg-white text-gray-400 border-gray-200'}`}
                            >
                              {status}
                            </button>
                          ))}
                        </div>
                      </div>

                      <div className="pt-2 pb-4">
                        <button 
                          onClick={() => setIsSellerSettingOpen(false)}
                          className="w-full bg-gray-900 text-white font-black py-4 rounded-2xl shadow-xl active:scale-95 transition-all"
                        >
                          설정 완료
                        </button>
                      </div>
                    </div>
                  </div>
                </div>
              )}
            </>
          ) : (
            <SellerBroadcastUI />
          )}
        </div>
      </MobileFrame>

      {/* 3. 메뉴얼 */}
      <div className="flex flex-col items-center gap-4 flex-shrink-0 self-start sticky top-8 z-[100] text-left">
        <h2 className="text-white font-black text-lg bg-indigo-600 px-8 py-1.5 rounded-full shadow-2xl tracking-tight">SYSTEM MANUAL</h2>
        <div className="flex flex-col gap-3 py-4 p-2 bg-white/5 rounded-3xl border border-white/10 shadow-2xl backdrop-blur-md">
          {manualItems.map((item, idx) => (
            <div key={idx} className="relative group flex justify-end">
              {visibleBanners[idx] && (
                <div 
                  className="fixed bg-indigo-700 text-white px-8 py-6 rounded-3xl shadow-[0_30px_60px_rgba(0,0,0,0.5)] animate-in slide-in-from-right-10 duration-300 z-[2000] flex flex-col gap-3 border border-white/20 backdrop-blur-xl"
                  style={{ top: '20%', right: '420px', width: '450px' }}
                >
                  <h4 className="text-lg font-black border-b border-white/20 pb-2 flex items-center gap-2"><InfoIcon size={20} /> {item.label}</h4>
                  <p className="text-[15px] whitespace-pre-wrap leading-relaxed opacity-95 font-medium">{item.text}</p>
                  <button onClick={() => toggleBanner(idx)} className="absolute top-4 right-4 text-white/50 hover:text-white transition-colors"><X size={20}/></button>
                  <div className="absolute right-[-8px] top-10 w-4 h-4 bg-indigo-700 rotate-45 border-r border-t border-white/20"></div>
                </div>
              )}
              <button
                onClick={() => toggleBanner(idx)}
                className={`flex items-center gap-3 px-5 py-4 rounded-2xl font-black text-[12px] transition-all shadow-xl active:scale-95 border ${
                  visibleBanners[idx] ? 'bg-indigo-600 text-white border-white/20 scale-105' : 'bg-white text-gray-900 border-transparent hover:bg-gray-100'
                }`}
              >
                <FileText size={16} /> {item.label}
              </button>
            </div>
          ))}
        </div>
      </div>

      <style>{`
        @keyframes bounce-subtle { 0%, 100% { transform: translateY(0); } 50% { transform: translateY(-4px); } }
        .bounce-subtle-animation { animation: bounce-subtle 3s infinite ease-in-out; }
        .line-clamp-2 { display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden; }
        .scrollbar-hide::-webkit-scrollbar { display: none; }
        .scrollbar-hide { -ms-overflow-style: none; scrollbar-width: none; }
      `}</style>
    </div>
  );
};

export default App;