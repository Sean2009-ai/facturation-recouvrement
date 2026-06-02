import { useState, useEffect, useRef } from "react";
import { LineChart, Line, BarChart, Bar, XAxis, YAxis, Tooltip, ResponsiveContainer, Cell } from "recharts";

// ─── DESIGN TOKENS ───────────────────────────────────────────────
const T = {
  bg:      "#07080A",
  surface: "#0D0F13",
  card:    "#12151B",
  raised:  "#181C24",
  border:  "#1E2330",
  borderHi:"#2C3347",
  gold:    "#C9963A",
  goldHi:  "#EDB96A",
  teal:    "#2ABFAA",
  red:     "#E0504A",
  amber:   "#E09030",
  blue:    "#4A7CF5",
  purple:  "#8A5CF5",
  green:   "#28C47A",
  text:    "#E8E2D4",
  sub:     "#9A9080",
  dim:     "#5A5448",
};

const fmt  = (n) => new Intl.NumberFormat("fr-FR").format(Math.round(n)) + " FCFA";
const fmtM = (n) => (n >= 1_000_000 ? (n / 1_000_000).toFixed(1) + "M" : (n / 1000).toFixed(0) + "k") + " FCFA";

// ─── SEED DATA ────────────────────────────────────────────────────
const USERS_INIT = [
  { id: "U001", name: "Koné Aminata",    email: "a.kone@recouvr.io",    role: "admin",   status: "active",   company: "Recouvr HQ",        joined: "2024-01-10", invoices: 24, collected: 18200000, avatar: "KA" },
  { id: "U002", name: "Diallo Moussa",   email: "m.diallo@recouvr.io",  role: "manager", status: "active",   company: "Recouvr Dakar",     joined: "2024-02-14", invoices: 18, collected: 9400000,  avatar: "DM" },
  { id: "U003", name: "Asante Kwame",    email: "k.asante@recouvr.io",  role: "agent",   status: "active",   company: "Recouvr Ghana",     joined: "2024-03-01", invoices: 11, collected: 5100000,  avatar: "AK" },
  { id: "U004", name: "Traoré Fatima",   email: "f.traore@recouvr.io",  role: "agent",   status: "inactive", company: "Recouvr Mali",      joined: "2024-03-22", invoices: 6,  collected: 1800000,  avatar: "TF" },
  { id: "U005", name: "Mensah Esi",      email: "e.mensah@recouvr.io",  role: "manager", status: "active",   company: "Recouvr Ghana",     joined: "2024-04-05", invoices: 14, collected: 7600000,  avatar: "ME" },
];

const CLIENTS_INIT = [
  { id: "C001", name: "Groupe Bolloré CI",    email: "compta@bollore-ci.com",   phone: "+225 07 12 34 567", country: "🇨🇮", tier: "vip",      totalBilled: 12500000, totalPaid: 8000000, overdueCount: 1, lastPayment: "2024-03-15", risk: "medium" },
  { id: "C002", name: "CFAO Motors Dakar",    email: "finance@cfao-dakar.sn",   phone: "+221 77 456 78 90", country: "🇸🇳", tier: "standard", totalBilled: 4200000,  totalPaid: 2400000, overdueCount: 0, lastPayment: "2024-04-20", risk: "low"    },
  { id: "C003", name: "MTN Business Ghana",   email: "accounts@mtnbiz.gh",      phone: "+233 24 567 8901",  country: "🇬🇭", tier: "vip",      totalBilled: 21000000, totalPaid: 21000000,overdueCount: 0, lastPayment: "2024-04-28", risk: "low"    },
  { id: "C004", name: "Orange Business Mali", email: "factures@orange-mali.ml", phone: "+223 76 543 210",   country: "🇲🇱", tier: "risky",    totalBilled: 9300000,  totalPaid: 3200000, overdueCount: 2, lastPayment: "2024-02-10", risk: "high"   },
  { id: "C005", name: "Ecobank Togo",         email: "tresorerie@ecobank.tg",   phone: "+228 90 12 34 56",  country: "🇹🇬", tier: "standard", totalBilled: 2850000,  totalPaid: 2850000, overdueCount: 0, lastPayment: "2024-04-30", risk: "low"    },
  { id: "C006", name: "Air Côte d'Ivoire",    email: "ap@airci.net",            phone: "+225 05 98 76 543", country: "🇨🇮", tier: "vip",      totalBilled: 6750000,  totalPaid: 4500000, overdueCount: 0, lastPayment: "2024-04-18", risk: "low"    },
];

const RULES_INIT = [
  { id: "R001", name: "Rappel J+3",         trigger: 3,  tone: "Courtois", active: true,  channel: "email", sent: 42, opened: 31, paid: 8  },
  { id: "R002", name: "Relance J+7",         trigger: 7,  tone: "Ferme",    active: true,  channel: "email", sent: 28, opened: 19, paid: 11 },
  { id: "R003", name: "Mise en demeure J+15",trigger: 15, tone: "Urgent",   active: true,  channel: "email", sent: 14, opened: 12, paid: 6  },
  { id: "R004", name: "Escalade J+30",       trigger: 30, tone: "Légal",    active: false, channel: "email", sent: 5,  opened: 5,  paid: 2  },
];

const EMAIL_LOG_INIT = [
  { id: "EL001", client: "Groupe Bolloré CI",    invoice: "FAC-2024-001", rule: "Relance J+7",          tone: "Ferme",    status: "opened", sentAt: "2024-05-19 09:14", amount: 4500000 },
  { id: "EL002", client: "Orange Business Mali",  invoice: "FAC-2024-005", rule: "Mise en demeure J+15", tone: "Urgent",   status: "paid",   sentAt: "2024-05-08 11:30", amount: 3100000 },
  { id: "EL003", client: "CFAO Motors Dakar",     invoice: "FAC-2024-002", rule: "Rappel J+3",           tone: "Courtois", status: "sent",   sentAt: "2024-05-22 08:00", amount: 1800000 },
  { id: "EL004", client: "Groupe Bolloré CI",    invoice: "FAC-2024-001", rule: "Rappel J+3",           tone: "Courtois", status: "opened", sentAt: "2024-05-15 09:00", amount: 4500000 },
  { id: "EL005", client: "Orange Business Mali",  invoice: "FAC-2024-005", rule: "Relance J+7",          tone: "Ferme",    status: "opened", sentAt: "2024-05-12 10:45", amount: 3100000 },
];

const CHART_DATA = [
  { month: "Jan", collected: 8200000,  billed: 12000000 },
  { month: "Fév", collected: 11400000, billed: 15000000 },
  { month: "Mar", collected: 9800000,  billed: 13500000 },
  { month: "Avr", collected: 14200000, billed: 18000000 },
  { month: "Mai", collected: 7600000,  billed: 19800000 },
];

// ─── HELPERS ──────────────────────────────────────────────────────
const Avatar = ({ initials, size = 34, color = T.gold }) => (
  <div style={{ width: size, height: size, borderRadius: size * 0.28, background: `linear-gradient(135deg, ${color}30, ${color}18)`, border: `1.5px solid ${color}50`, display: "flex", alignItems: "center", justifyContent: "center", fontSize: size * 0.34, fontWeight: 800, color, flexShrink: 0, fontFamily: "monospace" }}>
    {initials}
  </div>
);

const Badge = ({ label, color }) => (
  <span style={{ padding: "3px 9px", borderRadius: 20, background: `${color}18`, color, fontSize: 11, fontWeight: 700, border: `1px solid ${color}30` }}>{label}</span>
);

const RiskDot = ({ risk }) => {
  const c = { low: T.green, medium: T.amber, high: T.red }[risk] || T.dim;
  return <span style={{ display: "inline-block", width: 8, height: 8, borderRadius: "50%", background: c, marginRight: 5, boxShadow: `0 0 6px ${c}80` }} />;
};

const ToneTag = ({ tone }) => {
  const cfg = { Courtois: T.teal, Ferme: T.amber, Urgent: T.red, Légal: T.purple };
  const c = cfg[tone] || T.sub;
  return <Badge label={tone} color={c} />;
};

const Stat = ({ icon, label, value, sub, color, delta }) => (
  <div style={{ background: T.card, border: `1px solid ${T.border}`, borderRadius: 14, padding: "18px 20px", position: "relative", overflow: "hidden" }}>
    <div style={{ position: "absolute", inset: 0, background: `radial-gradient(ellipse at top right, ${color}08 0%, transparent 70%)`, pointerEvents: "none" }} />
    <div style={{ display: "flex", justifyContent: "space-between", alignItems: "flex-start", marginBottom: 10 }}>
      <span style={{ fontSize: 20 }}>{icon}</span>
      {delta && <span style={{ fontSize: 11, fontWeight: 700, color: delta > 0 ? T.green : T.red }}>{delta > 0 ? "+" : ""}{delta}%</span>}
    </div>
    <div style={{ color, fontSize: 22, fontWeight: 900, letterSpacing: "-0.03em", marginBottom: 3 }}>{value}</div>
    <div style={{ color: T.text, fontSize: 12, fontWeight: 600, marginBottom: 2 }}>{label}</div>
    {sub && <div style={{ color: T.sub, fontSize: 11 }}>{sub}</div>}
  </div>
);

// ─── AI EMAIL COMPOSER (calls Claude API) ─────────────────────────
const AIEmailComposer = ({ client, rule, onClose, onSent }) => {
  const [loading, setLoading] = useState(true);
  const [email, setEmail] = useState(null);
  const [sending, setSending] = useState(false);
  const [sent, setSent] = useState(false);
  const [error, setError] = useState(false);
  const [editMode, setEditMode] = useState(false);
  const [editedBody, setEditedBody] = useState("");

  useEffect(() => { generateEmail(); }, []);

  const generateEmail = async () => {
    setLoading(true); setError(false);
    try {
      const res = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          model: "claude-sonnet-4-20250514",
          max_tokens: 1000,
          messages: [{
            role: "user",
            content: `Tu es un expert en recouvrement B2B en Afrique de l'Ouest francophone.
Génère un email de relance professionnel pour :
- Client : ${client.name}
- Email : ${client.email}
- Pays : ${client.country}
- Profil risque : ${client.risk}
- Tier : ${client.tier}
- Encours impayé : ${fmt(client.totalBilled - client.totalPaid)}
- Règle déclenchée : ${rule.name} (${rule.trigger} jours de retard)
- Ton demandé : ${rule.tone}
- Dernière relance : il y a ${rule.trigger} jours

Réponds UNIQUEMENT en JSON valide sans backticks :
{
  "subject": "objet de l'email",
  "preview": "première phrase de l'email (aperçu)",
  "body": "corps complet de l'email en HTML simple (p, strong, br uniquement), signé 'L'Équipe Recouvr'",
  "callToAction": "texte du bouton CTA (ex: Régulariser ma situation)",
  "reasoning": "1 phrase expliquant le choix de ton et d'approche"
}`
          }]
        })
      });
      const data = await res.json();
      const raw = data.content.map(b => b.text || "").join("");
      const parsed = JSON.parse(raw.replace(/```json|```/g, "").trim());
      setEmail(parsed);
      setEditedBody(parsed.body);
    } catch { setError(true); }
    setLoading(false);
  };

  const handleSend = () => {
    setSending(true);
    setTimeout(() => { setSent(true); setSending(false); setTimeout(() => { onSent(); onClose(); }, 1400); }, 2000);
  };

  return (
    <div style={{ position: "fixed", inset: 0, background: "rgba(0,0,0,0.9)", display: "flex", alignItems: "center", justifyContent: "center", zIndex: 2000, backdropFilter: "blur(8px)" }}>
      <div style={{ background: T.card, border: `1px solid ${T.borderHi}`, borderRadius: 20, width: 640, maxWidth: "96vw", maxHeight: "92vh", overflow: "auto", padding: 32, boxShadow: "0 40px 100px rgba(0,0,0,0.8)" }}>

        {/* Header */}
        <div style={{ display: "flex", justifyContent: "space-between", alignItems: "flex-start", marginBottom: 24 }}>
          <div>
            <div style={{ display: "flex", alignItems: "center", gap: 8, marginBottom: 4 }}>
              <div style={{ width: 30, height: 30, background: `linear-gradient(135deg, ${T.teal}, ${T.blue})`, borderRadius: 8, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 15 }}>✉</div>
              <div style={{ color: T.goldHi, fontWeight: 800, fontSize: 16 }}>Email IA — Aperçu & Envoi</div>
            </div>
            <div style={{ color: T.sub, fontSize: 12 }}>{client.name} · {rule.name}</div>
          </div>
          <button onClick={onClose} style={{ background: T.raised, border: "none", color: T.sub, borderRadius: 8, width: 32, height: 32, cursor: "pointer", fontSize: 16, display: "flex", alignItems: "center", justifyContent: "center" }}>✕</button>
        </div>

        {loading && (
          <div style={{ textAlign: "center", padding: "48px 0" }}>
            <div style={{ width: 44, height: 44, border: `3px solid ${T.border}`, borderTopColor: T.teal, borderRadius: "50%", margin: "0 auto 16px", animation: "spin 0.8s linear infinite" }} />
            <div style={{ color: T.sub, fontSize: 13 }}>Claude rédige l'email selon le profil client et la règle...</div>
          </div>
        )}

        {error && (
          <div style={{ textAlign: "center", padding: 24 }}>
            <div style={{ color: T.red, fontSize: 14, marginBottom: 16 }}>⚠ Erreur de génération</div>
            <button onClick={generateEmail} style={{ background: T.raised, border: `1px solid ${T.border}`, color: T.text, borderRadius: 8, padding: "8px 20px", cursor: "pointer", fontSize: 13 }}>Réessayer</button>
          </div>
        )}

        {email && !loading && (
          <>
            {/* Reasoning IA */}
            <div style={{ background: `linear-gradient(135deg, ${T.teal}12, ${T.blue}08)`, border: `1px solid ${T.teal}30`, borderRadius: 12, padding: 12, marginBottom: 20, display: "flex", gap: 10, alignItems: "flex-start" }}>
              <span style={{ fontSize: 15, flexShrink: 0 }}>🧠</span>
              <div style={{ color: T.text, fontSize: 12, lineHeight: 1.7 }}>{email.reasoning}</div>
            </div>

            {/* Aperçu email */}
            <div style={{ background: T.surface, border: `1px solid ${T.border}`, borderRadius: 14, overflow: "hidden", marginBottom: 20 }}>
              {/* Email header */}
              <div style={{ padding: "14px 18px", borderBottom: `1px solid ${T.border}`, display: "flex", flexDirection: "column", gap: 6 }}>
                <div style={{ display: "flex", gap: 8 }}>
                  <span style={{ color: T.sub, fontSize: 12, minWidth: 44 }}>À :</span>
                  <span style={{ color: T.teal, fontSize: 12, fontWeight: 600 }}>{client.email}</span>
                </div>
                <div style={{ display: "flex", gap: 8 }}>
                  <span style={{ color: T.sub, fontSize: 12, minWidth: 44 }}>Objet :</span>
                  <span style={{ color: T.text, fontSize: 12, fontWeight: 700 }}>{email.subject}</span>
                </div>
                <div style={{ display: "flex", gap: 8, alignItems: "center" }}>
                  <span style={{ color: T.sub, fontSize: 12, minWidth: 44 }}>Ton :</span>
                  <ToneTag tone={rule.tone} />
                </div>
              </div>

              {/* Email body */}
              <div style={{ padding: "18px 18px 0" }}>
                {editMode ? (
                  <textarea
                    value={editedBody}
                    onChange={e => setEditedBody(e.target.value)}
                    style={{ width: "100%", minHeight: 200, background: T.card, border: `1px solid ${T.borderHi}`, borderRadius: 8, color: T.text, fontSize: 12, lineHeight: 1.8, padding: 12, outline: "none", resize: "vertical", boxSizing: "border-box", fontFamily: "monospace" }}
                  />
                ) : (
                  <div style={{ color: T.sub, fontSize: 13, lineHeight: 1.85, maxHeight: 220, overflow: "auto" }}
                    dangerouslySetInnerHTML={{ __html: editedBody }} />
                )}
              </div>

              {/* CTA button preview */}
              <div style={{ padding: 18, display: "flex", justifyContent: "center" }}>
                <div style={{ padding: "11px 28px", borderRadius: 10, background: `linear-gradient(135deg, ${T.gold}, ${T.amber})`, color: "#000", fontWeight: 800, fontSize: 13, userSelect: "none" }}>
                  {email.callToAction} →
                </div>
              </div>
            </div>

            {/* Actions */}
            <div style={{ display: "flex", gap: 10 }}>
              <button onClick={() => setEditMode(m => !m)} style={{ flex: 1, padding: "11px", borderRadius: 10, border: `1px solid ${T.borderHi}`, background: editMode ? `${T.blue}20` : T.raised, color: editMode ? T.blue : T.sub, fontWeight: 600, fontSize: 13, cursor: "pointer" }}>
                {editMode ? "✓ Terminer l'édition" : "✏ Modifier"}
              </button>
              <button onClick={generateEmail} style={{ flex: 1, padding: "11px", borderRadius: 10, border: `1px solid ${T.border}`, background: T.raised, color: T.sub, fontWeight: 600, fontSize: 13, cursor: "pointer" }}>
                ↺ Regénérer
              </button>
              <button onClick={handleSend} disabled={sending || sent} style={{ flex: 2, padding: "11px", borderRadius: 10, border: "none", background: sent ? T.green : `linear-gradient(135deg, ${T.teal}, ${T.blue})`, color: "#fff", fontWeight: 800, fontSize: 13, cursor: "pointer", transition: "all 0.3s" }}>
                {sent ? "✓ Email envoyé !" : sending ? "Envoi en cours..." : `📤 Envoyer à ${client.email}`}
              </button>
            </div>
          </>
        )}
      </div>
    </div>
  );
};

// ─── ADD RULE MODAL ───────────────────────────────────────────────
const AddRuleModal = ({ onClose, onAdd }) => {
  const [form, setForm] = useState({ name: "", trigger: 5, tone: "Courtois", channel: "email", active: true });
  const set = (k, v) => setForm(f => ({ ...f, [k]: v }));

  return (
    <div style={{ position: "fixed", inset: 0, background: "rgba(0,0,0,0.85)", display: "flex", alignItems: "center", justifyContent: "center", zIndex: 2000, backdropFilter: "blur(6px)" }}>
      <div style={{ background: T.card, border: `1px solid ${T.borderHi}`, borderRadius: 18, width: 460, padding: 28, boxShadow: "0 32px 80px rgba(0,0,0,0.7)" }}>
        <div style={{ display: "flex", justifyContent: "space-between", marginBottom: 24 }}>
          <div style={{ color: T.goldHi, fontWeight: 800, fontSize: 16 }}>➕ Nouvelle règle automatique</div>
          <button onClick={onClose} style={{ background: T.raised, border: "none", color: T.sub, borderRadius: 8, width: 32, height: 32, cursor: "pointer", fontSize: 16 }}>✕</button>
        </div>

        {[
          { label: "Nom de la règle", key: "name", type: "text", placeholder: "ex: Rappel J+5" },
        ].map(f => (
          <div key={f.key} style={{ marginBottom: 16 }}>
            <div style={{ color: T.sub, fontSize: 12, marginBottom: 6, fontWeight: 600 }}>{f.label}</div>
            <input value={form[f.key]} onChange={e => set(f.key, e.target.value)} placeholder={f.placeholder}
              style={{ width: "100%", padding: "11px 14px", borderRadius: 10, border: `1.5px solid ${T.border}`, background: T.surface, color: T.text, fontSize: 13, outline: "none", boxSizing: "border-box" }} />
          </div>
        ))}

        <div style={{ marginBottom: 16 }}>
          <div style={{ color: T.sub, fontSize: 12, marginBottom: 6, fontWeight: 600 }}>Déclencher après (jours de retard)</div>
          <div style={{ display: "flex", alignItems: "center", gap: 12 }}>
            <input type="range" min="1" max="60" value={form.trigger} onChange={e => set("trigger", +e.target.value)}
              style={{ flex: 1, accentColor: T.gold }} />
            <div style={{ background: T.raised, border: `1px solid ${T.border}`, borderRadius: 8, padding: "6px 14px", color: T.goldHi, fontWeight: 800, minWidth: 52, textAlign: "center" }}>J+{form.trigger}</div>
          </div>
        </div>

        <div style={{ marginBottom: 16 }}>
          <div style={{ color: T.sub, fontSize: 12, marginBottom: 8, fontWeight: 600 }}>Ton de l'email IA</div>
          <div style={{ display: "flex", gap: 8 }}>
            {["Courtois", "Ferme", "Urgent", "Légal"].map(t => {
              const c = { Courtois: T.teal, Ferme: T.amber, Urgent: T.red, Légal: T.purple }[t];
              return (
                <button key={t} onClick={() => set("tone", t)} style={{ flex: 1, padding: "8px 4px", borderRadius: 8, border: `1.5px solid ${form.tone === t ? c : T.border}`, background: form.tone === t ? `${c}18` : T.surface, color: form.tone === t ? c : T.sub, fontWeight: 700, fontSize: 12, cursor: "pointer" }}>{t}</button>
              );
            })}
          </div>
        </div>

        <button
          onClick={() => { if (form.name && form.trigger) { onAdd({ ...form, id: `R00${Date.now()}`, sent: 0, opened: 0, paid: 0 }); onClose(); } }}
          style={{ width: "100%", padding: "13px", borderRadius: 10, border: "none", background: form.name ? `linear-gradient(135deg, ${T.gold}, ${T.amber})` : T.border, color: form.name ? "#000" : T.sub, fontWeight: 800, fontSize: 14, cursor: form.name ? "pointer" : "not-allowed" }}>
          Créer la règle
        </button>
      </div>
    </div>
  );
};

// ─── TOAST ────────────────────────────────────────────────────────
const useToast = () => {
  const [toast, setToast] = useState(null);
  const show = (msg, color = T.green) => { setToast({ msg, color }); setTimeout(() => setToast(null), 3500); };
  const Toast = () => toast ? (
    <div style={{ position: "fixed", top: 24, right: 24, zIndex: 9999, background: T.card, border: `1px solid ${toast.color}40`, borderRadius: 12, padding: "12px 20px", color: toast.color, fontWeight: 700, fontSize: 13, animation: "toastIn 0.3s ease", boxShadow: `0 8px 32px rgba(0,0,0,0.6)`, display: "flex", alignItems: "center", gap: 8 }}>
      {toast.msg}
    </div>
  ) : null;
  return { show, Toast };
};

// ─── MAIN ADMIN APP ───────────────────────────────────────────────
export default function AdminDashboard() {
  const [page, setPage] = useState("overview");
  const [users, setUsers] = useState(USERS_INIT);
  const [clients, setClients] = useState(CLIENTS_INIT);
  const [rules, setRules] = useState(RULES_INIT);
  const [emailLog, setEmailLog] = useState(EMAIL_LOG_INIT);
  const [aiComposer, setAiComposer] = useState(null); // { client, rule }
  const [showAddRule, setShowAddRule] = useState(false);
  const { show: showToast, Toast } = useToast();

  const globalStats = {
    totalUsers: users.length,
    activeUsers: users.filter(u => u.status === "active").length,
    totalClients: clients.length,
    totalBilled: clients.reduce((s, c) => s + c.totalBilled, 0),
    totalCollected: clients.reduce((s, c) => s + c.totalPaid, 0),
    emailsSent: emailLog.length,
    emailsPaid: emailLog.filter(e => e.status === "paid").length,
    rateCollect: Math.round(clients.reduce((s,c)=>s+c.totalPaid,0) / clients.reduce((s,c)=>s+c.totalBilled,0) * 100),
  };

  const NAV = [
    { id: "overview",  label: "Vue globale",   icon: "◈" },
    { id: "users",     label: "Utilisateurs",  icon: "👥" },
    { id: "clients",   label: "Clients",       icon: "🏢" },
    { id: "automations", label: "Règles IA",   icon: "⚡" },
    { id: "emaillog",  label: "Emails envoyés",icon: "📨" },
  ];

  return (
    <div style={{ background: T.bg, minHeight: "100vh", display: "flex", fontFamily: "'DM Mono', 'Courier New', monospace", color: T.text }}>
      <style>{`
        @keyframes spin    { to { transform: rotate(360deg); } }
        @keyframes fadeIn  { from{opacity:0;transform:translateY(10px)} to{opacity:1;transform:translateY(0)} }
        @keyframes toastIn { from{opacity:0;transform:translateX(40px)} to{opacity:1;transform:translateX(0)} }
        * { box-sizing: border-box; }
        ::-webkit-scrollbar { width: 4px; }
        ::-webkit-scrollbar-thumb { background: ${T.borderHi}; border-radius: 4px; }
        input:focus { border-color: ${T.gold} !important; }
        button:hover { opacity: 0.88; }
      `}</style>

      <Toast />

      {/* ── SIDEBAR ── */}
      <div style={{ width: 220, background: T.surface, borderRight: `1px solid ${T.border}`, display: "flex", flexDirection: "column", flexShrink: 0, position: "sticky", top: 0, height: "100vh" }}>
        {/* Logo */}
        <div style={{ padding: "24px 20px 20px", borderBottom: `1px solid ${T.border}` }}>
          <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
            <div style={{ width: 32, height: 32, background: `linear-gradient(135deg, ${T.gold}, ${T.amber})`, borderRadius: 9, display: "flex", alignItems: "center", justifyContent: "center", fontWeight: 900, color: "#000", fontSize: 16 }}>R</div>
            <div>
              <div style={{ color: T.goldHi, fontWeight: 800, fontSize: 15, letterSpacing: "-0.02em" }}>Recouvr</div>
              <div style={{ color: T.dim, fontSize: 10 }}>Admin Console</div>
            </div>
          </div>
        </div>

        {/* Nav */}
        <nav style={{ flex: 1, padding: "12px 10px" }}>
          {NAV.map(n => (
            <button key={n.id} onClick={() => setPage(n.id)} style={{
              width: "100%", padding: "10px 12px", borderRadius: 10, border: "none", textAlign: "left",
              background: page === n.id ? `${T.gold}18` : "transparent",
              color: page === n.id ? T.goldHi : T.sub,
              fontWeight: page === n.id ? 700 : 500, fontSize: 13, cursor: "pointer",
              display: "flex", alignItems: "center", gap: 10, marginBottom: 2,
              borderLeft: `2px solid ${page === n.id ? T.gold : "transparent"}`,
              transition: "all 0.15s", fontFamily: "inherit"
            }}>
              <span style={{ fontSize: 15 }}>{n.icon}</span>
              {n.label}
            </button>
          ))}
        </nav>

        {/* Admin profile */}
        <div style={{ padding: "16px 20px", borderTop: `1px solid ${T.border}`, display: "flex", alignItems: "center", gap: 10 }}>
          <Avatar initials="KA" size={32} color={T.gold} />
          <div>
            <div style={{ color: T.text, fontSize: 12, fontWeight: 700 }}>Koné Aminata</div>
            <div style={{ color: T.dim, fontSize: 10 }}>Super Admin</div>
          </div>
        </div>
      </div>

      {/* ── MAIN CONTENT ── */}
      <div style={{ flex: 1, overflow: "auto" }}>
        <div style={{ padding: "32px 32px", maxWidth: 1100, animation: "fadeIn 0.3s ease" }}>

          {/* ═══ OVERVIEW ═══ */}
          {page === "overview" && (
            <>
              <div style={{ marginBottom: 28 }}>
                <div style={{ color: T.goldHi, fontSize: 22, fontWeight: 900, letterSpacing: "-0.03em", marginBottom: 4 }}>Vue d'ensemble</div>
                <div style={{ color: T.sub, fontSize: 13 }}>Tableau de bord administrateur · Recouvr Africa</div>
              </div>

              {/* KPIs row 1 */}
              <div style={{ display: "grid", gridTemplateColumns: "repeat(4, 1fr)", gap: 14, marginBottom: 14 }}>
                <Stat icon="👥" label="Utilisateurs actifs" value={`${globalStats.activeUsers}/${globalStats.totalUsers}`} sub="agents & managers" color={T.blue} delta={12} />
                <Stat icon="🏢" label="Clients gérés" value={globalStats.totalClients} sub="6 pays couverts" color={T.teal} delta={8} />
                <Stat icon="📨" label="Emails IA envoyés" value={globalStats.emailsSent} sub={`${globalStats.emailsPaid} ont déclenché un paiement`} color={T.purple} delta={24} />
                <Stat icon="🎯" label="Taux de recouvrement" value={`${globalStats.rateCollect}%`} sub="sur encours total" color={T.gold} delta={5} />
              </div>

              {/* KPIs row 2 */}
              <div style={{ display: "grid", gridTemplateColumns: "repeat(2, 1fr)", gap: 14, marginBottom: 28 }}>
                <Stat icon="💰" label="Total facturé (cumulé)" value={fmtM(globalStats.totalBilled)} color={T.amber} />
                <Stat icon="✅" label="Total collecté (cumulé)" value={fmtM(globalStats.totalCollected)} sub={`${fmtM(globalStats.totalBilled - globalStats.totalCollected)} en cours`} color={T.green} delta={18} />
              </div>

              {/* Charts */}
              <div style={{ display: "grid", gridTemplateColumns: "2fr 1fr", gap: 16, marginBottom: 28 }}>
                {/* Line chart */}
                <div style={{ background: T.card, border: `1px solid ${T.border}`, borderRadius: 14, padding: "20px 24px" }}>
                  <div style={{ color: T.text, fontWeight: 700, fontSize: 14, marginBottom: 4 }}>Facturation vs Recouvrement</div>
                  <div style={{ color: T.sub, fontSize: 11, marginBottom: 20 }}>5 derniers mois</div>
                  <ResponsiveContainer width="100%" height={180}>
                    <LineChart data={CHART_DATA}>
                      <XAxis dataKey="month" tick={{ fill: T.dim, fontSize: 11 }} axisLine={false} tickLine={false} />
                      <YAxis tickFormatter={v => (v/1000000).toFixed(0)+"M"} tick={{ fill: T.dim, fontSize: 10 }} axisLine={false} tickLine={false} />
                      <Tooltip formatter={(v, n) => [fmtM(v), n === "collected" ? "Collecté" : "Facturé"]} contentStyle={{ background: T.raised, border: `1px solid ${T.border}`, borderRadius: 8, fontSize: 12 }} labelStyle={{ color: T.text }} />
                      <Line type="monotone" dataKey="billed" stroke={T.border} strokeWidth={2} dot={false} />
                      <Line type="monotone" dataKey="collected" stroke={T.gold} strokeWidth={2.5} dot={{ fill: T.gold, r: 4 }} />
                    </LineChart>
                  </ResponsiveContainer>
                </div>

                {/* Bar chart — emails */}
                <div style={{ background: T.card, border: `1px solid ${T.border}`, borderRadius: 14, padding: "20px 24px" }}>
                  <div style={{ color: T.text, fontWeight: 700, fontSize: 14, marginBottom: 4 }}>Performance emails IA</div>
                  <div style={{ color: T.sub, fontSize: 11, marginBottom: 20 }}>Par règle automatique</div>
                  <ResponsiveContainer width="100%" height={180}>
                    <BarChart data={rules.filter(r=>r.sent>0)} barSize={18} barGap={4}>
                      <XAxis dataKey="trigger" tickFormatter={v=>`J+${v}`} tick={{ fill: T.dim, fontSize: 10 }} axisLine={false} tickLine={false} />
                      <YAxis tick={{ fill: T.dim, fontSize: 10 }} axisLine={false} tickLine={false} />
                      <Tooltip contentStyle={{ background: T.raised, border: `1px solid ${T.border}`, borderRadius: 8, fontSize: 12 }} labelFormatter={v=>`Règle J+${v}`} />
                      <Bar dataKey="sent" name="Envoyés" fill={T.borderHi} radius={[4,4,0,0]} />
                      <Bar dataKey="opened" name="Ouverts" fill={T.blue} radius={[4,4,0,0]} />
                      <Bar dataKey="paid" name="Convertis" fill={T.gold} radius={[4,4,0,0]} />
                    </BarChart>
                  </ResponsiveContainer>
                </div>
              </div>

              {/* Recent email log */}
              <div style={{ background: T.card, border: `1px solid ${T.border}`, borderRadius: 14, overflow: "hidden" }}>
                <div style={{ padding: "16px 20px", borderBottom: `1px solid ${T.border}`, display: "flex", justifyContent: "space-between", alignItems: "center" }}>
                  <div style={{ color: T.text, fontWeight: 700, fontSize: 14 }}>Derniers emails envoyés</div>
                  <button onClick={() => setPage("emaillog")} style={{ background: "none", border: "none", color: T.gold, fontSize: 12, cursor: "pointer", fontWeight: 600 }}>Voir tout →</button>
                </div>
                {emailLog.slice(0,4).map((e, i) => (
                  <div key={e.id} style={{ display: "flex", alignItems: "center", gap: 16, padding: "14px 20px", borderBottom: i < 3 ? `1px solid ${T.border}` : "none" }}>
                    <div style={{ width: 8, height: 8, borderRadius: "50%", background: e.status === "paid" ? T.green : e.status === "opened" ? T.blue : T.dim, flexShrink: 0 }} />
                    <div style={{ flex: 1 }}>
                      <div style={{ color: T.text, fontSize: 12, fontWeight: 600 }}>{e.client}</div>
                      <div style={{ color: T.sub, fontSize: 11 }}>{e.rule} · {e.sentAt}</div>
                    </div>
                    <ToneTag tone={e.tone} />
                    <Badge label={e.status === "paid" ? "✓ Payé" : e.status === "opened" ? "Ouvert" : "Envoyé"} color={e.status === "paid" ? T.green : e.status === "opened" ? T.blue : T.dim} />
                  </div>
                ))}
              </div>
            </>
          )}

          {/* ═══ USERS ═══ */}
          {page === "users" && (
            <>
              <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 28 }}>
                <div>
                  <div style={{ color: T.goldHi, fontSize: 22, fontWeight: 900, letterSpacing: "-0.03em", marginBottom: 4 }}>Utilisateurs</div>
                  <div style={{ color: T.sub, fontSize: 13 }}>{globalStats.activeUsers} actifs · {users.length} total</div>
                </div>
                <button onClick={() => showToast("Fonctionnalité bientôt disponible", T.blue)} style={{ padding: "10px 20px", borderRadius: 10, border: `1px solid ${T.gold}50`, background: `${T.gold}15`, color: T.goldHi, fontWeight: 700, fontSize: 13, cursor: "pointer" }}>
                  + Inviter un agent
                </button>
              </div>

              <div style={{ background: T.card, border: `1px solid ${T.border}`, borderRadius: 14, overflow: "hidden" }}>
                <div style={{ display: "grid", gridTemplateColumns: "2fr 1.2fr 1fr 1fr 1fr 0.8fr", padding: "12px 20px", borderBottom: `1px solid ${T.border}`, background: T.surface }}>
                  {["Agent", "Société", "Rôle", "Factures", "Collecté", "Statut"].map(h => (
                    <div key={h} style={{ color: T.dim, fontSize: 11, fontWeight: 700, letterSpacing: "0.06em", textTransform: "uppercase" }}>{h}</div>
                  ))}
                </div>
                {users.map((u, i) => (
                  <div key={u.id}
                    style={{ display: "grid", gridTemplateColumns: "2fr 1.2fr 1fr 1fr 1fr 0.8fr", padding: "15px 20px", borderBottom: i < users.length - 1 ? `1px solid ${T.border}` : "none", alignItems: "center", transition: "background 0.15s" }}
                    onMouseEnter={e => e.currentTarget.style.background = T.raised}
                    onMouseLeave={e => e.currentTarget.style.background = "transparent"}>
                    <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
                      <Avatar initials={u.avatar} size={34} color={u.role === "admin" ? T.gold : u.role === "manager" ? T.teal : T.blue} />
                      <div>
                        <div style={{ color: T.text, fontWeight: 700, fontSize: 13 }}>{u.name}</div>
                        <div style={{ color: T.sub, fontSize: 11 }}>{u.email}</div>
                      </div>
                    </div>
                    <div style={{ color: T.sub, fontSize: 12 }}>{u.company}</div>
                    <div><Badge label={u.role} color={u.role === "admin" ? T.gold : u.role === "manager" ? T.teal : T.blue} /></div>
                    <div style={{ color: T.text, fontSize: 13, fontWeight: 600 }}>{u.invoices}</div>
                    <div style={{ color: T.green, fontSize: 12, fontWeight: 700 }}>{fmtM(u.collected)}</div>
                    <div>
                      <button onClick={() => { setUsers(prev => prev.map(uu => uu.id === u.id ? { ...uu, status: uu.status === "active" ? "inactive" : "active" } : uu)); showToast(`Statut mis à jour`); }}
                        style={{ padding: "4px 10px", borderRadius: 20, border: `1px solid ${u.status === "active" ? T.green : T.dim}30`, background: `${u.status === "active" ? T.green : T.dim}15`, color: u.status === "active" ? T.green : T.dim, fontSize: 11, fontWeight: 700, cursor: "pointer" }}>
                        {u.status === "active" ? "Actif" : "Inactif"}
                      </button>
                    </div>
                  </div>
                ))}
              </div>
            </>
          )}

          {/* ═══ CLIENTS ═══ */}
          {page === "clients" && (
            <>
              <div style={{ marginBottom: 28 }}>
                <div style={{ color: T.goldHi, fontSize: 22, fontWeight: 900, letterSpacing: "-0.03em", marginBottom: 4 }}>Clients</div>
                <div style={{ color: T.sub, fontSize: 13 }}>{clients.length} entreprises · {clients.filter(c=>c.risk==="high").length} à haut risque</div>
              </div>

              <div style={{ background: T.card, border: `1px solid ${T.border}`, borderRadius: 14, overflow: "hidden" }}>
                <div style={{ display: "grid", gridTemplateColumns: "2fr 1fr 1fr 1fr 1fr 1.2fr", padding: "12px 20px", borderBottom: `1px solid ${T.border}`, background: T.surface }}>
                  {["Client", "Facturé", "Collecté", "Impayés", "Risque", "Action IA"].map(h => (
                    <div key={h} style={{ color: T.dim, fontSize: 11, fontWeight: 700, letterSpacing: "0.06em", textTransform: "uppercase" }}>{h}</div>
                  ))}
                </div>
                {clients.map((c, i) => {
                  const due = c.totalBilled - c.totalPaid;
                  const pct = Math.round(c.totalPaid / c.totalBilled * 100);
                  return (
                    <div key={c.id}
                      style={{ display: "grid", gridTemplateColumns: "2fr 1fr 1fr 1fr 1fr 1.2fr", padding: "15px 20px", borderBottom: i < clients.length - 1 ? `1px solid ${T.border}` : "none", alignItems: "center", transition: "background 0.15s" }}
                      onMouseEnter={e => e.currentTarget.style.background = T.raised}
                      onMouseLeave={e => e.currentTarget.style.background = "transparent"}>
                      <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
                        <div style={{ fontSize: 20, flexShrink: 0 }}>{c.country}</div>
                        <div>
                          <div style={{ display: "flex", alignItems: "center", gap: 6, marginBottom: 2 }}>
                            <div style={{ color: T.text, fontWeight: 700, fontSize: 13 }}>{c.name}</div>
                            {c.tier === "vip" && <Badge label="VIP" color={T.gold} />}
                            {c.tier === "risky" && <Badge label="⚠" color={T.red} />}
                          </div>
                          <div style={{ color: T.sub, fontSize: 11 }}>{c.email}</div>
                        </div>
                      </div>
                      <div style={{ color: T.text, fontSize: 12, fontWeight: 600 }}>{fmtM(c.totalBilled)}</div>
                      <div>
                        <div style={{ color: T.green, fontSize: 12, fontWeight: 700 }}>{fmtM(c.totalPaid)}</div>
                        <div style={{ height: 3, background: T.border, borderRadius: 2, marginTop: 5, width: 60 }}>
                          <div style={{ height: "100%", width: `${pct}%`, background: pct > 80 ? T.green : pct > 50 ? T.amber : T.red, borderRadius: 2 }} />
                        </div>
                      </div>
                      <div style={{ color: due > 0 ? T.red : T.green, fontSize: 12, fontWeight: 700 }}>{due > 0 ? fmtM(due) : "—"}</div>
                      <div style={{ display: "flex", alignItems: "center" }}>
                        <RiskDot risk={c.risk} />
                        <span style={{ color: { low: T.green, medium: T.amber, high: T.red }[c.risk], fontSize: 12, fontWeight: 600, textTransform: "capitalize" }}>{c.risk}</span>
                      </div>
                      <div>
                        {due > 0 ? (
                          <button
                            onClick={() => setAiComposer({ client: c, rule: rules.find(r => r.active) || rules[0] })}
                            style={{ padding: "7px 12px", borderRadius: 8, border: `1px solid ${T.purple}50`, background: `${T.purple}15`, color: T.purple, fontSize: 11, fontWeight: 700, cursor: "pointer", whiteSpace: "nowrap" }}>
                            ✦ Email IA
                          </button>
                        ) : <span style={{ color: T.green, fontSize: 12 }}>✓ À jour</span>}
                      </div>
                    </div>
                  );
                })}
              </div>
            </>
          )}

          {/* ═══ AUTOMATIONS ═══ */}
          {page === "automations" && (
            <>
              <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 28 }}>
                <div>
                  <div style={{ color: T.goldHi, fontSize: 22, fontWeight: 900, letterSpacing: "-0.03em", marginBottom: 4 }}>Règles automatiques IA</div>
                  <div style={{ color: T.sub, fontSize: 13 }}>Les emails sont générés et envoyés automatiquement par Claude</div>
                </div>
                <button onClick={() => setShowAddRule(true)} style={{ padding: "10px 20px", borderRadius: 10, border: "none", background: `linear-gradient(135deg, ${T.gold}, ${T.amber})`, color: "#000", fontWeight: 800, fontSize: 13, cursor: "pointer" }}>
                  + Nouvelle règle
                </button>
              </div>

              {/* How it works */}
              <div style={{ background: `linear-gradient(135deg, ${T.purple}10, ${T.blue}08)`, border: `1px solid ${T.purple}30`, borderRadius: 14, padding: "16px 20px", marginBottom: 24, display: "flex", gap: 16, alignItems: "center" }}>
                <span style={{ fontSize: 24, flexShrink: 0 }}>⚡</span>
                <div>
                  <div style={{ color: T.text, fontWeight: 700, fontSize: 13, marginBottom: 4 }}>Comment ça marche</div>
                  <div style={{ color: T.sub, fontSize: 12, lineHeight: 1.7 }}>
                    Chaque nuit à minuit, le moteur vérifie toutes les factures impayées. Si une facture dépasse le seuil de jours défini, Claude génère un email personnalisé selon le ton et le profil client, puis l'envoie automatiquement à l'adresse enregistrée.
                  </div>
                </div>
              </div>

              {/* Pipeline visuel */}
              <div style={{ display: "flex", alignItems: "center", gap: 0, marginBottom: 28, overflowX: "auto", padding: "4px 0" }}>
                {["Facture impayée", "Vérification nightly", "Règle déclenchée", "Claude génère l'email", "Envoi automatique", "Suivi (ouvert / payé)"].map((step, i, arr) => (
                  <div key={i} style={{ display: "flex", alignItems: "center" }}>
                    <div style={{ background: T.card, border: `1px solid ${T.border}`, borderRadius: 10, padding: "10px 14px", fontSize: 12, color: T.sub, fontWeight: 600, whiteSpace: "nowrap", textAlign: "center", minWidth: 120 }}>
                      <div style={{ width: 24, height: 24, borderRadius: "50%", background: `${T.gold}20`, border: `1px solid ${T.gold}40`, color: T.gold, fontWeight: 800, fontSize: 11, display: "flex", alignItems: "center", justifyContent: "center", margin: "0 auto 6px" }}>{i+1}</div>
                      {step}
                    </div>
                    {i < arr.length - 1 && <div style={{ color: T.dim, fontSize: 18, padding: "0 4px", flexShrink: 0 }}>→</div>}
                  </div>
                ))}
              </div>

              {/* Règles */}
              <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
                {rules.map(rule => (
                  <div key={rule.id} style={{ background: T.card, border: `1px solid ${rule.active ? T.borderHi : T.border}`, borderRadius: 14, padding: "18px 22px", opacity: rule.active ? 1 : 0.55, transition: "all 0.2s" }}>
                    <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 14 }}>
                      <div style={{ display: "flex", alignItems: "center", gap: 12 }}>
                        <div style={{ background: `${T.gold}20`, border: `1px solid ${T.gold}40`, borderRadius: 10, padding: "8px 14px", color: T.goldHi, fontWeight: 900, fontSize: 16 }}>J+{rule.trigger}</div>
                        <div>
                          <div style={{ color: T.text, fontWeight: 700, fontSize: 14 }}>{rule.name}</div>
                          <div style={{ display: "flex", gap: 8, marginTop: 4 }}>
                            <ToneTag tone={rule.tone} />
                            <Badge label={`📧 ${rule.channel}`} color={T.blue} />
                          </div>
                        </div>
                      </div>
                      <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
                        {/* Test manuel */}
                        <button
                          onClick={() => setAiComposer({ client: clients[0], rule })}
                          style={{ padding: "7px 14px", borderRadius: 8, border: `1px solid ${T.teal}40`, background: `${T.teal}12`, color: T.teal, fontSize: 12, fontWeight: 700, cursor: "pointer" }}>
                          Tester
                        </button>
                        {/* Toggle actif */}
                        <button
                          onClick={() => { setRules(prev => prev.map(r => r.id === rule.id ? { ...r, active: !r.active } : r)); showToast(`Règle ${rule.active ? "désactivée" : "activée"}`); }}
                          style={{ width: 48, height: 26, borderRadius: 13, border: "none", background: rule.active ? T.gold : T.border, cursor: "pointer", position: "relative", transition: "all 0.3s" }}>
                          <div style={{ position: "absolute", top: 3, left: rule.active ? 24 : 3, width: 20, height: 20, borderRadius: "50%", background: "#fff", transition: "left 0.3s", boxShadow: "0 1px 4px rgba(0,0,0,0.4)" }} />
                        </button>
                      </div>
                    </div>

                    {/* Stats règle */}
                    <div style={{ display: "grid", gridTemplateColumns: "repeat(3, 1fr)", gap: 10 }}>
                      {[
                        { label: "Emails envoyés", value: rule.sent, color: T.sub },
                        { label: "Taux d'ouverture", value: rule.sent > 0 ? `${Math.round(rule.opened/rule.sent*100)}%` : "—", color: T.blue },
                        { label: "Taux de conversion", value: rule.sent > 0 ? `${Math.round(rule.paid/rule.sent*100)}%` : "—", color: T.green },
                      ].map(s => (
                        <div key={s.label} style={{ background: T.surface, borderRadius: 8, padding: "10px 14px" }}>
                          <div style={{ color: T.dim, fontSize: 11, marginBottom: 4 }}>{s.label}</div>
                          <div style={{ color: s.color, fontSize: 18, fontWeight: 800 }}>{s.value}</div>
                        </div>
                      ))}
                    </div>
                  </div>
                ))}
              </div>
            </>
          )}

          {/* ═══ EMAIL LOG ═══ */}
          {page === "emaillog" && (
            <>
              <div style={{ marginBottom: 28 }}>
                <div style={{ color: T.goldHi, fontSize: 22, fontWeight: 900, letterSpacing: "-0.03em", marginBottom: 4 }}>Journal des emails IA</div>
                <div style={{ color: T.sub, fontSize: 13 }}>{emailLog.length} emails envoyés · {emailLog.filter(e=>e.status==="paid").length} paiements déclenchés</div>
              </div>

              {/* Summary pills */}
              <div style={{ display: "flex", gap: 10, marginBottom: 22, flexWrap: "wrap" }}>
                {[
                  { label: `${emailLog.length} envoyés`, color: T.sub },
                  { label: `${emailLog.filter(e=>e.status==="opened").length} ouverts`, color: T.blue },
                  { label: `${emailLog.filter(e=>e.status==="paid").length} convertis`, color: T.green },
                  { label: `Taux ${Math.round(emailLog.filter(e=>e.status==="paid").length/emailLog.length*100)}%`, color: T.gold },
                ].map(p => <Badge key={p.label} label={p.label} color={p.color} />)}
              </div>

              <div style={{ background: T.card, border: `1px solid ${T.border}`, borderRadius: 14, overflow: "hidden" }}>
                <div style={{ display: "grid", gridTemplateColumns: "2fr 1.2fr 1.5fr 1fr 0.8fr 0.8fr", padding: "12px 20px", borderBottom: `1px solid ${T.border}`, background: T.surface }}>
                  {["Client", "Facture", "Règle", "Ton", "Montant", "Statut"].map(h => (
                    <div key={h} style={{ color: T.dim, fontSize: 11, fontWeight: 700, letterSpacing: "0.06em", textTransform: "uppercase" }}>{h}</div>
                  ))}
                </div>
                {emailLog.map((e, i) => (
                  <div key={e.id}
                    style={{ display: "grid", gridTemplateColumns: "2fr 1.2fr 1.5fr 1fr 0.8fr 0.8fr", padding: "14px 20px", borderBottom: i < emailLog.length - 1 ? `1px solid ${T.border}` : "none", alignItems: "center", transition: "background 0.15s" }}
                    onMouseEnter={el => el.currentTarget.style.background = T.raised}
                    onMouseLeave={el => el.currentTarget.style.background = "transparent"}>
                    <div>
                      <div style={{ color: T.text, fontWeight: 600, fontSize: 13 }}>{e.client}</div>
                      <div style={{ color: T.sub, fontSize: 11 }}>{e.sentAt}</div>
                    </div>
                    <div style={{ color: T.sub, fontSize: 12 }}>{e.invoice}</div>
                    <div style={{ color: T.text, fontSize: 12 }}>{e.rule}</div>
                    <ToneTag tone={e.tone} />
                    <div style={{ color: T.amber, fontSize: 12, fontWeight: 700 }}>{fmtM(e.amount)}</div>
                    <Badge
                      label={e.status === "paid" ? "✓ Payé" : e.status === "opened" ? "Ouvert" : "Envoyé"}
                      color={e.status === "paid" ? T.green : e.status === "opened" ? T.blue : T.dim}
                    />
                  </div>
                ))}
              </div>
            </>
          )}

        </div>
      </div>

      {/* ── MODALS ── */}
      {aiComposer && (
        <AIEmailComposer
          client={aiComposer.client}
          rule={aiComposer.rule}
          onClose={() => setAiComposer(null)}
          onSent={() => {
            setEmailLog(prev => [{
              id: `EL${Date.now()}`, client: aiComposer.client.name,
              invoice: `FAC-2024-00${Math.floor(Math.random()*9)+1}`,
              rule: aiComposer.rule.name, tone: aiComposer.rule.tone,
              status: "sent", sentAt: new Date().toLocaleString("fr-FR").slice(0,16),
              amount: aiComposer.client.totalBilled - aiComposer.client.totalPaid
            }, ...prev]);
            setRules(prev => prev.map(r => r.id === aiComposer.rule.id ? { ...r, sent: r.sent+1 } : r));
            showToast(`✉ Email envoyé à ${aiComposer.client.email}`);
          }}
        />
      )}
      {showAddRule && (
        <AddRuleModal
          onClose={() => setShowAddRule(false)}
          onAdd={rule => { setRules(prev => [...prev, rule]); showToast("✓ Règle créée avec succès"); }}
        />
      )}
    </div>
  );
}
