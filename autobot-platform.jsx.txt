import { useState, useEffect, useRef } from "react";

// ─────────────────────────────────────────────
// AUTOBOT — Instagram DM Automation AI SaaS
// Complete Frontend (React/JSX Artifact)
// ─────────────────────────────────────────────

const CLAUDE_API = "https://api.anthropic.com/v1/messages";

// ── Color tokens ──────────────────────────────
const THEME = {
  dark: {
    bg: "#050508",
    surface: "#0d0d14",
    card: "#12121e",
    border: "#1e1e30",
    accent: "#7c3aed",
    accentGlow: "#a855f7",
    accentSoft: "#1a0a2e",
    green: "#10b981",
    pink: "#ec4899",
    blue: "#3b82f6",
    text: "#f8fafc",
    muted: "#94a3b8",
    dim: "#475569",
  },
  light: {
    bg: "#f8f7ff",
    surface: "#ffffff",
    card: "#f1f0ff",
    border: "#e2e0ff",
    accent: "#7c3aed",
    accentGlow: "#6d28d9",
    accentSoft: "#ede9fe",
    green: "#059669",
    pink: "#db2777",
    blue: "#2563eb",
    text: "#1e1b4b",
    muted: "#6b7280",
    dim: "#9ca3af",
  },
};

// ── Fake data ─────────────────────────────────
const FAKE_CONVERSATIONS = [
  { id: 1, name: "Priya Sharma", handle: "@priya.creates", avatar: "PS", lastMsg: "Hey! I saw your reel 🔥", time: "2m", unread: 3, status: "ai_replied", platform: "ig" },
  { id: 2, name: "Rohan Mehra", handle: "@rohan.dev", avatar: "RM", lastMsg: "What's the price for collab?", time: "14m", unread: 1, status: "pending", platform: "ig" },
  { id: 3, name: "Aisha Khan", handle: "@aisha.vibes", avatar: "AK", lastMsg: "Thanks for the quick reply!", time: "1h", unread: 0, status: "ai_replied", platform: "ig" },
  { id: 4, name: "Dev Kapoor", handle: "@devk", avatar: "DK", lastMsg: "Can you send the link again?", time: "3h", unread: 0, status: "manual", platform: "ig" },
  { id: 5, name: "Nisha Patel", handle: "@nisha.patel", avatar: "NP", lastMsg: "Loved your post btw!", time: "5h", unread: 2, status: "ai_replied", platform: "ig" },
];

const FAKE_MESSAGES = {
  1: [
    { id: 1, from: "user", text: "Hey! I saw your reel 🔥", time: "2:10 PM" },
    { id: 2, from: "bot", text: "Thanks so much! 🙏 Which reel are you referring to? I'd love to know which one caught your eye!", time: "2:10 PM", ai: true },
    { id: 3, from: "user", text: "The editing one! Do you offer editing services?", time: "2:11 PM" },
    { id: 4, from: "bot", text: "Yes we do! 🎬 We offer professional video editing starting at ₹999. DM the word PRICING to get our full rate card!", time: "2:11 PM", ai: true },
  ],
  2: [
    { id: 1, from: "user", text: "What's the price for collab?", time: "1:56 PM" },
    { id: 2, from: "bot", text: "Hey Rohan! 👋 Our collaboration packages start at ₹5,000/month. Want me to send the detailed deck?", time: "1:56 PM", ai: true },
  ],
};

const FAKE_AUTOMATIONS = [
  { id: 1, keyword: "PRICING", reply: "Hey! Here's our pricing deck 👇 [link]. Reply TALK to speak to our team!", status: true, triggered: 142 },
  { id: 2, keyword: "COLLAB", reply: "We'd love to collab! 🤝 Please fill this form and we'll get back in 24h: [link]", status: true, triggered: 87 },
  { id: 3, keyword: "FREE", reply: "We have a FREE starter guide! 📖 Reply EMAIL to get it sent directly to your inbox.", status: false, triggered: 34 },
  { id: 4, keyword: "LINK", reply: "Here's the link you asked for! 🔗 bit.ly/autobot-ig — Let me know if you need anything else.", status: true, triggered: 211 },
];

// ── Tiny components ───────────────────────────
const Badge = ({ children, color = "accent", theme }) => {
  const colors = {
    accent: { bg: theme.accentSoft, text: theme.accentGlow },
    green: { bg: "#052e16", text: theme.green },
    pink: { bg: "#500724", text: theme.pink },
    blue: { bg: "#1e3a5f", text: theme.blue },
  };
  if (theme.bg === THEME.light.bg) {
    colors.green = { bg: "#d1fae5", text: "#065f46" };
    colors.pink = { bg: "#fce7f3", text: "#9d174d" };
    colors.blue = { bg: "#dbeafe", text: "#1e40af" };
  }
  const c = colors[color];
  return (
    <span style={{
      background: c.bg, color: c.text,
      padding: "2px 10px", borderRadius: 20, fontSize: 11, fontWeight: 700, letterSpacing: 0.5
    }}>{children}</span>
  );
};

const Toggle = ({ value, onChange, theme }) => (
  <div onClick={() => onChange(!value)} style={{
    width: 44, height: 24, borderRadius: 12,
    background: value ? theme.accent : theme.border,
    cursor: "pointer", position: "relative", transition: "background 0.2s",
    boxShadow: value ? `0 0 12px ${theme.accent}66` : "none",
  }}>
    <div style={{
      position: "absolute", top: 3, left: value ? 23 : 3,
      width: 18, height: 18, borderRadius: "50%", background: "#fff",
      transition: "left 0.2s", boxShadow: "0 1px 4px #0005"
    }} />
  </div>
);

const Stat = ({ label, value, delta, icon, theme }) => (
  <div style={{
    background: theme.card, border: `1px solid ${theme.border}`,
    borderRadius: 16, padding: "20px 24px",
    boxShadow: theme.bg === THEME.dark.bg ? `0 4px 24px #0009` : "0 2px 12px #0001"
  }}>
    <div style={{ fontSize: 24, marginBottom: 8 }}>{icon}</div>
    <div style={{ fontSize: 28, fontWeight: 800, color: theme.text, fontFamily: "'Syne', sans-serif" }}>{value}</div>
    <div style={{ fontSize: 13, color: theme.muted, marginTop: 2 }}>{label}</div>
    {delta && <div style={{ fontSize: 12, color: theme.green, marginTop: 6, fontWeight: 600 }}>↑ {delta} this week</div>}
  </div>
);

// ── AI Reply Generator (calls Claude API) ─────
const generateAIReply = async (message, context = "") => {
  const res = await fetch(CLAUDE_API, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      model: "claude-sonnet-4-20250514",
      max_tokens: 1000,
      system: `You are an AI assistant for Instagram DM automation for a creator/business account. 
Generate SHORT, friendly, conversational Instagram DM replies (max 2-3 sentences). 
Be warm, use 1-2 relevant emojis, and always end with a soft CTA or question.
Context about this account: ${context || "A creator/agency that offers video editing, content creation, and social media services."}
Reply in the same language as the user message. Keep it natural, not robotic.`,
      messages: [{ role: "user", content: `User DM: "${message}"\n\nGenerate an appropriate auto-reply for this Instagram DM.` }]
    })
  });
  const data = await res.json();
  return data.content?.[0]?.text || "Thanks for reaching out! We'll get back to you soon 🙏";
};

// ── Main App ──────────────────────────────────
export default function Autobot() {
  const [isDark, setIsDark] = useState(true);
  const [page, setPage] = useState("dashboard");
  const [conversations, setConversations] = useState(FAKE_CONVERSATIONS);
  const [messages, setMessages] = useState(FAKE_MESSAGES);
  const [selectedConv, setSelectedConv] = useState(1);
  const [inputMsg, setInputMsg] = useState("");
  const [automations, setAutomations] = useState(FAKE_AUTOMATIONS);
  const [aiEnabled, setAiEnabled] = useState(true);
  const [aiGenerating, setAiGenerating] = useState(false);
  const [newKeyword, setNewKeyword] = useState("");
  const [newReply, setNewReply] = useState("");
  const [showAddAutomation, setShowAddAutomation] = useState(false);
  const [aiTestInput, setAiTestInput] = useState("");
  const [aiTestOutput, setAiTestOutput] = useState("");
  const [aiTesting, setAiTesting] = useState(false);
  const [plan, setPlan] = useState("pro");
  const [igConnected, setIgConnected] = useState(true);
  const [notification, setNotification] = useState(null);
  const msgEndRef = useRef(null);

  const t = isDark ? THEME.dark : THEME.light;

  useEffect(() => {
    msgEndRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages, selectedConv]);

  const notify = (msg, type = "success") => {
    setNotification({ msg, type });
    setTimeout(() => setNotification(null), 3000);
  };

  const sendMessage = async () => {
    if (!inputMsg.trim()) return;
    const userMsg = { id: Date.now(), from: "me", text: inputMsg, time: new Date().toLocaleTimeString([], { hour: "2-digit", minute: "2-digit" }) };
    setMessages(prev => ({ ...prev, [selectedConv]: [...(prev[selectedConv] || []), userMsg] }));
    setInputMsg("");

    if (aiEnabled) {
      setAiGenerating(true);
      try {
        const conv = conversations.find(c => c.id === selectedConv);
        const reply = await generateAIReply(inputMsg, `Talking to ${conv?.name}`);
        const botMsg = { id: Date.now() + 1, from: "bot", text: reply, time: new Date().toLocaleTimeString([], { hour: "2-digit", minute: "2-digit" }), ai: true };
        setMessages(prev => ({ ...prev, [selectedConv]: [...(prev[selectedConv] || []), botMsg] }));
      } catch {
        notify("AI reply failed — fallback used", "error");
      }
      setAiGenerating(false);
    }
  };

  const testAI = async () => {
    if (!aiTestInput.trim()) return;
    setAiTesting(true);
    try {
      const reply = await generateAIReply(aiTestInput);
      setAiTestOutput(reply);
    } catch {
      setAiTestOutput("Error calling AI. Check your API connection.");
    }
    setAiTesting(false);
  };

  const addAutomation = () => {
    if (!newKeyword || !newReply) return;
    setAutomations(prev => [...prev, { id: Date.now(), keyword: newKeyword.toUpperCase(), reply: newReply, status: true, triggered: 0 }]);
    setNewKeyword(""); setNewReply("");
    setShowAddAutomation(false);
    notify("Automation rule created!");
  };

  // ── Layout ─────────────────────────────────
  const navItems = [
    { id: "dashboard", icon: "◈", label: "Dashboard" },
    { id: "inbox", icon: "✉", label: "Inbox" },
    { id: "automation", icon: "⚡", label: "Automation" },
    { id: "ai", icon: "🤖", label: "AI Engine" },
    { id: "billing", icon: "💳", label: "Billing" },
    { id: "settings", icon: "⚙", label: "Settings" },
  ];

  return (
    <div style={{
      minHeight: "100vh", background: t.bg, color: t.text,
      fontFamily: "'DM Sans', 'Segoe UI', sans-serif",
      display: "flex", flexDirection: "column",
    }}>
      {/* Google Fonts */}
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Syne:wght@700;800&family=DM+Sans:wght@400;500;600;700&display=swap');
        * { box-sizing: border-box; margin: 0; padding: 0; }
        ::-webkit-scrollbar { width: 4px; height: 4px; }
        ::-webkit-scrollbar-track { background: transparent; }
        ::-webkit-scrollbar-thumb { background: ${t.border}; border-radius: 99px; }
        textarea, input { outline: none; font-family: inherit; }
        @keyframes pulse { 0%,100%{opacity:1} 50%{opacity:0.4} }
        @keyframes glow { 0%,100%{box-shadow:0 0 16px ${t.accent}44} 50%{box-shadow:0 0 32px ${t.accent}88} }
        @keyframes slideIn { from{transform:translateY(-12px);opacity:0} to{transform:translateY(0);opacity:1} }
        @keyframes fadeIn { from{opacity:0} to{opacity:1} }
        .nav-item:hover { background: ${t.accentSoft} !important; color: ${t.accent} !important; }
        .conv-item:hover { background: ${isDark ? "#1a1a28" : "#f5f3ff"} !important; }
        .btn-primary:hover { filter: brightness(1.15); transform: translateY(-1px); box-shadow: 0 8px 24px ${t.accent}55 !important; }
        .btn-ghost:hover { background: ${t.accentSoft} !important; }
        .automation-row:hover { background: ${isDark ? "#1a1a28" : "#f5f3ff"} !important; }
      `}</style>

      {/* Notification Toast */}
      {notification && (
        <div style={{
          position: "fixed", top: 20, right: 20, zIndex: 9999,
          background: notification.type === "error" ? "#7f1d1d" : t.accent,
          color: "#fff", padding: "12px 20px", borderRadius: 12,
          fontSize: 14, fontWeight: 600, animation: "slideIn 0.3s ease",
          boxShadow: `0 8px 32px ${notification.type === "error" ? "#f008" : t.accent + "88"}`
        }}>{notification.msg}</div>
      )}

      {/* Top Nav */}
      <header style={{
        height: 60, background: t.surface, borderBottom: `1px solid ${t.border}`,
        display: "flex", alignItems: "center", justifyContent: "space-between",
        padding: "0 24px", position: "sticky", top: 0, zIndex: 100,
        backdropFilter: "blur(12px)",
      }}>
        <div style={{ display: "flex", alignItems: "center", gap: 12 }}>
          <div style={{
            width: 32, height: 32, borderRadius: 10,
            background: `linear-gradient(135deg, ${t.accent}, ${t.pink})`,
            display: "flex", alignItems: "center", justifyContent: "center",
            fontSize: 16, fontWeight: 900, color: "#fff",
            boxShadow: `0 4px 16px ${t.accent}66`,
          }}>A</div>
          <span style={{ fontFamily: "'Syne', sans-serif", fontWeight: 800, fontSize: 20, letterSpacing: -0.5 }}>
            Auto<span style={{ color: t.accent }}>bot</span>
          </span>
          <div style={{
            marginLeft: 8, background: t.accentSoft, color: t.accentGlow,
            fontSize: 10, fontWeight: 700, padding: "2px 8px", borderRadius: 20,
            letterSpacing: 1, textTransform: "uppercase"
          }}>BETA</div>
        </div>

        <nav style={{ display: "flex", gap: 4 }}>
          {navItems.map(n => (
            <button key={n.id} className="nav-item" onClick={() => setPage(n.id)} style={{
              background: page === n.id ? t.accentSoft : "transparent",
              color: page === n.id ? t.accent : t.muted,
              border: "none", borderRadius: 8, padding: "6px 14px",
              cursor: "pointer", fontSize: 13, fontWeight: 600,
              display: "flex", alignItems: "center", gap: 6, transition: "all 0.15s",
            }}>
              <span style={{ fontSize: 14 }}>{n.icon}</span>
              <span style={{ display: window.innerWidth < 900 ? "none" : "inline" }}>{n.label}</span>
            </button>
          ))}
        </nav>

        <div style={{ display: "flex", alignItems: "center", gap: 12 }}>
          <button onClick={() => setIsDark(!isDark)} style={{
            background: t.card, border: `1px solid ${t.border}`, borderRadius: 8,
            padding: "6px 12px", cursor: "pointer", color: t.muted, fontSize: 16
          }}>{isDark ? "☀️" : "🌙"}</button>
          <div style={{
            width: 34, height: 34, borderRadius: "50%",
            background: `linear-gradient(135deg, ${t.accent}, ${t.blue})`,
            display: "flex", alignItems: "center", justifyContent: "center",
            color: "#fff", fontWeight: 700, fontSize: 13, cursor: "pointer"
          }}>MJ</div>
        </div>
      </header>

      {/* Page Content */}
      <main style={{ flex: 1, padding: page === "inbox" ? 0 : "28px 32px", overflow: "auto" }}>

        {/* ── DASHBOARD ── */}
        {page === "dashboard" && (
          <div style={{ animation: "fadeIn 0.3s ease" }}>
            <div style={{ marginBottom: 28 }}>
              <h1 style={{ fontFamily: "'Syne', sans-serif", fontSize: 28, fontWeight: 800, letterSpacing: -0.5 }}>
                Good morning, Maurya Ji 👋
              </h1>
              <p style={{ color: t.muted, marginTop: 4, fontSize: 14 }}>Here's what's happening with your automations today.</p>
            </div>

            {/* Stats Grid */}
            <div style={{ display: "grid", gridTemplateColumns: "repeat(4, 1fr)", gap: 16, marginBottom: 28 }}>
              <Stat theme={t} icon="✉️" label="Total Messages" value="2,841" delta="312" />
              <Stat theme={t} icon="🤖" label="AI Replies Sent" value="1,930" delta="208" />
              <Stat theme={t} icon="⚡" label="Auto Triggered" value="476" delta="54" />
              <Stat theme={t} icon="📈" label="Response Rate" value="98.2%" delta="1.4%" />
            </div>

            {/* Charts Row */}
            <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: 16, marginBottom: 28 }}>
              {/* Activity Chart */}
              <div style={{ background: t.card, border: `1px solid ${t.border}`, borderRadius: 16, padding: 24 }}>
                <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 20 }}>
                  <h3 style={{ fontWeight: 700, fontSize: 15 }}>Message Activity</h3>
                  <Badge theme={t} color="accent">Last 7 days</Badge>
                </div>
                {/* SVG Bar Chart */}
                <svg viewBox="0 0 340 100" style={{ width: "100%", height: 100 }}>
                  {[42, 78, 55, 91, 67, 110, 88].map((v, i) => (
                    <g key={i}>
                      <rect x={i * 48 + 4} y={110 - v} width={36} height={v}
                        rx={6} fill={`url(#grad${i})`} opacity={0.9} />
                      <defs>
                        <linearGradient id={`grad${i}`} x1="0" y1="0" x2="0" y2="1">
                          <stop offset="0%" stopColor={t.accent} />
                          <stop offset="100%" stopColor={t.pink} stopOpacity={0.6} />
                        </linearGradient>
                      </defs>
                    </g>
                  ))}
                  {["M", "T", "W", "T", "F", "S", "S"].map((d, i) => (
                    <text key={i} x={i * 48 + 22} y={115} textAnchor="middle" fill={t.dim} fontSize={10}>{d}</text>
                  ))}
                </svg>
              </div>

              {/* Top Automations */}
              <div style={{ background: t.card, border: `1px solid ${t.border}`, borderRadius: 16, padding: 24 }}>
                <h3 style={{ fontWeight: 700, fontSize: 15, marginBottom: 16 }}>Top Automations</h3>
                {automations.slice(0, 4).map((a, i) => (
                  <div key={a.id} style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 12 }}>
                    <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
                      <div style={{ width: 28, height: 28, borderRadius: 8, background: t.accentSoft, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 10, fontWeight: 700, color: t.accent }}>⚡</div>
                      <div>
                        <div style={{ fontSize: 13, fontWeight: 600 }}>{a.keyword}</div>
                        <div style={{ fontSize: 11, color: t.muted }}>{a.triggered} triggered</div>
                      </div>
                    </div>
                    <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
                      <div style={{ width: 80, height: 4, background: t.border, borderRadius: 4 }}>
                        <div style={{ width: `${Math.min(a.triggered / 2.5, 100)}%`, height: "100%", background: t.accent, borderRadius: 4 }} />
                      </div>
                    </div>
                  </div>
                ))}
              </div>
            </div>

            {/* Recent Conversations */}
            <div style={{ background: t.card, border: `1px solid ${t.border}`, borderRadius: 16, padding: 24 }}>
              <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 16 }}>
                <h3 style={{ fontWeight: 700, fontSize: 15 }}>Recent Conversations</h3>
                <button className="btn-ghost" onClick={() => setPage("inbox")} style={{
                  background: "transparent", border: `1px solid ${t.border}`, color: t.accent,
                  borderRadius: 8, padding: "6px 14px", cursor: "pointer", fontSize: 13, fontWeight: 600, transition: "all 0.15s"
                }}>View All</button>
              </div>
              {conversations.slice(0, 4).map(c => (
                <div key={c.id} className="conv-item" onClick={() => { setPage("inbox"); setSelectedConv(c.id); }}
                  style={{ display: "flex", alignItems: "center", gap: 12, padding: "10px 12px", borderRadius: 10, cursor: "pointer", transition: "background 0.15s" }}>
                  <div style={{ width: 38, height: 38, borderRadius: "50%", background: `linear-gradient(135deg, ${t.accent}, ${t.pink})`, display: "flex", alignItems: "center", justifyContent: "center", color: "#fff", fontWeight: 700, fontSize: 13, flexShrink: 0 }}>{c.avatar}</div>
                  <div style={{ flex: 1, minWidth: 0 }}>
                    <div style={{ display: "flex", justifyContent: "space-between" }}>
                      <span style={{ fontWeight: 600, fontSize: 14 }}>{c.name}</span>
                      <span style={{ fontSize: 11, color: t.dim }}>{c.time}</span>
                    </div>
                    <div style={{ fontSize: 12, color: t.muted, overflow: "hidden", textOverflow: "ellipsis", whiteSpace: "nowrap" }}>{c.lastMsg}</div>
                  </div>
                  {c.status === "ai_replied" && <Badge theme={t} color="accent">AI</Badge>}
                  {c.unread > 0 && <div style={{ background: t.accent, color: "#fff", borderRadius: "50%", width: 18, height: 18, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 10, fontWeight: 700, flexShrink: 0 }}>{c.unread}</div>}
                </div>
              ))}
            </div>
          </div>
        )}

        {/* ── INBOX ── */}
        {page === "inbox" && (
          <div style={{ display: "flex", height: "calc(100vh - 60px)", animation: "fadeIn 0.3s ease" }}>
            {/* Sidebar */}
            <div style={{ width: 320, borderRight: `1px solid ${t.border}`, display: "flex", flexDirection: "column", background: t.surface }}>
              <div style={{ padding: "16px 16px 12px", borderBottom: `1px solid ${t.border}` }}>
                <div style={{ fontFamily: "'Syne', sans-serif", fontWeight: 800, fontSize: 16, marginBottom: 10 }}>Inbox</div>
                <div style={{ position: "relative" }}>
                  <input placeholder="Search conversations..." style={{
                    width: "100%", padding: "8px 12px 8px 34px", borderRadius: 10,
                    background: t.card, border: `1px solid ${t.border}`, color: t.text, fontSize: 13
                  }} />
                  <span style={{ position: "absolute", left: 11, top: 9, color: t.dim, fontSize: 13 }}>🔍</span>
                </div>
              </div>
              <div style={{ flex: 1, overflowY: "auto" }}>
                {conversations.map(c => (
                  <div key={c.id} className="conv-item" onClick={() => setSelectedConv(c.id)}
                    style={{
                      display: "flex", alignItems: "center", gap: 12, padding: "12px 16px",
                      cursor: "pointer", transition: "background 0.15s",
                      background: selectedConv === c.id ? (isDark ? "#1a1a28" : "#f5f3ff") : "transparent",
                      borderLeft: selectedConv === c.id ? `3px solid ${t.accent}` : "3px solid transparent",
                    }}>
                    <div style={{ position: "relative", flexShrink: 0 }}>
                      <div style={{ width: 40, height: 40, borderRadius: "50%", background: `linear-gradient(135deg, ${t.accent}, ${t.blue})`, display: "flex", alignItems: "center", justifyContent: "center", color: "#fff", fontWeight: 700, fontSize: 13 }}>{c.avatar}</div>
                      {c.status === "ai_replied" && <div style={{ position: "absolute", bottom: 0, right: 0, width: 12, height: 12, background: t.green, borderRadius: "50%", border: `2px solid ${t.surface}` }} />}
                    </div>
                    <div style={{ flex: 1, minWidth: 0 }}>
                      <div style={{ display: "flex", justifyContent: "space-between", alignItems: "baseline" }}>
                        <span style={{ fontWeight: 600, fontSize: 13 }}>{c.name}</span>
                        <span style={{ fontSize: 11, color: t.dim }}>{c.time}</span>
                      </div>
                      <div style={{ fontSize: 12, color: t.muted, overflow: "hidden", textOverflow: "ellipsis", whiteSpace: "nowrap" }}>{c.lastMsg}</div>
                    </div>
                    {c.unread > 0 && <div style={{ background: t.accent, color: "#fff", borderRadius: "50%", width: 18, height: 18, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 10, fontWeight: 700, flexShrink: 0 }}>{c.unread}</div>}
                  </div>
                ))}
              </div>
            </div>

            {/* Chat Area */}
            <div style={{ flex: 1, display: "flex", flexDirection: "column", background: t.bg }}>
              {/* Chat Header */}
              {(() => {
                const conv = conversations.find(c => c.id === selectedConv);
                return (
                  <div style={{ padding: "14px 20px", borderBottom: `1px solid ${t.border}`, display: "flex", alignItems: "center", justifyContent: "space-between", background: t.surface }}>
                    <div style={{ display: "flex", alignItems: "center", gap: 12 }}>
                      <div style={{ width: 38, height: 38, borderRadius: "50%", background: `linear-gradient(135deg, ${t.accent}, ${t.pink})`, display: "flex", alignItems: "center", justifyContent: "center", color: "#fff", fontWeight: 700, fontSize: 13 }}>{conv?.avatar}</div>
                      <div>
                        <div style={{ fontWeight: 700, fontSize: 14 }}>{conv?.name}</div>
                        <div style={{ fontSize: 12, color: t.green }}>● Active now</div>
                      </div>
                    </div>
                    <div style={{ display: "flex", gap: 8 }}>
                      <Badge theme={t} color={aiEnabled ? "accent" : "blue"}>AI {aiEnabled ? "ON" : "OFF"}</Badge>
                      <Toggle value={aiEnabled} onChange={setAiEnabled} theme={t} />
                    </div>
                  </div>
                );
              })()}

              {/* Messages */}
              <div style={{ flex: 1, overflowY: "auto", padding: "20px 24px", display: "flex", flexDirection: "column", gap: 12 }}>
                {(messages[selectedConv] || []).map(msg => (
                  <div key={msg.id} style={{ display: "flex", flexDirection: msg.from === "user" ? "row" : "row-reverse", gap: 8, alignItems: "flex-end" }}>
                    {msg.from === "user" && <div style={{ width: 28, height: 28, borderRadius: "50%", background: `linear-gradient(135deg, ${t.blue}, ${t.pink})`, flexShrink: 0, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 10, color: "#fff", fontWeight: 700 }}>U</div>}
                    <div style={{ maxWidth: "65%" }}>
                      <div style={{
                        padding: "10px 14px", borderRadius: msg.from === "user" ? "16px 16px 16px 4px" : "16px 16px 4px 16px",
                        background: msg.from === "user" ? (isDark ? "#1e1e30" : "#e0e7ff") : msg.ai ? `linear-gradient(135deg, ${t.accent}cc, ${t.accentGlow}cc)` : t.card,
                        color: msg.from !== "user" && msg.ai ? "#fff" : t.text,
                        fontSize: 14, lineHeight: 1.5, boxShadow: msg.ai ? `0 4px 16px ${t.accent}44` : "none"
                      }}>
                        {msg.text}
                        {msg.ai && <div style={{ fontSize: 10, opacity: 0.7, marginTop: 4 }}>🤖 AI Reply</div>}
                      </div>
                      <div style={{ fontSize: 10, color: t.dim, marginTop: 3, textAlign: msg.from !== "user" ? "left" : "right" }}>{msg.time}</div>
                    </div>
                  </div>
                ))}
                {aiGenerating && (
                  <div style={{ display: "flex", flexDirection: "row-reverse", gap: 8 }}>
                    <div style={{ padding: "12px 16px", background: t.card, borderRadius: "16px 16px 4px 16px", fontSize: 14, color: t.muted }}>
                      <span style={{ animation: "pulse 1.2s infinite" }}>🤖 AI is typing...</span>
                    </div>
                  </div>
                )}
                <div ref={msgEndRef} />
              </div>

              {/* Input */}
              <div style={{ padding: "12px 20px", borderTop: `1px solid ${t.border}`, display: "flex", gap: 10, background: t.surface }}>
                <input value={inputMsg} onChange={e => setInputMsg(e.target.value)}
                  onKeyDown={e => e.key === "Enter" && sendMessage()}
                  placeholder="Type a message (AI will auto-reply)..."
                  style={{ flex: 1, padding: "10px 16px", borderRadius: 12, background: t.card, border: `1px solid ${t.border}`, color: t.text, fontSize: 14 }} />
                <button className="btn-primary" onClick={sendMessage} style={{
                  background: `linear-gradient(135deg, ${t.accent}, ${t.accentGlow})`,
                  border: "none", borderRadius: 12, padding: "10px 20px", color: "#fff",
                  fontWeight: 700, cursor: "pointer", fontSize: 14, transition: "all 0.2s",
                }}>Send</button>
              </div>
            </div>
          </div>
        )}

        {/* ── AUTOMATION ── */}
        {page === "automation" && (
          <div style={{ animation: "fadeIn 0.3s ease" }}>
            <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 28 }}>
              <div>
                <h1 style={{ fontFamily: "'Syne', sans-serif", fontSize: 26, fontWeight: 800 }}>Automation Rules</h1>
                <p style={{ color: t.muted, fontSize: 13, marginTop: 4 }}>Keyword triggers that auto-reply instantly to incoming DMs</p>
              </div>
              <button className="btn-primary" onClick={() => setShowAddAutomation(true)} style={{
                background: `linear-gradient(135deg, ${t.accent}, ${t.accentGlow})`,
                border: "none", borderRadius: 12, padding: "10px 20px", color: "#fff",
                fontWeight: 700, cursor: "pointer", fontSize: 14, transition: "all 0.2s",
                boxShadow: `0 4px 16px ${t.accent}44`,
              }}>+ New Rule</button>
            </div>

            {/* Add Automation Modal */}
            {showAddAutomation && (
              <div style={{ position: "fixed", inset: 0, background: "#000a", zIndex: 200, display: "flex", alignItems: "center", justifyContent: "center" }}>
                <div style={{ background: t.card, border: `1px solid ${t.border}`, borderRadius: 20, padding: 28, width: 480, animation: "slideIn 0.3s ease" }}>
                  <h3 style={{ fontFamily: "'Syne', sans-serif", fontWeight: 800, fontSize: 18, marginBottom: 20 }}>Create Automation Rule</h3>
                  <div style={{ marginBottom: 16 }}>
                    <label style={{ fontSize: 12, fontWeight: 600, color: t.muted, letterSpacing: 1, textTransform: "uppercase" }}>Trigger Keyword</label>
                    <input value={newKeyword} onChange={e => setNewKeyword(e.target.value)} placeholder="e.g. PRICING, COLLAB, LINK"
                      style={{ width: "100%", padding: "10px 14px", borderRadius: 10, background: t.bg, border: `1px solid ${t.border}`, color: t.text, fontSize: 14, marginTop: 6 }} />
                  </div>
                  <div style={{ marginBottom: 20 }}>
                    <label style={{ fontSize: 12, fontWeight: 600, color: t.muted, letterSpacing: 1, textTransform: "uppercase" }}>Auto Reply Message</label>
                    <textarea value={newReply} onChange={e => setNewReply(e.target.value)} placeholder="Type the reply to send when keyword is detected..."
                      rows={4} style={{ width: "100%", padding: "10px 14px", borderRadius: 10, background: t.bg, border: `1px solid ${t.border}`, color: t.text, fontSize: 14, marginTop: 6, resize: "none" }} />
                  </div>
                  <div style={{ display: "flex", gap: 10, justifyContent: "flex-end" }}>
                    <button className="btn-ghost" onClick={() => setShowAddAutomation(false)} style={{ background: "transparent", border: `1px solid ${t.border}`, color: t.muted, borderRadius: 10, padding: "8px 18px", cursor: "pointer", fontWeight: 600, transition: "all 0.15s" }}>Cancel</button>
                    <button className="btn-primary" onClick={addAutomation} style={{ background: `linear-gradient(135deg, ${t.accent}, ${t.accentGlow})`, border: "none", borderRadius: 10, padding: "8px 20px", color: "#fff", fontWeight: 700, cursor: "pointer", transition: "all 0.2s" }}>Create Rule</button>
                  </div>
                </div>
              </div>
            )}

            <div style={{ background: t.card, border: `1px solid ${t.border}`, borderRadius: 16, overflow: "hidden" }}>
              <div style={{ display: "grid", gridTemplateColumns: "1fr 2fr 80px 80px 60px", gap: 0, padding: "12px 20px", borderBottom: `1px solid ${t.border}`, fontSize: 11, fontWeight: 700, color: t.dim, letterSpacing: 1, textTransform: "uppercase" }}>
                <div>Keyword</div><div>Reply Preview</div><div>Triggered</div><div>Status</div><div></div>
              </div>
              {automations.map(a => (
                <div key={a.id} className="automation-row" style={{
                  display: "grid", gridTemplateColumns: "1fr 2fr 80px 80px 60px",
                  padding: "16px 20px", borderBottom: `1px solid ${t.border}`, alignItems: "center", transition: "background 0.15s"
                }}>
                  <div>
                    <span style={{ background: t.accentSoft, color: t.accentGlow, padding: "4px 10px", borderRadius: 8, fontSize: 13, fontWeight: 700 }}>
                      ⚡ {a.keyword}
                    </span>
                  </div>
                  <div style={{ fontSize: 13, color: t.muted, overflow: "hidden", textOverflow: "ellipsis", whiteSpace: "nowrap", paddingRight: 16 }}>{a.reply}</div>
                  <div style={{ fontSize: 14, fontWeight: 700 }}>{a.triggered}</div>
                  <div><Toggle value={a.status} onChange={v => setAutomations(prev => prev.map(x => x.id === a.id ? { ...x, status: v } : x))} theme={t} /></div>
                  <div>
                    <button onClick={() => { setAutomations(prev => prev.filter(x => x.id !== a.id)); notify("Rule deleted", "error"); }}
                      style={{ background: "transparent", border: "none", color: t.dim, cursor: "pointer", fontSize: 16, padding: 4 }}>🗑</button>
                  </div>
                </div>
              ))}
            </div>

            {/* Flow Builder Teaser */}
            <div style={{ marginTop: 20, background: t.card, border: `1px dashed ${t.border}`, borderRadius: 16, padding: 28, textAlign: "center" }}>
              <div style={{ fontSize: 32, marginBottom: 8 }}>🔀</div>
              <div style={{ fontWeight: 700, fontSize: 16, marginBottom: 4 }}>Visual Flow Builder</div>
              <div style={{ color: t.muted, fontSize: 13, marginBottom: 14 }}>Build multi-step conversation flows with conditions and delays</div>
              <span style={{ background: t.accentSoft, color: t.accent, padding: "6px 16px", borderRadius: 20, fontSize: 12, fontWeight: 700 }}>PRO FEATURE — Upgrade to unlock</span>
            </div>
          </div>
        )}

        {/* ── AI ENGINE ── */}
        {page === "ai" && (
          <div style={{ animation: "fadeIn 0.3s ease", maxWidth: 800 }}>
            <h1 style={{ fontFamily: "'Syne', sans-serif", fontSize: 26, fontWeight: 800, marginBottom: 6 }}>AI Engine</h1>
            <p style={{ color: t.muted, fontSize: 14, marginBottom: 28 }}>Powered by Claude — intelligent, context-aware DM automation</p>

            {/* AI Status */}
            <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: 16, marginBottom: 24 }}>
              <div style={{ background: t.card, border: `1px solid ${t.border}`, borderRadius: 16, padding: 20 }}>
                <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 12 }}>
                  <span style={{ fontWeight: 700 }}>AI Auto-Reply</span>
                  <Toggle value={aiEnabled} onChange={setAiEnabled} theme={t} />
                </div>
                <p style={{ color: t.muted, fontSize: 12, lineHeight: 1.5 }}>When enabled, AI generates contextual replies for all incoming DMs that don't match keyword rules.</p>
              </div>
              <div style={{ background: t.card, border: `1px solid ${t.border}`, borderRadius: 16, padding: 20 }}>
                <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 8 }}>
                  <span style={{ fontWeight: 700 }}>AI Model</span>
                  <Badge theme={t} color="green">Active</Badge>
                </div>
                <div style={{ fontSize: 13, color: t.accent, fontWeight: 600 }}>Claude Sonnet 4</div>
                <p style={{ color: t.muted, fontSize: 12, marginTop: 4 }}>Fast, accurate, cost-effective for high-volume DMs</p>
              </div>
            </div>

            {/* Live AI Tester */}
            <div style={{ background: t.card, border: `1px solid ${t.border}`, borderRadius: 16, padding: 24, marginBottom: 20 }}>
              <h3 style={{ fontWeight: 700, fontSize: 15, marginBottom: 4 }}>🧪 Live AI Tester</h3>
              <p style={{ color: t.muted, fontSize: 13, marginBottom: 16 }}>Simulate an incoming DM and see what AI would reply in real-time</p>
              <div style={{ display: "flex", gap: 10, marginBottom: 14 }}>
                <input value={aiTestInput} onChange={e => setAiTestInput(e.target.value)}
                  onKeyDown={e => e.key === "Enter" && testAI()}
                  placeholder="Enter a test DM message..."
                  style={{ flex: 1, padding: "10px 14px", borderRadius: 10, background: t.bg, border: `1px solid ${t.border}`, color: t.text, fontSize: 14 }} />
                <button className="btn-primary" onClick={testAI} disabled={aiTesting} style={{
                  background: `linear-gradient(135deg, ${t.accent}, ${t.accentGlow})`,
                  border: "none", borderRadius: 10, padding: "10px 20px", color: "#fff",
                  fontWeight: 700, cursor: aiTesting ? "not-allowed" : "pointer", fontSize: 14,
                  opacity: aiTesting ? 0.7 : 1, transition: "all 0.2s",
                }}>{aiTesting ? "Generating..." : "Test AI →"}</button>
              </div>
              {aiTestOutput && (
                <div style={{ padding: "14px 16px", background: `linear-gradient(135deg, ${t.accent}22, ${t.accentGlow}11)`, border: `1px solid ${t.accent}44`, borderRadius: 12 }}>
                  <div style={{ fontSize: 11, color: t.accent, fontWeight: 700, marginBottom: 6 }}>🤖 AI REPLY PREVIEW</div>
                  <div style={{ fontSize: 14, lineHeight: 1.6 }}>{aiTestOutput}</div>
                </div>
              )}
            </div>

            {/* Prompt Customization */}
            <div style={{ background: t.card, border: `1px solid ${t.border}`, borderRadius: 16, padding: 24 }}>
              <h3 style={{ fontWeight: 700, fontSize: 15, marginBottom: 4 }}>System Prompt</h3>
              <p style={{ color: t.muted, fontSize: 13, marginBottom: 14 }}>Tell the AI about your brand voice, products, and how to handle conversations</p>
              <textarea defaultValue="You are a helpful assistant for my Instagram account. I am a content creator / agency offering video editing, content creation, and social media growth services. Be friendly, professional, and always end with a soft CTA. Use 1-2 emojis per message. Reply in the same language as the user."
                rows={5} style={{ width: "100%", padding: "12px 14px", borderRadius: 10, background: t.bg, border: `1px solid ${t.border}`, color: t.text, fontSize: 14, resize: "none", lineHeight: 1.6 }} />
              <button className="btn-primary" onClick={() => notify("Prompt saved!")} style={{
                marginTop: 12, background: `linear-gradient(135deg, ${t.accent}, ${t.accentGlow})`,
                border: "none", borderRadius: 10, padding: "9px 20px", color: "#fff", fontWeight: 700, cursor: "pointer", fontSize: 14
              }}>Save Prompt</button>
            </div>
          </div>
        )}

        {/* ── BILLING ── */}
        {page === "billing" && (
          <div style={{ animation: "fadeIn 0.3s ease", maxWidth: 900 }}>
            <h1 style={{ fontFamily: "'Syne', sans-serif", fontSize: 26, fontWeight: 800, marginBottom: 6 }}>Billing & Plans</h1>
            <p style={{ color: t.muted, fontSize: 14, marginBottom: 28 }}>Choose the plan that scales with your growth</p>

            <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr 1fr", gap: 16 }}>
              {[
                { name: "Free", price: "₹0", period: "/mo", color: t.dim, features: ["500 AI replies/mo", "2 keyword rules", "1 IG account", "Basic analytics"], cta: "Current Plan", active: false, highlight: false },
                { name: "Pro", price: "₹999", period: "/mo", color: t.accent, features: ["10,000 AI replies/mo", "Unlimited rules", "3 IG accounts", "Advanced analytics", "Priority support"], cta: "Current Plan", active: true, highlight: true },
                { name: "Premium", price: "₹2,499", period: "/mo", color: t.pink, features: ["Unlimited AI replies", "Unlimited rules", "10 IG accounts", "Full analytics suite", "Visual flow builder", "White-label option"], cta: "Upgrade", active: false, highlight: false },
              ].map(p => (
                <div key={p.name} style={{
                  background: p.highlight ? `linear-gradient(160deg, ${t.accentSoft}, ${t.card})` : t.card,
                  border: `1px solid ${p.highlight ? t.accent : t.border}`,
                  borderRadius: 20, padding: 24, position: "relative",
                  boxShadow: p.highlight ? `0 8px 40px ${t.accent}33` : "none",
                  animation: p.highlight ? "glow 3s infinite" : "none",
                }}>
                  {p.highlight && <div style={{ position: "absolute", top: -10, right: 20, background: `linear-gradient(135deg, ${t.accent}, ${t.pink})`, color: "#fff", fontSize: 10, fontWeight: 700, padding: "4px 12px", borderRadius: 20, letterSpacing: 1 }}>MOST POPULAR</div>}
                  <div style={{ fontSize: 16, fontWeight: 800, marginBottom: 4 }}>{p.name}</div>
                  <div style={{ fontFamily: "'Syne', sans-serif", fontSize: 36, fontWeight: 800, color: p.color }}>
                    {p.price}<span style={{ fontSize: 14, color: t.muted, fontFamily: "DM Sans" }}>{p.period}</span>
                  </div>
                  <div style={{ margin: "20px 0", borderTop: `1px solid ${t.border}` }} />
                  {p.features.map(f => (
                    <div key={f} style={{ display: "flex", gap: 8, alignItems: "center", marginBottom: 10, fontSize: 13 }}>
                      <span style={{ color: t.green }}>✓</span>
                      <span style={{ color: t.muted }}>{f}</span>
                    </div>
                  ))}
                  <button onClick={() => { if (p.cta === "Upgrade") { setPlan(p.name.toLowerCase()); notify(`Upgraded to ${p.name}!`); } }}
                    className={p.active ? "" : "btn-primary"}
                    style={{
                      marginTop: 16, width: "100%", padding: "11px", borderRadius: 12,
                      background: p.active ? "transparent" : `linear-gradient(135deg, ${p.color === t.pink ? t.pink : t.accent}, ${t.accentGlow})`,
                      border: p.active ? `1px solid ${t.border}` : "none",
                      color: p.active ? t.muted : "#fff", fontWeight: 700, fontSize: 14,
                      cursor: p.active ? "default" : "pointer", transition: "all 0.2s",
                    }}>{p.active ? "✓ Active Plan" : p.cta}</button>
                </div>
              ))}
            </div>

            <div style={{ marginTop: 24, background: t.card, border: `1px solid ${t.border}`, borderRadius: 16, padding: 20 }}>
              <div style={{ fontWeight: 700, marginBottom: 8 }}>Payment Methods</div>
              <div style={{ display: "flex", gap: 10 }}>
                <div style={{ background: t.bg, border: `1px solid ${t.border}`, borderRadius: 10, padding: "8px 16px", fontSize: 13, display: "flex", gap: 6, alignItems: "center" }}>
                  <span>💳</span> <span style={{ color: t.muted }}>Razorpay (UPI / Cards / NetBanking)</span>
                </div>
                <div style={{ background: t.bg, border: `1px solid ${t.border}`, borderRadius: 10, padding: "8px 16px", fontSize: 13, display: "flex", gap: 6, alignItems: "center" }}>
                  <span>🌐</span> <span style={{ color: t.muted }}>Stripe (International)</span>
                </div>
              </div>
            </div>
          </div>
        )}

        {/* ── SETTINGS ── */}
        {page === "settings" && (
          <div style={{ animation: "fadeIn 0.3s ease", maxWidth: 700 }}>
            <h1 style={{ fontFamily: "'Syne', sans-serif", fontSize: 26, fontWeight: 800, marginBottom: 28 }}>Settings</h1>

            {/* Instagram Connection */}
            <div style={{ background: t.card, border: `1px solid ${t.border}`, borderRadius: 16, padding: 24, marginBottom: 16 }}>
              <h3 style={{ fontWeight: 700, fontSize: 15, marginBottom: 16 }}>📱 Instagram Accounts</h3>
              <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", padding: "12px 16px", background: t.bg, borderRadius: 12, marginBottom: 10 }}>
                <div style={{ display: "flex", gap: 12, alignItems: "center" }}>
                  <div style={{ width: 36, height: 36, borderRadius: "50%", background: `linear-gradient(135deg, #f09433, #e6683c, #dc2743, #cc2366)`, display: "flex", alignItems: "center", justifyContent: "center", color: "#fff", fontSize: 16 }}>📸</div>
                  <div>
                    <div style={{ fontWeight: 600, fontSize: 14 }}>@gigamerge.studio</div>
                    <div style={{ fontSize: 11, color: t.green }}>● Connected · 12.4K followers</div>
                  </div>
                </div>
                <div style={{ display: "flex", gap: 8 }}>
                  <Badge theme={t} color="green">Active</Badge>
                  <button style={{ background: "transparent", border: `1px solid ${t.border}`, color: t.dim, borderRadius: 8, padding: "4px 10px", cursor: "pointer", fontSize: 12 }}>Disconnect</button>
                </div>
              </div>
              <button className="btn-ghost" style={{ background: "transparent", border: `1px dashed ${t.border}`, color: t.muted, borderRadius: 12, padding: "10px 16px", cursor: "pointer", fontSize: 13, width: "100%", fontWeight: 600, transition: "all 0.15s" }}>
                + Connect Another Instagram Account
              </button>
            </div>

            {/* API Settings */}
            <div style={{ background: t.card, border: `1px solid ${t.border}`, borderRadius: 16, padding: 24, marginBottom: 16 }}>
              <h3 style={{ fontWeight: 700, fontSize: 15, marginBottom: 16 }}>🔑 API Configuration</h3>
              {[
                { label: "OpenAI API Key", placeholder: "sk-••••••••••••••••••••••••" },
                { label: "Meta App ID", placeholder: "1234567890" },
                { label: "Meta App Secret", placeholder: "••••••••••••••••" },
                { label: "Webhook Verify Token", placeholder: "your_webhook_token" },
              ].map(field => (
                <div key={field.label} style={{ marginBottom: 14 }}>
                  <label style={{ fontSize: 12, fontWeight: 600, color: t.muted, letterSpacing: 0.5, textTransform: "uppercase" }}>{field.label}</label>
                  <input type="password" placeholder={field.placeholder}
                    style={{ width: "100%", padding: "9px 13px", borderRadius: 10, background: t.bg, border: `1px solid ${t.border}`, color: t.text, fontSize: 14, marginTop: 6 }} />
                </div>
              ))}
              <button className="btn-primary" onClick={() => notify("API keys saved securely!")} style={{
                background: `linear-gradient(135deg, ${t.accent}, ${t.accentGlow})`,
                border: "none", borderRadius: 10, padding: "9px 20px", color: "#fff", fontWeight: 700, cursor: "pointer", fontSize: 14
              }}>Save API Keys</button>
            </div>

            {/* Notification & Feature Toggles */}
            <div style={{ background: t.card, border: `1px solid ${t.border}`, borderRadius: 16, padding: 24 }}>
              <h3 style={{ fontWeight: 700, fontSize: 15, marginBottom: 16 }}>⚙️ Feature Toggles</h3>
              {[
                { label: "AI Auto-Reply", desc: "Let AI handle incoming DMs automatically", value: aiEnabled, set: setAiEnabled },
                { label: "Keyword Automation", desc: "Trigger predefined replies on keywords", value: true, set: () => {} },
                { label: "Email Notifications", desc: "Get alerts for unread messages", value: false, set: () => {} },
                { label: "Safety Moderation", desc: "Filter harmful content before AI replies", value: true, set: () => {} },
              ].map(item => (
                <div key={item.label} style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 16 }}>
                  <div>
                    <div style={{ fontSize: 14, fontWeight: 600 }}>{item.label}</div>
                    <div style={{ fontSize: 12, color: t.muted }}>{item.desc}</div>
                  </div>
                  <Toggle value={item.value} onChange={item.set} theme={t} />
                </div>
              ))}
            </div>
          </div>
        )}
      </main>

      {/* Footer */}
      {page !== "inbox" && (
        <footer style={{ padding: "12px 32px", borderTop: `1px solid ${t.border}`, display: "flex", justifyContent: "space-between", alignItems: "center", fontSize: 12, color: t.dim }}>
          <span>Autobot © 2025 — Built for GigaMerge Studio</span>
          <div style={{ display: "flex", gap: 16 }}>
            <span style={{ color: t.green }}>● System Operational</span>
            <span>v1.0.0-beta</span>
          </div>
        </footer>
      )}
    </div>
  );
}
