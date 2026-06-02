import { useState, useEffect, useRef } from "react";

const C = {
  bg: "#0A0C0F", surface: "#111318", card: "#181B21", border: "#232830",
  gold: "#D4A843", goldLight: "#F0C96B", amber: "#E8841A",
  green: "#2ECC8A", red: "#E05252", blue: "#4B8BF5", purple: "#9B6DFF",
  text: "#EEE8DA", muted: "#7A7566", subtle: "#1E2128",
};

const fmt = (n) => new Intl.NumberFormat("fr-FR").format(n) + " FCFA";

const INVOICES_INIT = [
  { id: "FAC-2024-001", client: "Groupe Bolloré CI", amount: 4500000, due: "2024-05-15", status: "overdue", daysLate: 11, reminders: 3, phone: "+225 07 12 34 567", email: "compta@bollore-ci.com", history: ["PAYÉ 2M FCFA en Jan", "Retard habituel Q2"], tier: "vip" },
  { id: "FAC-2024-002", client: "CFAO Motors Dakar", amount: 1800000, due: "2024-05-22", status: "pending", daysLate: 0, reminders: 1, phone: "+221 77 456 78 90", email: "finance@cfao-dakar.sn", history: ["Nouveau client", "1er contrat"], tier: "standard" },
  { id: "FAC-2024-003", client: "MTN Business Ghana", amount: 7200000, due: "2024-05-28", status: "upcoming", daysLate: 0, reminders: 0, phone: "+233 24 567 8901", email: "accounts@mtnbiz.gh", history: ["Toujours ponctuel"], tier: "vip" },
  { id: "FAC-2024-004", client: "Ecobank Togo", amount: 950000, due: "2024-04-30", status: "paid", daysLate: 0, reminders: 2, phone: "+228 90 12 34 56", email: "tresorerie@ecobank.tg", history: ["Payé après 2 relances"], tier: "standard" },
  { id: "FAC-2024-005", client: "Orange Business Mali", amount: 3100000, due: "2024-05-10", status: "overdue", daysLate: 16, reminders: 4, phone: "+223 76 543 210", email: "factures@orange-mali.ml", history: ["3 impayés en 2023", "Litige en cours"], tier: "risky" },
  { id: "FAC-2024-006", client: "Air Côte d'Ivoire", amount: 2250000, due: "2024-06-01", status: "upcoming", daysLate: 0, reminders: 0, phone: "+225 05 98 76 543", email: "ap@airci.net", history: ["Client fidèle 3 ans"], tier: "vip" },
];

// ──────────────────────────────────────────────
// AI REMINDER MODAL
// ──────────────────────────────────────────────
const AIReminderModal = ({ invoice, onClose, onSend }) => {
  const [loading, setLoading] = useState(false);
  const [suggestions, setSuggestions] = useState(null);
  const [selected, setSelected] = useState(null);
  const [channel, setChannel] = useState("whatsapp");
  const [sending, setSending] = useState(false);
  const [sent, setSent] = useState(false);

  const tierLabel = { vip: "client VIP / grand compte", standard: "client standard", risky: "client à risque avec historique d'impayés" };

  const generateSuggestions = async () => {
    setLoading(true);
    try {
      const prompt = `Tu es un expert en recouvrement B2B en Afrique de l'Ouest. Génère 3 messages de relance pour la facture suivante.

Contexte client:
- Nom: ${invoice.client}
- Montant: ${fmt(invoice.amount)}
- Facture: ${invoice.id}
- Statut: ${invoice.status === "overdue" ? `En retard de ${invoice.daysLate} jours` : "Approche de l'échéance le " + invoice.due}
- Profil: ${tierLabel[invoice.tier]}
- Historique: ${invoice.history.join(", ")}
- Nombre de relances déjà envoyées: ${invoice.reminders}
- Lien de paiement: pay.recouvr.io/${invoice.id.replace("FAC-", "").replace("-", "")}

Génère 3 variantes adaptées au profil et au contexte. Réponds UNIQUEMENT en JSON valide, sans backticks, sans texte avant ou après:
{
  "analysis": "2 phrases analysant la situation et recommandant une approche",
  "channel": "whatsapp ou sms ou email (recommandation)",
  "messages": [
    {"tone": "Courtois", "label": "Rappel doux", "text": "message complet prêt à envoyer"},
    {"tone": "Ferme", "label": "Rappel ferme", "text": "message complet prêt à envoyer"},
    {"tone": "Urgent", "label": "Mise en demeure", "text": "message complet prêt à envoyer"}
  ]
}`;

      const res = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          model: "claude-sonnet-4-20250514",
          max_tokens: 1000,
          messages: [{ role: "user", content: prompt }]
        })
      });
      const data = await res.json();
      const raw = data.content.map(b => b.text || "").join("");
      const clean = raw.replace(/```json|```/g, "").trim();
      const parsed = JSON.parse(clean);
      setSuggestions(parsed);
      setSelected(0);
      setChannel(parsed.channel || "whatsapp");
    } catch (e) {
      setSuggestions({ error: true });
    }
    setLoading(false);
  };

  useEffect(() => { generateSuggestions(); }, []);

  const handleSend = () => {
    setSending(true);
    setTimeout(() => { setSending(false); setSent(true); setTimeout(() => { onSend(invoice.id, channel); onClose(); }, 1200); }, 1800);
  };

  const channels = [
    { id: "whatsapp", icon: "💬", label: "WhatsApp", color: "#25D366" },
    { id: "sms", icon: "📱", label: "SMS", color: C.amber },
    { id: "email", icon: "📧", label: "Email", color: C.blue },
  ];

  const toneColors = { "Courtois": C.green, "Ferme": C.amber, "Urgent": C.red };

  return (
    <div style={{ position: "fixed", inset: 0, background: "rgba(0,0,0,0.85)", display: "flex", alignItems: "center", justifyContent: "center", zIndex: 1000, backdropFilter: "blur(6px)" }}>
      <div style={{ background: C.card, border: `1px solid ${C.border}`, borderRadius: 18, width: 580, maxWidth: "95vw", maxHeight: "90vh", overflow: "auto", padding: 28, boxShadow: "0 32px 80px rgba(0,0,0,0.7)" }}>

        {/* Header */}
        <div style={{ display: "flex", justifyContent: "space-between", alignItems: "flex-start", marginBottom: 20 }}>
          <div>
            <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
              <div style={{ background: `linear-gradient(135deg, ${C.purple}, ${C.blue})`, borderRadius: 8, width: 28, height: 28, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 14 }}>✦</div>
              <div style={{ color: C.goldLight, fontWeight: 800, fontSize: 16 }}>Relance intelligente</div>
            </div>
            <div style={{ color: C.muted, fontSize: 12, marginTop: 4 }}>{invoice.client} · {invoice.id}</div>
          </div>
          <button onClick={onClose} style={{ background: C.subtle, border: "none", color: C.muted, borderRadius: 8, width: 32, height: 32, cursor: "pointer", fontSize: 16 }}>✕</button>
        </div>

        {loading && (
          <div style={{ textAlign: "center", padding: "40px 0" }}>
            <div style={{ width: 40, height: 40, border: `3px solid ${C.border}`, borderTopColor: C.purple, borderRadius: "50%", margin: "0 auto 16px", animation: "spin 0.8s linear infinite" }} />
            <div style={{ color: C.muted, fontSize: 13 }}>L'IA analyse le profil client et rédige les messages...</div>
          </div>
        )}

        {suggestions?.error && (
          <div style={{ color: C.red, textAlign: "center", padding: 20 }}>Erreur de génération. Vérifie ta connexion.</div>
        )}

        {suggestions && !suggestions.error && !loading && (
          <>
            <div style={{ background: `linear-gradient(135deg, ${C.purple}15, ${C.blue}10)`, border: `1px solid ${C.purple}40`, borderRadius: 12, padding: 14, marginBottom: 18 }}>
              <div style={{ display: "flex", gap: 8, alignItems: "flex-start" }}>
                <span style={{ fontSize: 16, flexShrink: 0 }}>🧠</span>
                <div style={{ color: C.text, fontSize: 12, lineHeight: 1.7 }}>{suggestions.analysis}</div>
              </div>
            </div>

            <div style={{ display: "flex", gap: 8, marginBottom: 16 }}>
              {channels.map(ch => (
                <button key={ch.id} onClick={() => setChannel(ch.id)} style={{ flex: 1, padding: "9px 6px", borderRadius: 10, border: `1.5px solid ${channel === ch.id ? ch.color : C.border}`, background: channel === ch.id ? `${ch.color}18` : C.surface, color: channel === ch.id ? ch.color : C.muted, cursor: "pointer", fontSize: 12, fontWeight: 600, transition: "all 0.2s" }}>
                  <div style={{ fontSize: 16, marginBottom: 2 }}>{ch.icon}</div>
                  {ch.label}
                  {suggestions.channel === ch.id && <div style={{ fontSize: 9, marginTop: 2, opacity: 0.7 }}>✦ recommandé</div>}
                </button>
              ))}
            </div>

            <div style={{ marginBottom: 16 }}>
              <div style={{ color: C.muted, fontSize: 11, fontWeight: 600, letterSpacing: "0.06em", textTransform: "uppercase", marginBottom: 10 }}>Choisir une variante</div>
              <div style={{ display: "flex", gap: 8, marginBottom: 12 }}>
                {suggestions.messages.map((msg, i) => (
                  <button key={i} onClick={() => setSelected(i)} style={{ flex: 1, padding: "9px 8px", borderRadius: 10, border: `1.5px solid ${selected === i ? (toneColors[msg.tone] || C.gold) : C.border}`, background: selected === i ? `${toneColors[msg.tone] || C.gold}15` : C.surface, color: selected === i ? (toneColors[msg.tone] || C.gold) : C.muted, cursor: "pointer", fontSize: 12, fontWeight: 700, transition: "all 0.2s" }}>
                    {msg.label}
                  </button>
                ))}
              </div>
              {selected !== null && (
                <div style={{ background: C.surface, borderRadius: 10, padding: 14, fontSize: 12, color: C.muted, lineHeight: 1.8, border: `1px solid ${C.border}`, whiteSpace: "pre-wrap", maxHeight: 160, overflow: "auto" }}>
                  {suggestions.messages[selected].text}
                </div>
              )}
            </div>

            <button onClick={handleSend} disabled={sending || sent || selected === null} style={{ width: "100%", padding: "13px", borderRadius: 10, border: "none", background: sent ? C.green : `linear-gradient(135deg, ${C.purple}, ${C.blue})`, color: "#fff", fontWeight: 700, fontSize: 14, cursor: "pointer", transition: "all 0.3s" }}>
              {sent ? "✓ Envoyé !" : sending ? "Envoi en cours..." : `Envoyer via ${channels.find(c => c.id === channel)?.label}`}
            </button>
          </>
        )}
      </div>
    </div>
  );
};

// ──────────────────────────────────────────────
// PAGE PAIEMENT DÉBITEUR
// ──────────────────────────────────────────────
const PaymentPage = ({ invoice, onClose, onPaid }) => {
  const [step, setStep] = useState("details");
  const [method, setMethod] = useState(null);
  const [phone, setPhone] = useState("");
  const [processing, setProcessing] = useState(false);

  const methods = [
    { id: "mtn", label: "MTN MoMo", icon: "🟡", color: "#FFC403", countries: "CI · GH · CM" },
    { id: "orange", label: "Orange Money", icon: "🟠", color: "#FF6600", countries: "CI · SN · ML" },
    { id: "wave", label: "Wave", icon: "🌊", color: "#1BC5BD", countries: "CI · SN" },
    { id: "moov", label: "Moov Money", icon: "🔵", color: "#0052CC", countries: "CI · BJ · TG" },
  ];

  const handlePay = () => {
    setProcessing(true);
    setTimeout(() => { setStep("success"); setProcessing(false); setTimeout(() => { onPaid(invoice.id); onClose(); }, 2500); }, 2800);
  };

  const selectedMethod = methods.find(m => m.id === method);

  return (
    <div style={{ position: "fixed", inset: 0, background: "#070809", display: "flex", flexDirection: "column", zIndex: 2000, overflowY: "auto" }}>
      <style>{`
        @keyframes spin { to { transform: rotate(360deg); } }
        @keyframes pulse { 0%,100%{opacity:1} 50%{opacity:0.5} }
        @keyframes checkPop { 0%{transform:scale(0)} 60%{transform:scale(1.2)} 100%{transform:scale(1)} }
        @keyframes fadeIn { from{opacity:0;transform:translateY(12px)} to{opacity:1;transform:translateY(0)} }
        @keyframes shimmer { 0%{background-position:-400px 0} 100%{background-position:400px 0} }
      `}</style>

      {/* Header débiteur */}
      <div style={{ padding: "16px 24px", borderBottom: `1px solid #1A1D22`, display: "flex", justifyContent: "space-between", alignItems: "center", background: "#0A0C0F" }}>
        <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
          <div style={{ width: 28, height: 28, background: `linear-gradient(135deg, ${C.gold}, ${C.amber})`, borderRadius: 7, display: "flex", alignItems: "center", justifyContent: "center", fontWeight: 800, color: "#000", fontSize: 14 }}>R</div>
          <div style={{ color: C.goldLight, fontWeight: 700, fontSize: 14 }}>Recouvr Pay</div>
        </div>
        <button onClick={onClose} style={{ background: "#1A1D22", border: "none", color: C.muted, borderRadius: 8, padding: "6px 12px", cursor: "pointer", fontSize: 12 }}>← Retour dashboard</button>
      </div>

      <div style={{ flex: 1, display: "flex", alignItems: "center", justifyContent: "center", padding: "32px 16px" }}>
        <div style={{ width: "100%", maxWidth: 440 }}>

          {/* SUCCÈS */}
          {step === "success" && (
            <div style={{ textAlign: "center", animation: "fadeIn 0.4s ease" }}>
              <div style={{ width: 80, height: 80, background: `${C.green}20`, borderRadius: "50%", display: "flex", alignItems: "center", justifyContent: "center", margin: "0 auto 20px", animation: "checkPop 0.5s ease" }}>
                <div style={{ fontSize: 36 }}>✅</div>
              </div>
              <div style={{ color: C.green, fontWeight: 800, fontSize: 24, marginBottom: 8 }}>Paiement confirmé !</div>
              <div style={{ color: C.text, fontSize: 16, fontWeight: 600, marginBottom: 4 }}>{fmt(invoice.amount)}</div>
              <div style={{ color: C.muted, fontSize: 13 }}>reçus par votre fournisseur</div>
              <div style={{ marginTop: 24, background: "#111318", border: `1px solid #1E2128`, borderRadius: 12, padding: 16, textAlign: "left" }}>
                <div style={{ display: "flex", justifyContent: "space-between", marginBottom: 8 }}>
                  <span style={{ color: C.muted, fontSize: 12 }}>Réf. transaction</span>
                  <span style={{ color: C.text, fontSize: 12, fontWeight: 600 }}>TXN-{Date.now().toString().slice(-8)}</span>
                </div>
                <div style={{ display: "flex", justifyContent: "space-between", marginBottom: 8 }}>
                  <span style={{ color: C.muted, fontSize: 12 }}>Facture</span>
                  <span style={{ color: C.text, fontSize: 12, fontWeight: 600 }}>{invoice.id}</span>
                </div>
                <div style={{ display: "flex", justifyContent: "space-between" }}>
                  <span style={{ color: C.muted, fontSize: 12 }}>Opérateur</span>
                  <span style={{ color: C.text, fontSize: 12, fontWeight: 600 }}>{selectedMethod?.label}</span>
                </div>
              </div>
              <div style={{ color: C.muted, fontSize: 11, marginTop: 16, animation: "pulse 1.5s infinite" }}>Redirection en cours...</div>
            </div>
          )}

          {/* TRAITEMENT EN COURS */}
          {step === "processing" && (
            <div style={{ textAlign: "center", padding: "60px 0" }}>
              <div style={{ width: 56, height: 56, border: `4px solid ${C.border}`, borderTopColor: selectedMethod?.color || C.gold, borderRadius: "50%", margin: "0 auto 20px", animation: "spin 0.9s linear infinite" }} />
              <div style={{ color: C.text, fontWeight: 700, fontSize: 16, marginBottom: 8 }}>Traitement en cours...</div>
              <div style={{ color: C.muted, fontSize: 13 }}>Vérification auprès de {selectedMethod?.label}</div>
              <div style={{ marginTop: 24, color: C.muted, fontSize: 11, animation: "pulse 1.5s infinite" }}>Ne fermez pas cette fenêtre</div>
            </div>
          )}

          {step !== "success" && step !== "processing" && (
            <>
              {/* Carte facture */}
              <div style={{ background: `linear-gradient(135deg, #1A1D24, #13161C)`, border: `1px solid #2A2E38`, borderRadius: 16, padding: 24, marginBottom: 24, position: "relative", overflow: "hidden" }}>
                <div style={{ position: "absolute", top: -20, right: -20, width: 100, height: 100, borderRadius: "50%", background: `${C.gold}08` }} />
                <div style={{ color: C.muted, fontSize: 11, fontWeight: 600, letterSpacing: "0.1em", textTransform: "uppercase", marginBottom: 16 }}>Facture à régler</div>
                <div style={{ color: C.goldLight, fontSize: 32, fontWeight: 900, letterSpacing: "-0.03em", marginBottom: 4 }}>{fmt(invoice.amount)}</div>
                <div style={{ color: C.text, fontSize: 15, fontWeight: 600, marginBottom: 12 }}>{invoice.client}</div>
                <div style={{ display: "flex", gap: 12 }}>
                  <div style={{ background: "#0A0C0F", borderRadius: 8, padding: "8px 12px", flex: 1 }}>
                    <div style={{ color: C.muted, fontSize: 10, marginBottom: 2 }}>Réf. facture</div>
                    <div style={{ color: C.text, fontSize: 12, fontWeight: 600 }}>{invoice.id}</div>
                  </div>
                  <div style={{ background: "#0A0C0F", borderRadius: 8, padding: "8px 12px", flex: 1 }}>
                    <div style={{ color: C.muted, fontSize: 10, marginBottom: 2 }}>Échéance</div>
                    <div style={{ color: invoice.status === "overdue" ? C.red : C.text, fontSize: 12, fontWeight: 600 }}>{invoice.due}</div>
                  </div>
                  <div style={{ background: "#0A0C0F", borderRadius: 8, padding: "8px 12px", flex: 1 }}>
                    <div style={{ color: C.muted, fontSize: 10, marginBottom: 2 }}>Statut</div>
                    <div style={{ color: invoice.status === "overdue" ? C.red : C.amber, fontSize: 12, fontWeight: 600 }}>
                      {invoice.status === "overdue" ? `${invoice.daysLate}j de retard` : "En attente"}
                    </div>
                  </div>
                </div>
              </div>

              {/* ÉTAPE 1 : Choix opérateur */}
              {step === "details" && (
                <>
                  <div style={{ color: C.muted, fontSize: 12, fontWeight: 600, letterSpacing: "0.06em", textTransform: "uppercase", marginBottom: 12 }}>Choisir votre opérateur</div>
                  <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: 10, marginBottom: 20 }}>
                    {methods.map(m => (
                      <button key={m.id} onClick={() => { setMethod(m.id); setStep("method"); }}
                        style={{ padding: "16px 12px", borderRadius: 12, border: `1.5px solid ${C.border}`, background: C.card, color: C.text, cursor: "pointer", textAlign: "left", transition: "all 0.2s", display: "flex", flexDirection: "column", gap: 4 }}
                        onMouseEnter={e => { e.currentTarget.style.borderColor = m.color; e.currentTarget.style.background = `${m.color}10`; }}
                        onMouseLeave={e => { e.currentTarget.style.borderColor = C.border; e.currentTarget.style.background = C.card; }}>
                        <div style={{ fontSize: 22 }}>{m.icon}</div>
                        <div style={{ fontWeight: 700, fontSize: 13 }}>{m.label}</div>
                        <div style={{ color: C.muted, fontSize: 10 }}>{m.countries}</div>
                      </button>
                    ))}
                  </div>
                  <div style={{ display: "flex", alignItems: "center", gap: 8, justifyContent: "center" }}>
                    <div style={{ color: C.muted, fontSize: 11 }}>🔒 Paiement sécurisé · SSL · PCI-DSS</div>
                  </div>
                </>
              )}

              {/* ÉTAPE 2 : Saisie numéro */}
              {step === "method" && selectedMethod && (
                <>
                  <button onClick={() => setStep("details")} style={{ background: "none", border: "none", color: C.muted, cursor: "pointer", fontSize: 13, marginBottom: 16, display: "flex", alignItems: "center", gap: 4 }}>← Changer d'opérateur</button>

                  <div style={{ background: C.card, border: `1.5px solid ${selectedMethod.color}40`, borderRadius: 14, padding: 20, marginBottom: 20 }}>
                    <div style={{ display: "flex", alignItems: "center", gap: 10, marginBottom: 16 }}>
                      <div style={{ fontSize: 28 }}>{selectedMethod.icon}</div>
                      <div>
                        <div style={{ color: C.text, fontWeight: 700 }}>{selectedMethod.label}</div>
                        <div style={{ color: C.muted, fontSize: 11 }}>{selectedMethod.countries}</div>
                      </div>
                    </div>

                    <div style={{ marginBottom: 16 }}>
                      <div style={{ color: C.muted, fontSize: 12, marginBottom: 6 }}>Votre numéro {selectedMethod.label}</div>
                      <input
                        value={phone}
                        onChange={e => setPhone(e.target.value)}
                        placeholder="+225 07 XX XX XXX"
                        style={{
                          width: "100%", padding: "13px 16px", borderRadius: 10,
                          border: `1.5px solid ${phone ? selectedMethod.color + "80" : C.border}`,
                          background: C.surface, color: C.text, fontSize: 15, fontWeight: 600,
                          outline: "none", boxSizing: "border-box", letterSpacing: "0.05em",
                          transition: "border-color 0.2s"
                        }}
                      />
                    </div>

                    <div style={{ background: C.surface, borderRadius: 10, padding: "10px 14px", fontSize: 12, color: C.muted, lineHeight: 1.6 }}>
                      📲 Vous recevrez une notification de validation sur ce numéro. Confirmez le paiement depuis votre téléphone.
                    </div>
                  </div>

                  {/* Récapitulatif avant paiement */}
                  <div style={{ background: C.subtle, borderRadius: 12, padding: "14px 16px", marginBottom: 16 }}>
                    <div style={{ display: "flex", justifyContent: "space-between", marginBottom: 6 }}>
                      <span style={{ color: C.muted, fontSize: 12 }}>Montant à payer</span>
                      <span style={{ color: C.goldLight, fontSize: 13, fontWeight: 800 }}>{fmt(invoice.amount)}</span>
                    </div>
                    <div style={{ display: "flex", justifyContent: "space-between", marginBottom: 6 }}>
                      <span style={{ color: C.muted, fontSize: 12 }}>Frais de transaction</span>
                      <span style={{ color: C.green, fontSize: 12, fontWeight: 600 }}>Offerts</span>
                    </div>
                    <div style={{ height: 1, background: C.border, margin: "10px 0" }} />
                    <div style={{ display: "flex", justifyContent: "space-between" }}>
                      <span style={{ color: C.text, fontSize: 13, fontWeight: 700 }}>Total débité</span>
                      <span style={{ color: C.text, fontSize: 13, fontWeight: 800 }}>{fmt(invoice.amount)}</span>
                    </div>
                  </div>

                  <button
                    onClick={handlePay}
                    disabled={!phone || phone.length < 8 || processing}
                    style={{
                      width: "100%", padding: "15px", borderRadius: 12, border: "none",
                      background: phone && phone.length >= 8 ? `linear-gradient(135deg, ${selectedMethod.color}, ${selectedMethod.color}CC)` : C.border,
                      color: phone && phone.length >= 8 ? "#000" : C.muted,
                      fontWeight: 800, fontSize: 15, cursor: phone && phone.length >= 8 ? "pointer" : "not-allowed",
                      transition: "all 0.3s", letterSpacing: "0.02em"
                    }}>
                    {processing ? "Traitement..." : `Payer ${fmt(invoice.amount)}`}
                  </button>

                  <div style={{ textAlign: "center", marginTop: 12, color: C.muted, fontSize: 11 }}>
                    🔒 Crypté SSL · Aucune donnée stockée
                  </div>
                </>
              )}
            </>
          )}
        </div>
      </div>
    </div>
  );
};

// ──────────────────────────────────────────────
// BADGE STATUT
// ──────────────────────────────────────────────
const StatusBadge = ({ status, daysLate }) => {
  const cfg = {
    overdue: { color: C.red, bg: `${C.red}18`, label: daysLate ? `${daysLate}j de retard` : "En retard" },
    pending: { color: C.amber, bg: `${C.amber}18`, label: "En attente" },
    upcoming: { color: C.blue, bg: `${C.blue}18`, label: "À venir" },
    paid: { color: C.green, bg: `${C.green}18`, label: "Payé ✓" },
  };
  const c = cfg[status] || cfg.pending;
  return (
    <span style={{ padding: "4px 10px", borderRadius: 20, background: c.bg, color: c.color, fontSize: 11, fontWeight: 700, whiteSpace: "nowrap" }}>
      {c.label}
    </span>
  );
};

// ──────────────────────────────────────────────
// BADGE TIER
// ──────────────────────────────────────────────
const TierBadge = ({ tier }) => {
  const cfg = {
    vip: { label: "VIP", color: C.gold },
    standard: { label: "STD", color: C.muted },
    risky: { label: "⚠ RISK", color: C.red },
  };
  const c = cfg[tier] || cfg.standard;
  return (
    <span style={{ fontSize: 10, fontWeight: 700, color: c.color, letterSpacing: "0.05em" }}>{c.label}</span>
  );
};

// ──────────────────────────────────────────────
// DASHBOARD PRINCIPAL
// ──────────────────────────────────────────────
export default function App() {
  const [invoices, setInvoices] = useState(INVOICES_INIT);
  const [aiModal, setAiModal] = useState(null);
  const [payModal, setPayModal] = useState(null);
  const [filter, setFilter] = useState("all");
  const [toast, setToast] = useState(null);

  const showToast = (msg, color = C.green) => {
    setToast({ msg, color });
    setTimeout(() => setToast(null), 3000);
  };

  const handleSendReminder = (id, channel) => {
    setInvoices(prev => prev.map(inv => inv.id === id ? { ...inv, reminders: inv.reminders + 1 } : inv));
    const chLabel = { whatsapp: "WhatsApp", sms: "SMS", email: "Email" }[channel] || channel;
    showToast(`✓ Relance envoyée via ${chLabel}`);
  };

  const handlePaid = (id) => {
    setInvoices(prev => prev.map(inv => inv.id === id ? { ...inv, status: "paid", daysLate: 0 } : inv));
    showToast("🎉 Paiement enregistré avec succès !", C.green);
  };

  const filtered = filter === "all" ? invoices : invoices.filter(i => i.status === filter);

  const stats = {
    totalDue: invoices.filter(i => i.status !== "paid").reduce((s, i) => s + i.amount, 0),
    overdue: invoices.filter(i => i.status === "overdue").reduce((s, i) => s + i.amount, 0),
    collected: invoices.filter(i => i.status === "paid").reduce((s, i) => s + i.amount, 0),
    overdueCount: invoices.filter(i => i.status === "overdue").length,
  };

  const filters = [
    { id: "all", label: "Tout" },
    { id: "overdue", label: "En retard" },
    { id: "pending", label: "En attente" },
    { id: "upcoming", label: "À venir" },
    { id: "paid", label: "Payés" },
  ];

  return (
    <div style={{ background: C.bg, minHeight: "100vh", fontFamily: "'DM Sans', 'Helvetica Neue', sans-serif", color: C.text }}>
      <style>{`
        @keyframes spin { to { transform: rotate(360deg); } }
        @keyframes fadeIn { from{opacity:0;transform:translateY(8px)} to{opacity:1;transform:translateY(0)} }
        @keyframes toastIn { from{opacity:0;transform:translateX(40px)} to{opacity:1;transform:translateX(0)} }
        * { box-sizing: border-box; }
        ::-webkit-scrollbar { width: 4px; height: 4px; }
        ::-webkit-scrollbar-track { background: transparent; }
        ::-webkit-scrollbar-thumb { background: #2A2E38; border-radius: 4px; }
      `}</style>

      {/* Toast */}
      {toast && (
        <div style={{ position: "fixed", top: 24, right: 24, zIndex: 9999, background: C.card, border: `1px solid ${toast.color}40`, borderRadius: 12, padding: "12px 18px", color: toast.color, fontWeight: 700, fontSize: 13, animation: "toastIn 0.3s ease", boxShadow: `0 8px 32px rgba(0,0,0,0.5)` }}>
          {toast.msg}
        </div>
      )}

      {/* Header */}
      <div style={{ padding: "20px 32px", borderBottom: `1px solid ${C.border}`, display: "flex", alignItems: "center", justifyContent: "space-between", background: C.surface }}>
        <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
          <div style={{ width: 34, height: 34, background: `linear-gradient(135deg, ${C.gold}, ${C.amber})`, borderRadius: 9, display: "flex", alignItems: "center", justifyContent: "center", fontWeight: 900, color: "#000", fontSize: 16 }}>R</div>
          <div>
            <div style={{ color: C.goldLight, fontWeight: 800, fontSize: 17, letterSpacing: "-0.02em" }}>Recouvr</div>
            <div style={{ color: C.muted, fontSize: 10, marginTop: -2 }}>Recouvrement B2B · Afrique de l'Ouest</div>
          </div>
        </div>
        <div style={{ display: "flex", alignItems: "center", gap: 12 }}>
          <div style={{ background: `${C.red}18`, border: `1px solid ${C.red}30`, borderRadius: 8, padding: "6px 12px", color: C.red, fontSize: 12, fontWeight: 700 }}>
            ⚡ {stats.overdueCount} facture{stats.overdueCount > 1 ? "s" : ""} en retard
          </div>
          <div style={{ width: 34, height: 34, background: C.card, borderRadius: "50%", display: "flex", alignItems: "center", justifyContent: "center", fontSize: 16, border: `1px solid ${C.border}`, cursor: "pointer" }}>👤</div>
        </div>
      </div>

      <div style={{ maxWidth: 1100, margin: "0 auto", padding: "32px 24px" }}>

        {/* KPIs */}
        <div style={{ display: "grid", gridTemplateColumns: "repeat(3, 1fr)", gap: 16, marginBottom: 32 }}>
          {[
            { label: "Encours total", value: fmt(stats.totalDue), sub: `${invoices.filter(i => i.status !== "paid").length} factures actives`, color: C.gold, icon: "📋" },
            { label: "Montant en retard", value: fmt(stats.overdue), sub: `${stats.overdueCount} client${stats.overdueCount > 1 ? "s" : ""} concerné${stats.overdueCount > 1 ? "s" : ""}`, color: C.red, icon: "⚠️" },
            { label: "Collecté ce mois", value: fmt(stats.collected), sub: "Taux de recouvrement 72%", color: C.green, icon: "✅" },
          ].map((kpi, i) => (
            <div key={i} style={{ background: C.card, border: `1px solid ${C.border}`, borderRadius: 14, padding: "20px 22px", position: "relative", overflow: "hidden" }}>
              <div style={{ position: "absolute", top: -10, right: -10, width: 70, height: 70, borderRadius: "50%", background: `${kpi.color}08` }} />
              <div style={{ display: "flex", justifyContent: "space-between", alignItems: "flex-start", marginBottom: 10 }}>
                <div style={{ color: C.muted, fontSize: 12, fontWeight: 600 }}>{kpi.label}</div>
                <span style={{ fontSize: 18 }}>{kpi.icon}</span>
              </div>
              <div style={{ color: kpi.color, fontSize: 22, fontWeight: 900, letterSpacing: "-0.03em", marginBottom: 4 }}>{kpi.value}</div>
              <div style={{ color: C.muted, fontSize: 11 }}>{kpi.sub}</div>
            </div>
          ))}
        </div>

        {/* Filtres */}
        <div style={{ display: "flex", gap: 8, marginBottom: 20, flexWrap: "wrap" }}>
          {filters.map(f => {
            const count = f.id === "all" ? invoices.length : invoices.filter(i => i.status === f.id).length;
            const active = filter === f.id;
            return (
              <button key={f.id} onClick={() => setFilter(f.id)} style={{
                padding: "8px 16px", borderRadius: 20, border: `1.5px solid ${active ? C.gold : C.border}`,
                background: active ? `${C.gold}18` : C.surface, color: active ? C.goldLight : C.muted,
                fontWeight: active ? 700 : 500, fontSize: 13, cursor: "pointer", transition: "all 0.2s",
                display: "flex", alignItems: "center", gap: 6
              }}>
                {f.label}
                <span style={{ background: active ? `${C.gold}30` : C.border, borderRadius: 10, padding: "1px 7px", fontSize: 11, fontWeight: 700, color: active ? C.gold : C.muted }}>{count}</span>
              </button>
            );
          })}
        </div>

        {/* Table factures */}
        <div style={{ background: C.card, border: `1px solid ${C.border}`, borderRadius: 16, overflow: "hidden" }}>
          {/* En-tête table */}
          <div style={{ display: "grid", gridTemplateColumns: "1.8fr 1.2fr 1fr 0.9fr 0.8fr 1.2fr", gap: 0, padding: "12px 20px", borderBottom: `1px solid ${C.border}`, background: C.surface }}>
            {["Client", "Montant", "Échéance", "Statut", "Relances", "Actions"].map(h => (
              <div key={h} style={{ color: C.muted, fontSize: 11, fontWeight: 600, letterSpacing: "0.06em", textTransform: "uppercase" }}>{h}</div>
            ))}
          </div>

          {/* Lignes */}
          {filtered.length === 0 && (
            <div style={{ textAlign: "center", padding: "48px 0", color: C.muted, fontSize: 14 }}>Aucune facture dans cette catégorie</div>
          )}
          {filtered.map((inv, idx) => (
            <div key={inv.id}
              style={{ display: "grid", gridTemplateColumns: "1.8fr 1.2fr 1fr 0.9fr 0.8fr 1.2fr", gap: 0, padding: "16px 20px", borderBottom: idx < filtered.length - 1 ? `1px solid ${C.border}` : "none", alignItems: "center", transition: "background 0.15s", cursor: "default", animation: "fadeIn 0.3s ease" }}
              onMouseEnter={e => e.currentTarget.style.background = C.subtle}
              onMouseLeave={e => e.currentTarget.style.background = "transparent"}>

              {/* Client */}
              <div>
                <div style={{ display: "flex", alignItems: "center", gap: 8, marginBottom: 3 }}>
                  <div style={{ width: 30, height: 30, borderRadius: 8, background: `linear-gradient(135deg, ${C.border}, ${C.subtle})`, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 13, fontWeight: 800, color: C.muted, flexShrink: 0 }}>
                    {inv.client.charAt(0)}
                  </div>
                  <div>
                    <div style={{ fontWeight: 700, fontSize: 13, color: C.text }}>{inv.client}</div>
                    <div style={{ display: "flex", alignItems: "center", gap: 6, marginTop: 1 }}>
                      <span style={{ color: C.muted, fontSize: 10 }}>{inv.id}</span>
                      <TierBadge tier={inv.tier} />
                    </div>
                  </div>
                </div>
              </div>

              {/* Montant */}
              <div style={{ color: C.text, fontWeight: 700, fontSize: 13 }}>{fmt(inv.amount)}</div>

              {/* Échéance */}
              <div style={{ color: inv.status === "overdue" ? C.red : C.muted, fontSize: 12, fontWeight: inv.status === "overdue" ? 700 : 400 }}>{inv.due}</div>

              {/* Statut */}
              <div><StatusBadge status={inv.status} daysLate={inv.daysLate} /></div>

              {/* Relances */}
              <div style={{ display: "flex", alignItems: "center", gap: 6 }}>
                <div style={{ display: "flex", gap: 2 }}>
                  {[...Array(Math.min(inv.reminders, 5))].map((_, i) => (
                    <div key={i} style={{ width: 6, height: 6, borderRadius: "50%", background: inv.reminders >= 4 ? C.red : inv.reminders >= 2 ? C.amber : C.muted }} />
                  ))}
                  {inv.reminders === 0 && <span style={{ color: C.muted, fontSize: 11 }}>—</span>}
                </div>
                {inv.reminders > 0 && <span style={{ color: C.muted, fontSize: 11 }}>{inv.reminders}</span>}
              </div>

              {/* Actions */}
              <div style={{ display: "flex", gap: 6 }}>
                {inv.status !== "paid" && (
                  <>
                    <button
                      onClick={() => setAiModal(inv)}
                      title="Envoyer une relance IA"
                      style={{ display: "flex", alignItems: "center", gap: 4, padding: "6px 10px", borderRadius: 8, border: `1px solid ${C.purple}50`, background: `${C.purple}12`, color: C.purple, cursor: "pointer", fontSize: 11, fontWeight: 700, transition: "all 0.2s", whiteSpace: "nowrap" }}
                      onMouseEnter={e => { e.currentTarget.style.background = `${C.purple}25`; }}
                      onMouseLeave={e => { e.currentTarget.style.background = `${C.purple}12`; }}>
                      ✦ Relancer
                    </button>
                    <button
                      onClick={() => setPayModal(inv)}
                      title="Simuler page paiement"
                      style={{ padding: "6px 10px", borderRadius: 8, border: `1px solid ${C.green}50`, background: `${C.green}12`, color: C.green, cursor: "pointer", fontSize: 11, fontWeight: 700, transition: "all 0.2s" }}
                      onMouseEnter={e => { e.currentTarget.style.background = `${C.green}25`; }}
                      onMouseLeave={e => { e.currentTarget.style.background = `${C.green}12`; }}>
                      Payer
                    </button>
                  </>
                )}
                {inv.status === "paid" && (
                  <span style={{ color: C.green, fontSize: 12, fontWeight: 700 }}>✓ Soldé</span>
                )}
              </div>
            </div>
          ))}
        </div>

        {/* Footer stats */}
        <div style={{ marginTop: 20, display: "flex", justifyContent: "space-between", alignItems: "center" }}>
          <div style={{ color: C.muted, fontSize: 12 }}>{filtered.length} facture{filtered.length > 1 ? "s" : ""} affichée{filtered.length > 1 ? "s" : ""}</div>
          <div style={{ display: "flex", gap: 16 }}>
            <span style={{ color: C.muted, fontSize: 12 }}>Total filtré : <strong style={{ color: C.text }}>{fmt(filtered.reduce((s, i) => s + i.amount, 0))}</strong></span>
          </div>
        </div>

      </div>

      {/* Modals */}
      {aiModal && (
        <AIReminderModal
          invoice={aiModal}
          onClose={() => setAiModal(null)}
          onSend={handleSendReminder}
        />
      )}
      {payModal && (
        <PaymentPage
          invoice={payModal}
          onClose={() => setPayModal(null)}
          onPaid={handlePaid}
        />
      )}
    </div>
  );
}
