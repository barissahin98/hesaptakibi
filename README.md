<!DOCTYPE html>
<html lang="tr">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
<title>₺ Hesap Takibi</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
  html, body { background: #0b1520; overflow-x: hidden; }
  input[type="number"]::-webkit-inner-spin-button,
  input[type="number"]::-webkit-outer-spin-button { opacity: 1; }
  select option { background: #0b1520; color: #38bdf8; }
</style>
</head>
<body>
<div id="root"></div>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react/18.2.0/umd/react.production.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react-dom/18.2.0/umd/react-dom.production.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/babel-standalone/7.23.9/babel.min.js"></script>
<script type="text/babel">
const { useState, useEffect, useMemo } = React;

const DEF_G = ["Satış", "Hizmet", "Faiz", "Kira Geliri", "Borç", "Diğer Gelir"];
const DEF_Gi = ["Kira", "Maaş", "Fatura", "Malzeme", "Ulaşım", "Vergi", "Borç", "Diğer Gider"];

function fmt(v) { return new Intl.NumberFormat("tr-TR", { style: "currency", currency: "TRY", maximumFractionDigits: 0 }).format(v); }
function fmt2(v) { return new Intl.NumberFormat("tr-TR", { style: "currency", currency: "TRY" }).format(v); }
function fmtD(d) { return new Date(d).toLocaleDateString("tr-TR", { day: "numeric", month: "short" }); }
function fmtC(v) { return (v > 0 ? "+" : "") + new Intl.NumberFormat("tr-TR", { notation: "compact", maximumFractionDigits: 1 }).format(v); }
function gid() { return Date.now().toString(36) + Math.random().toString(36).slice(2, 7); }
function td() { return new Date().toISOString().slice(0, 10); }

async function loadDB() {
  try { const r = localStorage.getItem("muh-v6"); return r ? JSON.parse(r) : null; } catch { return null; }
}
async function saveDB(d) {
  try { localStorage.setItem("muh-v6", JSON.stringify(d)); } catch (e) { /* silent */ }
}

function App() {
  /* ═══ STATE ═══ */
  const [txns, setTxns] = useState([]);
  const [plans, setPlans] = useState([]);
  const [cariler, setCariler] = useState([]);
  const [cards, setCards] = useState([]);
  const [hesaplar, setHesaplar] = useState([]);
  const [page, setPage] = useState("ekle");
  const emptyForm = { type: "gelir", amount: "", desc: "", category: "Satış", date: td(), cariId: "", masraf: "", masrafCat: "Diğer Gider", hasMasraf: false, hesapId: "", hesapIdFrom: "", vadeli: false };
  const emptyPlan = { type: "gelir", amount: "", desc: "", category: "Satış", date: "", cariId: "", masraf: "", masrafCat: "Diğer Gider", hasMasraf: false, status: "odenmedi", recurring: false, recurFreq: "aylik" };
  const [form, setForm] = useState({ ...emptyForm });
  const [planForm, setPlanForm] = useState({ ...emptyPlan });
  const [cariForm, setCariForm] = useState({ name: "", note: "" });
  const [cardForm, setCardForm] = useState({ name: "", last4: "", paymentDay: "1" });
  const [hesapForm, setHesapForm] = useState({ name: "", type: "banka", last4: "", paymentDay: "1" });
  const [editHesapId, setEditHesapId] = useState(null);
  const [filter, setFilter] = useState({ type: "all", date: "", search: "", category: "all", dateType: "ay" });
  const [showSearch, setShowSearch] = useState(false);
  const [editId, setEditId] = useState(null);
  const [editPlanId, setEditPlanId] = useState(null);
  const [editCariId, setEditCariId] = useState(null);
  const [editCardId, setEditCardId] = useState(null);
  const [selectedCari, setSelectedCari] = useState(null);
  const [selectedDay, setSelectedDay] = useState(null);
  const [realizingPlan, setRealizingPlan] = useState(null);
  const [calMonth, setCalMonth] = useState(() => ({ year: new Date().getFullYear(), month: new Date().getMonth() }));
  const [includeBal, setIncludeBal] = useState(false);
  const [custG, setCustG] = useState([]);
  const [custGi, setCustGi] = useState([]);
  const [remG, setRemG] = useState([]);
  const [remGi, setRemGi] = useState([]);
  const [showNewCat, setShowNewCat] = useState(null);
  const [newCatVal, setNewCatVal] = useState("");
  const [loaded, setLoaded] = useState(false);
  const [toast, setToast] = useState("");

  /* ═══ PERSISTENCE ═══ */
  useEffect(() => {
    loadDB().then(d => {
      if (!d) { setLoaded(true); return; }
      if (d.txns) setTxns(d.txns);
      if (d.plans) setPlans(d.plans);
      if (d.cariler) setCariler(d.cariler);
      if (d.cards) setCards(d.cards);
      if (d.hesaplar) setHesaplar(d.hesaplar);
      if (d.custG) setCustG(d.custG);
      if (d.custGi) setCustGi(d.custGi);
      if (d.remG) setRemG(d.remG);
      if (d.remGi) setRemGi(d.remGi);
      setLoaded(true);
    });
  }, []);

  useEffect(() => {
    if (loaded) saveDB({ txns, plans, cariler, cards, hesaplar, custG, custGi, remG, remGi });
  }, [txns, plans, cariler, cards, hesaplar, custG, custGi, remG, remGi, loaded]);

  /* ═══ HELPERS ═══ */
  const sho = (m) => { setToast(m); setTimeout(() => setToast(""), 1800); };
  const cn = (id) => cariler.find(c => c.id === id)?.name || "";
  const cdn = (id) => { const c = cards.find(x => x.id === id); return c ? c.name + "(*" + c.last4 + ")" : ""; };
  const hn = (id) => { const c = cards.find(x => x.id === id); if (c) return "💳 " + c.name; const h = hesaplar.find(x => x.id === id); if (h) return (h.type === "banka" ? "🏦 " : "💵 ") + h.name; return ""; };
  const isCardId = (id) => cards.some(c => c.id === id);
  const netA = (t) => t.type === "gelir" ? t.amount - (t.masraf || 0) : t.amount;

  const allG = [...DEF_G.filter(c => !remG.includes(c)), ...custG];
  const allGi = [...DEF_Gi.filter(c => !remGi.includes(c)), ...custGi];

  /* ═══ CALCULATIONS ═══ */
  const cashT = txns.filter(t => t.type !== "virman" && !isCardId(t.hesapId || t.cardId || "") && !t.vadeli);
  const ccT = txns.filter(t => t.type !== "virman" && isCardId(t.hesapId || t.cardId || ""));
  const vadeliT = txns.filter(t => !!t.vadeli);
  const tot = cashT.reduce((a, t) => { if (t.type === "gelir") { a.g += t.amount; a.m += (t.masraf || 0); } else { a.gi += t.amount; } return a; }, { g: 0, gi: 0, m: 0 });
  const bal = tot.g - tot.m - tot.gi;
  const pTot = plans.reduce((a, t) => { if (t.type === "gelir") { a.g += t.amount; a.m += (t.masraf || 0); } else { a.gi += t.amount; } return a; }, { g: 0, gi: 0, m: 0 });
  const pBal = pTot.g - pTot.m - pTot.gi;

  const ccBal = {};
  ccT.forEach(t => { const cid = t.hesapId || t.cardId; ccBal[cid] = (ccBal[cid] || 0) + t.amount; });
  const ccDebt = Object.values(ccBal).reduce((s, v) => s + v, 0);

  const ccPlans = useMemo(() => {
    const res = [];
    cards.forEach(card => {
      const debt = ccBal[card.id] || 0;
      if (debt <= 0) return;
      const now = new Date();
      let pm = now.getDate() <= card.paymentDay ? now.getMonth() : now.getMonth() + 1;
      let py = now.getFullYear();
      if (pm > 11) { pm = 0; py++; }
      const mx = new Date(py, pm + 1, 0).getDate();
      const ad = Math.min(card.paymentDay, mx);
      const ds = py + "-" + String(pm + 1).padStart(2, "0") + "-" + String(ad).padStart(2, "0");
      res.push({ id: "cc-" + card.id, type: "gider", amount: debt, desc: card.name + "(*" + card.last4 + ") kart ödemesi", category: "Kredi Kartı", date: ds, cariId: "", masraf: 0, masrafCat: "", isCC: true, cardId: card.id });
    });
    return res;
  }, [cards, JSON.stringify(ccBal)]);

  const cariBals = {};
  [...txns, ...plans].forEach(t => {
    if (!t.cariId) return;
    if (!cariBals[t.cariId]) cariBals[t.cariId] = 0;
    cariBals[t.cariId] += t.type === "gelir" ? (t.category === "Borç" ? -netA(t) : netA(t)) : (t.category === "Borç" ? t.amount : -t.amount);
  });

  const moData = txns.reduce((a, t) => {
    if (t.type === "virman" || isCardId(t.hesapId || t.cardId || "") || t.vadeli || !t.date) return a;
    const m = t.date.slice(0, 7);
    if (!a[m]) a[m] = { g: 0, gi: 0 };
    if (t.type === "gelir") { a[m].g += t.amount; a[m].gi += (t.masraf || 0); }
    else { a[m].gi += t.amount; }
    return a;
  }, {});

  const usedCats = [...new Set(txns.map(t => t.category))].sort();
  const filtered = txns.filter(t => {
    if (filter.type !== "all" && t.type !== filter.type) return false;
    if (filter.category !== "all" && t.category !== filter.category) return false;
    if (filter.date && t.date && !t.date.startsWith(filter.date)) return false;
    if (filter.search) {
      const q = filter.search.toLowerCase();
      const hay = [t.desc, t.category, t.date, t.type, t.cariId ? cn(t.cariId) : "", (t.hesapId || t.cardId) ? hn(t.hesapId || t.cardId) : ""].join(" ").toLowerCase();
      if (!hay.includes(q)) return false;
    }
    return true;
  }).sort((a, b) => (b.date || "").localeCompare(a.date || ""));

  // Calendar
  const calAll = [...cashT.filter(t => t.date), ...plans.filter(t => t.date), ...ccPlans].sort((a, b) => a.date.localeCompare(b.date));
  const ddMap = {};
  calAll.forEach(t => { ddMap[t.date] = (ddMap[t.date] || 0) + (t.type === "gelir" ? netA(t) : -t.amount); });
  const sortDates = Object.keys(ddMap).sort();
  const cumBal = {};
  let runB = 0;
  sortDates.forEach(d => { runB += ddMap[d]; cumBal[d] = runB; });
  const balAt = (ds) => { let b = 0; for (const d of sortDates) { if (d > ds) break; b = cumBal[d]; } return b; };
  const txAt = (ds) => calAll.filter(t => t.date === ds);
  const cDays = new Date(calMonth.year, calMonth.month + 1, 0).getDate();
  const cStart = (() => { const d = new Date(calMonth.year, calMonth.month, 1).getDay(); return d === 0 ? 6 : d - 1; })();
  const cLabel = new Date(calMonth.year, calMonth.month).toLocaleDateString("tr-TR", { month: "long", year: "numeric" });
  const months = Object.keys(moData).sort();
  const cariTxns = selectedCari ? [...txns, ...plans].filter(t => t.cariId === selectedCari).sort((a, b) => (b.date || "").localeCompare(a.date || "")) : [];

  /* ═══ HANDLERS ═══ */
  const subTxn = () => {
    if (!form.amount || isNaN(form.amount) || +form.amount <= 0) return sho("Geçerli tutar girin");
    if (form.date > td()) return sho("Gelecek tarih için Plan kullanın");
    if (!form.date) return sho("Tarih seçin");
    if (form.type === "virman") {
      if (!form.hesapIdFrom) return sho("Kaynak hesap seçin");
      if (!form.hesapId) return sho("Hedef hesap seçin");
      if (form.hesapIdFrom === form.hesapId) return sho("Kaynak ve hedef hesap farklı olmalı");
      const e = { id: editId || gid(), type: "virman", amount: +form.amount, desc: form.desc || "Virman", category: "Virman", date: form.date, cariId: "", masraf: 0, masrafCat: "", hesapId: form.hesapId, hesapIdFrom: form.hesapIdFrom, vadeli: false };
      if (editId) { setTxns(p => p.map(t => t.id === editId ? e : t)); setEditId(null); sho("Güncellendi"); }
      else { setTxns(p => [...p, e]); sho("Virman kaydedildi"); }
      setForm({ ...emptyForm });
      return;
    }
    if (form.category === "Borç" && !form.cariId) return sho("Borç kategorisi için cari seçimi zorunludur");
    if (!form.hesapId) return sho("Hesap seçimi zorunludur");
    const m = form.hasMasraf && form.masraf && !isNaN(form.masraf) ? +form.masraf : 0;
    const e = { id: editId || gid(), type: form.type, amount: +form.amount, desc: form.desc, category: form.category, date: form.date, cariId: form.cariId || "", masraf: m, masrafCat: form.hasMasraf ? (form.masrafCat || "Diğer Gider") : "", hesapId: form.hesapId || "", vadeli: !!(form.vadeli && form.cariId && form.category !== "Borç") };
    if (editId) { setTxns(p => p.map(t => t.id === editId ? e : t)); setEditId(null); sho("Güncellendi"); }
    else { setTxns(p => [...p, e]); sho("Eklendi"); }

    // Plan gerçekleştirme
    if (realizingPlan) {
      const { planId, partial, originalAmount } = realizingPlan;
      if (partial && +form.amount < originalAmount) {
        const remaining = originalAmount - +form.amount;
        setPlans(p => p.map(t => t.id === planId ? { ...t, amount: remaining, status: "kismen" } : t));
        sho("Kısmi ödeme kaydedildi, kalan: " + fmt(remaining));
      } else {
        const plan = plans.find(t => t.id === planId);
        if (plan && plan.recurring && plan.date) {
          const freq = plan.recurFreq || "aylik";
          const d = new Date(plan.date);
          if (freq === "gunluk") d.setDate(d.getDate() + 1);
          else if (freq === "haftalik") d.setDate(d.getDate() + 7);
          else if (freq === "aylik") d.setMonth(d.getMonth() + 1);
          else if (freq === "yillik") d.setFullYear(d.getFullYear() + 1);
          const newDate = d.toISOString().slice(0, 10);
          setPlans(p => p.map(t => t.id === planId ? { ...t, date: newDate, status: "odenmedi", amount: plan.amount } : t));
        } else {
          setPlans(p => p.filter(t => t.id !== planId));
        }
      }
      setRealizingPlan(null);
    }

    setForm({ ...emptyForm });
  };

  const subPlan = () => {
    if (!planForm.amount || isNaN(planForm.amount) || +planForm.amount <= 0) return sho("Geçerli tutar girin");
    if (!planForm.date) return sho("Tarih seçin");
    const m = planForm.hasMasraf && planForm.masraf && !isNaN(planForm.masraf) ? +planForm.masraf : 0;
    const e = { id: editPlanId || gid(), type: planForm.type, amount: +planForm.amount, desc: planForm.desc, category: planForm.category, date: planForm.date, cariId: planForm.cariId || "", masraf: m, masrafCat: planForm.hasMasraf ? (planForm.masrafCat || "Diğer Gider") : "", status: planForm.status || "odenmedi", recurring: !!planForm.recurring, recurFreq: planForm.recurFreq || "aylik" };
    if (editPlanId) { setPlans(p => p.map(t => t.id === editPlanId ? e : t)); setEditPlanId(null); sho("Güncellendi"); }
    else { setPlans(p => [...p, e]); sho("Eklendi"); }
    setPlanForm({ ...emptyPlan });
  };

  const realizePlan = (plan, partial) => {
    setForm({
      type: plan.type, amount: partial ? "" : String(plan.amount), desc: plan.desc, category: plan.category,
      date: td(), cariId: plan.cariId || "", masraf: plan.masraf ? String(plan.masraf) : "", masrafCat: plan.masrafCat || "Diğer Gider",
      hasMasraf: !!(plan.masraf > 0), hesapId: "", vadeli: false
    });
    setRealizingPlan({ planId: plan.id, partial, originalAmount: plan.amount });
    setEditId(null);
    setPage("ekle");
  };

  const realizeCC = (ccPlan, partial) => {
    setForm({
      type: "gider", amount: partial ? "" : String(ccPlan.amount), desc: ccPlan.desc || "Kart Ödemesi",
      category: "Kredi Kartı", date: td(), cariId: "", masraf: "", masrafCat: "Diğer Gider",
      hasMasraf: false, hesapId: "", hesapIdFrom: "", vadeli: false
    });
    setEditId(null);
    setRealizingPlan(null);
    setPage("ekle");
    sho("Kart ödemesi için hesap seçin");
  };

  const subCari = () => {
    if (!cariForm.name.trim()) return sho("İsim girin");
    const e = { id: editCariId || gid(), name: cariForm.name.trim(), note: cariForm.note.trim() };
    if (editCariId) { setCariler(p => p.map(c => c.id === editCariId ? e : c)); setEditCariId(null); sho("Güncellendi"); }
    else { setCariler(p => [...p, e]); sho("Eklendi"); }
    setCariForm({ name: "", note: "" });
  };

  const subCard = () => {
    if (!cardForm.name.trim()) return sho("Kart adı girin");
    const day = Math.max(1, Math.min(31, parseInt(cardForm.paymentDay) || 1));
    const e = { id: editCardId || gid(), name: cardForm.name.trim(), last4: cardForm.last4.trim(), paymentDay: day };
    if (editCardId) { setCards(p => p.map(c => c.id === editCardId ? e : c)); setEditCardId(null); sho("Güncellendi"); }
    else { setCards(p => [...p, e]); sho("Eklendi"); }
    setCardForm({ name: "", last4: "", paymentDay: "1" });
  };

  const eTxn = (t) => { setForm({ type: t.type, amount: String(t.amount), desc: t.desc, category: t.category, date: t.date || td(), cariId: t.cariId || "", masraf: t.masraf ? String(t.masraf) : "", masrafCat: t.masrafCat || "Diğer Gider", hasMasraf: !!(t.masraf > 0), hesapId: t.hesapId || t.cardId || "", hesapIdFrom: t.hesapIdFrom || "", vadeli: !!t.vadeli }); setEditId(t.id); setPage("ekle"); };
  const ePlan = (t) => { setPlanForm({ type: t.type, amount: String(t.amount), desc: t.desc, category: t.category, date: t.date || "", cariId: t.cariId || "", masraf: t.masraf ? String(t.masraf) : "", masrafCat: t.masrafCat || "Diğer Gider", hasMasraf: !!(t.masraf > 0), status: t.status || "odenmedi", recurring: !!t.recurring, recurFreq: t.recurFreq || "aylik" }); setEditPlanId(t.id); };
  const dTxn = (id) => { setTxns(p => p.filter(t => t.id !== id)); sho("Silindi"); };
  const dPlan = (id) => { setPlans(p => p.filter(t => t.id !== id)); sho("Silindi"); };
  const dCari = (id) => { setCariler(p => p.filter(c => c.id !== id)); setTxns(p => p.map(t => t.cariId === id ? { ...t, cariId: "" } : t)); sho("Silindi"); };
  const dCard = (id) => { setCards(p => p.filter(c => c.id !== id)); setTxns(p => p.map(t => (t.hesapId === id || t.cardId === id) ? { ...t, hesapId: "", cardId: "" } : t)); sho("Silindi"); };
  const subHesap = () => {
    if (!hesapForm.name.trim()) return sho("Hesap adı girin");
    if (hesapForm.type === "kredi") {
      const day = Math.max(1, Math.min(31, parseInt(hesapForm.paymentDay) || 1));
      const e = { id: editHesapId || gid(), name: hesapForm.name.trim(), last4: hesapForm.last4.trim(), paymentDay: day };
      if (editHesapId) {
        if (cards.some(c => c.id === editHesapId)) { setCards(p => p.map(c => c.id === editHesapId ? e : c)); }
        else { setHesaplar(p => p.filter(h => h.id !== editHesapId)); setCards(p => [...p, e]); }
        setEditHesapId(null); sho("Güncellendi");
      } else { setCards(p => [...p, e]); sho("Eklendi"); }
    } else {
      const e = { id: editHesapId || gid(), name: hesapForm.name.trim(), type: hesapForm.type };
      if (editHesapId) {
        if (hesaplar.some(h => h.id === editHesapId)) { setHesaplar(p => p.map(h => h.id === editHesapId ? e : h)); }
        else { setCards(p => p.filter(c => c.id !== editHesapId)); setHesaplar(p => [...p, e]); }
        setEditHesapId(null); sho("Güncellendi");
      } else { setHesaplar(p => [...p, e]); sho("Eklendi"); }
    }
    setHesapForm({ name: "", type: "banka", last4: "", paymentDay: "1" });
  };
  const dHesap = (id) => { setHesaplar(p => p.filter(h => h.id !== id)); setTxns(p => p.map(t => t.hesapId === id ? { ...t, hesapId: "" } : t)); sho("Silindi"); };
  const clearAll = () => { if (confirm("Tüm veriler silinecek?")) { setTxns([]); setPlans([]); setCariler([]); setCards([]); setHesaplar([]); setCustG([]); setCustGi([]); setRemG([]); setRemGi([]); sho("Silindi"); } };

  /* ═══ STYLES ═══ */
  const st = {
    inp: { width: "100%", padding: "13px 14px", borderRadius: 10, border: "1px solid rgba(100,200,255,0.08)", background: "rgba(0,0,0,0.22)", color: "#e0e8f0", fontSize: 15, fontFamily: "inherit", outline: "none", boxSizing: "border-box", minHeight: 46 },
    card: { background: "rgba(255,255,255,0.03)", borderRadius: 14, padding: 16, border: "1px solid rgba(100,200,255,0.04)", marginBottom: 12 },
    btn: (c) => ({ padding: 14, borderRadius: 12, border: "none", background: c || "#38bdf8", color: ["#38bdf8", "#a78bfa", "#f59e0b"].includes(c || "#38bdf8") ? "#0a1520" : "#fff", fontWeight: 700, fontSize: 15, cursor: "pointer", fontFamily: "inherit", width: "100%", minHeight: 48 }),
    tag: (t) => ({ display: "inline-block", padding: "2px 8px", borderRadius: 10, fontSize: 10, fontWeight: 700, background: t === "gelir" ? "rgba(52,211,153,0.1)" : "rgba(251,113,133,0.1)", color: t === "gelir" ? "#34d399" : "#fb7185" }),
    sc: (c) => ({ background: "linear-gradient(135deg," + c + "10," + c + "05)", borderRadius: 12, padding: "11px 12px", border: "1px solid " + c + "18", flex: 1, minWidth: 0 }),
  };

  const Lb = ({ children }) => <label style={{ fontSize: 11, color: "#6b8096", fontWeight: 600, display: "block", marginBottom: 3 }}>{children}</label>;

  const SumBar = () => (
    <div style={{ display: "flex", gap: 6, marginBottom: 12 }}>
      <div style={st.sc("#34d399")}><div style={{ fontSize: 9, color: "#34d399", fontWeight: 700, letterSpacing: 0.5 }}>GELİR</div><div style={{ fontSize: 15, fontWeight: 800, marginTop: 2 }}>{fmt(tot.g)}</div></div>
      <div style={st.sc("#fb7185")}><div style={{ fontSize: 9, color: "#fb7185", fontWeight: 700, letterSpacing: 0.5 }}>GİDER</div><div style={{ fontSize: 15, fontWeight: 800, marginTop: 2 }}>{fmt(tot.gi)}</div></div>
      <div style={st.sc(bal >= 0 ? "#38bdf8" : "#f59e0b")}><div style={{ fontSize: 9, color: bal >= 0 ? "#38bdf8" : "#f59e0b", fontWeight: 700, letterSpacing: 0.5 }}>BAKİYE</div><div style={{ fontSize: 15, fontWeight: 800, marginTop: 2 }}>{fmt(bal)}</div></div>
    </div>
  );

  const CariSel = ({ v, set }) => (
    <div style={{ marginBottom: 10 }}><Lb>Cari / Kişi</Lb>
      <select value={v} onChange={e => set(e.target.value)} style={{ ...st.inp, appearance: "auto" }}>
        <option value="" style={{ background: "#0b1520", color: "#38bdf8" }}>— Yok —</option>
        {cariler.map(c => <option key={c.id} value={c.id} style={{ background: "#0b1520", color: "#38bdf8", fontWeight: 600 }}>{c.name}</option>)}
      </select>
    </div>
  );

  const HesapSel = () => (
    <div style={{ marginBottom: 10 }}><Lb>Hesap</Lb>
      <select value={form.hesapId} onChange={e => setForm(p => ({ ...p, hesapId: e.target.value }))} style={{ ...st.inp, appearance: "auto" }}>
        <option value="" style={{ background: "#0b1520", color: "#38bdf8" }}>— Hesap seçilmedi —</option>
        {hesaplar.filter(h => h.type === "nakit").length > 0 ? <optgroup label="💵 Nakit" style={{ background: "#38bdf8", color: "#0b1520", fontWeight: 700 }}>{hesaplar.filter(h => h.type === "nakit").map(h => <option key={h.id} value={h.id} style={{ background: "#0b1520", color: "#38bdf8", fontWeight: 600 }}>{h.name}</option>)}</optgroup> : null}
        {hesaplar.filter(h => h.type === "banka").length > 0 ? <optgroup label="🏦 Banka" style={{ background: "#38bdf8", color: "#0b1520", fontWeight: 700 }}>{hesaplar.filter(h => h.type === "banka").map(h => <option key={h.id} value={h.id} style={{ background: "#0b1520", color: "#38bdf8", fontWeight: 600 }}>{h.name}</option>)}</optgroup> : null}
        {cards.length > 0 ? <optgroup label="💳 Kredi Kartı" style={{ background: "#38bdf8", color: "#0b1520", fontWeight: 700 }}>{cards.map(c => <option key={c.id} value={c.id} style={{ background: "#0b1520", color: "#38bdf8", fontWeight: 600 }}>{c.name} (*{c.last4})</option>)}</optgroup> : null}
      </select>
      {isCardId(form.hesapId) ? <div style={{ fontSize: 10, color: "#818cf8", marginTop: 6, padding: "6px 10px", background: "rgba(129,140,248,0.05)", borderRadius: 6 }}>💳 Kart ödemesi gününde bakiyeye yansır</div> : null}
    </div>
  );

  const CatSel = ({ cats, v, set, fk }) => {
    const isNew = showNewCat === fk;
    return (
      <div style={{ marginBottom: 10 }}><Lb>Kategori</Lb>
        {isNew ? (
          <div style={{ display: "flex", gap: 6 }}>
            <input placeholder="Yeni kategori" value={newCatVal} onChange={e => setNewCatVal(e.target.value)} style={{ ...st.inp, flex: 1 }} autoFocus />
            <button onClick={() => { const n = newCatVal.trim(); if (!n) return; const tp = fk === "form" ? form.type : planForm.type; if (tp === "gelir") { if (!allG.includes(n)) setCustG(p => [...p, n]); } else { if (!allGi.includes(n)) setCustGi(p => [...p, n]); } set(n); setNewCatVal(""); setShowNewCat(null); }} style={{ padding: "12px 16px", borderRadius: 10, border: "none", background: "#38bdf8", color: "#0a1520", fontWeight: 700, fontSize: 13, cursor: "pointer", fontFamily: "inherit" }}>Ekle</button>
            <button onClick={() => { setShowNewCat(null); setNewCatVal(""); }} style={{ padding: "12px", borderRadius: 10, border: "1px solid rgba(100,200,255,0.08)", background: "transparent", color: "#526478", fontSize: 13, cursor: "pointer", fontFamily: "inherit" }}>✕</button>
          </div>
        ) : (
          <select value={v} onChange={e => { if (e.target.value === "__new__") { setShowNewCat(fk); setNewCatVal(""); } else set(e.target.value); }} style={{ ...st.inp, appearance: "auto" }}>
            {cats.map(c => <option key={c} value={c} style={{ background: "#0b1520", color: "#38bdf8", fontWeight: 600 }}>{c}</option>)}
            <option value="__new__" style={{ background: "#0b1520", color: "#38bdf8", fontWeight: 700 }}>+ Yeni Kategori</option>
          </select>
        )}
      </div>
    );
  };

  const MasrafBox = ({ f, setF }) => {
    if (f.type !== "gelir") return null;
    return (
      <div style={{ marginBottom: 10 }}>
        <label onClick={() => setF(p => ({ ...p, hasMasraf: !p.hasMasraf, masraf: "", masrafCat: "Diğer Gider" }))} style={{ display: "flex", alignItems: "center", gap: 8, cursor: "pointer", fontSize: 13, color: f.hasMasraf ? "#fb7185" : "#526478", fontWeight: 600, marginBottom: f.hasMasraf ? 10 : 0, padding: "4px 0" }}>
          <div style={{ width: 20, height: 20, borderRadius: 5, border: "2px solid " + (f.hasMasraf ? "#fb7185" : "rgba(100,200,255,0.12)"), background: f.hasMasraf ? "rgba(251,113,133,0.12)" : "transparent", display: "flex", alignItems: "center", justifyContent: "center", fontSize: 12, flexShrink: 0 }}>{f.hasMasraf ? "✓" : ""}</div>
          Bu gelirin masrafı var
        </label>
        {f.hasMasraf && (
          <div style={{ background: "rgba(251,113,133,0.04)", borderRadius: 12, padding: 14, border: "1px solid rgba(251,113,133,0.06)" }}>
            <div style={{ marginBottom: 8 }}><Lb>Masraf (₺)</Lb><input type="number" placeholder="0" value={f.masraf} onChange={e => setF(p => ({ ...p, masraf: e.target.value }))} style={st.inp} /></div>
            <div><Lb>Masraf Kategorisi</Lb><select value={f.masrafCat} onChange={e => setF(p => ({ ...p, masrafCat: e.target.value }))} style={{ ...st.inp, appearance: "auto" }}>{allGi.map(c => <option key={c} value={c}>{c}</option>)}</select></div>
            {f.masraf && +f.masraf > 0 && f.amount && +f.amount > 0 && <div style={{ marginTop: 10, fontSize: 12, color: "#6b8096" }}>Net: <strong style={{ color: "#34d399" }}>{fmt2(+f.amount - +f.masraf)}</strong></div>}
          </div>
        )}
      </div>
    );
  };

  const TxnRow = ({ t, onE, onD, sc }) => (
    <div style={{ ...st.card, display: "flex", alignItems: "center", gap: 10, padding: "12px 14px" }}>
      <div style={{ width: 38, height: 38, borderRadius: 10, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 16, background: t.type === "virman" ? "rgba(56,189,248,0.08)" : t.type === "gelir" ? "rgba(52,211,153,0.08)" : "rgba(251,113,133,0.08)", flexShrink: 0 }}>{t.type === "virman" ? "⇄" : t.type === "gelir" ? "↑" : "↓"}</div>
      <div style={{ flex: 1, minWidth: 0 }}>
        <div style={{ fontSize: 13, fontWeight: 700, whiteSpace: "nowrap", overflow: "hidden", textOverflow: "ellipsis" }}>{t.desc || t.category}</div>
        <div style={{ fontSize: 10, color: "#526478", marginTop: 2 }}>
          <span style={{ ...st.tag(t.type), ...(t.type === "virman" ? { background: "rgba(56,189,248,0.1)", color: "#38bdf8" } : {}) }}>{t.category}</span>
          <span style={{ marginLeft: 6 }}>{t.date ? fmtD(t.date) : ""}</span>
          {t.cariId ? <span style={{ marginLeft: 4, color: "#f59e0b" }}>·{cn(t.cariId)}</span> : null}
          {t.type === "virman" && t.hesapIdFrom ? <span style={{ marginLeft: 4, color: "#38bdf8", fontSize: 9 }}>{hn(t.hesapIdFrom)} → {hn(t.hesapId)}</span> : sc && (t.hesapId || t.cardId) ? <span style={{ marginLeft: 4, color: isCardId(t.hesapId || t.cardId) ? "#818cf8" : "#38bdf8", fontSize: 9 }}>{hn(t.hesapId || t.cardId)}</span> : null}
          {t.vadeli ? <span style={{ marginLeft: 4, color: "#f59e0b" }}>⏳</span> : null}
        </div>
      </div>
      <div style={{ textAlign: "right", flexShrink: 0 }}>
        <div style={{ fontSize: 14, fontWeight: 800, color: t.type === "virman" ? "#38bdf8" : t.type === "gelir" ? "#34d399" : "#fb7185" }}>{t.type === "virman" ? "" : t.type === "gelir" ? "+" : "-"}{fmt(t.amount)}</div>
        {t.masraf > 0 ? <div style={{ fontSize: 9, color: "#fb7185" }}>-{fmt(t.masraf)} {t.masrafCat}</div> : null}
        {(onE || onD) ? <div style={{ display: "flex", gap: 10, marginTop: 4, justifyContent: "flex-end" }}>
          {onE ? <button onClick={() => onE(t)} style={{ background: "none", border: "none", color: "#38bdf8", fontSize: 11, cursor: "pointer", padding: "2px 0", fontFamily: "inherit" }}>Düzenle</button> : null}
          {onD ? <button onClick={() => onD(t.id)} style={{ background: "none", border: "none", color: "#fb7185", fontSize: 11, cursor: "pointer", padding: "2px 0", fontFamily: "inherit" }}>Sil</button> : null}
        </div> : null}
      </div>
    </div>
  );

  const P = 14;
  const fCats = form.type === "gelir" ? allG : allGi;
  const pFCats = planForm.type === "gelir" ? allG : allGi;

  /* ═══ RENDER ═══ */
  return (
    <div style={{ fontFamily: "'Nunito',system-ui,sans-serif", maxWidth: 430, margin: "0 auto", minHeight: "100dvh", background: "linear-gradient(165deg,#0b1520,#142233 50%,#0b1520)", color: "#e0e8f0", paddingBottom: 62, position: "relative", overflowX: "hidden" }}>
      <link href="https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;700;800&display=swap" rel="stylesheet" />
      {toast ? <div style={{ position: "fixed", bottom: 72, left: "50%", transform: "translateX(-50%)", background: "#38bdf8", color: "#0a1520", padding: "9px 22px", borderRadius: 22, fontWeight: 700, fontSize: 13, zIndex: 999, whiteSpace: "nowrap", boxShadow: "0 4px 20px rgba(56,189,248,0.2)" }}>{toast}</div> : null}

      <div style={{ position: "sticky", top: 0, zIndex: 50, background: "rgba(11,21,32,0.96)", backdropFilter: "blur(14px)", padding: "12px 14px 8px", display: "flex", justifyContent: "center", alignItems: "center" }}>
        <div style={{ fontSize: 20, fontWeight: 800, letterSpacing: -0.3 }}><span style={{ color: "#38bdf8" }}>₺</span> Hesap Takibi</div>
        <button onClick={() => setPage("ayarlar")} style={{ position: "absolute", right: 14, background: "none", border: "none", color: page === "ayarlar" ? "#38bdf8" : "#526478", fontSize: 20, cursor: "pointer", padding: 4 }}>⚙️</button>
      </div>

      {/* ═══ EKLE ═══ */}
      {page === "ekle" ? <div style={{ padding: "0 " + P + "px" }}>
        <SumBar />
        <div style={st.card}>
          <div style={{ fontSize: 15, fontWeight: 700, marginBottom: 14 }}>{editId ? "İşlemi Düzenle" : realizingPlan ? "Plan → İşlem Kaydet" : "Yeni İşlem"}</div>
          {realizingPlan ? <div style={{ fontSize: 11, color: "#a78bfa", marginBottom: 12, padding: "8px 12px", background: "rgba(167,139,250,0.06)", borderRadius: 8, display: "flex", justifyContent: "space-between", alignItems: "center" }}>
            <span>{realizingPlan.partial ? "Kısmi ödeme — tutarı girin" : "Plan gerçekleştiriliyor"}</span>
            <button onClick={() => { setRealizingPlan(null); setForm({ ...emptyForm }); }} style={{ background: "none", border: "none", color: "#fb7185", fontSize: 11, cursor: "pointer", fontFamily: "inherit" }}>İptal</button>
          </div> : null}
          <div style={{ display: "flex", gap: 6, marginBottom: 14 }}>
            {[["gelir", "↑ Gelir", "#34d399", "rgba(52,211,153,0.15)"], ["gider", "↓ Gider", "#fb7185", "rgba(251,113,133,0.15)"], ["virman", "⇄ Virman", "#38bdf8", "rgba(56,189,248,0.15)"]].map(([t, l, c, bg]) => <button key={t} onClick={() => setForm(p => ({ ...p, type: t, category: t === "gelir" ? DEF_G[0] : t === "gider" ? DEF_Gi[0] : "Virman", hesapId: "", hesapIdFrom: "", cariId: "", hasMasraf: false, vadeli: false }))} style={{ flex: 1, padding: 10, borderRadius: 10, border: form.type === t ? "none" : "1px solid rgba(100,200,255,0.06)", background: form.type === t ? bg : "transparent", color: form.type === t ? c : "#526478", fontWeight: 700, fontSize: 13, cursor: "pointer", fontFamily: "inherit" }}>{l}</button>)}
          </div>
          <div style={{ marginBottom: 10 }}><Lb>Tutar (₺)</Lb><input type="number" inputMode="decimal" placeholder="0" value={form.amount} onChange={e => setForm(p => ({ ...p, amount: e.target.value }))} style={{ ...st.inp, fontSize: 18, fontWeight: 700 }} /></div>
          {form.type === "virman" ? <>
            <div style={{ marginBottom: 10 }}><Lb>Kaynak Hesap</Lb>
              <select value={form.hesapIdFrom} onChange={e => setForm(p => ({ ...p, hesapIdFrom: e.target.value }))} style={{ ...st.inp, appearance: "auto" }}>
                <option value="" style={{ background: "#0b1520", color: "#38bdf8" }}>— Nereden —</option>
                {hesaplar.filter(h => h.type === "nakit").length > 0 ? <optgroup label="💵 Nakit" style={{ background: "#38bdf8", color: "#0b1520", fontWeight: 700 }}>{hesaplar.filter(h => h.type === "nakit").map(h => <option key={h.id} value={h.id} style={{ background: "#0b1520", color: "#38bdf8", fontWeight: 600 }}>{h.name}</option>)}</optgroup> : null}
                {hesaplar.filter(h => h.type === "banka").length > 0 ? <optgroup label="🏦 Banka" style={{ background: "#38bdf8", color: "#0b1520", fontWeight: 700 }}>{hesaplar.filter(h => h.type === "banka").map(h => <option key={h.id} value={h.id} style={{ background: "#0b1520", color: "#38bdf8", fontWeight: 600 }}>{h.name}</option>)}</optgroup> : null}
              </select>
            </div>
            <div style={{ textAlign: "center", color: "#38bdf8", fontSize: 16, margin: "4px 0" }}>↓</div>
            <div style={{ marginBottom: 10 }}><Lb>Hedef Hesap</Lb>
              <select value={form.hesapId} onChange={e => setForm(p => ({ ...p, hesapId: e.target.value }))} style={{ ...st.inp, appearance: "auto" }}>
                <option value="" style={{ background: "#0b1520", color: "#38bdf8" }}>— Nereye —</option>
                {hesaplar.filter(h => h.type === "nakit").length > 0 ? <optgroup label="💵 Nakit" style={{ background: "#38bdf8", color: "#0b1520", fontWeight: 700 }}>{hesaplar.filter(h => h.type === "nakit").map(h => <option key={h.id} value={h.id} style={{ background: "#0b1520", color: "#38bdf8", fontWeight: 600 }}>{h.name}</option>)}</optgroup> : null}
                {hesaplar.filter(h => h.type === "banka").length > 0 ? <optgroup label="🏦 Banka" style={{ background: "#38bdf8", color: "#0b1520", fontWeight: 700 }}>{hesaplar.filter(h => h.type === "banka").map(h => <option key={h.id} value={h.id} style={{ background: "#0b1520", color: "#38bdf8", fontWeight: 600 }}>{h.name}</option>)}</optgroup> : null}
              </select>
            </div>
          </> : <>
            <MasrafBox f={form} setF={setForm} />
            <CatSel cats={fCats} v={form.category} set={v => setForm(p => ({ ...p, category: v }))} fk="form" />
            <CariSel v={form.cariId} set={v => setForm(p => ({ ...p, cariId: v }))} />
            <HesapSel />
          </>}
          <div style={{ marginBottom: 10 }}><Lb>Açıklama</Lb><input placeholder="Opsiyonel" value={form.desc} onChange={e => setForm(p => ({ ...p, desc: e.target.value }))} style={st.inp} /></div>
          <div style={{ marginBottom: 14 }}>
            <Lb>Tarih</Lb><input type="date" max={td()} value={form.date} onChange={e => setForm(p => ({ ...p, date: e.target.value }))} style={st.inp} />
          </div>
          {form.type !== "virman" && form.cariId && form.category !== "Borç" ? <div style={{ marginBottom: 14 }}>
            <label onClick={() => setForm(p => ({ ...p, vadeli: !p.vadeli }))} style={{ display: "flex", alignItems: "center", gap: 8, cursor: "pointer", fontSize: 13, color: form.vadeli ? "#f59e0b" : "#526478", fontWeight: 600, padding: "6px 0" }}>
              <div style={{ width: 20, height: 20, borderRadius: 5, border: "2px solid " + (form.vadeli ? "#f59e0b" : "rgba(100,200,255,0.12)"), background: form.vadeli ? "rgba(245,158,11,0.12)" : "transparent", display: "flex", alignItems: "center", justifyContent: "center", fontSize: 12, flexShrink: 0 }}>{form.vadeli ? "✓" : ""}</div>
              ⏳ Vadeli işlem (bakiyeye yansımaz, cariye yazılır)
            </label>
          </div> : null}
          <button onClick={subTxn} style={st.btn()}>{editId ? "Güncelle" : "Kaydet"}</button>
          {editId ? <button onClick={() => { setEditId(null); setForm({ ...emptyForm }); }} style={{ ...st.btn("transparent"), color: "#526478", border: "1px solid rgba(100,200,255,0.06)", marginTop: 8 }}>İptal</button> : null}
        </div>
      </div> : null}

      {/* ═══ ÖZET ═══ */}
      {page === "dashboard" ? <div style={{ padding: "0 " + P + "px" }}>
        <SumBar />
        {ccDebt > 0 ? <div style={{ ...st.sc("#818cf8"), marginBottom: 12 }}><div style={{ fontSize: 9, color: "#818cf8", fontWeight: 700, letterSpacing: 0.5 }}>💳 KK BORCU</div><div style={{ fontSize: 20, fontWeight: 800, marginTop: 2 }}>{fmt(ccDebt)}</div><div style={{ fontSize: 10, color: "#526478", marginTop: 4 }}>{cards.map(c => { const b = ccBal[c.id] || 0; return b > 0 ? <div key={c.id}>{c.name}: {fmt(b)}</div> : null; })}</div></div> : null}

        {vadeliT.length > 0 ? (() => { const vG = vadeliT.filter(t => t.type === "gelir").reduce((s, t) => s + t.amount, 0); const vGi = vadeliT.filter(t => t.type === "gider").reduce((s, t) => s + t.amount, 0); return <div style={{ ...st.sc("#f59e0b"), marginBottom: 12 }}><div style={{ fontSize: 9, color: "#f59e0b", fontWeight: 700, letterSpacing: 0.5 }}>⏳ VADELİ İŞLEMLER</div><div style={{ fontSize: 10, color: "#526478", marginTop: 4 }}>{vG > 0 ? <span style={{ color: "#34d399" }}>Alacak: {fmt(vG)} </span> : null}{vGi > 0 ? <span style={{ color: "#fb7185" }}>Borç: {fmt(vGi)}</span> : null}</div></div>; })() : null}

        {(() => {
          const activeCari = cariler.filter(c => Math.round((cariBals[c.id] || 0) * 100) !== 0);
          if (activeCari.length === 0) return null;
          let cariNet = 0;
          activeCari.forEach(c => { cariNet += cariBals[c.id] || 0; });
          return <div style={st.card}>
              <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 10 }}>
                <div style={{ fontSize: 13, fontWeight: 700, color: "#8a9db0" }}>Cari Durum</div>
                <div style={{ fontSize: 12, fontWeight: 700, color: cariNet >= 0 ? "#34d399" : "#fb7185" }}>Denge: {cariNet >= 0 ? "+" : ""}{fmt(cariNet)}</div>
              </div>
              {activeCari.map(c => { const b = cariBals[c.id] || 0; return <div key={c.id} style={{ display: "flex", justifyContent: "space-between", alignItems: "center", padding: "7px 0", borderBottom: "1px solid rgba(100,200,255,0.03)" }}><div><div style={{ fontSize: 13, fontWeight: 700 }}>{c.name}</div><div style={{ fontSize: 10, color: "#526478" }}>{b > 0 ? "Size borçlu" : "Borçlusunuz"}</div></div><div style={{ fontSize: 13, fontWeight: 800, color: b > 0 ? "#34d399" : "#fb7185" }}>{fmt(Math.abs(b))}</div></div>; })}
            </div>;
        })()}


        {txns.length === 0 ? <div style={{ textAlign: "center", padding: "36px 0", color: "#4a6275" }}><div style={{ fontSize: 36, marginBottom: 8 }}>📊</div><div style={{ fontSize: 14, fontWeight: 600 }}>Henüz işlem yok</div></div> : null}

        {txns.length > 0 ? <>
          <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 8 }}>
            <div style={{ fontSize: 14, fontWeight: 700, color: "#8a9db0" }}>İşlemler</div>
            <button onClick={() => { setShowSearch(p => !p); if (!showSearch) setFilter(p => ({ ...p, search: "" })); }} style={{ background: showSearch ? "rgba(56,189,248,0.1)" : "rgba(255,255,255,0.04)", border: "none", color: showSearch ? "#38bdf8" : "#526478", width: 34, height: 34, borderRadius: 8, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 15, cursor: "pointer" }}>🔍</button>
          </div>
          {showSearch ? <input placeholder="Ara..." value={filter.search} onChange={e => setFilter(p => ({ ...p, search: e.target.value }))} style={{ ...st.inp, marginBottom: 8, fontSize: 14 }} autoFocus /> : null}
          <div style={{ display: "flex", gap: 5, marginBottom: 6, alignItems: "center" }}>
            <div style={{ display: "flex", borderRadius: 8, overflow: "hidden", border: "1px solid rgba(100,200,255,0.06)", flex: 1 }}>
              {[["all", "Tümü", "#8a9db0"], ["gelir", "Gelir", "#34d399"], ["gider", "Gider", "#fb7185"]].map(([v, l, c]) => <button key={v} onClick={() => setFilter(p => ({ ...p, type: v }))} style={{ flex: 1, padding: "9px 2px", border: "none", background: filter.type === v ? (v === "gelir" ? "rgba(52,211,153,0.15)" : v === "gider" ? "rgba(251,113,133,0.15)" : "rgba(56,189,248,0.1)") : "rgba(0,0,0,0.15)", color: filter.type === v ? c : "#4a6275", fontWeight: filter.type === v ? 700 : 500, fontSize: 11, cursor: "pointer", fontFamily: "inherit" }}>{l}</button>)}
            </div>
            <select value={filter.category} onChange={e => setFilter(p => ({ ...p, category: e.target.value }))} style={{ ...st.inp, flex: 1, padding: "9px 6px", fontSize: 11, minHeight: 36 }}><option value="all" style={{ background: "#0b1520", color: "#38bdf8" }}>Kategoriler</option>{usedCats.map(c => <option key={c} value={c} style={{ background: "#0b1520", color: "#38bdf8", fontWeight: 600 }}>{c}</option>)}</select>
          </div>
          <div style={{ display: "flex", gap: 5, marginBottom: 10, alignItems: "center" }}>
            <div style={{ display: "flex", borderRadius: 6, overflow: "hidden", border: "1px solid rgba(100,200,255,0.06)", flexShrink: 0 }}>
              {[["ay", "Ay"], ["gun", "Gün"]].map(([v, l]) => <button key={v} onClick={() => setFilter(p => ({ ...p, dateType: v, date: "" }))} style={{ padding: "7px 10px", border: "none", background: filter.dateType === v ? "rgba(56,189,248,0.1)" : "rgba(0,0,0,0.15)", color: filter.dateType === v ? "#38bdf8" : "#4a6275", fontWeight: filter.dateType === v ? 700 : 500, fontSize: 10, cursor: "pointer", fontFamily: "inherit" }}>{l}</button>)}
            </div>
            <input type={filter.dateType === "gun" ? "date" : "month"} value={filter.date} onChange={e => setFilter(p => ({ ...p, date: e.target.value }))} style={{ ...st.inp, flex: 1, padding: "7px 6px", fontSize: 11, minHeight: 34 }} />
          </div>
          {/* Denge */}
          {(() => {
            const fG = filtered.filter(t => t.type === "gelir").reduce((s, t) => s + netA(t), 0);
            const fGi = filtered.filter(t => t.type === "gider").reduce((s, t) => s + t.amount, 0);
            const fBal = fG - fGi;
            return <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 8, padding: "6px 10px", borderRadius: 8, background: "rgba(255,255,255,0.02)", border: "1px solid rgba(100,200,255,0.04)" }}>
              <div style={{ display: "flex", gap: 10, fontSize: 11 }}>
                <span style={{ color: "#34d399" }}>↑{fmt(fG)}</span>
                <span style={{ color: "#fb7185" }}>↓{fmt(fGi)}</span>
              </div>
              <span style={{ fontSize: 12, fontWeight: 800, color: fBal >= 0 ? "#34d399" : "#fb7185" }}>Denge: {fBal >= 0 ? "+" : ""}{fmt(fBal)}</span>
            </div>;
          })()}
          {(filter.search || filter.type !== "all" || filter.category !== "all" || filter.date) ? <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 8 }}><span style={{ fontSize: 11, color: "#526478" }}>{filtered.length} sonuç</span><button onClick={() => setFilter({ type: "all", date: "", search: "", category: "all", dateType: "ay" })} style={{ background: "none", border: "none", color: "#38bdf8", fontSize: 11, cursor: "pointer", padding: 0, fontFamily: "inherit" }}>Temizle</button></div> : null}
          {filtered.length === 0 ? <div style={{ textAlign: "center", padding: "20px 0", color: "#4a6275", fontSize: 12 }}>Sonuç yok</div> : null}
          {filtered.map(t => <TxnRow key={t.id} t={t} onE={eTxn} onD={dTxn} sc />)}
        </> : null}
      </div> : null}

      {/* ═══ PLAN ═══ */}
      {page === "plan" ? <div style={{ padding: "0 " + P + "px" }}>
        <div style={{ display: "flex", gap: 6, marginBottom: 6 }}>
          <div style={st.sc("#a78bfa")}><div style={{ fontSize: 9, color: "#a78bfa", fontWeight: 700 }}>PLAN GELİR</div><div style={{ fontSize: 15, fontWeight: 800, marginTop: 2 }}>{fmt(pTot.g)}</div></div>
          <div style={st.sc("#f472b6")}><div style={{ fontSize: 9, color: "#f472b6", fontWeight: 700 }}>PLAN GİDER</div><div style={{ fontSize: 15, fontWeight: 800, marginTop: 2 }}>{fmt(pTot.gi)}</div></div>
          <div style={st.sc("#c084fc")}><div style={{ fontSize: 9, color: "#c084fc", fontWeight: 700 }}>TAHMİNİ</div><div style={{ fontSize: 15, fontWeight: 800, marginTop: 2 }}>{fmt((includeBal ? bal : 0) + pBal - ccDebt)}</div></div>
        </div>
        <div style={{ ...st.card, display: "flex", justifyContent: "space-between", alignItems: "center", padding: "10px 14px", marginBottom: 12 }}>
          <div><div style={{ fontSize: 9, color: "#38bdf8", fontWeight: 700, letterSpacing: 0.5 }}>BUGÜNKÜ BAKİYE</div><div style={{ fontSize: 16, fontWeight: 800, marginTop: 2, color: bal >= 0 ? "#34d399" : "#fb7185" }}>{fmt(bal)}</div></div>
          <label onClick={() => setIncludeBal(p => !p)} style={{ display: "flex", alignItems: "center", gap: 6, cursor: "pointer", fontSize: 11, color: includeBal ? "#38bdf8" : "#526478", fontWeight: 600 }}>
            <div style={{ width: 20, height: 20, borderRadius: 5, border: "2px solid " + (includeBal ? "#38bdf8" : "rgba(100,200,255,0.12)"), background: includeBal ? "rgba(56,189,248,0.12)" : "transparent", display: "flex", alignItems: "center", justifyContent: "center", fontSize: 12, flexShrink: 0 }}>{includeBal ? "✓" : ""}</div>
            Tahmine ekle
          </label>
        </div>

        {/* Takvim */}
        <div style={st.card}>
          <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 10 }}>
            <button onClick={() => { setCalMonth(p => p.month === 0 ? { year: p.year - 1, month: 11 } : { ...p, month: p.month - 1 }); setSelectedDay(null); }} style={{ background: "none", border: "none", color: "#38bdf8", fontSize: 20, cursor: "pointer", padding: "4px 10px" }}>‹</button>
            <div style={{ fontSize: 14, fontWeight: 700, color: "#8a9db0", textTransform: "capitalize" }}>{cLabel}</div>
            <button onClick={() => { setCalMonth(p => p.month === 11 ? { year: p.year + 1, month: 0 } : { ...p, month: p.month + 1 }); setSelectedDay(null); }} style={{ background: "none", border: "none", color: "#38bdf8", fontSize: 20, cursor: "pointer", padding: "4px 10px" }}>›</button>
          </div>
          <div style={{ display: "grid", gridTemplateColumns: "repeat(7,1fr)", gap: 3, marginBottom: 4 }}>
            {["Pt", "Sa", "Ça", "Pe", "Cu", "Ct", "Pz"].map(d => <div key={d} style={{ textAlign: "center", fontSize: 10, color: "#4a6275", fontWeight: 700, padding: "3px 0" }}>{d}</div>)}
          </div>
          <div style={{ display: "grid", gridTemplateColumns: "repeat(7,1fr)", gap: 3 }}>
            {Array.from({ length: cStart }).map((_, i) => <div key={"e" + i} />)}
            {Array.from({ length: cDays }).map((_, i) => {
              const day = i + 1;
              const ds = calMonth.year + "-" + String(calMonth.month + 1).padStart(2, "0") + "-" + String(day).padStart(2, "0");
              const b = balAt(ds);
              const dt = txAt(ds);
              const isT = ds === td();
              const isSel = ds === selectedDay;
              const bg = isSel ? "rgba(56,189,248,0.2)" : b > 0 ? "rgba(52,211,153,0.08)" : b < 0 ? "rgba(251,113,133,0.08)" : "rgba(255,255,255,0.01)";
              const bc = b > 0 ? "#34d399" : b < 0 ? "#fb7185" : "#3a5060";
              return (
                <div key={day} onClick={() => setSelectedDay(isSel ? null : ds)} style={{ background: bg, borderRadius: 8, padding: "4px 2px", minHeight: 48, display: "flex", flexDirection: "column", alignItems: "center", border: isSel ? "1.5px solid #38bdf8" : isT ? "1.5px solid rgba(56,189,248,0.4)" : "1.5px solid transparent", cursor: "pointer" }}>
                  <div style={{ fontSize: 11, fontWeight: isT || isSel ? 800 : 600, color: isSel ? "#38bdf8" : isT ? "#38bdf8" : "#526478" }}>{day}</div>
                  {dt.length > 0 ? <div style={{ display: "flex", gap: 2, marginTop: 2, flexWrap: "wrap", justifyContent: "center" }}>{dt.slice(0, 3).map((t, j) => <div key={j} style={{ width: 4, height: 4, borderRadius: "50%", background: t.type === "gelir" ? "#34d399" : "#fb7185" }} />)}</div> : null}
                  <div style={{ fontSize: 7, fontWeight: 700, color: bc, marginTop: "auto", whiteSpace: "nowrap" }}>{b !== 0 ? fmtC(b) : ""}</div>
                </div>
              );
            })}
          </div>
        </div>

        {/* Seçilen günün planları */}
        {selectedDay ? (() => {
          const dayPlans = [...plans, ...ccPlans].filter(t => t.date === selectedDay);
          return <div style={st.card}>
            <div style={{ fontSize: 13, fontWeight: 700, marginBottom: 10, color: "#38bdf8" }}>📅 {fmtD(selectedDay)} Planları</div>
            {dayPlans.length === 0 ? <div style={{ fontSize: 12, color: "#4a6275", padding: "8px 0" }}>Bu güne ait plan yok</div> : null}
            {dayPlans.map(t => <div key={t.id} style={{ display: "flex", alignItems: "center", gap: 8, padding: "8px 0", borderBottom: "1px solid rgba(100,200,255,0.03)" }}>
              <div style={{ width: 28, height: 28, borderRadius: 7, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 12, background: t.type === "gelir" ? "rgba(167,139,250,0.1)" : "rgba(244,114,182,0.1)", flexShrink: 0 }}>{t.type === "gelir" ? "↑" : "↓"}</div>
              <div style={{ flex: 1, minWidth: 0 }}>
                <div style={{ fontSize: 12, fontWeight: 700 }}>{t.desc || t.category}</div>
                <div style={{ fontSize: 10, color: "#526478" }}>{t.category}{t.cariId ? " · " + cn(t.cariId) : ""}</div>
              </div>
              <div style={{ fontSize: 13, fontWeight: 800, color: t.type === "gelir" ? "#a78bfa" : "#f472b6" }}>{t.type === "gelir" ? "+" : "-"}{fmt(t.amount)}</div>
            </div>)}
          </div>;
        })() : null}


        <div style={st.card}>
          <div style={{ fontSize: 15, fontWeight: 700, marginBottom: 14 }}>{editPlanId ? "Planı Düzenle" : "Yeni Plan"}</div>
          <div style={{ display: "flex", gap: 6, marginBottom: 14 }}>
            {["gelir", "gider"].map(t => <button key={t} onClick={() => setPlanForm(p => ({ ...p, type: t, category: t === "gelir" ? DEF_G[0] : DEF_Gi[0] }))} style={{ flex: 1, padding: 10, borderRadius: 10, border: planForm.type === t ? "none" : "1px solid rgba(100,200,255,0.06)", background: planForm.type === t ? (t === "gelir" ? "rgba(167,139,250,0.15)" : "rgba(244,114,182,0.15)") : "transparent", color: planForm.type === t ? (t === "gelir" ? "#a78bfa" : "#f472b6") : "#526478", fontWeight: 700, fontSize: 14, cursor: "pointer", fontFamily: "inherit" }}>{t === "gelir" ? "↑ Gelir" : "↓ Gider"}</button>)}
          </div>
          <div style={{ marginBottom: 10 }}><Lb>Tutar (₺)</Lb><input type="number" inputMode="decimal" placeholder="0" value={planForm.amount} onChange={e => setPlanForm(p => ({ ...p, amount: e.target.value }))} style={{ ...st.inp, fontSize: 18, fontWeight: 700 }} /></div>
          <MasrafBox f={planForm} setF={setPlanForm} />
          <CatSel cats={pFCats} v={planForm.category} set={v => setPlanForm(p => ({ ...p, category: v }))} fk="planForm" />
          <CariSel v={planForm.cariId} set={v => setPlanForm(p => ({ ...p, cariId: v }))} />
          <div style={{ marginBottom: 10 }}><Lb>Açıklama</Lb><input placeholder="Opsiyonel" value={planForm.desc} onChange={e => setPlanForm(p => ({ ...p, desc: e.target.value }))} style={st.inp} /></div>
          <div style={{ marginBottom: 14 }}><Lb>Tarih</Lb><input type="date" value={planForm.date} onChange={e => setPlanForm(p => ({ ...p, date: e.target.value }))} style={st.inp} /></div>
          <div style={{ marginBottom: 14 }}>
            <label onClick={() => setPlanForm(p => ({ ...p, recurring: !p.recurring }))} style={{ display: "flex", alignItems: "center", gap: 8, cursor: "pointer", fontSize: 13, color: planForm.recurring ? "#a78bfa" : "#526478", fontWeight: 600, padding: "4px 0" }}>
              <div style={{ width: 20, height: 20, borderRadius: 5, border: "2px solid " + (planForm.recurring ? "#a78bfa" : "rgba(100,200,255,0.12)"), background: planForm.recurring ? "rgba(167,139,250,0.12)" : "transparent", display: "flex", alignItems: "center", justifyContent: "center", fontSize: 12, flexShrink: 0 }}>{planForm.recurring ? "✓" : ""}</div>
              Tekrarlanan işlem
            </label>
            {planForm.recurring ? <select value={planForm.recurFreq} onChange={e => setPlanForm(p => ({ ...p, recurFreq: e.target.value }))} style={{ ...st.inp, appearance: "auto", marginTop: 8 }}>
              <option value="gunluk" style={{ background: "#0b1520", color: "#38bdf8" }}>Günlük</option>
              <option value="haftalik" style={{ background: "#0b1520", color: "#38bdf8" }}>Haftalık</option>
              <option value="aylik" style={{ background: "#0b1520", color: "#38bdf8" }}>Aylık</option>
              <option value="yillik" style={{ background: "#0b1520", color: "#38bdf8" }}>Yıllık</option>
            </select> : null}
          </div>
          <button onClick={subPlan} style={{ ...st.btn("#a78bfa"), color: "#fff" }}>{editPlanId ? "Güncelle" : "Plan Ekle"}</button>
          {editPlanId ? <button onClick={() => { setEditPlanId(null); setPlanForm({ ...emptyPlan }); }} style={{ ...st.btn("transparent"), color: "#526478", border: "1px solid rgba(100,200,255,0.06)", marginTop: 8 }}>İptal</button> : null}
        </div>

        {/* Geciken Ödemeler */}
        {(() => {
          const overdue = plans.filter(t => t.date && t.date < td() && (t.status || "odenmedi") !== "odendi");
          if (overdue.length === 0) return null;
          return <div style={{ ...st.card, border: "1px solid rgba(251,113,133,0.15)", background: "rgba(251,113,133,0.03)" }}>
            <div style={{ fontSize: 13, fontWeight: 700, marginBottom: 10, color: "#fb7185" }}>⚠ Geciken Ödemeler</div>
            {overdue.sort((a, b) => a.date.localeCompare(b.date)).map(t => {
              const isGelir = t.type === "gelir";
              const fullLabel = isGelir ? "Geldi" : "Ödendi";
              return <div key={t.id} style={{ padding: "8px 0", borderBottom: "1px solid rgba(251,113,133,0.06)" }}>
                <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
                  <div style={{ color: "#fb7185", fontSize: 16, flexShrink: 0 }}>⚠</div>
                  <div style={{ flex: 1, minWidth: 0 }}>
                    <div style={{ fontSize: 12, fontWeight: 700 }}>{t.desc || t.category}</div>
                    <div style={{ fontSize: 10, color: "#526478" }}>{fmtD(t.date)} · {t.category}{t.status === "kismen" ? " · Kalan" : ""}</div>
                  </div>
                  <div style={{ fontSize: 13, fontWeight: 800, color: isGelir ? "#a78bfa" : "#f472b6" }}>{isGelir ? "+" : "-"}{fmt(t.amount)}</div>
                </div>
                <div style={{ display: "flex", gap: 4, marginTop: 6 }}>
                  <button onClick={() => realizePlan(t, false)} style={{ flex: 1, padding: "7px 4px", borderRadius: 6, border: "none", background: "rgba(52,211,153,0.15)", color: "#34d399", fontWeight: 700, fontSize: 11, cursor: "pointer", fontFamily: "inherit" }}>{fullLabel}</button>
                  <button onClick={() => realizePlan(t, true)} style={{ flex: 1, padding: "7px 4px", borderRadius: 6, border: "none", background: "rgba(245,158,11,0.15)", color: "#f59e0b", fontWeight: 700, fontSize: 11, cursor: "pointer", fontFamily: "inherit" }}>Kısmen</button>
                </div>
              </div>;
            })}
          </div>;
        })()}

        {(plans.length > 0 || ccPlans.length > 0) ? <div style={st.card}><div style={{ fontSize: 13, fontWeight: 700, marginBottom: 10, color: "#8a9db0" }}>Tüm Planlar</div>
          {ccPlans.map(t => <div key={t.id} style={{ padding: "8px 0", borderBottom: "1px solid rgba(100,200,255,0.03)" }}>
            <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
              <div style={{ width: 30, height: 30, borderRadius: 8, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 13, background: "rgba(129,140,248,0.1)", flexShrink: 0 }}>💳</div>
              <div style={{ flex: 1, minWidth: 0 }}><div style={{ fontSize: 13, fontWeight: 700, whiteSpace: "nowrap", overflow: "hidden", textOverflow: "ellipsis" }}>{t.desc}</div><div style={{ fontSize: 10, color: "#526478" }}>{fmtD(t.date)} · Kart Ödemesi</div></div>
              <div style={{ fontSize: 13, fontWeight: 800, color: "#818cf8" }}>-{fmt(t.amount)}</div>
            </div>
            <div style={{ display: "flex", gap: 4, marginTop: 6 }}>
              <button onClick={() => realizeCC(t, false)} style={{ flex: 1, padding: "6px 4px", borderRadius: 6, border: "none", background: "rgba(52,211,153,0.12)", color: "#34d399", fontWeight: 700, fontSize: 10, cursor: "pointer", fontFamily: "inherit" }}>Ödendi</button>
              <button onClick={() => realizeCC(t, true)} style={{ flex: 1, padding: "6px 4px", borderRadius: 6, border: "none", background: "rgba(245,158,11,0.12)", color: "#f59e0b", fontWeight: 700, fontSize: 10, cursor: "pointer", fontFamily: "inherit" }}>Kısmen</button>
            </div>
          </div>)}
          {[...plans].sort((a, b) => (a.date || "").localeCompare(b.date || "")).map(t => {
          const isGelir = t.type === "gelir";
          const isOverdue = t.date && t.date < td() && (t.status || "odenmedi") !== "odendi";
          const fullLabel = isGelir ? "Geldi" : "Ödendi";
          const freqLabel = t.recurring ? (t.recurFreq === "gunluk" ? "🔄 Günlük" : t.recurFreq === "haftalik" ? "🔄 Haftalık" : t.recurFreq === "yillik" ? "🔄 Yıllık" : "🔄 Aylık") : "";
          return <div key={t.id} style={{ padding: "8px 0", borderBottom: "1px solid rgba(100,200,255,0.03)" }}>
            <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
              {isOverdue ? <div style={{ fontSize: 14, flexShrink: 0, color: "#fb7185" }}>⚠</div> : <div style={{ width: 30, height: 30, borderRadius: 8, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 13, background: isGelir ? "rgba(167,139,250,0.1)" : "rgba(244,114,182,0.1)", flexShrink: 0 }}>{isGelir ? "↑" : "↓"}</div>}
              <div style={{ flex: 1, minWidth: 0 }}>
                <div style={{ fontSize: 13, fontWeight: 700, whiteSpace: "nowrap", overflow: "hidden", textOverflow: "ellipsis" }}>{t.desc || t.category}</div>
                <div style={{ fontSize: 10, color: "#526478" }}>{t.category}{t.date ? " · " + fmtD(t.date) : ""}{t.cariId ? " · " + cn(t.cariId) : ""}{t.status === "kismen" ? " · Kalan" : ""}{freqLabel ? " · " + freqLabel : ""}</div>
              </div>
              <div style={{ textAlign: "right", flexShrink: 0 }}>
                <div style={{ fontSize: 13, fontWeight: 800, color: isGelir ? "#a78bfa" : "#f472b6" }}>{isGelir ? "+" : "-"}{fmt(t.amount)}</div>
              </div>
            </div>
            <div style={{ display: "flex", gap: 4, marginTop: 6 }}>
              <button onClick={() => realizePlan(t, false)} style={{ flex: 1, padding: "6px 4px", borderRadius: 6, border: "none", background: "rgba(52,211,153,0.12)", color: "#34d399", fontWeight: 700, fontSize: 10, cursor: "pointer", fontFamily: "inherit" }}>{fullLabel}</button>
              <button onClick={() => realizePlan(t, true)} style={{ flex: 1, padding: "6px 4px", borderRadius: 6, border: "none", background: "rgba(245,158,11,0.12)", color: "#f59e0b", fontWeight: 700, fontSize: 10, cursor: "pointer", fontFamily: "inherit" }}>Kısmen</button>
              <button onClick={() => ePlan(t)} style={{ padding: "6px 10px", borderRadius: 6, border: "none", background: "rgba(100,200,255,0.06)", color: "#a78bfa", fontWeight: 600, fontSize: 10, cursor: "pointer", fontFamily: "inherit" }}>✎</button>
              <button onClick={() => dPlan(t.id)} style={{ padding: "6px 10px", borderRadius: 6, border: "none", background: "rgba(251,113,133,0.06)", color: "#fb7185", fontWeight: 600, fontSize: 10, cursor: "pointer", fontFamily: "inherit" }}>✕</button>
            </div>
          </div>;
        })}</div> : null}
      </div> : null}

      {/* ═══ KART & HESAP ═══ */}
      {page === "kart" ? <div style={{ padding: "0 " + P + "px" }}>
        {/* Birleşik Hesap/Kart Ekleme */}
        <div style={st.card}>
          <div style={{ fontSize: 15, fontWeight: 700, marginBottom: 14 }}>{editHesapId ? "Düzenle" : "Yeni Hesap"}</div>
          <div style={{ marginBottom: 10 }}><Lb>Hesap Adı *</Lb><input placeholder="Örn: Ziraat Bankası, Cüzdan" value={hesapForm.name} onChange={e => setHesapForm(p => ({ ...p, name: e.target.value }))} style={st.inp} /></div>
          <div style={{ marginBottom: 10 }}><Lb>Hesap Tipi</Lb>
            <div style={{ display: "flex", gap: 6 }}>
              {[["banka", "🏦 Banka"], ["nakit", "💵 Nakit"], ["kredi", "💳 Kredi Kartı"]].map(([v, l]) => <button key={v} onClick={() => setHesapForm(p => ({ ...p, type: v }))} style={{ flex: 1, padding: 10, borderRadius: 8, border: hesapForm.type === v ? "none" : "1px solid rgba(100,200,255,0.06)", background: hesapForm.type === v ? (v === "kredi" ? "rgba(129,140,248,0.15)" : "rgba(56,189,248,0.15)") : "transparent", color: hesapForm.type === v ? (v === "kredi" ? "#818cf8" : "#38bdf8") : "#526478", fontWeight: 700, fontSize: 11, cursor: "pointer", fontFamily: "inherit" }}>{l}</button>)}
            </div>
          </div>
          {hesapForm.type === "kredi" ? <>
            <div style={{ marginBottom: 10 }}><Lb>Son 4 Hane</Lb><input placeholder="1234" inputMode="numeric" maxLength={4} value={hesapForm.last4} onChange={e => setHesapForm(p => ({ ...p, last4: e.target.value.replace(/\D/g, "").slice(0, 4) }))} style={st.inp} /></div>
            <div style={{ marginBottom: 14 }}><Lb>Ödeme Günü (1-31)</Lb><input type="number" inputMode="numeric" min={1} max={31} value={hesapForm.paymentDay} onChange={e => setHesapForm(p => ({ ...p, paymentDay: e.target.value }))} style={st.inp} /></div>
          </> : null}
          <button onClick={subHesap} style={{ ...st.btn(hesapForm.type === "kredi" ? "#818cf8" : "#38bdf8"), color: hesapForm.type === "kredi" ? "#fff" : "#0a1520" }}>{editHesapId ? "Güncelle" : "Ekle"}</button>
          {editHesapId ? <button onClick={() => { setEditHesapId(null); setHesapForm({ name: "", type: "banka", last4: "", paymentDay: "1" }); }} style={{ ...st.btn("transparent"), color: "#526478", border: "1px solid rgba(100,200,255,0.06)", marginTop: 8 }}>İptal</button> : null}
        </div>

        {/* Hesap Listesi */}
        {hesaplar.length > 0 ? (() => {
          let totalHBal = 0;
          const hBals = {};
          hesaplar.forEach(h => {
            let hb = 0;
            txns.forEach(t => {
              if (t.type === "virman") { if (t.hesapId === h.id) hb += t.amount; if (t.hesapIdFrom === h.id) hb -= t.amount; }
              else if ((t.hesapId || t.cardId) === h.id && !t.vadeli) { hb += t.type === "gelir" ? t.amount - (t.masraf || 0) : -t.amount; }
            });
            hBals[h.id] = hb;
            totalHBal += hb;
          });
          return <div style={st.card}>
            <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 10 }}>
              <div style={{ fontSize: 13, fontWeight: 700, color: "#8a9db0" }}>Hesaplar</div>
              <div style={{ fontSize: 12, fontWeight: 700, color: totalHBal >= 0 ? "#34d399" : "#fb7185" }}>Denge: {totalHBal >= 0 ? "+" : ""}{fmt(totalHBal)}</div>
            </div>
            {hesaplar.map(h => {
              const hBal = hBals[h.id];
              return <div key={h.id} style={{ display: "flex", justifyContent: "space-between", alignItems: "center", padding: "8px 0", borderBottom: "1px solid rgba(100,200,255,0.03)" }}>
                <div><div style={{ fontSize: 13, fontWeight: 700 }}>{h.type === "banka" ? "🏦" : "💵"} {h.name}</div><div style={{ fontSize: 10, color: "#526478" }}>{h.type === "banka" ? "Banka Hesabı" : "Nakit"}</div></div>
                <div style={{ display: "flex", alignItems: "center", gap: 12 }}>
                  <div style={{ fontSize: 14, fontWeight: 800, color: hBal >= 0 ? "#34d399" : "#fb7185" }}>{fmt(hBal)}</div>
                  <div style={{ display: "flex", gap: 8 }}>
                    <button onClick={() => { setHesapForm({ name: h.name, type: h.type, last4: "", paymentDay: "1" }); setEditHesapId(h.id); }} style={{ background: "none", border: "none", color: "#38bdf8", fontSize: 11, cursor: "pointer", padding: "2px 0" }}>Düzenle</button>
                    <button onClick={() => dHesap(h.id)} style={{ background: "none", border: "none", color: "#fb7185", fontSize: 11, cursor: "pointer", padding: "2px 0" }}>Sil</button>
                  </div>
                </div>
              </div>;
            })}
          </div>;
        })() : null}

        {/* Kredi Kartları */}
        {cards.length > 0 ? <div style={st.card}>
          <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 10 }}>
            <div style={{ fontSize: 13, fontWeight: 700, color: "#818cf8" }}>💳 Kredi Kartları</div>
            {ccDebt > 0 ? <div style={{ fontSize: 12, fontWeight: 700, color: "#fb7185" }}>Toplam: {fmt(ccDebt)}</div> : null}
          </div>
          {cards.map(c => { const debt = ccBal[c.id] || 0; const ct = ccT.filter(t => (t.hesapId || t.cardId) === c.id).sort((a, b) => b.date.localeCompare(a.date)); return <div key={c.id} style={{ padding: "8px 0", borderBottom: "1px solid rgba(100,200,255,0.03)" }}>
            <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center" }}>
              <div><div style={{ fontSize: 13, fontWeight: 700 }}>{c.name}</div><div style={{ fontSize: 10, color: "#526478" }}>*{c.last4} · Ödeme: {c.paymentDay}. gün</div></div>
              <div style={{ display: "flex", alignItems: "center", gap: 12 }}>
                <div style={{ fontSize: 14, fontWeight: 800, color: debt > 0 ? "#fb7185" : "#34d399" }}>{fmt(debt)}</div>
                <div style={{ display: "flex", gap: 8 }}>
                  <button onClick={() => { setHesapForm({ name: c.name, type: "kredi", last4: c.last4 || "", paymentDay: String(c.paymentDay) }); setEditHesapId(c.id); }} style={{ background: "none", border: "none", color: "#818cf8", fontSize: 11, cursor: "pointer", padding: "2px 0" }}>Düzenle</button>
                  <button onClick={() => dCard(c.id)} style={{ background: "none", border: "none", color: "#fb7185", fontSize: 11, cursor: "pointer", padding: "2px 0" }}>Sil</button>
                </div>
              </div>
            </div>
            {ct.length > 0 ? <div style={{ marginTop: 6, paddingTop: 6, borderTop: "1px solid rgba(100,200,255,0.03)" }}>{ct.map(t => <div key={t.id} style={{ display: "flex", justifyContent: "space-between", padding: "4px 0" }}><div style={{ fontSize: 11 }}>{t.desc || t.category} <span style={{ color: "#526478" }}>· {fmtD(t.date)}</span></div><div style={{ fontSize: 11, fontWeight: 700, color: "#fb7185" }}>{fmt(t.amount)}</div></div>)}</div> : null}
          </div>; })}
        </div> : null}

        {cards.length === 0 && hesaplar.length === 0 ? <div style={{ textAlign: "center", padding: "36px 0", color: "#4a6275" }}><div style={{ fontSize: 36, marginBottom: 8 }}>🏦</div><div style={{ fontSize: 14, fontWeight: 600 }}>Henüz hesap veya kart yok</div></div> : null}
      </div> : null}

      {/* ═══ CARİ ═══ */}
      {page === "cari" && !selectedCari ? <div style={{ padding: "0 " + P + "px" }}>
        <div style={st.card}>
          <div style={{ fontSize: 15, fontWeight: 700, marginBottom: 14 }}>{editCariId ? "Cariyi Düzenle" : "Yeni Cari"}</div>
          <div style={{ marginBottom: 10 }}><Lb>İsim *</Lb><input placeholder="Firma / kişi" value={cariForm.name} onChange={e => setCariForm(p => ({ ...p, name: e.target.value }))} style={st.inp} /></div>
          <div style={{ marginBottom: 14 }}><Lb>Not</Lb><input placeholder="Opsiyonel" value={cariForm.note} onChange={e => setCariForm(p => ({ ...p, note: e.target.value }))} style={st.inp} /></div>
          <button onClick={subCari} style={{ ...st.btn("#f59e0b"), color: "#0a1520" }}>{editCariId ? "Güncelle" : "Cari Ekle"}</button>
          {editCariId ? <button onClick={() => { setEditCariId(null); setCariForm({ name: "", note: "" }); }} style={{ ...st.btn("transparent"), color: "#526478", border: "1px solid rgba(100,200,255,0.06)", marginTop: 8 }}>İptal</button> : null}
        </div>
        {cariler.length > 0 ? (() => { let tA = 0, tB = 0; cariler.forEach(c => { const b = cariBals[c.id] || 0; if (b > 0) tA += b; else tB += Math.abs(b); }); return <div style={{ display: "flex", gap: 6, marginBottom: 12 }}><div style={st.sc("#34d399")}><div style={{ fontSize: 9, color: "#34d399", fontWeight: 700 }}>ALACAK</div><div style={{ fontSize: 15, fontWeight: 800, marginTop: 2 }}>{fmt(tA)}</div></div><div style={st.sc("#fb7185")}><div style={{ fontSize: 9, color: "#fb7185", fontWeight: 700 }}>BORÇ</div><div style={{ fontSize: 15, fontWeight: 800, marginTop: 2 }}>{fmt(tB)}</div></div></div>; })() : null}
        {cariler.map(c => { const b = cariBals[c.id] || 0; return <div key={c.id} style={{ ...st.card, padding: "12px 14px", cursor: "pointer" }} onClick={() => setSelectedCari(c.id)}><div style={{ display: "flex", alignItems: "center", gap: 10 }}><div style={{ width: 40, height: 40, borderRadius: 12, background: b > 0 ? "rgba(52,211,153,0.08)" : b < 0 ? "rgba(251,113,133,0.08)" : "rgba(249,168,37,0.08)", display: "flex", alignItems: "center", justifyContent: "center", fontSize: 16, flexShrink: 0, fontWeight: 800 }}>{c.name.charAt(0).toUpperCase()}</div><div style={{ flex: 1, minWidth: 0 }}><div style={{ fontSize: 14, fontWeight: 700 }}>{c.name}</div><div style={{ fontSize: 11, color: "#526478" }}>{b > 0 ? "Size borçlu" : b < 0 ? "Borçlusunuz" : "Eşit"}</div></div><div style={{ textAlign: "right", flexShrink: 0 }}><div style={{ fontSize: 15, fontWeight: 800, color: b > 0 ? "#34d399" : b < 0 ? "#fb7185" : "#526478" }}>{fmt(Math.abs(b))}</div><div style={{ display: "flex", gap: 10, marginTop: 4, justifyContent: "flex-end" }}><button onClick={e => { e.stopPropagation(); setCariForm({ name: c.name, note: c.note || "" }); setEditCariId(c.id); }} style={{ background: "none", border: "none", color: "#f59e0b", fontSize: 11, cursor: "pointer", padding: "2px 0" }}>Düzenle</button><button onClick={e => { e.stopPropagation(); dCari(c.id); }} style={{ background: "none", border: "none", color: "#fb7185", fontSize: 11, cursor: "pointer", padding: "2px 0" }}>Sil</button></div></div></div></div>; })}
      </div> : null}

      {page === "cari" && selectedCari ? (() => { const c = cariler.find(x => x.id === selectedCari); if (!c) { setSelectedCari(null); return null; } const b = cariBals[c.id] || 0; return <div style={{ padding: "0 " + P + "px" }}><button onClick={() => setSelectedCari(null)} style={{ background: "none", border: "none", color: "#38bdf8", fontSize: 13, cursor: "pointer", padding: "6px 0", fontFamily: "inherit", fontWeight: 600 }}>← Geri</button><div style={{ ...st.sc(b >= 0 ? "#34d399" : "#fb7185"), marginBottom: 12, padding: 18 }}><div style={{ fontSize: 18, fontWeight: 800, marginBottom: 4 }}>{c.name}</div><div style={{ fontSize: 9, color: b > 0 ? "#34d399" : b < 0 ? "#fb7185" : "#526478", fontWeight: 700, marginTop: 8, letterSpacing: 0.5 }}>{b > 0 ? "SİZE BORÇLU" : b < 0 ? "BORÇLUSUNUZ" : "EŞİT"}</div><div style={{ fontSize: 26, fontWeight: 800, marginTop: 2 }}>{fmt(Math.abs(b))}</div></div>{cariTxns.length === 0 ? <div style={{ textAlign: "center", padding: 20, color: "#4a6275", fontSize: 13 }}>İşlem yok</div> : null}{cariTxns.map(t => <TxnRow key={t.id} t={t} />)}</div>; })() : null}

      {/* ═══ AYARLAR ═══ */}
      {page === "ayarlar" ? <div style={{ padding: "0 " + P + "px" }}>
        <div style={st.card}>
          <div style={{ fontSize: 15, fontWeight: 700, marginBottom: 14 }}>Ayarlar</div>
          <div style={{ fontSize: 12, color: "#526478", marginBottom: 4 }}>İşlem: <strong style={{ color: "#e0e8f0" }}>{txns.length}</strong> · Plan: <strong style={{ color: "#e0e8f0" }}>{plans.length}</strong> · Cari: <strong style={{ color: "#e0e8f0" }}>{cariler.length}</strong> · Hesap: <strong style={{ color: "#e0e8f0" }}>{hesaplar.length}</strong> · Kart: <strong style={{ color: "#e0e8f0" }}>{cards.length}</strong></div>
          <div style={{ fontSize: 12, color: "#526478", marginBottom: 16 }}>Veriler otomatik kaydedilir.</div>
          <button onClick={clearAll} style={st.btn("#ef4444")}>Tüm Verileri Sil</button>
        </div>
        {(() => { const used = new Set([...txns, ...plans].filter(t => t.type === "gelir").map(t => t.category)); return <div style={st.card}><div style={{ fontSize: 13, fontWeight: 700, marginBottom: 10, color: "#34d399" }}>Gelir Kategorileri</div><div style={{ display: "flex", flexWrap: "wrap", gap: 6, marginBottom: 12 }}>{allG.map(c => { const u = used.has(c); return <div key={c} style={{ display: "flex", alignItems: "center", gap: 4, padding: "6px 12px", borderRadius: 20, background: "rgba(52,211,153,0.06)", border: "1px solid rgba(52,211,153,0.08)", fontSize: 12, fontWeight: 600, color: "#34d399" }}>{c}{u ? <span style={{ fontSize: 9, color: "#4a6275", marginLeft: 3 }}>✓</span> : <button onClick={() => { if (DEF_G.includes(c)) setRemG(p => [...p, c]); else setCustG(p => p.filter(x => x !== c)); sho("Silindi"); }} style={{ background: "none", border: "none", color: "#fb7185", fontSize: 13, cursor: "pointer", padding: 0, marginLeft: 2, lineHeight: 1 }}>×</button>}</div>; })}</div><div style={{ display: "flex", gap: 6 }}><input placeholder="Yeni kategori" id="nGC" style={{ ...st.inp, flex: 1 }} onKeyDown={e => { if (e.key === "Enter") { const v = e.target.value.trim(); if (v && !allG.includes(v)) { setCustG(p => [...p, v]); setRemG(p => p.filter(x => x !== v)); e.target.value = ""; sho("Eklendi"); } } }} /><button onClick={() => { const el = document.getElementById("nGC"); const v = el.value.trim(); if (v && !allG.includes(v)) { setCustG(p => [...p, v]); setRemG(p => p.filter(x => x !== v)); el.value = ""; sho("Eklendi"); } }} style={{ padding: "12px 18px", borderRadius: 10, border: "none", background: "#34d399", color: "#0a1520", fontWeight: 700, fontSize: 13, cursor: "pointer", fontFamily: "inherit" }}>Ekle</button></div></div>; })()}
        {(() => { const used = new Set([...txns, ...plans].filter(t => t.type === "gider").map(t => t.category)); [...txns, ...plans].forEach(t => { if (t.masrafCat) used.add(t.masrafCat); }); return <div style={st.card}><div style={{ fontSize: 13, fontWeight: 700, marginBottom: 10, color: "#fb7185" }}>Gider Kategorileri</div><div style={{ display: "flex", flexWrap: "wrap", gap: 6, marginBottom: 12 }}>{allGi.map(c => { const u = used.has(c); return <div key={c} style={{ display: "flex", alignItems: "center", gap: 4, padding: "6px 12px", borderRadius: 20, background: "rgba(251,113,133,0.06)", border: "1px solid rgba(251,113,133,0.08)", fontSize: 12, fontWeight: 600, color: "#fb7185" }}>{c}{u ? <span style={{ fontSize: 9, color: "#4a6275", marginLeft: 3 }}>✓</span> : <button onClick={() => { if (DEF_Gi.includes(c)) setRemGi(p => [...p, c]); else setCustGi(p => p.filter(x => x !== c)); sho("Silindi"); }} style={{ background: "none", border: "none", color: "#fb7185", fontSize: 13, cursor: "pointer", padding: 0, marginLeft: 2, lineHeight: 1 }}>×</button>}</div>; })}</div><div style={{ display: "flex", gap: 6 }}><input placeholder="Yeni kategori" id="nGiC" style={{ ...st.inp, flex: 1 }} onKeyDown={e => { if (e.key === "Enter") { const v = e.target.value.trim(); if (v && !allGi.includes(v)) { setCustGi(p => [...p, v]); setRemGi(p => p.filter(x => x !== v)); e.target.value = ""; sho("Eklendi"); } } }} /><button onClick={() => { const el = document.getElementById("nGiC"); const v = el.value.trim(); if (v && !allGi.includes(v)) { setCustGi(p => [...p, v]); setRemGi(p => p.filter(x => x !== v)); el.value = ""; sho("Eklendi"); } }} style={{ padding: "12px 18px", borderRadius: 10, border: "none", background: "#fb7185", color: "#fff", fontWeight: 700, fontSize: 13, cursor: "pointer", fontFamily: "inherit" }}>Ekle</button></div></div>; })()}
      </div> : null}

      {/* ═══ NAV ═══ */}
      <div style={{ display: "flex", alignItems: "stretch", background: "rgba(11,21,32,0.96)", borderTop: "1px solid rgba(100,200,255,0.06)", position: "fixed", bottom: 0, left: 0, right: 0, maxWidth: 430, margin: "0 auto", zIndex: 100, backdropFilter: "blur(14px)", paddingBottom: "env(safe-area-inset-bottom, 0px)" }}>
        {[["plan", "Plan", "📅"], ["dashboard", "Özet", "📊"], ["ekle", "Ekle", "✏️"], ["kart", "Hesap", "🏦"], ["cari", "Cari", "👥"]].map(([k, l, ic]) => (
          <button key={k} onClick={() => { setPage(k); if (k === "cari") setSelectedCari(null); }} style={{ flex: 1, display: "flex", flexDirection: "column", alignItems: "center", justifyContent: "center", gap: 2, padding: "7px 0 5px", border: "none", borderTop: page === k ? "2.5px solid #38bdf8" : "2.5px solid transparent", background: page === k ? "rgba(56,189,248,0.08)" : "transparent", color: page === k ? "#38bdf8" : "#526478", fontWeight: page === k ? 700 : 500, fontSize: 9, cursor: "pointer", fontFamily: "inherit" }}>
            <span style={{ fontSize: 17, lineHeight: 1 }}>{ic}</span><span>{l}</span>
          </button>
        ))}
      </div>
    </div>
  );
}

ReactDOM.createRoot(document.getElementById("root")).render(<App />);
</script>
</body>
</html>
