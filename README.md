# smartervitabot
smarter vita bot
/** 
 * VitaDashboard.jsx
 * -----------------
 * Container/View architecture with Context for state management.
 *
 * 🎨 **Fiverr Designer Handoff Guide**
 *
 * 1. Design Tokens & Tailwind
 *    - Use provided `tailwind.config.js` for colors, spacing, typography, and breakpoints.
 *    - Reference design tokens: primary (charcoal), secondary (sand), accent (eucalyptus), cta, warning, danger.
 *    - Consistent padding/margin (e.g. p-4/m-4), grid gaps (gap-6).
 *
 * 2. Component Library & Props
 *    - Create Storybook stories for each presentational component:
 *      • `<ChatBox chatHistory={[{type:'user',text:'Hi'}]} />`
 *      • `<ScanUploader onUpload={fn} />`
 *      • `<PDFParser userId="uid" />`
 *      • `<GoalWizard goals={...} onChange={fn} />`
 *      • `<SupplementList suggestions={[{name:'Omega-3',id:'1'}]} reorderMap={{}} />`
 *      • `<DailyHabits habits={['Hydrate']} userId="uid" />`
 *      • `<MindfulnessBox />`, `<ProgressCharts userId="uid" />`
 *      • `<VoiceInput onResult={fn} />`, `<ResetButton onReset={fn} />`
 *    - Each must accept a `className` prop and apply it to the root element.
 *
 * 3. Responsive Layout
 *    - Mobile: single-column (`grid-cols-1`).
 *    - Desktop: two-column (`md:grid-cols-2`).
 *    - Header: flex with wrap under small widths.
 *
 * 4. Animations & Interactions
 *    - Framer Motion for section entrances:  
 *      `initial={{opacity:0,y:20}} animate={{opacity:1,y:0}} transition={{duration:0.6}}`
 *    - Button hover/active: subtle scale or color change.
 *
 * 5. Accessibility & ARIA
 *    - All interactive elements need `aria-label` or visible labels.
 *    - ChatBox updates wrapped in `<div aria-live="polite">…</div>`.
 *    - Maintain visible focus outlines (avoid removing `outline` without replacement).
 *
 * 6. Figma Deliverables
 *    - Annotated screens (mobile & desktop).
 *    - Component variants (default/hover/active/disabled).
 *    - Style guide page: color swatches, typography scale, spacing, icon library.
 *
 * 7. Testing & QA
 *    - Unit tests for hooks (`usePdfScanData`, `useDynamicGoals`, etc.).
 *    - E2E smoke tests for upload→parse→chat flows.
 *
 * 8. README & Repo
 *    - Setup instructions (`npm install`, `.env` variables, `npm start`).
 *    - API contract examples (mock JSON for /parse-scan, /vita-chat, /vita-coaching, /vita-forecast).
 */

import React, { createContext, useContext, useReducer, useEffect, useCallback, Suspense, lazy } from 'react';
import { useTranslation } from 'react-i18next';
import { motion } from 'framer-motion';
import html2pdf from 'html2pdf.js';
import { getAuth, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, getDoc, collection, query, where, getDocs } from 'firebase/firestore';

// Presentational components
import ChatBox from './components/ChatBox';
import ScanUploader from './components/ScanUploader';
import SupplementList from './components/SupplementList';
import AutoshipGate from './components/AutoshipGate';
import DailyHabits from './components/DailyHabits';
import MindfulnessBox from './components/MindfulnessBox';
import ProgressCharts from './components/ProgressCharts';
import VoiceInput from './components/VoiceInput';
import ResetButton from './components/ResetButton';

// Lazy-loaded components
const MealPlanner  = lazy(() => import('./components/MealPlanner'));
const WellnessTip  = lazy(() => import('./components/WellnessTip'));
const CoachingPlan = lazy(() => import('./components/CoachingPlan'));
const ForecastChart= lazy(() => import('./components/ForecastChart'));
const PDFParser    = lazy(() => import('./components/PDFParser'));
const GoalWizard   = lazy(() => import('./components/GoalWizard'));

// Firebase instances
const auth = getAuth();
const db   = getFirestore();

// Hook: Parse uploaded PDF scan into structured data
function usePdfScanData(userId) {
  const [scanData, setScanData] = React.useState(null);
  useEffect(() => {
    if (!userId) return;
    async function handleScan(file) {
      const resp = await fetch(`${process.env.REACT_APP_API_BASE_URL}/parse-scan`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ fileName: file.name })
      });
      const parsed = await resp.json();
      setScanData(parsed);
    }
    window.addEventListener('scanParsed', e => handleScan(e.detail));
    return () => window.removeEventListener('scanParsed', handleScan);
  }, [userId]);
  return scanData;
}

// Hook: Generate dynamic weekly goals based on scan and coaching plan
function useDynamicGoals(scanData, coachingPlan) {
  const [goals, setGoals] = React.useState(null);
  useEffect(() => {
    if (!scanData || !coachingPlan) return;
    const newGoals = {
      bodyFat:    (scanData.bodyFatPercent || 0) - (coachingPlan.weeklyFatLossPercent || 1),
      muscleMass: (scanData.muscleMassKg    || 0) + (coachingPlan.weeklyMuscleGainKg || 0.5),
      hydration:  scanData.hydrationLiters || 2
    };
    setGoals(newGoals);
  }, [scanData, coachingPlan]);
  return goals;
}

// Global state context
const initialState = { chatInput:'', chatHistory:[], reorderMap:{}, isLoading:false };
function reducer(state, action) {
  switch (action.type) {
    case 'SET_INPUT':      return { ...state, chatInput: action.payload };
    case 'ADD_MESSAGE':    return { ...state, chatHistory: [...state.chatHistory, action.payload] };
    case 'SET_LOADING':    return { ...state, isLoading: action.payload };
    case 'UPDATE_REORDER': return { ...state, reorderMap: {...state.reorderMap, ...action.payload} };
    case 'RESET_CHAT':     return {...initialState};
    default: return state;
  }
}
const VitaContext = createContext();
export function useVita() { return useContext(VitaContext); }

function VitaDashboardContainer() {
  const [state, dispatch] = useReducer(reducer, initialState);
  const { t, i18n }     = useTranslation();
  const [user, setUser] = React.useState(null);
  const [loadingProfile, setLoadingProfile] = React.useState(true);
  const [hasShippingOrder, setHasShippingOrder] = React.useState(false);
  const [motivationalQuote, setMotivationalQuote] = React.useState('');
  const [supplements, setSupplements] = React.useState([]);
  const [coachingPlan, setCoachingPlan]   = React.useState(null);
  const [forecastData, setForecastData]   = React.useState(null);

  // PDF scan & dynamic goals
  const scanData    = usePdfScanData(user?.uid);
  const dynamicGoals= useDynamicGoals(scanData, coachingPlan);

  // Auth & profile
  useEffect(() => {
    const unsub = onAuthStateChanged(auth, async u => {
      if (u) {
        const snap = await getDoc(doc(db, 'userProfiles', u.uid));
        const data = snap.exists() ? snap.data() : {};
        setUser({ uid:u.uid, name:data.name, autoshipActive:data.autoshipActive });
      } else setUser({ autoshipActive:false });
      setLoadingProfile(false);
    });
    return unsub;
  }, []);

  // Orders gate
  useEffect(() => {
    if (!user?.uid) return;
    (async () => {
      const now = new Date();
      const snaps = await getDocs(
        query(
          collection(db, 'orders'),
          where('userId','==',user.uid),
          where('status','==','shipping'),
          where('shipDate','>=', now)
        )
      );
      setHasShippingOrder(!snaps.empty);
    })();
  }, [user]);

  // Motivation fetch
  useEffect(() => {
    if (!user) return;
    (async () => {
      try {
        const res = await fetch(`${process.env.REACT_APP_API_BASE_URL}/vita-motivation`,{
          method:'POST', headers:{'Content-Type':'application/json'},
          body:JSON.stringify({prompt:'Short wellness motivational quote.'})
        });
        const { quote } = await res.json();
        setMotivationalQuote(quote);
      } catch {
        setMotivationalQuote(t('defaultQuote'));
      }
    })();
  }, [user, t]);

  // Supplements load
  useEffect(() => {
    if (!user?.uid) return;
    (async () => {
      const snaps = await getDocs(collection(db,'supplements'));
      setSupplements(snaps.docs.map(d=>({id:d.id,...d.data()})));
    })();
  }, [user]);

  // Adaptive coaching
  useEffect(() => {
    if (!user?.uid || !state.chatHistory.length) return;
    (async () => {
      const res = await fetch(`${process.env.REACT_APP_API_BASE_URL}/vita-coaching`,{
        method:'POST', headers:{'Content-Type':'application/json'},
        body:JSON.stringify({userId:user.uid, chatHistory: state.chatHistory, supplements})
      });
      const { plan } = await res.json();
      setCoachingPlan(plan);
    })();
  }, [user, state.chatHistory, supplements]);

  // Predictive forecast
  useEffect(() => {
    if (!user?.uid) return;
    (async () => {
      const res = await fetch(`${process.env.REACT_APP_API_BASE_URL}/vita-forecast`,{
        method:'POST', headers:{'Content-Type':'application/json'},
        body:JSON.stringify({userId:user.uid})
      });
      const { forecast } = await res.json();
      setForecastData(forecast);
    })();
  }, [user]);

  // PDF export
  const exportPdf = useCallback((id, filename) =>
    setTimeout(()=>{
      const el = document.getElementById(id);
      if(el) html2pdf().from(el).save(filename);
    },0),
  []);

  // Handlers
  const handleLanguage    = lng => i18n.changeLanguage(lng);
  const handleChatSubmit  = async () => {
    if (!state.chatInput.trim()) return;
    dispatch({type:'SET_LOADING', payload:true});
    dispatch({type:'ADD_MESSAGE', payload:{type:'user', text:state.chatInput}});
    try {
      const res = await fetch(`${process.env.REACT_APP_API_BASE_URL}/vita-chat`,{
        method:'POST', headers:{'Content-Type':'application/json'},
        body:JSON.stringify({
          model: 'gpt-4',
          systemPrompt: t('systemPrompt'),
          symptoms: state.chatInput,
          supplements
        })
      });
      const { message } = await res.json();
      dispatch({type:'ADD_MESSAGE', payload:{type:'vita', text:message}});
      // reorder logic
      const lower = message.toLowerCase();
      const updates = {};
      supplements.forEach(s => {
        if (lower.includes(s.name.toLowerCase())) {
          updates[s.name] = (state.reorderMap[s.name]||0) + 1;
        }
      });
      dispatch({type:'UPDATE_REORDER', payload:updates});
    } catch {
      dispatch({type:'ADD_MESSAGE', payload:{type:'vita', text:t('aiUnavailable')}});
    }
    dispatch({type:'SET_LOADING', payload:false});
    dispatch({type:'SET_INPUT', payload:''});
  };

  return (
    <VitaContext.Provider
      value={{
        state, dispatch, user, loadingProfile, hasShippingOrder,
        motivationalQuote, supplements, coachingPlan, forecastData,
        scanData, dynamicGoals, exportPdf, handleLanguage, handleChatSubmit, t
      }}
    >
      <VitaDashboardView />
    </VitaContext.Provider>
  );
}

function VitaDashboardView() {
  const {
    state, dispatch, user, loadingProfile, hasShippingOrder,
    motivationalQuote, supplements, coachingPlan, forecastData,
    scanData, dynamicGoals, exportPdf, handleLanguage, handleChatSubmit, t
  } = useVita();

  if (loadingProfile || user == null) return <div>{t('loadingProfile')}</div>;
  if (!user.autoshipActive || !hasShippingOrder) return <AutoshipGate />;

  return (
    <div className="min-h-screen bg-sand text-charcoal font-sans p-6">
      {/* HEADER */}
      <header className="flex flex-col sm:flex-row justify-between items-start sm:items-center border-b border-eucalyptus pb-4 mb-6 gap-4 sm:gap-0">
        <div>
          <motion.h1
            initial={{opacity:0,y:-20}}
            animate={{opacity:1,y:0}}
            transition={{duration:0.6}}
            className="text-h1 font-serif text-charcoal"
          >
            ✨ {t('dashboardTitle')}
          </motion.h1>
          <p className="mt-2 text-base font-medium">{t('greeting',{name:user.name})}</p>
          <p className="text-sm text-charcoal/70">{motivationalQuote}</p>
        </div>
        <div className="flex gap-4 items-center">
          <select
            value={i18n.language}
            onChange={e=>handleLanguage(e.target.value)}
            aria-label={t('selectLanguage')}
            className="border border-eucalyptus p-2 rounded focus:ring-2 focus:ring-eucalyptus"
          >
            <option value="en">English</option>
            <option value="fr">Français</option>
            <option value="es">Español</option>
          </select>
          <button
            onClick={()=>exportPdf('vita-summary','VITA_Summary.pdf')}
            className="bg-cta hover:bg-cta/90 text-white px-4 py-2 rounded-lg font-medium text-base transition"
          >
            {t('exportSummary')}
          </button>
        </div>
      </header>

      {/* MAIN GRID */}
      <div id="vita-summary" className="grid grid-cols-1 md:grid-cols-2 gap-6">
        {/* 1. PDF Scan Parser */}
        <Suspense fallback={<div>{t('loadingScan')}</div>}>
          <PDFParser userId={user.uid} />
        </Suspense>

        <ScanUploader />
        <DailyHabits userId={user.uid} />

        {/* 2. Dynamic Goals Wizard */}
        <Suspense fallback={<div>{t('loadingGoals')}</div>}>
          <GoalWizard goals={dynamicGoals} onChange={g=>{/*...*/}} />
        </Suspense>

        {/* Chat & Supplements */}
        <section className="bg-white p-6 rounded-2xl shadow-lg">
          <h2 className="text-h2 font-bold mb-4 text-eucalyptus">{t('manualInput')}</h2>
          <textarea
            className="w-full border border-charcoal/20 p-3 rounded text-base focus:outline-none focus:ring-2 focus:ring-eucalyptus"
            rows={4}
            placeholder={t('manualPlaceholder')}
            value={state.chatInput}
            onChange={e=>dispatch({type:'SET_INPUT',payload:e.target.value})}
          />
          <VoiceInput onResult={text=>dispatch({type:'SET_INPUT',payload:state.chatInput+' '+text})} />
          <button
            onClick={handleChatSubmit}
            disabled={state.isLoading}
            className="mt-4 bg-cta hover:bg-cta/90 text-white px-6 py-2 rounded-lg w-full transition"
          >
            {state.isLoading ? t('thinking') : t('submitToVita')}
          </button>
        </section>

        <ChatBox chatHistory={state.chatHistory} />
        <SupplementList suggestions={supplements} reorderMap={state.reorderMap} />

        <Suspense fallback={<div>{t('loadingPlanner')}</div>}>
          <MealPlanner userId={user.uid} />
        </Suspense>
        <Suspense fallback={<div>{t('loadingTips')}</div>}>
          <WellnessTip />
        </Suspense>

        <MindfulnessBox />
        <ProgressCharts userId={user.uid} />

        <Suspense fallback={null}>
          <CoachingPlan plan={coachingPlan} />
        </Suspense>
        <Suspense fallback={null}>
          <ForecastChart data={forecastData} />
        </Suspense>

        <ResetButton onReset={()=>dispatch({type:'RESET_CHAT'})} />
      </div>
    </div>
  );
}

// Top-level export
export default function VitaDashboard() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <VitaDashboardContainer />
    </Suspense>
  );
}
import React, { useReducer, useEffect, useCallback, Suspense, lazy, createContext, useContext } from 'react';
import { BrowserRouter as Router, Routes, Route, NavLink } from 'react-router-dom';
import { QueryClient, QueryClientProvider, useQuery } from 'react-query';
import { useTranslation } from 'react-i18next';
import { motion } from 'framer-motion';
import html2pdf from 'html2pdf.js';
import { getAuth, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, getDoc, collection, query, where, getDocs } from 'firebase/firestore';

// React Query client
const queryClient = new QueryClient();

// Lazy-loaded pages
const DashboardHome = lazy(() => import('./pages/DashboardHome'));
const ScanPage      = lazy(() => import('./pages/ScanPage'));
const ChatPage      = lazy(() => import('./pages/ChatPage'));
const MealPlanPage  = lazy(() => import('./pages/MealPlanPage'));
const AnalyticsPage = lazy(() => import('./pages/AnalyticsPage'));
const SettingsPage  = lazy(() => import('./pages/SettingsPage'));
const AutoshipGate  = lazy(() => import('./components/AutoshipGate'));

const auth = getAuth();
const db   = getFirestore();

// ... (Query fetchers and context omitted for brevity)
const VitaContext = createContext();
export function useVita() { return useContext(VitaContext); }

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Router>
        <MainLayout />
      </Router>
    </QueryClientProvider>
  );
}

function MainLayout() {
  const [user, setUser] = React.useState(null);
  const [loadingAuth, setLoadingAuth] = React.useState(true);

  useEffect(() => {
    return onAuthStateChanged(auth, u => {
      setUser(u);
      setLoadingAuth(false);
    });
  }, []);

  if (loadingAuth) return <div>Loading...</div>;
  if (!user) return <LoginPage />; // assume you have a login page

  return (
    <div className="min-h-screen flex bg-sand">
      <nav className="w-64 bg-white border-r p-6">
        <h2 className="text-xl font-bold mb-6 text-eucalyptus">VITA</h2>
        {['Home','Scan','Chat','Meals','Analytics','Settings'].map((p) => (
          <NavLink
            key={p}
            to={`/${p.toLowerCase()}`}
            className={({isActive}) => `block py-2 px-4 rounded ${isActive?'bg-eucalyptus text-white':'text-charcoal'}`}
          >
            {p}
          </NavLink>
        ))}
      </nav>
      <main className="flex-1 p-6 overflow-auto">
        <Suspense fallback={<div>Loading page...</div>}>
          <Routes>
            <Route path="/home"      element={<DashboardHome />} />
            <Route path="/scan"      element={<ScanPage />} />
            <Route path="/chat"      element={<ChatPage />} />
            <Route path="/meals"     element={<MealPlanPage />} />
            <Route path="/analytics" element={<AnalyticsPage />} />
            <Route path="/settings"  element={<SettingsPage />} />
            <Route path="/*"          element={<DashboardHome />} />
          </Routes>
        </Suspense>
      </main>
    </div>
  );
}
