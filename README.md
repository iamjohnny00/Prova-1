import React, { useMemo, useState } from "react";
import {
  ResponsiveContainer,
  LineChart,
  Line,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  Legend,
} from "recharts";
import { Shield, ShieldCheck, ShieldOff, Euro, CalendarRange } from "lucide-react";

// ---------------------------------------------------------------------------
// Design tokens
// ---------------------------------------------------------------------------
const T = {
  paper: "#EDF0EC",
  paperRaised: "#F7F8F5",
  ink: "#182524",
  inkSoft: "#4B5C58",
  hair: "#D3D9D2",
  navy: "#1F3A3D",
  gold: "#A9812F",
  goldSoft: "#C9A85C",
  burgundy: "#7A3030",
  emerald: "#3B6E5C",
  clay: "#8C6A4F",
};

const serif = "'Iowan Old Style', 'Palatino Linotype', Georgia, serif";
const sans = "ui-sans-serif, system-ui, -apple-system, 'Segoe UI', sans-serif";

// ---------------------------------------------------------------------------
// Fixed product parameters (uguali per tutti i clienti)
// ---------------------------------------------------------------------------
const CARICAMENTO = 0.03; // 3% sul premio investibile
const COPERTURE = {
  nessuna: { label: "Nessuna", costo: 0, Icon: ShieldOff },
  light: { label: "Light", costo: 20, Icon: Shield },
  full: { label: "Full", costo: 30, Icon: ShieldCheck },
};

const GS_LORDO = 0.031;
const GS_TRATTENUTA = 0.01;
const GS_NETTO = GS_LORDO - GS_TRATTENUTA; // 2.1%

const SCENARI = [
  { key: "prudente", label: "Prudente", lordo: 0.04, trattenuta: 0.01, color: T.clay },
  { key: "realistico", label: "Realistico", lordo: 0.06, trattenuta: 0.01, color: T.emerald },
  { key: "favorevole", label: "Favorevole", lordo: 0.08, trattenuta: 0.01, color: T.gold },
];

const SPLIT_GS = 0.5;
const SPLIT_OICR = 0.5;

const fmt = (n) =>
  n.toLocaleString("it-IT", { minimumFractionDigits: 0, maximumFractionDigits: 0 });
const fmtDec = (n) =>
  n.toLocaleString("it-IT", { minimumFractionDigits: 2, maximumFractionDigits: 2 });

// ---------------------------------------------------------------------------
// Simulation
// ---------------------------------------------------------------------------
function simulate(importoMensile, coperturaKey, anni) {
  const costoCopertura = COPERTURE[coperturaKey].costo;
  const monthlyOICR = Object.fromEntries(
    SCENARI.map((s) => [s.key, Math.pow(1 + (s.lordo - s.trattenuta), 1 / 12) - 1])
  );
  // Fattore di rateo medio: un versamento mensile, distribuito uniformemente durante
  // l'anno, matura in media interesse come se fosse investito a metà anno (5,5/12).
  const PRORATA_GS = 5.5 / 12;

  let balGS = 0;
  const balOICR = Object.fromEntries(SCENARI.map((s) => [s.key, 0]));

  let premiLordiCum = 0;
  let premiNettiCum = 0;
  let costiCoperturaCum = 0;
  let caricamentiCum = 0;

  const yearly = [];

  for (let anno = 1; anno <= anni; anno++) {
    let gsContribAnno = 0;

    for (let m = 1; m <= 12; m++) {
      const premioInvestibile = importoMensile - costoCopertura;
      const caricamentoImporto = Math.max(premioInvestibile, 0) * CARICAMENTO;
      const premioNetto = Math.max(premioInvestibile - caricamentoImporto, 0);

      premiLordiCum += importoMensile;
      costiCoperturaCum += costoCopertura;
      caricamentiCum += caricamentoImporto;
      premiNettiCum += premioNetto;

      const contribGS = premioNetto * SPLIT_GS;
      const contribOICR = premioNetto * SPLIT_OICR;
      gsContribAnno += contribGS;

      // OICR: valorizzazione giornaliera -> capitalizzazione continua sul tasso annuo netto
      SCENARI.forEach((s) => {
        balOICR[s.key] = (balOICR[s.key] + contribOICR) * (1 + monthlyOICR[s.key]);
      });
    }

    // Gestione Separata: rivalutazione ANNUALE (art. 5 CGA).
    // Il capitale già presente matura il tasso pieno; i nuovi versamenti dell'anno
    // maturano un rateo pro-quota fino alla ricorrenza annuale.
    const interessePregresso = balGS * GS_NETTO;
    const interesseNuoviVersamenti = gsContribAnno * GS_NETTO * PRORATA_GS;
    balGS = balGS + interessePregresso + gsContribAnno + interesseNuoviVersamenti;

    const row = {
      anno,
      premiVersati: premiLordiCum,
      premiNetti: premiNettiCum,
      montanteGS: balGS,
    };
    SCENARI.forEach((s) => {
      row[s.key] = balGS + balOICR[s.key];
    });
    yearly.push(row);
  }

  return {
    yearly,
    totals: {
      premiLordiCum,
      premiNettiCum,
      costiCoperturaCum,
      caricamentiCum,
      balGS,
      balOICR,
    },
    premioInvestibileMensile: importoMensile - costoCopertura,
    caricamentoMensile: Math.max(importoMensile - costoCopertura, 0) * CARICAMENTO,
    premioNettoMensile:
      Math.max(importoMensile - costoCopertura, 0) * (1 - CARICAMENTO),
  };
}

// ---------------------------------------------------------------------------
// Bridge / flow bar — signature element showing where the monthly premium goes
// ---------------------------------------------------------------------------
function FlowBar({ importoMensile, coperturaKey }) {
  const costo = COPERTURE[coperturaKey].costo;
  const investibile = Math.max(importoMensile - costo, 0);
  const caricamento = investibile * CARICAMENTO;
  const netto = investibile - caricamento;

  const segs = [
    { label: "Copertura", val: costo, color: T.burgundy },
    { label: "Caricamento 3%", val: caricamento, color: T.goldSoft },
    { label: "Capitale investito", val: netto, color: T.emerald },
  ].filter((s) => s.val > 0.001);

  return (
    <div>
      <div className="flex items-baseline justify-between mb-2">
        <span style={{ fontFamily: sans, color: T.inkSoft, fontSize: 12, letterSpacing: "0.04em" }}>
          COMPOSIZIONE DEL PREMIO MENSILE — € {fmtDec(importoMensile)}
        </span>
      </div>
      <div className="w-full h-8 flex overflow-hidden" style={{ borderRadius: 3 }}>
        {segs.map((s, i) => (
          <div
            key={i}
            style={{
              width: `${(s.val / importoMensile) * 100}%`,
              background: s.color,
            }}
            title={`${s.label}: €${fmtDec(s.val)}`}
          />
        ))}
      </div>
      <div className="flex flex-wrap gap-x-6 gap-y-1 mt-3">
        {segs.map((s, i) => (
          <div key={i} className="flex items-center gap-2">
            <span style={{ width: 10, height: 10, background: s.color, display: "inline-block", borderRadius: 2 }} />
            <span style={{ fontFamily: sans, fontSize: 13, color: T.ink }}>
              {s.label}: <b>€ {fmtDec(s.val)}</b>
            </span>
          </div>
        ))}
      </div>
    </div>
  );
}

// ---------------------------------------------------------------------------
// Main
// ---------------------------------------------------------------------------
export default function SimulatorePianoAccumulo() {
  const [importoMensile, setImportoMensile] = useState(150);
  const [coperturaKey, setCoperturaKey] = useState("full");
  const [anni, setAnni] = useState(10);

  const result = useMemo(
    () => simulate(importoMensile, coperturaKey, anni),
    [importoMensile, coperturaKey, anni]
  );

  const lastRow = result.yearly[result.yearly.length - 1];

  return (
    <div style={{ background: T.paper, minHeight: "100%", fontFamily: sans, color: T.ink }} className="w-full p-5 sm:p-8">
      {/* Header */}
      <div className="mb-6 pb-5" style={{ borderBottom: `1px solid ${T.hair}` }}>
        <div style={{ fontFamily: sans, fontSize: 11, letterSpacing: "0.12em", color: T.gold, fontWeight: 600 }}>
          PIANO DI ACCUMULO — SIMULAZIONE
        </div>
        <h1 style={{ fontFamily: serif, fontSize: 30, color: T.navy, margin: "4px 0 0" }}>
          Proiezione del capitale nel tempo
        </h1>
        <p style={{ color: T.inkSoft, fontSize: 14, marginTop: 6, maxWidth: 640 }}>
          Imposta l'importo mensile e la copertura complementare per il cliente:
          costi, caricamenti e rendimenti sono calcolati automaticamente sui tre scenari.
        </p>
      </div>

      {/* Controls */}
      <div
        className="grid grid-cols-1 sm:grid-cols-3 gap-4 p-5 mb-6"
        style={{ background: T.paperRaised, border: `1px solid ${T.hair}`, borderRadius: 6 }}
      >
        <div>
          <label className="flex items-center gap-2 mb-2" style={{ fontSize: 12, letterSpacing: "0.05em", color: T.inkSoft }}>
            <Euro size={14} /> IMPORTO MENSILE
          </label>
          <div className="flex items-center gap-2">
            <span style={{ fontFamily: serif, fontSize: 20, color: T.navy }}>€</span>
            <input
              type="number"
              min={20}
              step={10}
              value={importoMensile}
              onChange={(e) => setImportoMensile(Math.max(0, Number(e.target.value)))}
              style={{
                fontFamily: serif,
                fontSize: 22,
                color: T.navy,
                background: "transparent",
                border: "none",
                borderBottom: `2px solid ${T.navy}`,
                width: 120,
                outline: "none",
              }}
            />
          </div>
          <input
            type="range"
            min={20}
            max={1000}
            step={10}
            value={importoMensile}
            onChange={(e) => setImportoMensile(Number(e.target.value))}
            className="w-full mt-3"
          />
        </div>

        <div>
          <label className="flex items-center gap-2 mb-2" style={{ fontSize: 12, letterSpacing: "0.05em", color: T.inkSoft }}>
            <Shield size={14} /> COPERTURA COMPLEMENTARE
          </label>
          <div className="flex gap-2">
            {Object.entries(COPERTURE).map(([key, c]) => {
              const active = coperturaKey === key;
              const Icon = c.Icon;
              return (
                <button
                  key={key}
                  onClick={() => setCoperturaKey(key)}
                  className="flex-1 flex flex-col items-center gap-1 py-2"
                  style={{
                    borderRadius: 5,
                    border: `1px solid ${active ? T.navy : T.hair}`,
                    background: active ? T.navy : "transparent",
                    color: active ? T.paperRaised : T.ink,
                    cursor: "pointer",
                  }}
                >
                  <Icon size={16} />
                  <span style={{ fontSize: 12 }}>{c.label}</span>
                  <span style={{ fontSize: 11, opacity: 0.8 }}>
                    {c.costo === 0 ? "—" : `€ ${c.costo}/mese`}
                  </span>
                </button>
              );
            })}
          </div>
        </div>

        <div>
          <label className="flex items-center gap-2 mb-2" style={{ fontSize: 12, letterSpacing: "0.05em", color: T.inkSoft }}>
            <CalendarRange size={14} /> ORIZZONTE TEMPORALE
          </label>
          <div className="flex gap-2 flex-wrap">
            {Array.from({ length: 6 }, (_, i) => i + 5).map((y) => (
              <button
                key={y}
                onClick={() => setAnni(y)}
                style={{
                  padding: "6px 12px",
                  borderRadius: 5,
                  border: `1px solid ${anni === y ? T.navy : T.hair}`,
                  background: anni === y ? T.navy : "transparent",
                  color: anni === y ? T.paperRaised : T.ink,
                  fontSize: 13,
                  cursor: "pointer",
                }}
              >
                {y} anni
              </button>
            ))}
          </div>
        </div>
      </div>

      {/* Flow bar */}
      <div className="p-5 mb-6" style={{ background: T.paperRaised, border: `1px solid ${T.hair}`, borderRadius: 6 }}>
        <FlowBar importoMensile={importoMensile} coperturaKey={coperturaKey} />
        <div className="mt-4 pt-4 grid grid-cols-2 sm:grid-cols-4 gap-4" style={{ borderTop: `1px solid ${T.hair}` }}>
          <Stat label="50% Gestione separata" value={`3,1% lordo · 2,1% netto`} sub="rivalutazione annuale" />
          <Stat label="50% OICR — Prudente" value="4% lordo · 3% netto" sub="valorizzazione giornaliera" color={T.clay} />
          <Stat label="50% OICR — Realistico" value="6% lordo · 5% netto" sub="valorizzazione giornaliera" color={T.emerald} />
          <Stat label="50% OICR — Favorevole" value="8% lordo · 7% netto" sub="valorizzazione giornaliera" color={T.gold} />
        </div>
      </div>

      {/* Summary at horizon end */}
      <div className="grid grid-cols-2 sm:grid-cols-5 gap-3 mb-6">
        <BigStat label={`Premi versati (${anni} anni)`} value={`€ ${fmt(lastRow.premiVersati)}`} />
        <BigStat label="Capitale netto investito" value={`€ ${fmt(lastRow.premiNetti)}`} />
        {SCENARI.map((s) => (
          <BigStat
            key={s.key}
            label={`Montante — ${s.label}`}
            value={`€ ${fmt(lastRow[s.key])}`}
            color={s.color}
            emphasis
          />
        ))}
      </div>

      {/* Chart */}
      <div className="p-5 mb-6" style={{ background: T.paperRaised, border: `1px solid ${T.hair}`, borderRadius: 6 }}>
        <h2 style={{ fontFamily: serif, fontSize: 18, color: T.navy, marginBottom: 12 }}>
          Crescita del capitale, anno per anno
        </h2>
        <ResponsiveContainer width="100%" height={340}>
          <LineChart data={result.yearly} margin={{ top: 5, right: 15, left: 0, bottom: 5 }}>
            <CartesianGrid stroke={T.hair} strokeDasharray="3 3" vertical={false} />
            <XAxis
              dataKey="anno"
              tickFormatter={(a) => `Anno ${a}`}
              stroke={T.inkSoft}
              tick={{ fontSize: 12, fontFamily: sans }}
            />
            <YAxis
              tickFormatter={(v) => `€${Math.round(v / 1000)}k`}
              stroke={T.inkSoft}
              tick={{ fontSize: 12, fontFamily: sans }}
              width={55}
            />
            <Tooltip
              formatter={(v) => `€ ${fmtDec(v)}`}
              labelFormatter={(a) => `Anno ${a}`}
              contentStyle={{ fontFamily: sans, fontSize: 13, borderRadius: 4, border: `1px solid ${T.hair}` }}
            />
            <Legend wrapperStyle={{ fontFamily: sans, fontSize: 13, paddingTop: 10 }} />
            <Line
              type="monotone"
              dataKey="premiVersati"
              name="Premi versati"
              stroke={T.inkSoft}
              strokeDasharray="4 3"
              strokeWidth={1.5}
              dot={false}
            />
            {SCENARI.map((s) => (
              <Line
                key={s.key}
                type="monotone"
                dataKey={s.key}
                name={s.label}
                stroke={s.color}
                strokeWidth={2.5}
                dot={{ r: 3 }}
              />
            ))}
          </LineChart>
        </ResponsiveContainer>
      </div>

      {/* Table */}
      <div className="p-5" style={{ background: T.paperRaised, border: `1px solid ${T.hair}`, borderRadius: 6 }}>
        <h2 style={{ fontFamily: serif, fontSize: 18, color: T.navy, marginBottom: 12 }}>
          Dettaglio anno per anno
        </h2>
        <div className="overflow-x-auto">
          <table className="w-full" style={{ fontSize: 13, borderCollapse: "collapse" }}>
            <thead>
              <tr style={{ borderBottom: `1px solid ${T.hair}`, color: T.inkSoft, textAlign: "right" }}>
                <th style={{ textAlign: "left", padding: "6px 8px", fontWeight: 500 }}>Anno</th>
                <th style={{ padding: "6px 8px", fontWeight: 500 }}>Premi versati</th>
                <th style={{ padding: "6px 8px", fontWeight: 500 }}>Capitale netto investito</th>
                {SCENARI.map((s) => (
                  <th key={s.key} style={{ padding: "6px 8px", fontWeight: 500, color: s.color }}>
                    {s.label}
                  </th>
                ))}
              </tr>
            </thead>
            <tbody>
              {result.yearly.map((row) => (
                <tr key={row.anno} style={{ borderBottom: `1px solid ${T.hair}`, textAlign: "right" }}>
                  <td style={{ textAlign: "left", padding: "6px 8px", fontFamily: serif, color: T.navy }}>
                    {row.anno}
                  </td>
                  <td style={{ padding: "6px 8px", fontVariantNumeric: "tabular-nums" }}>
                    € {fmt(row.premiVersati)}
                  </td>
                  <td style={{ padding: "6px 8px", fontVariantNumeric: "tabular-nums" }}>
                    € {fmt(row.premiNetti)}
                  </td>
                  {SCENARI.map((s) => (
                    <td
                      key={s.key}
                      style={{
                        padding: "6px 8px",
                        fontVariantNumeric: "tabular-nums",
                        fontWeight: 600,
                        color: T.ink,
                      }}
                    >
                      € {fmt(row[s.key])}
                    </td>
                  ))}
                </tr>
              ))}
            </tbody>
          </table>
        </div>
        <p style={{ color: T.inkSoft, fontSize: 11, marginTop: 12 }}>
          Simulazione a scopo illustrativo. Caricamento 3% sul premio al netto della copertura.
          Gestione Separata: rivalutazione annuale, 3,1% lordo / 2,1% netto; i versamenti
          dell'anno maturano un rateo pro-quota fino alla ricorrenza annuale, poi si
          compongono col tasso pieno. OICR: valorizzazione giornaliera — Prudente 4%
          lordo/3% netto, Realistico 6% lordo/5% netto, Favorevole 8% lordo/7% netto.
          Split 50% GS / 50% OICR. Rendimenti passati o ipotizzati non sono indicativi
          di rendimenti futuri.
        </p>
      </div>
    </div>
  );
}

function Stat({ label, value, sub, color }) {
  return (
    <div>
      <div style={{ fontSize: 11, color: T.inkSoft, letterSpacing: "0.03em" }}>{label}</div>
      <div style={{ fontFamily: serif, fontSize: 15, color: color || T.navy, marginTop: 2 }}>{value}</div>
    </div>
  );
}

function BigStat({ label, value, color, emphasis }) {
  return (
    <div
      className="p-4"
      style={{
        background: emphasis ? T.navy : T.paperRaised,
        border: `1px solid ${emphasis ? T.navy : T.hair}`,
        borderRadius: 6,
      }}
    >
      <div style={{ fontSize: 11, color: emphasis ? "#C9D3D0" : T.inkSoft, letterSpacing: "0.03em" }}>
        {label}
      </div>
      <div
        style={{
          fontFamily: serif,
          fontSize: 20,
          color: emphasis ? (color || T.goldSoft) : T.navy,
          marginTop: 4,
        }}
      >
        {value}
      </div>
    </div>
  );
}
