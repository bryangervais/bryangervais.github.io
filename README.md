import React, { useMemo, useState } from "react";

// NBC GARUDA Calculator – single-file React component
// Drop into a Vite/Next/CRA project, or a CodeSandbox. Tailwind recommended.
// Outputs 6 probabilities based on provided logistic-style formulas.

// Helpers
const clamp = (v, min, max) => (v === null || v === undefined || Number.isNaN(v) ? null : Math.min(Math.max(v, min), max));
const num = (v) => (v === "" || v === null || v === undefined ? null : Number(v));
const yesNoUnknown = [
  { label: "Unknown", value: "unknown" },
  { label: "No", value: 0 },
  { label: "Yes", value: 1 },
];

const wfnsOptions = [
  { label: "Unknown", value: "unknown" },
  { label: "I (1)", value: 1 },
  { label: "II (2)", value: 2 },
  { label: "III (3)", value: 3 },
  { label: "IV (4)", value: 4 },
  { label: "V (5)", value: 5 },
];

const locationOptions = [
  { label: "Unknown", value: "unknown" },
  { label: "AChA", value: "acha" },
  { label: "ACom", value: "acom" },
  { label: "Basilar", value: "basilar" },
  { label: "ICA", value: "ica" },
  { label: "MCA", value: "mca" },
  { label: "PCoA", value: "pcoa" },
  { label: "Other", value: "other" },
];

function sigmoid(z) {
  return 1 / (1 + Math.exp(-z));
}

function Field({ label, children, tooltip }) {
  return (
    <div className="flex flex-col gap-1">
      <div className="flex items-center gap-2">
        <label className="text-sm font-medium text-gray-800">{label}</label>
        {tooltip && (
          <span className="text-xs text-gray-500">{tooltip}</span>
        )}
      </div>
      {children}
    </div>
  );
}

function NumberInput({ value, onChange, placeholder, step = 1, min, max }) {
  return (
    <input
      type="number"
      value={value ?? ""}
      onChange={(e) => onChange(e.target.value === "" ? null : Number(e.target.value))}
      placeholder={placeholder}
      step={step}
      min={min}
      max={max}
      className="w-full rounded-xl border border-gray-300 px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-indigo-500"
    />
  );
}

function Select({ value, onChange, options }) {
  return (
    <select
      value={value ?? "unknown"}
      onChange={(e) => {
        const v = e.target.value;
        if (v === "unknown") onChange("unknown");
        else onChange(Number.isNaN(Number(v)) ? v : Number(v));
      }}
      className="w-full rounded-xl border border-gray-300 px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-indigo-500"
    >
      {options.map((opt) => (
        <option key={opt.value} value={opt.value}>
          {opt.label}
        </option>
      ))}
    </select>
  );
}

function Badge({ children, tone = "slate" }) {
  const map = {
    slate: "bg-slate-100 text-slate-800",
    green: "bg-green-100 text-green-800",
    red: "bg-red-100 text-red-800",
    amber: "bg-amber-100 text-amber-800",
    blue: "bg-blue-100 text-blue-800",
    violet: "bg-violet-100 text-violet-800",
  };
  return (
    <span className={`inline-flex items-center rounded-full px-2 py-0.5 text-xs font-medium ${map[tone]}`}>{children}</span>
  );
}

function Card({ title, children, footer, tone = "white" }) {
  const bg = tone === "white" ? "bg-white" : tone;
  return (
    <div className={`rounded-2xl ${bg} shadow-sm ring-1 ring-gray-200 p-4 md:p-6 flex flex-col gap-3`}>
      <div className="flex items-center justify-between">
        <h3 className="text-base md:text-lg font-semibold text-gray-900">{title}</h3>
      </div>
      <div className="text-sm md:text-base">{children}</div>
      {footer && <div className="pt-2">{footer}</div>}
    </div>
  );
}

export default function GarudaCalculator() {
  const [state, setState] = useState({
    age: null, // years
    htn: "unknown",
    cvd: "unknown",
    smoke: "unknown",
    famHist: "unknown",
    gcs: null, // 3-15
    wfns: "unknown",
    hemiparesis: "unknown",
    ruptured: "unknown",
    dm: "unknown",
    location: "unknown",
    daughter: "unknown",
    ptosis: "unknown",
    seizure: "unknown",
    multiple: "unknown",
    iom: "unknown",
    dome: null, // mm
    neck: null, // mm
  });

  const set = (key) => (v) => setState((s) => ({ ...s, [key]: v }));

  // Derived flags
  const gcsLT15 = useMemo(() => (state.gcs == null ? "unknown" : state.gcs <= 14 ? 1 : 0), [state.gcs]);
  const dnRatio = useMemo(() => (state.dome != null && state.neck != null && state.neck > 0 ? state.dome / state.neck : null), [state.dome, state.neck]);
  const neckGT4 = useMemo(() => (state.neck == null ? "unknown" : state.neck > 4 ? 1 : 0), [state.neck]);

  // Utility: get numeric value or 0 if unknown (and track missingness)
  function getVal(v) {
    if (v === "unknown" || v === null || v === undefined || Number.isNaN(v)) return { val: 0, missing: true };
    return { val: Number(v), missing: false };
  }

  // Location coefficients for Outcome 4
  function locCoef() {
    switch (state.location) {
      case "acha":
        return { coef: 1.272, missing: false };
      case "acom":
        return { coef: -0.608, missing: false };
      case "basilar":
        return { coef: 0.383, missing: false };
      case "ica":
        return { coef: -0.019, missing: false };
      case "mca":
        return { coef: -1.061, missing: false };
      case "pcoa":
        return { coef: -0.411, missing: false };
      case "other":
      case "unknown":
      default:
        return { coef: 0, missing: true };
    }
  }

  // Compute 6 outcomes
  function computeOutcomes() {
    const missing = new Set();

    const { val: age, missing: mAge } = getVal(state.age);
    if (mAge) missing.add("Age");

    const { val: htn, missing: mHtn } = getVal(state.htn);
    if (mHtn) missing.add("Hypertension");

    const { val: wfns, missing: mWfns } = getVal(state.wfns);
    if (mWfns) missing.add("WFNS grade");

    const { val: dome, missing: mDome } = getVal(state.dome);
    if (mDome) missing.add("Dome height (mm)");

    const { val: cvd, missing: mCvd } = getVal(state.cvd);
    if (mCvd) missing.add("CVD");

    const { val: smoke, missing: mSmoke } = getVal(state.smoke);
    if (mSmoke) missing.add("Current smoking");

    const { val: fam, missing: mFam } = getVal(state.famHist);
    if (mFam) missing.add("Family history");

    const { val: gcs15, missing: mGcs } = getVal(gcsLT15);
    if (mGcs) missing.add("GCS < 15");

    const { val: iom, missing: mIom } = getVal(state.iom);
    if (mIom) missing.add("IOM used");

    const { val: dn, missing: mDn } = getVal(dnRatio);
    if (mDn) missing.add("Dome/Neck ratio");

    const { val: hemi, missing: mHemi } = getVal(state.hemiparesis);
    if (mHemi) missing.add("Hemiparesis");

    const { val: rupt, missing: mRupt } = getVal(state.ruptured);
    if (mRupt) missing.add("Ruptured aneurysm");

    const { val: dm, missing: mDm } = getVal(state.dm);
    if (mDm) missing.add("Diabetes");

    const { coef: lcoef, missing: mLoc } = locCoef();
    if (mLoc) missing.add("Location of aneurysm");

    const { val: daughter, missing: mDaughter } = getVal(state.daughter);
    if (mDaughter) missing.add("Daughter aneurysm");

    const { val: ptosis, missing: mPtosis } = getVal(state.ptosis);
    if (mPtosis) missing.add("Ptosis");

    const { val: seizure, missing: mSeizure } = getVal(state.seizure);
    if (mSeizure) missing.add("Seizure");

    const { val: multiple, missing: mMultiple } = getVal(state.multiple);
    if (mMultiple) missing.add("Multiple aneurysms");

    const { val: neck4, missing: mNeck4 } = getVal(neckGT4);
    if (mNeck4) missing.add("Neck > 4 mm");

    // 1) Mortality after coiling
    const z1 = -5.073 + 0.022 * age + 1.008 * htn + 0.525 * wfns + 0.061 * dome;
    const p1 = sigmoid(z1);

    // 2) Mortality after clipping
    const z2 = -1.198 + (-0.019) * age + 1.678 * cvd + (-1.878) * smoke + 2.157 * fam + 1.653 * gcs15 + (-1.119) * iom + 0.059 * dome + (-0.233) * dn;
    const p2 = sigmoid(z2);

    // 3) Good GOSE after coiling
    const z3 = 2.19 + (-0.013) * age + (-0.626) * hemi + (-0.493) * wfns + (-0.703) * rupt;
    const p3 = sigmoid(z3);

    // 4) Good GOSE after clipping
    const z4 = 2.781 + (-0.043) * age + 1.439 * dm + (-1.363) * gcs15 + lcoef + (-0.856) * daughter;
    const p4 = sigmoid(z4);

    // 5) GCS recovery to baseline after coiling
    const z5 = 3.198 + (-0.033) * age + (-1.143) * dm + (-0.94) * ptosis + (-0.977) * seizure + 0.621 * multiple + (-0.029) * dome + (-0.232) * dn;
    const p5 = sigmoid(z5);

    // 6) GCS recovery to baseline after clipping
    const z6 = 3.334 + (-0.033) * age + (-1) * cvd + (-0.517) * gcs15 + 0.843 * iom + (-1.542) * rupt + (-0.179) * dome + 0.646 * neck4;
    const p6 = sigmoid(z6);

    const pct = (p) => (Number.isFinite(p) ? (p * 100).toFixed(1) + "%" : "–");

    // Track per-outcome missingness (only for variables each model uses)
    const missingByOutcome = [
      [mAge && "Age", mHtn && "Hypertension", mWfns && "WFNS", mDome && "Dome height"].filter(Boolean),
      [mAge && "Age", mCvd && "CVD", mSmoke && "Smoking", mFam && "Family history", mGcs && "GCS < 15", mIom && "IOM used", mDome && "Dome height", mDn && "Dome/Neck ratio"].filter(Boolean),
      [mAge && "Age", mHemi && "Hemiparesis", mWfns && "WFNS", mRupt && "Ruptured"].filter(Boolean),
      [mAge && "Age", mDm && "Diabetes", mGcs && "GCS < 15", mLoc && "Location", mDaughter && "Daughter sac"].filter(Boolean),
      [mAge && "Age", mDm && "Diabetes", mPtosis && "Ptosis", mSeizure && "Seizure", mMultiple && "Multiple aneurysms", mDome && "Dome height", mDn && "Dome/Neck ratio"].filter(Boolean),
      [mAge && "Age", mCvd && "CVD", mGcs && "GCS < 15", mIom && "IOM used", mRupt && "Ruptured", mDome && "Dome height", mNeck4 && ">4 mm neck"].filter(Boolean),
    ];

    return {
      probs: [p1, p2, p3, p4, p5, p6],
      logits: [z1, z2, z3, z4, z5, z6],
      percents: [pct(p1), pct(p2), pct(p3), pct(p4), pct(p5), pct(p6)],
      missingByOutcome,
    };
  }

  const results = useMemo(() => computeOutcomes(), [state, gcsLT15, dnRatio, neckGT4]);

  // UI bits
  const reset = () =>
    setState({
      age: null,
      htn: "unknown",
      cvd: "unknown",
      smoke: "unknown",
      famHist: "unknown",
      gcs: null,
      wfns: "unknown",
      hemiparesis: "unknown",
      ruptured: "unknown",
      dm: "unknown",
      location: "unknown",
      daughter: "unknown",
      ptosis: "unknown",
      seizure: "unknown",
      multiple: "unknown",
      iom: "unknown",
      dome: null,
      neck: null,
    });

  return (
    <div className="min-h-screen bg-gradient-to-b from-gray-50 to-white text-gray-900">
      <div className="mx-auto max-w-6xl px-4 py-6 md:py-10">
        <header className="mb-6 md:mb-10 flex flex-col gap-2">
          <h1 className="text-2xl md:text-3xl font-bold tracking-tight">NBC GARUDA – General Aneurysm Risk Utility for Decision Aid</h1>
          <p className="text-sm md:text-base text-gray-600">Enter all available data once. The calculator uses what each model needs. Unknowns are treated as 0 in the math; we flag any assumed values per outcome.</p>
        </header>

        <div className="grid grid-cols-1 lg:grid-cols-2 gap-4 md:gap-6">
          <Card title="Patient & Presentation">
            <div className="grid grid-cols-1 md:grid-cols-3 gap-3">
              <Field label="Age (years)"><NumberInput value={state.age} onChange={set("age")} placeholder="e.g., 58" min={0} max={120} /></Field>
              <Field label="GCS total (3–15)"><NumberInput value={state.gcs} onChange={set("gcs")} placeholder="15" min={3} max={15} /></Field>
              <Field label="WFNS grade"><Select value={state.wfns} onChange={set("wfns")} options={wfnsOptions} /></Field>

              <Field label="Hypertension"><Select value={state.htn} onChange={set("htn")} options={yesNoUnknown} /></Field>
              <Field label="Diabetes"><Select value={state.dm} onChange={set("dm")} options={yesNoUnknown} /></Field>
              <Field label="CVD (IHD/CHF/arrhythmia/PAD)"><Select value={state.cvd} onChange={set("cvd")} options={yesNoUnknown} /></Field>

              <Field label="Current smoking"><Select value={state.smoke} onChange={set("smoke")} options={yesNoUnknown} /></Field>
              <Field label="Family history (aneurysm/SAH)"><Select value={state.famHist} onChange={set("famHist")} options={yesNoUnknown} /></Field>
              <Field label="Ruptured aneurysm"><Select value={state.ruptured} onChange={set("ruptured")} options={yesNoUnknown} /></Field>

              <Field label="Hemiparesis"><Select value={state.hemiparesis} onChange={set("hemiparesis")} options={yesNoUnknown} /></Field>
              <Field label="Ptosis"><Select value={state.ptosis} onChange={set("ptosis")} options={yesNoUnknown} /></Field>
              <Field label="Seizure (pre-op)"><Select value={state.seizure} onChange={set("seizure")} options={yesNoUnknown} /></Field>

              <Field label="Multiple aneurysms"><Select value={state.multiple} onChange={set("multiple")} options={yesNoUnknown} /></Field>
              <Field label="IOM used"><Select value={state.iom} onChange={set("iom")} options={yesNoUnknown} /></Field>
              <Field label="Location of aneurysm"><Select value={state.location} onChange={set("location")} options={locationOptions} /></Field>
            </div>

            <div className="mt-3 flex flex-wrap items-center gap-2">
              <Badge tone={gcsLT15 === 1 ? "red" : gcsLT15 === 0 ? "green" : "amber"}>
                GCS &lt; 15: {gcsLT15 === 1 ? "Yes" : gcsLT15 === 0 ? "No" : "Unknown"}
              </Badge>
            </div>
          </Card>

          <Card title="Aneurysm Morphology">
            <div className="grid grid-cols-1 md:grid-cols-3 gap-3">
              <Field label="Dome height (mm)"><NumberInput value={state.dome} onChange={set("dome")} placeholder="e.g., 6.0" step={0.1} min={0} max={50} /></Field>
              <Field label="Neck width (mm)"><NumberInput value={state.neck} onChange={set("neck")} placeholder="e.g., 3.0" step={0.1} min={0} max={20} /></Field>
              <Field label="Dome/Neck ratio (auto)" tooltip="Computed from dome / neck">
                <input
                  className="w-full rounded-xl border border-dashed border-gray-300 bg-gray-50 px-3 py-2 text-sm"
                  readOnly
                  value={dnRatio == null || !Number.isFinite(dnRatio) ? "—" : dnRatio.toFixed(2)}
                />
              </Field>

              <Field label="Neck > 4 mm (auto)">
                <input
                  className="w-full rounded-xl border border-dashed border-gray-300 bg-gray-50 px-3 py-2 text-sm"
                  readOnly
                  value={neckGT4 === "unknown" ? "Unknown" : neckGT4 ? "Yes" : "No"}
                />
              </Field>
              <Field label="Daughter sac"><Select value={state.daughter} onChange={set("daughter")} options={yesNoUnknown} /></Field>
            </div>
          </Card>
        </div>

        <div className="mt-6 flex flex-wrap gap-3">
          <button onClick={reset} className="rounded-2xl bg-gray-900 px-4 py-2 text-white text-sm font-medium shadow hover:bg-black">Reset</button>
        </div>

        <section className="mt-8 grid grid-cols-1 md:grid-cols-2 xl:grid-cols-3 gap-4 md:gap-6">
          {[
            { title: "Mortality if COIL", idx: 0, auc: 0.826 },
            { title: "Mortality if CLIP", idx: 1, auc: 0.815 },
            { title: "Good GOSE (5–8) if COIL", idx: 2, auc: 0.707 },
            { title: "Good GOSE (5–8) if CLIP", idx: 3, auc: 0.793 },
            { title: "GCS back-to-baseline if COIL", idx: 4, auc: 0.768 },
            { title: "GCS back-to-baseline if CLIP", idx: 5, auc: 0.765 },
          ].map((cfg) => {
            const p = results.percents[cfg.idx];
            const logit = results.logits[cfg.idx];
            const missing = results.missingByOutcome[cfg.idx];
            const complete = missing.length === 0;
            return (
              <Card
                key={cfg.idx}
                title={`${cfg.title} – ${p}`}
                footer={
                  <div className="flex items-center justify-between">
                    <div className="flex items-center gap-2">
                      <Badge tone="blue">AUC {cfg.auc}</Badge>
                      <Badge tone={complete ? "green" : "amber"}>{complete ? "All predictors present" : `${missing.length} missing`}</Badge>
                    </div>
                    <div className="text-xs text-gray-500">logit: {Number.isFinite(logit) ? logit.toFixed(3) : "—"}</div>
                  </div>
                }
              >
                {complete ? (
                  <p className="text-gray-700">Probability shown uses all available predictors.</p>
                ) : (
                  <div className="text-gray-700">
                    <p className="mb-1">Computed treating these as 0 (Unknown):</p>
                    <ul className="list-disc pl-5 text-sm">
                      {missing.map((m, i) => (
                        <li key={i}>{m}</li>
                      ))}
                    </ul>
                  </div>
                )}
              </Card>
            );
          })}
        </section>

        <footer className="mt-10 text-xs text-gray-500">
          <p>
            Notes: Outputs are transformed via logistic function p = 1/(1+e^(−z)). Use as decision support only, not a substitute for clinical judgment. Coefficients and AUC values provided by the NBC GARUDA team.
          </p>
        </footer>
      </div>
    </div>
  );
}
