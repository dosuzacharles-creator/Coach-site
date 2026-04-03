import { useState, useEffect, useRef } from "react";

// ── CONFIG ─────────────────────────────────────────────────────────────────
const ANTHROPIC_MODEL = "claude-sonnet-4-20250514";

const SYSTEM_PROMPT = `Tu es APEX, un coach personnel d'élite spécialisé en transformation physique, discipline mentale et nutrition. Tu parles en français.

Ton style : direct, motivant, sérieux mais encourageant. Pas de blabla inutile. Comme un coach sportif de haut niveau.

Tu analyses les données de l'utilisateur (routines, sport, alimentation, hydratation, streak) et tu donnes des conseils personnalisés.

Exemples de ton style :
- "Lève-toi, aujourd'hui tu progresses."
- "Bon choix, continue comme ça."
- "Tu veux changer ton physique ? Alors reste constant."
- "Attention, tu es en train de dériver."

Pour le scanner alimentaire, quand l'utilisateur décrit un repas, réponds avec :
FORMAT JSON STRICT :
{"verdict": "OK" ou "EVITER", "score": 1-10, "explication": "...", "alternative": "..."}

Pour le reste, réponds en texte direct, max 3 phrases percutantes.`;

const FOOD_PROMPT = `Tu es un nutritionniste expert. L'utilisateur te décrit ou montre un repas.
Réponds UNIQUEMENT en JSON valide, sans markdown, sans backticks :
{"verdict":"OK","score":7,"explication":"Bon apport protéique mais manque de légumes.","alternative":"Ajoute une salade verte ou des légumes vapeur."}
Le verdict est "OK" si score >= 6, sinon "EVITER".`;

// ── DATA ────────────────────────────────────────────────────────────────────
const MORNING_TASKS = [
  { id: "wake", icon: "☀️", label: "Réveil à l'heure" },
  { id: "water_am", icon: "💧", label: "Grand verre d'eau" },
  { id: "shower_am", icon: "🚿", label: "Douche froide" },
  { id: "skincare", icon: "🧴", label: "Skincare" },
  { id: "teeth", icon: "🦷", label: "Brossage dents" },
  { id: "outfit", icon: "👕", label: "Tenue propre / soignée" },
  { id: "posture", icon: "🧘", label: "Posture 2 min" },
];
const EVENING_TASKS = [
  { id: "shower_pm", icon: "🚿", label: "Douche soir" },
  { id: "skincare_pm", icon: "🧴", label: "Skincare soir" },
  { id: "prep_tomorrow", icon: "📋", label: "Préparer demain" },
  { id: "no_screen", icon: "📵", label: "Pas d'écran 30 min" },
  { id: "sleep", icon: "😴", label: "Couché avant 23h" },
];
const WORKOUT_A = [
  { id: "pushups", icon: "💪", label: "Pompes", sets: "4×15", rest: 60 },
  { id: "squats", icon: "🦵", label: "Squats", sets: "4×20", rest: 60 },
  { id: "plank", icon: "🏋️", label: "Gainage", sets: "3×45s", rest: 45 },
  { id: "abs", icon: "🔥", label: "Abdos", sets: "3×20", rest: 60 },
];
const WORKOUT_B = [
  { id: "run", icon: "🏃", label: "Course / Marche rapide", sets: "30 min", rest: 0 },
  { id: "jumps", icon: "⚡", label: "Sauts burpees", sets: "3×10", rest: 90 },
  { id: "lunges", icon: "🦵", label: "Fentes alternées", sets: "3×12", rest: 60 },
  { id: "stretch", icon: "🧘", label: "Étirements", sets: "10 min", rest: 0 },
];
const LEVELS = [
  { name: "Novice", min: 0, color: "#64748b" },
  { name: "Débutant", min: 100, color: "#3b82f6" },
  { name: "Intermédiaire", min: 300, color: "#8b5cf6" },
  { name: "Avancé", min: 600, color: "#f59e0b" },
  { name: "Élite", min: 1000, color: "#ef4444" },
  { name: "APEX", min: 2000, color: "#10b981" },
];

// ── HELPERS ─────────────────────────────────────────────────────────────────
function getLevel(xp) {
  let lv = LEVELS[0];
  for (const l of LEVELS) if (xp >= l.min) lv = l;
  return lv;
}
function nextLevel(xp) {
  for (let i = LEVELS.length - 1; i >= 0; i--) {
    if (xp >= LEVELS[i].min) {
      return LEVELS[i + 1] || LEVELS[LEVELS.length - 1];
    }
  }
  return LEVELS[1];
}
async function callAI(messages, system = SYSTEM_PROMPT) {
  const res = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      model: ANTHROPIC_MODEL,
      max_tokens: 1000,
      system,
      messages,
    }),
  });
  const data = await res.json();
  return data.content?.[0]?.text || "";
}

// ── ICONS ────────────────────────────────────────────────────────────────────
const Icon = ({ name, size = 20, color = "currentColor" }) => {
  const icons = {
    home: <svg width={size} height={size} viewBox="0 0 24 24" fill="none" stroke={color} strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M3 9l9-7 9 7v11a2 2 0 01-2 2H5a2 2 0 01-2-2z"/><polyline points="9 22 9 12 15 12 15 22"/></svg>,
    list: <svg width={size} height={size} viewBox="0 0 24 24" fill="none" stroke={color} strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><line x1="8" y1="6" x2="21" y2="6"/><line x1="8" y1="12" x2="21" y2="12"/><line x1="8" y1="18" x2="21" y2="18"/><line x1="3" y1="6" x2="3.01" y2="6"/><line x1="3" y1="12" x2="3.01" y2="12"/><line x1="3" y1="18" x2="3.01" y2="18"/></svg>,
    camera: <svg width={size} height={size} viewBox="0 0 24 24" fill="none" stroke={color} strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M23 19a2 2 0 01-2 2H3a2 2 0 01-2-2V8a2 2 0 012-2h4l2-3h6l2 3h4a2 2 0 012 2z"/><circle cx="12" cy="13" r="4"/></svg>,
    dumbbell: <svg width={size} height={size} viewBox="0 0 24 24" fill="none" stroke={color} strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M6.5 6.5h11"/><path d="M6.5 17.5h11"/><line x1="12" y1="6.5" x2="12" y2="17.5"/><rect x="2" y="5" width="3" height="3" rx="1"/><rect x="2" y="16" width="3" height="3" rx="1"/><rect x="19" y="5" width="3" height="3" rx="1"/><rect x="19" y="16" width="3" height="3" rx="1"/></svg>,
    droplet: <svg width={size} height={size} viewBox="0 0 24 24" fill="none" stroke={color} strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M12 2.69l5.66 5.66a8 8 0 11-11.31 0z"/></svg>,
    bar: <svg width={size} height={size} viewBox="0 0 24 24" fill="none" stroke={color} strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><line x1="18" y1="20" x2="18" y2="10"/><line x1="12" y1="20" x2="12" y2="4"/><line x1="6" y1="20" x2="6" y2="14"/></svg>,
    bot: <svg width={size} height={size} viewBox="0 0 24 24" fill="none" stroke={color} strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><rect x="3" y="11" width="18" height="10" rx="2"/><circle cx="12" cy="5" r="2"/><path d="M12 7v4"/><line x1="8" y1="16" x2="8" y2="16"/><line x1="16" y1="16" x2="16" y2="16"/></svg>,
    settings: <svg width={size} height={size} viewBox="0 0 24 24" fill="none" stroke={color} strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><circle cx="12" cy="12" r="3"/><path d="M19.4 15a1.65 1.65 0 00.33 1.82l.06.06a2 2 0 010 2.83 2 2 0 01-2.83 0l-.06-.06a1.65 1.65 0 00-1.82-.33 1.65 1.65 0 00-1 1.51V21a2 2 0 01-4 0v-.09A1.65 1.65 0 009 19.4a1.65 1.65 0 00-1.82.33l-.06.06a2 2 0 01-2.83-2.83l.06-.06A1.65 1.65 0 004.68 15a1.65 1.65 0 00-1.51-1H3a2 2 0 010-4h.09A1.65 1.65 0 004.6 9a1.65 1.65 0 00-.33-1.82l-.06-.06a2 2 0 012.83-2.83l.06.06A1.65 1.65 0 009 4.68a1.65 1.65 0 001-1.51V3a2 2 0 014 0v.09a1.65 1.65 0 001 1.51 1.65 1.65 0 001.82-.33l.06-.06a2 2 0 012.83 2.83l-.06.06A1.65 1.65 0 0019.4 9a1.65 1.65 0 001.51 1H21a2 2 0 010 4h-.09a1.65 1.65 0 00-1.51 1z"/></svg>,
    send: <svg width={size} height={size} viewBox="0 0 24 24" fill="none" stroke={color} strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><line x1="22" y1="2" x2="11" y2="13"/><polygon points="22 2 15 22 11 13 2 9 22 2"/></svg>,
    check: <svg width={size} height={size} viewBox="0 0 24 24" fill="none" stroke={color} strokeWidth="2.5" strokeLinecap="round" strokeLinejoin="round"><polyline points="20 6 9 17 4 12"/></svg>,
    flame: <svg width={size} height={size} viewBox="0 0 24 24" fill="none" stroke={color} strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M8.5 14.5A2.5 2.5 0 0011 12c0-1.38-.5-2-1-3-1.072-2.143-.224-4.054 2-6 .5 2.5 2 4.9 4 6.5 2 1.6 3 3.5 3 5.5a7 7 0 01-7 7 7 7 0 01-7-7c0-1.153.433-2.294 1-3a2.5 2.5 0 002.5 2.5z"/></svg>,
    star: <svg width={size} height={size} viewBox="0 0 24 24" fill="none" stroke={color} strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><polygon points="12 2 15.09 8.26 22 9.27 17 14.14 18.18 21.02 12 17.77 5.82 21.02 7 14.14 2 9.27 8.91 8.26 12 2"/></svg>,
    zap: <svg width={size} height={size} viewBox="0 0 24 24" fill="none" stroke={color} strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><polygon points="13 2 3 14 12 14 11 22 21 10 12 10 13 2"/></svg>,
    plus: <svg width={size} height={size} viewBox="0 0 24 24" fill="none" stroke={color} strokeWidth="2.5" strokeLinecap="round"><line x1="12" y1="5" x2="12" y2="19"/><line x1="5" y1="12" x2="19" y2="12"/></svg>,
  };
  return icons[name] || null;
};

// ── MAIN APP ─────────────────────────────────────────────────────────────────
export default function App() {
  // ── STATE
  const [tab, setTab] = useState("home");
  const [darkMode, setDarkMode] = useState(true);
  const today = new Date().toDateString();

  // Persistent state (simulated localStorage via useState + effect)
  const [xp, setXp] = useState(150);
  const [streak, setStreak] = useState(7);
  const [water, setWater] = useState(3);
  const [checkedMorning, setCheckedMorning] = useState({});
  const [checkedEvening, setCheckedEvening] = useState({});
  const [workoutDay, setWorkoutDay] = useState("A");
  const [completedExercises, setCompletedExercises] = useState({});
  const [workoutDone, setWorkoutDone] = useState(false);
  const [coachMessages, setCoachMessages] = useState([
    { role: "assistant", text: "Salut champion. Je suis APEX, ton coach personnel. Prêt à transformer ton physique en 30 jours ? Dis-moi comment tu te sens aujourd'hui." },
  ]);
  const [chatInput, setChatInput] = useState("");
  const [chatLoading, setChatLoading] = useState(false);
  const [foodDesc, setFoodDesc] = useState("");
  const [foodResult, setFoodResult] = useState(null);
  const [foodLoading, setFoodLoading] = useState(false);
  const [coachMsg, setCoachMsg] = useState("Lève-toi, aujourd'hui tu progresses. Chaque action compte.");
  const [timerActive, setTimerActive] = useState(null);
  const [timerCount, setTimerCount] = useState(0);
  const [settings, setSettings] = useState({ wakeTime: "07:00", goal: "Prise de masse", difficulty: "Intermédiaire", notifs: true });

  // Stats
  const morningPct = Math.round((Object.values(checkedMorning).filter(Boolean).length / MORNING_TASKS.length) * 100);
  const eveningPct = Math.round((Object.values(checkedEvening).filter(Boolean).length / EVENING_TASKS.length) * 100);
  const routinePct = Math.round((morningPct + eveningPct) / 2);
  const waterPct = Math.min(100, Math.round((water / 8) * 100));
  const workoutExs = workoutDay === "A" ? WORKOUT_A : WORKOUT_B;
  const workoutPct = Math.round((Object.values(completedExercises).filter(Boolean).length / workoutExs.length) * 100);

  const lv = getLevel(xp);
  const nxt = nextLevel(xp);
  const xpPct = Math.min(100, Math.round(((xp - lv.min) / (nxt.min - lv.min)) * 100));

  // Timer
  useEffect(() => {
    let interval;
    if (timerActive !== null && timerCount > 0) {
      interval = setInterval(() => setTimerCount(c => {
        if (c <= 1) { setTimerActive(null); return 0; }
        return c - 1;
      }), 1000);
    }
    return () => clearInterval(interval);
  }, [timerActive, timerCount]);

  // ── STYLES
  const C = {
    bg: darkMode ? "#080c14" : "#f0f4f8",
    card: darkMode ? "#0f1624" : "#ffffff",
    card2: darkMode ? "#141d2e" : "#f8fafc",
    border: darkMode ? "#1e2d45" : "#e2e8f0",
    text: darkMode ? "#e8edf5" : "#1a202c",
    muted: darkMode ? "#4a6080" : "#718096",
    accent: "#3b82f6",
    green: "#10b981",
    red: "#ef4444",
    amber: "#f59e0b",
    purple: "#8b5cf6",
  };

  const styles = {
    app: { fontFamily: "'DM Sans', 'Inter', sans-serif", background: C.bg, minHeight: "100vh", maxWidth: 430, margin: "0 auto", position: "relative", overflow: "hidden", color: C.text },
    screen: { paddingBottom: 80, minHeight: "calc(100vh - 80px)", overflowY: "auto" },
    navbar: { position: "fixed", bottom: 0, left: "50%", transform: "translateX(-50%)", width: "100%", maxWidth: 430, background: darkMode ? "rgba(8,12,20,0.95)" : "rgba(255,255,255,0.95)", backdropFilter: "blur(20px)", borderTop: `1px solid ${C.border}`, display: "flex", zIndex: 100 },
    navBtn: (active) => ({ flex: 1, padding: "12px 4px 10px", display: "flex", flexDirection: "column", alignItems: "center", gap: 3, cursor: "pointer", background: "none", border: "none", color: active ? C.accent : C.muted, transition: "all 0.2s", fontSize: 9, fontWeight: active ? 700 : 500, letterSpacing: "0.05em", textTransform: "uppercase" }),
    card: { background: C.card, borderRadius: 20, padding: "18px 20px", marginBottom: 12, border: `1px solid ${C.border}` },
    card2: { background: C.card2, borderRadius: 16, padding: "14px 16px", marginBottom: 10, border: `1px solid ${C.border}` },
    header: { padding: "56px 20px 16px", },
    h1: { fontSize: 26, fontWeight: 800, letterSpacing: "-0.5px", margin: 0, lineHeight: 1.2 },
    h2: { fontSize: 20, fontWeight: 700, margin: 0 },
    h3: { fontSize: 15, fontWeight: 600, margin: 0 },
    label: { fontSize: 11, fontWeight: 700, letterSpacing: "0.1em", textTransform: "uppercase", color: C.muted },
    muted: { fontSize: 13, color: C.muted },
    btn: (color = C.accent, full = false) => ({ background: color, color: "#fff", border: "none", borderRadius: 14, padding: "14px 20px", fontSize: 15, fontWeight: 700, cursor: "pointer", width: full ? "100%" : "auto", letterSpacing: "-0.2px", transition: "all 0.15s", display: "flex", alignItems: "center", justifyContent: "center", gap: 8 }),
    btnOutline: { background: "transparent", color: C.accent, border: `1.5px solid ${C.accent}`, borderRadius: 14, padding: "12px 20px", fontSize: 14, fontWeight: 600, cursor: "pointer", letterSpacing: "-0.2px" },
    progressBar: (pct, color = C.accent, h = 8) => ({
      outer: { background: C.border, borderRadius: 999, height: h, overflow: "hidden", width: "100%" },
      inner: { background: color, width: `${pct}%`, height: "100%", borderRadius: 999, transition: "width 0.5s ease" },
    }),
    check: (done) => ({ width: 26, height: 26, borderRadius: 8, border: `2px solid ${done ? C.green : C.border}`, background: done ? C.green : "transparent", display: "flex", alignItems: "center", justifyContent: "center", cursor: "pointer", transition: "all 0.2s", flexShrink: 0 }),
    badge: (color) => ({ background: color + "22", color, borderRadius: 8, padding: "3px 10px", fontSize: 12, fontWeight: 700 }),
    input: { background: C.card2, border: `1px solid ${C.border}`, borderRadius: 14, padding: "12px 16px", fontSize: 14, color: C.text, width: "100%", outline: "none", resize: "none" },
    row: { display: "flex", alignItems: "center", gap: 10 },
    between: { display: "flex", alignItems: "center", justifyContent: "space-between" },
  };

  // ── FUNCTIONS
  const addWater = () => {
    if (water < 12) { setWater(w => w + 1); setXp(x => x + 5); }
  };
  const toggleMorning = (id) => {
    const val = !checkedMorning[id];
    setCheckedMorning(p => ({ ...p, [id]: val }));
    if (val) setXp(x => x + 10);
  };
  const toggleEvening = (id) => {
    const val = !checkedEvening[id];
    setCheckedEvening(p => ({ ...p, [id]: val }));
    if (val) setXp(x => x + 10);
  };
  const toggleExercise = (id) => {
    const val = !completedExercises[id];
    setCompletedExercises(p => ({ ...p, [id]: val }));
    if (val) setXp(x => x + 20);
  };

  const sendChat = async () => {
    if (!chatInput.trim() || chatLoading) return;
    const userMsg = chatInput.trim();
    setChatInput("");
    const newMsgs = [...coachMessages, { role: "user", text: userMsg }];
    setCoachMessages(newMsgs);
    setChatLoading(true);
    try {
      const contextMsg = `[Contexte utilisateur: Streak ${streak} jours | XP ${xp} | Eau ${water}/8 verres | Routine matin ${morningPct}% | Sport ${workoutPct}% | Niveau: ${lv.name}]\n\nMessage: ${userMsg}`;
      const apiMsgs = [{ role: "user", content: contextMsg }];
      const reply = await callAI(apiMsgs);
      setCoachMessages(p => [...p, { role: "assistant", text: reply }]);
    } catch {
      setCoachMessages(p => [...p, { role: "assistant", text: "Connexion IA momentanément indisponible. Reste focus !" }]);
    }
    setChatLoading(false);
  };

  const scanFood = async () => {
    if (!foodDesc.trim() || foodLoading) return;
    setFoodLoading(true);
    setFoodResult(null);
    try {
      const res = await callAI([{ role: "user", content: `Repas : ${foodDesc}` }], FOOD_PROMPT);
      const clean = res.replace(/```json|```/g, "").trim();
      const parsed = JSON.parse(clean);
      setFoodResult(parsed);
      setXp(x => x + 15);
    } catch {
      setFoodResult({ verdict: "OK", score: 5, explication: "Analyse IA indisponible. Mange équilibré : protéines + légumes + féculents.", alternative: "Blanc de poulet + riz + brocolis" });
    }
    setFoodLoading(false);
  };

  const startTimer = (sec, id) => {
    setTimerActive(id);
    setTimerCount(sec);
  };

  // ── SCREENS
  const screens = {
    home: <HomeScreen />,
    routine: <RoutineScreen />,
    food: <FoodScreen />,
    sport: <SportScreen />,
    water: <WaterScreen />,
    stats: <StatsScreen />,
    coach: <CoachScreen />,
    settings: <SettingsScreen />,
  };

  function HomeScreen() {
    return (
      <div style={styles.screen}>
        <div style={{ ...styles.header, paddingBottom: 20 }}>
          <div style={styles.between}>
            <div>
              <p style={{ ...styles.label, marginBottom: 6 }}>{new Date().toLocaleDateString("fr-FR", { weekday: "long", day: "numeric", month: "long" })}</p>
              <h1 style={styles.h1}>Bonjour, <span style={{ color: C.accent }}>Champion</span> 👊</h1>
            </div>
            <div style={{ textAlign: "center" }}>
              <div style={{ fontSize: 28, fontWeight: 900, color: C.amber, lineHeight: 1 }}>{streak}</div>
              <div style={{ fontSize: 10, color: C.muted, fontWeight: 700, letterSpacing: "0.1em" }}>STREAK</div>
            </div>
          </div>
        </div>

        <div style={{ padding: "0 16px" }}>
          {/* XP Card */}
          <div style={{ ...styles.card, background: `linear-gradient(135deg, ${lv.color}18, ${C.card})` }}>
            <div style={styles.between}>
              <div>
                <p style={styles.label}>Niveau</p>
                <div style={{ ...styles.row, marginTop: 4 }}>
                  <span style={{ fontSize: 22, fontWeight: 900, color: lv.color }}>{lv.name}</span>
                  <span style={styles.badge(lv.color)}>{xp} XP</span>
                </div>
              </div>
              <div style={{ fontSize: 36 }}><Icon name="zap" size={36} color={lv.color} /></div>
            </div>
            <div style={{ marginTop: 12 }}>
              <div style={styles.between}>
                <span style={styles.muted}>Vers {nxt.name}</span>
                <span style={{ fontSize: 12, color: lv.color, fontWeight: 700 }}>{xpPct}%</span>
              </div>
              <div style={{ ...styles.progressBar(xpPct, lv.color).outer, marginTop: 6 }}>
                <div style={styles.progressBar(xpPct, lv.color).inner} />
              </div>
            </div>
          </div>

          {/* Coach Message */}
          <div style={{ ...styles.card, background: `linear-gradient(135deg, ${C.accent}18, ${C.card})`, borderColor: C.accent + "40" }}>
            <div style={{ ...styles.row, marginBottom: 10 }}>
              <div style={{ background: C.accent + "20", borderRadius: 10, padding: 8 }}>
                <Icon name="bot" size={18} color={C.accent} />
              </div>
              <div>
                <p style={{ ...styles.label, color: C.accent }}>APEX Coach</p>
                <p style={{ margin: 0, fontSize: 13, color: C.muted }}>Message du jour</p>
              </div>
            </div>
            <p style={{ margin: 0, fontSize: 15, lineHeight: 1.6, fontStyle: "italic" }}>"{coachMsg}"</p>
          </div>

          {/* Progress Grid */}
          <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: 10, marginBottom: 12 }}>
            {[
              { label: "Routine", pct: routinePct, icon: "list", color: C.purple, tab: "routine" },
              { label: "Sport", pct: workoutPct, icon: "dumbbell", color: C.green, tab: "sport" },
              { label: "Hydratation", pct: waterPct, icon: "droplet", color: C.accent, tab: "water" },
              { label: "Nutrition", pct: foodResult ? 100 : 0, icon: "camera", color: C.amber, tab: "food" },
            ].map(({ label, pct, icon, color, tab: t }) => (
              <div key={label} style={{ ...styles.card2, cursor: "pointer" }} onClick={() => setTab(t)}>
                <div style={{ ...styles.row, marginBottom: 8 }}>
                  <div style={{ background: color + "20", borderRadius: 8, padding: 6 }}><Icon name={icon} size={14} color={color} /></div>
                  <span style={{ fontSize: 12, fontWeight: 600, color: C.muted }}>{label}</span>
                </div>
                <div style={{ fontSize: 22, fontWeight: 900, color: pct >= 100 ? C.green : C.text, marginBottom: 6 }}>{pct}%</div>
                <div style={styles.progressBar(pct, color, 4).outer}><div style={styles.progressBar(pct, color, 4).inner} /></div>
              </div>
            ))}
          </div>

          {/* Quick Actions */}
          <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: 10 }}>
            <button style={{ ...styles.btn(C.accent, true), borderRadius: 16 }} onClick={() => setTab("routine")}>
              <Icon name="list" size={16} /> Routine
            </button>
            <button style={{ ...styles.btn(C.green, true), borderRadius: 16 }} onClick={() => setTab("sport")}>
              <Icon name="dumbbell" size={16} /> Sport
            </button>
            <button style={{ ...styles.btn(C.amber, true), borderRadius: 16 }} onClick={() => setTab("food")}>
              <Icon name="camera" size={16} /> Scanner
            </button>
            <button style={{ ...styles.btn(C.purple, true), borderRadius: 16 }} onClick={() => setTab("coach")}>
              <Icon name="bot" size={16} /> Coach IA
            </button>
          </div>
        </div>
      </div>
    );
  }

  function RoutineScreen() {
    return (
      <div style={styles.screen}>
        <div style={styles.header}>
          <p style={styles.label}>Aujourd'hui</p>
          <h1 style={styles.h1}>Routine <span style={{ color: C.purple }}>Quotidienne</span></h1>
          <div style={{ ...styles.row, marginTop: 8 }}>
            <span style={styles.badge(C.purple)}>{routinePct}% complété</span>
            <span style={{ ...styles.muted, fontSize: 12 }}>+10 XP par tâche</span>
          </div>
        </div>
        <div style={{ padding: "0 16px" }}>
          <div style={{ ...styles.card, marginBottom: 16 }}>
            <div style={{ ...styles.between, marginBottom: 12 }}>
              <div style={styles.row}><span style={{ fontSize: 20 }}>🌅</span><h3 style={styles.h3}>Routine Matin</h3></div>
              <span style={{ ...styles.badge(C.amber), }}>{morningPct}%</span>
            </div>
            {MORNING_TASKS.map(t => (
              <div key={t.id} style={{ ...styles.between, padding: "10px 0", borderBottom: `1px solid ${C.border}` }} onClick={() => toggleMorning(t.id)}>
                <div style={styles.row}>
                  <span style={{ fontSize: 20 }}>{t.icon}</span>
                  <span style={{ fontSize: 15, fontWeight: checkedMorning[t.id] ? 700 : 400, color: checkedMorning[t.id] ? C.green : C.text, textDecoration: checkedMorning[t.id] ? "line-through" : "none", opacity: checkedMorning[t.id] ? 0.7 : 1 }}>{t.label}</span>
                </div>
                <div style={styles.check(checkedMorning[t.id])}>
                  {checkedMorning[t.id] && <Icon name="check" size={14} color="#fff" />}
                </div>
              </div>
            ))}
          </div>

          <div style={styles.card}>
            <div style={{ ...styles.between, marginBottom: 12 }}>
              <div style={styles.row}><span style={{ fontSize: 20 }}>🌙</span><h3 style={styles.h3}>Routine Soir</h3></div>
              <span style={styles.badge(C.purple)}>{eveningPct}%</span>
            </div>
            {EVENING_TASKS.map(t => (
              <div key={t.id} style={{ ...styles.between, padding: "10px 0", borderBottom: `1px solid ${C.border}` }} onClick={() => toggleEvening(t.id)}>
                <div style={styles.row}>
                  <span style={{ fontSize: 20 }}>{t.icon}</span>
                  <span style={{ fontSize: 15, fontWeight: checkedEvening[t.id] ? 700 : 400, color: checkedEvening[t.id] ? C.green : C.text, textDecoration: checkedEvening[t.id] ? "line-through" : "none", opacity: checkedEvening[t.id] ? 0.7 : 1 }}>{t.label}</span>
                </div>
                <div style={styles.check(checkedEvening[t.id])}>
                  {checkedEvening[t.id] && <Icon name="check" size={14} color="#fff" />}
                </div>
              </div>
            ))}
          </div>
        </div>
      </div>
    );
  }

  function FoodScreen() {
    return (
      <div style={styles.screen}>
        <div style={styles.header}>
          <p style={styles.label}>Nutrition</p>
          <h1 style={styles.h1}>Scanner <span style={{ color: C.amber }}>Alimentaire</span></h1>
          <p style={{ ...styles.muted, marginTop: 6 }}>Décris ton repas, l'IA l'analyse</p>
        </div>
        <div style={{ padding: "0 16px" }}>
          {/* Scanner Card */}
          <div style={{ ...styles.card, border: `1px solid ${C.amber}40` }}>
            <div style={{ background: C.amber + "15", borderRadius: 16, padding: 24, textAlign: "center", marginBottom: 16, cursor: "pointer" }}>
              <div style={{ fontSize: 48, marginBottom: 8 }}>📸</div>
              <p style={{ margin: 0, color: C.amber, fontWeight: 700, fontSize: 14 }}>Scanner par photo</p>
              <p style={{ ...styles.muted, fontSize: 12, marginTop: 4 }}>Décris ton repas ci-dessous</p>
            </div>
            <textarea
              style={{ ...styles.input, minHeight: 80, marginBottom: 12 }}
              placeholder="Ex: Pizza pepperoni, coca-cola, chips... ou Salade caesar avec poulet grillé"
              value={foodDesc}
              onChange={e => setFoodDesc(e.target.value)}
            />
            <button style={styles.btn(C.amber, true)} onClick={scanFood} disabled={foodLoading}>
              {foodLoading ? "⏳ Analyse en cours..." : "🔍 Analyser ce repas"}
            </button>
          </div>

          {/* Result */}
          {foodResult && (
            <div style={{ ...styles.card, border: `2px solid ${foodResult.verdict === "OK" ? C.green : C.red}50` }}>
              <div style={{ textAlign: "center", marginBottom: 16 }}>
                <div style={{ fontSize: 56, lineHeight: 1 }}>{foodResult.verdict === "OK" ? "✅" : "❌"}</div>
                <div style={{ fontSize: 28, fontWeight: 900, color: foodResult.verdict === "OK" ? C.green : C.red, marginTop: 8 }}>
                  {foodResult.verdict === "OK" ? "Bon choix !" : "À éviter"}
                </div>
              </div>
              <div style={{ ...styles.between, marginBottom: 12 }}>
                <span style={{ fontWeight: 700 }}>Score santé</span>
                <div style={{ ...styles.row }}>
                  {[...Array(10)].map((_, i) => (
                    <div key={i} style={{ width: 16, height: 16, borderRadius: 4, background: i < foodResult.score ? (foodResult.score >= 7 ? C.green : foodResult.score >= 5 ? C.amber : C.red) : C.border, marginLeft: 2 }} />
                  ))}
                  <span style={{ marginLeft: 8, fontWeight: 900, fontSize: 18 }}>{foodResult.score}/10</span>
                </div>
              </div>
              <div style={{ ...styles.card2, marginBottom: 10 }}>
                <p style={{ ...styles.label, marginBottom: 4 }}>Analyse</p>
                <p style={{ margin: 0, fontSize: 14, lineHeight: 1.6 }}>{foodResult.explication}</p>
              </div>
              {foodResult.alternative && (
                <div style={{ ...styles.card2, border: `1px solid ${C.green}40` }}>
                  <p style={{ ...styles.label, color: C.green, marginBottom: 4 }}>✨ Alternative proposée</p>
                  <p style={{ margin: 0, fontSize: 14, lineHeight: 1.6 }}>{foodResult.alternative}</p>
                </div>
              )}
            </div>
          )}

          {/* Tips */}
          <div style={styles.card}>
            <p style={{ ...styles.label, marginBottom: 12 }}>Conseils nutritionnels</p>
            {[
              { icon: "✅", label: "Favoriser", items: "Protéines · Légumes · Fruits · Féculents complets" },
              { icon: "❌", label: "Éviter", items: "Soda · Fast-food · Sucre industriel · Ultra-transformé" },
            ].map(({ icon, label, items }) => (
              <div key={label} style={{ ...styles.card2, marginBottom: 8 }}>
                <div style={styles.row}>
                  <span style={{ fontSize: 18 }}>{icon}</span>
                  <div>
                    <p style={{ ...styles.label, marginBottom: 2 }}>{label}</p>
                    <p style={{ ...styles.muted, margin: 0, fontSize: 13 }}>{items}</p>
                  </div>
                </div>
              </div>
            ))}
          </div>
        </div>
      </div>
    );
  }

  function SportScreen() {
    const exs = workoutDay === "A" ? WORKOUT_A : WORKOUT_B;
    return (
      <div style={styles.screen}>
        <div style={styles.header}>
          <p style={styles.label}>Training</p>
          <h1 style={styles.h1}>Sport <span style={{ color: C.green }}>du jour</span></h1>
        </div>
        <div style={{ padding: "0 16px" }}>
          {/* Day Selector */}
          <div style={{ ...styles.card, display: "flex", gap: 10 }}>
            {["A", "B"].map(d => (
              <button key={d} style={{ flex: 1, background: workoutDay === d ? C.green : C.card2, color: workoutDay === d ? "#fff" : C.muted, border: `1.5px solid ${workoutDay === d ? C.green : C.border}`, borderRadius: 12, padding: "12px", fontWeight: 700, cursor: "pointer", fontSize: 14 }} onClick={() => { setWorkoutDay(d); setCompletedExercises({}); }}>
                Jour {d} · {d === "A" ? "💪 Muscu" : "🏃 Cardio"}
              </button>
            ))}
          </div>

          {/* Progress */}
          <div style={{ ...styles.card2, marginBottom: 12 }}>
            <div style={styles.between}>
              <span style={{ fontWeight: 700 }}>Progression</span>
              <span style={{ color: C.green, fontWeight: 900 }}>{workoutPct}%</span>
            </div>
            <div style={{ ...styles.progressBar(workoutPct, C.green).outer, marginTop: 8 }}>
              <div style={styles.progressBar(workoutPct, C.green).inner} />
            </div>
          </div>

          {/* Timer */}
          {timerActive !== null && timerCount > 0 && (
            <div style={{ ...styles.card, textAlign: "center", background: `linear-gradient(135deg, ${C.accent}20, ${C.card})`, border: `1px solid ${C.accent}40` }}>
              <p style={{ ...styles.label, color: C.accent }}>⏱️ Repos</p>
              <div style={{ fontSize: 48, fontWeight: 900, color: C.accent }}>{timerCount}s</div>
            </div>
          )}

          {/* Exercises */}
          {exs.map(ex => (
            <div key={ex.id} style={{ ...styles.card, border: `1px solid ${completedExercises[ex.id] ? C.green + "60" : C.border}`, background: completedExercises[ex.id] ? C.green + "08" : C.card }}>
              <div style={styles.between}>
                <div style={styles.row}>
                  <span style={{ fontSize: 28 }}>{ex.icon}</span>
                  <div>
                    <p style={{ margin: 0, fontWeight: 700, fontSize: 16, color: completedExercises[ex.id] ? C.green : C.text }}>{ex.label}</p>
                    <p style={{ ...styles.muted, margin: 0 }}>{ex.sets} · +20 XP</p>
                  </div>
                </div>
                <div style={styles.check(completedExercises[ex.id])} onClick={() => { toggleExercise(ex.id); if (ex.rest > 0 && !completedExercises[ex.id]) startTimer(ex.rest, ex.id); }}>
                  {completedExercises[ex.id] && <Icon name="check" size={14} color="#fff" />}
                </div>
              </div>
              {ex.rest > 0 && !completedExercises[ex.id] && (
                <button style={{ ...styles.btnOutline, fontSize: 12, padding: "8px 14px", marginTop: 10 }} onClick={() => startTimer(ex.rest, ex.id)}>
                  ⏱️ Timer {ex.rest}s
                </button>
              )}
            </div>
          ))}

          {workoutPct === 100 && (
            <div style={{ ...styles.card, background: `linear-gradient(135deg, ${C.green}20, ${C.card})`, textAlign: "center", border: `1px solid ${C.green}50` }}>
              <div style={{ fontSize: 48 }}>🏆</div>
              <h3 style={{ ...styles.h3, color: C.green, fontSize: 18, marginTop: 8 }}>Workout terminé !</h3>
              <p style={{ ...styles.muted, marginTop: 4 }}>+{exs.length * 20} XP gagnés. Tu progresses.</p>
            </div>
          )}
        </div>
      </div>
    );
  }

  function WaterScreen() {
    return (
      <div style={styles.screen}>
        <div style={styles.header}>
          <p style={styles.label}>Hydratation</p>
          <h1 style={styles.h1}>Eau <span style={{ color: C.accent }}>du jour</span></h1>
        </div>
        <div style={{ padding: "0 16px" }}>
          <div style={{ ...styles.card, textAlign: "center", padding: 32 }}>
            <div style={{ fontSize: 72, lineHeight: 1, marginBottom: 8 }}>💧</div>
            <div style={{ fontSize: 56, fontWeight: 900, color: C.accent }}>{water}</div>
            <div style={{ ...styles.muted, fontSize: 16 }}>verres sur 8 (objectif 2L)</div>
            <div style={{ marginTop: 20, marginBottom: 20 }}>
              <div style={styles.progressBar(waterPct, C.accent, 12).outer}>
                <div style={styles.progressBar(waterPct, C.accent, 12).inner} />
              </div>
              <div style={{ ...styles.between, marginTop: 6 }}>
                <span style={styles.muted}>{water * 250}ml</span>
                <span style={{ color: C.accent, fontWeight: 700 }}>{waterPct}%</span>
                <span style={styles.muted}>2000ml</span>
              </div>
            </div>
            <button style={{ ...styles.btn(C.accent, true), fontSize: 18, padding: "18px", borderRadius: 18 }} onClick={addWater}>
              <Icon name="plus" size={22} color="#fff" /> Verre d'eau (+5 XP)
            </button>
          </div>

          <div style={{ display: "grid", gridTemplateColumns: "repeat(4, 1fr)", gap: 8, marginTop: 4 }}>
            {[...Array(8)].map((_, i) => (
              <div key={i} style={{ background: i < water ? C.accent + "30" : C.card2, border: `1.5px solid ${i < water ? C.accent : C.border}`, borderRadius: 14, padding: "16px 0", textAlign: "center", fontSize: 22 }}>
                {i < water ? "💧" : "○"}
              </div>
            ))}
          </div>

          <div style={{ ...styles.card, marginTop: 12 }}>
            <p style={{ ...styles.label, marginBottom: 12 }}>Pourquoi s'hydrater ?</p>
            {["Améliore la concentration et les performances", "Aide à la récupération musculaire", "Meilleure peau et énergie", "Objectif : 8 verres minimum / jour"].map(tip => (
              <div key={tip} style={{ ...styles.row, padding: "8px 0", borderBottom: `1px solid ${C.border}` }}>
                <span style={{ color: C.accent, fontWeight: 900 }}>›</span>
                <span style={{ fontSize: 14 }}>{tip}</span>
              </div>
            ))}
          </div>
        </div>
      </div>
    );
  }

  function StatsScreen() {
    const stats = [
      { label: "Streak", value: streak, suffix: "jours", icon: "flame", color: C.amber },
      { label: "XP Total", value: xp, suffix: "xp", icon: "zap", color: lv.color },
      { label: "Niveau", value: lv.name, suffix: "", icon: "star", color: C.purple },
      { label: "Routine", value: routinePct, suffix: "%", icon: "list", color: C.green },
      { label: "Sport", value: workoutPct, suffix: "%", icon: "dumbbell", color: C.green },
      { label: "Hydratation", value: waterPct, suffix: "%", icon: "droplet", color: C.accent },
    ];
    const weekData = [65, 80, 45, 90, 70, 100, routinePct];
    const days = ["L", "M", "M", "J", "V", "S", "D"];
    return (
      <div style={styles.screen}>
        <div style={styles.header}>
          <p style={styles.label}>Progression</p>
          <h1 style={styles.h1}>Mes <span style={{ color: C.green }}>Statistiques</span></h1>
        </div>
        <div style={{ padding: "0 16px" }}>
          <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: 10, marginBottom: 12 }}>
            {stats.map(({ label, value, suffix, icon, color }) => (
              <div key={label} style={{ ...styles.card2, textAlign: "center" }}>
                <div style={{ background: color + "20", borderRadius: 12, padding: 10, width: 44, height: 44, margin: "0 auto 10px", display: "flex", alignItems: "center", justifyContent: "center" }}>
                  <Icon name={icon} size={20} color={color} />
                </div>
                <div style={{ fontSize: 24, fontWeight: 900, color }}>{value}{suffix}</div>
                <div style={{ ...styles.muted, fontSize: 11, marginTop: 2 }}>{label}</div>
              </div>
            ))}
          </div>

          {/* Weekly Chart */}
          <div style={styles.card}>
            <p style={{ ...styles.label, marginBottom: 16 }}>Performance 7 derniers jours</p>
            <div style={{ display: "flex", alignItems: "flex-end", gap: 6, height: 100, paddingBottom: 8 }}>
              {weekData.map((v, i) => (
                <div key={i} style={{ flex: 1, display: "flex", flexDirection: "column", alignItems: "center", gap: 4 }}>
                  <div style={{ width: "100%", background: i === 6 ? C.accent : C.accent + "50", borderRadius: "6px 6px 0 0", height: `${v}%`, minHeight: 4, transition: "height 0.5s ease" }} />
                  <span style={{ fontSize: 11, color: i === 6 ? C.text : C.muted, fontWeight: i === 6 ? 700 : 400 }}>{days[i]}</span>
                </div>
              ))}
            </div>
            <div style={{ ...styles.row, justifyContent: "center", marginTop: 8 }}>
              <div style={{ width: 12, height: 12, borderRadius: 3, background: C.accent }} />
              <span style={{ ...styles.muted, fontSize: 12 }}>Routine complétée (%)</span>
            </div>
          </div>

          {/* Achievements */}
          <div style={styles.card}>
            <p style={{ ...styles.label, marginBottom: 12 }}>Badges</p>
            {[
              { icon: "🔥", label: "Streak 7 jours", done: streak >= 7 },
              { icon: "💧", label: "5 jours d'hydratation", done: true },
              { icon: "💪", label: "10 séances sport", done: xp >= 200 },
              { icon: "🏆", label: "Niveau Intermédiaire", done: xp >= 300 },
            ].map(({ icon, label, done }) => (
              <div key={label} style={{ ...styles.row, padding: "10px 0", borderBottom: `1px solid ${C.border}`, opacity: done ? 1 : 0.4 }}>
                <span style={{ fontSize: 22, filter: done ? "none" : "grayscale(1)" }}>{icon}</span>
                <span style={{ fontSize: 14, fontWeight: done ? 700 : 400 }}>{label}</span>
                {done && <span style={{ marginLeft: "auto", ...styles.badge(C.green) }}>✓</span>}
              </div>
            ))}
          </div>
        </div>
      </div>
    );
  }

  function CoachScreen() {
    const chatRef = useRef(null);
    useEffect(() => { chatRef.current?.scrollTo(0, chatRef.current.scrollHeight); }, [coachMessages, chatLoading]);
    return (
      <div style={{ ...styles.screen, display: "flex", flexDirection: "column", height: "calc(100vh - 80px)" }}>
        <div style={{ ...styles.header, paddingBottom: 16 }}>
          <div style={styles.row}>
            <div style={{ background: C.accent + "20", borderRadius: 14, padding: 10 }}>
              <Icon name="bot" size={24} color={C.accent} />
            </div>
            <div>
              <h2 style={styles.h2}>APEX Coach</h2>
              <p style={{ ...styles.muted, margin: 0, fontSize: 12 }}>IA personnelle · disponible 24/7</p>
            </div>
            <div style={{ marginLeft: "auto" }}><span style={styles.badge(C.green)}>● Online</span></div>
          </div>
        </div>

        <div ref={chatRef} style={{ flex: 1, overflowY: "auto", padding: "0 16px", display: "flex", flexDirection: "column", gap: 10 }}>
          {coachMessages.map((msg, i) => (
            <div key={i} style={{ display: "flex", justifyContent: msg.role === "user" ? "flex-end" : "flex-start" }}>
              <div style={{ maxWidth: "82%", background: msg.role === "user" ? C.accent : C.card, borderRadius: msg.role === "user" ? "18px 18px 4px 18px" : "18px 18px 18px 4px", padding: "12px 16px", border: msg.role === "assistant" ? `1px solid ${C.border}` : "none" }}>
                {msg.role === "assistant" && <p style={{ margin: "0 0 4px", fontSize: 10, fontWeight: 700, color: C.accent, letterSpacing: "0.1em" }}>APEX</p>}
                <p style={{ margin: 0, fontSize: 14, lineHeight: 1.6, color: msg.role === "user" ? "#fff" : C.text }}>{msg.text}</p>
              </div>
            </div>
          ))}
          {chatLoading && (
            <div style={{ display: "flex" }}>
              <div style={{ background: C.card, borderRadius: "18px 18px 18px 4px", padding: "12px 16px", border: `1px solid ${C.border}` }}>
                <div style={{ display: "flex", gap: 4 }}>
                  {[0, 1, 2].map(i => (
                    <div key={i} style={{ width: 8, height: 8, borderRadius: "50%", background: C.accent, animation: `bounce${i} 1s infinite`, opacity: 0.7 }} />
                  ))}
                </div>
              </div>
            </div>
          )}
        </div>

        {/* Suggestions rapides */}
        <div style={{ padding: "8px 16px", display: "flex", gap: 8, overflowX: "auto" }}>
          {["Comment progresser ?", "Mon plan de la semaine", "Conseils nutrition", "Motivation !"].map(s => (
            <button key={s} style={{ ...styles.btnOutline, fontSize: 12, padding: "7px 14px", whiteSpace: "nowrap", flexShrink: 0 }} onClick={() => { setChatInput(s); }}>
              {s}
            </button>
          ))}
        </div>

        <div style={{ padding: "8px 16px 16px", ...styles.row }}>
          <textarea style={{ ...styles.input, resize: "none", height: 44, paddingTop: 12 }} placeholder="Parle à ton coach..." value={chatInput} onChange={e => setChatInput(e.target.value)} onKeyDown={e => { if (e.key === "Enter" && !e.shiftKey) { e.preventDefault(); sendChat(); } }} />
          <button style={{ ...styles.btn(C.accent), borderRadius: 14, padding: "12px 16px", flexShrink: 0 }} onClick={sendChat} disabled={chatLoading}>
            <Icon name="send" size={18} />
          </button>
        </div>
        <style>{`@keyframes bounce0{0%,80%,100%{transform:translateY(0)}40%{transform:translateY(-6px)}} @keyframes bounce1{0%,80%,100%{transform:translateY(0)}40%{transform:translateY(-6px)}} @keyframes bounce2{0%,80%,100%{transform:translateY(0)}40%{transform:translateY(-6px)}} div[style*="bounce0"]{animation-delay:0s} div[style*="bounce1"]{animation-delay:.15s} div[style*="bounce2"]{animation-delay:.3s}`}</style>
      </div>
    );
  }

  function SettingsScreen() {
    return (
      <div style={styles.screen}>
        <div style={styles.header}>
          <p style={styles.label}>Personnalisation</p>
          <h1 style={styles.h1}>Paramètres</h1>
        </div>
        <div style={{ padding: "0 16px" }}>
          <div style={styles.card}>
            {[
              { label: "Heure de réveil", type: "time", key: "wakeTime" },
              { label: "Objectif physique", type: "select", key: "goal", opts: ["Prise de masse", "Perte de poids", "Endurance", "Rester en forme"] },
              { label: "Niveau de difficulté", type: "select", key: "difficulty", opts: ["Débutant", "Intermédiaire", "Avancé"] },
            ].map(({ label, type, key, opts }) => (
              <div key={key} style={{ padding: "14px 0", borderBottom: `1px solid ${C.border}` }}>
                <p style={{ ...styles.label, marginBottom: 8 }}>{label}</p>
                {type === "select" ? (
                  <select style={{ ...styles.input, appearance: "none" }} value={settings[key]} onChange={e => setSettings(s => ({ ...s, [key]: e.target.value }))}>
                    {opts.map(o => <option key={o}>{o}</option>)}
                  </select>
                ) : (
                  <input type="time" style={styles.input} value={settings[key]} onChange={e => setSettings(s => ({ ...s, [key]: e.target.value }))} />
                )}
              </div>
            ))}
            <div style={{ ...styles.between, padding: "14px 0", borderBottom: `1px solid ${C.border}` }}>
              <div>
                <p style={{ ...styles.label, marginBottom: 2 }}>Notifications</p>
                <p style={{ ...styles.muted, margin: 0, fontSize: 12 }}>Rappels quotidiens</p>
              </div>
              <div style={{ width: 50, height: 28, borderRadius: 14, background: settings.notifs ? C.accent : C.border, cursor: "pointer", position: "relative", transition: "background 0.2s" }} onClick={() => setSettings(s => ({ ...s, notifs: !s.notifs }))}>
                <div style={{ width: 22, height: 22, borderRadius: "50%", background: "#fff", position: "absolute", top: 3, left: settings.notifs ? 25 : 3, transition: "left 0.2s" }} />
              </div>
            </div>
            <div style={{ ...styles.between, padding: "14px 0" }}>
              <div>
                <p style={{ ...styles.label, marginBottom: 2 }}>Mode sombre</p>
                <p style={{ ...styles.muted, margin: 0, fontSize: 12 }}>Interface actuelle</p>
              </div>
              <div style={{ width: 50, height: 28, borderRadius: 14, background: darkMode ? C.accent : C.border, cursor: "pointer", position: "relative", transition: "background 0.2s" }} onClick={() => setDarkMode(d => !d)}>
                <div style={{ width: 22, height: 22, borderRadius: "50%", background: "#fff", position: "absolute", top: 3, left: darkMode ? 25 : 3, transition: "left 0.2s" }} />
              </div>
            </div>
          </div>

          <div style={{ ...styles.card, textAlign: "center" }}>
            <div style={{ fontSize: 32, marginBottom: 8 }}>⚡</div>
            <h3 style={{ ...styles.h3, marginBottom: 6, fontSize: 18 }}>APEX Lookmaxing</h3>
            <p style={{ ...styles.muted, fontSize: 13 }}>Coach IA · Transformation 30 jours</p>
            <p style={{ ...styles.muted, fontSize: 12, marginTop: 4 }}>v1.0 · Propulsé par Claude AI</p>
          </div>
        </div>
      </div>
    );
  }

  const navItems = [
    { id: "home", icon: "home", label: "Home" },
    { id: "routine", icon: "list", label: "Routine" },
    { id: "food", icon: "camera", label: "Scan" },
    { id: "sport", icon: "dumbbell", label: "Sport" },
    { id: "water", icon: "droplet", label: "Eau" },
    { id: "stats", icon: "bar", label: "Stats" },
    { id: "coach", icon: "bot", label: "Coach" },
    { id: "settings", icon: "settings", label: "Réglages" },
  ];

  return (
    <div style={styles.app}>
      <link href="https://fonts.googleapis.com/css2?family=DM+Sans:ital,wght@0,400;0,500;0,600;0,700;0,800;0,900;1,400&display=swap" rel="stylesheet" />
      {screens[tab]}
      <nav style={styles.navbar}>
        {navItems.map(({ id, icon, label }) => (
          <button key={id} style={styles.navBtn(tab === id)} onClick={() => setTab(id)}>
            <Icon name={icon} size={20} color={tab === id ? C.accent : C.muted} />
            <span>{label}</span>
          </button>
        ))}
      </nav>
    </div>
  );
}

