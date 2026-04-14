# GenLayer Studio

Lightweight developer tool for building **Intelligent Contracts** on GenLayer.
Includes libraries for weather, price feeds, and social media — plus a secure API key proxy.

> **How to use this file:** Copy each code block below into the path shown above it.
> Then run `npm install && npm run dev`.

---

## Project structure

```
genlayer-studio/
├── package.json
├── vercel.json
├── .env.example
├── pages/
│   ├── _app.js
│   ├── index.js
│   └── api/
│       └── proxy.js
├── styles/
│   ├── globals.css
│   └── studio.module.css
└── lib/
    └── contracts/
        ├── gl_weather.py
        ├── gl_pricefeed.py
        ├── gl_social.py
        ├── gl_http.py
        └── WeatherInsurance.py
```

---

## File 1 — `package.json`

```json
{
  "name": "genlayer-studio",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  },
  "dependencies": {
    "next": "14.2.3",
    "react": "^18",
    "react-dom": "^18",
    "genlayer-js": "^0.5.0"
  }
}
```

---

## File 2 — `vercel.json`

```json
{
  "functions": {
    "pages/api/proxy.js": {
      "runtime": "edge"
    }
  },
  "headers": [
    {
      "source": "/api/(.*)",
      "headers": [
        { "key": "Access-Control-Allow-Origin", "value": "*" },
        { "key": "Access-Control-Allow-Methods", "value": "GET,POST,OPTIONS" }
      ]
    }
  ]
}
```

---

## File 3 — `.env.example`

```
WEATHER_KEY=sk_live_your_openweathermap_key
COINGECKO_KEY=cg_demo_your_coingecko_key
TWITTER_BEARER=AAAAAAAAAAyour_twitter_bearer
CUSTOM_API_KEY=

NEXT_PUBLIC_GL_RPC=http://localhost:4000
PROXY_SECRET=
```

---

## File 4 — `pages/_app.js`

```js
import "../styles/globals.css";
export default function App({ Component, pageProps }) {
  return <Component {...pageProps} />;
}
```

---

## File 5 — `pages/api/proxy.js`

The security core. Resolves API keys from `process.env` server-side.
The raw key value **never** reaches the browser or contract bytecode.

```js
/**
 * GenLayer API Proxy — Vercel Edge Function
 *
 * Contracts call this with ?__key_ref=WEATHER_KEY
 * The edge resolves process.env[key_ref], injects it upstream,
 * and returns only the JSON response. Keys stay server-side.
 */

export const config = { runtime: "edge" };

const SERVICE_ROUTES = {
  weather: {
    base: "https://api.openweathermap.org/data/2.5",
    keyParam: "appid",
    defaultPath: "/weather",
  },
  forecast: {
    base: "https://api.openweathermap.org/data/2.5",
    keyParam: "appid",
    defaultPath: "/forecast",
  },
  pricefeed: {
    base: "https://api.coingecko.com/api/v3",
    keyParam: "x_cg_demo_api_key",
    defaultPath: "/simple/price",
  },
  social: {
    base: "https://api.twitter.com/2",
    keyParam: null,
    defaultPath: "/tweets/search/recent",
  },
  custom: {
    base: null,
    keyParam: null,
    defaultPath: null,
  },
};

const ALLOWED_KEY_REFS = new Set([
  "WEATHER_KEY",
  "COINGECKO_KEY",
  "TWITTER_BEARER",
  "CUSTOM_API_KEY",
]);

const rateLimitMap = new Map();
const RATE_LIMIT_WINDOW_MS = 60_000;
const RATE_LIMIT_MAX = 60;

function isRateLimited(ip) {
  const now = Date.now();
  const entry = rateLimitMap.get(ip) || { count: 0, ts: now };
  if (now - entry.ts > RATE_LIMIT_WINDOW_MS) {
    rateLimitMap.set(ip, { count: 1, ts: now });
    return false;
  }
  entry.count++;
  rateLimitMap.set(ip, entry);
  return entry.count > RATE_LIMIT_MAX;
}

export default async function handler(req) {
  if (req.method === "OPTIONS") return new Response(null, { status: 204 });

  const ip = req.headers.get("x-forwarded-for") ?? "unknown";
  if (isRateLimited(ip)) return json({ error: "rate_limited" }, 429);

  let params = {};
  const url = new URL(req.url);
  url.searchParams.forEach((v, k) => (params[k] = v));
  if (req.method === "POST") {
    try { params = { ...params, ...(await req.json()) }; } catch (_) {}
  }

  const { service, __key_ref, url: customUrl, ...rest } = params;

  if (!__key_ref || !ALLOWED_KEY_REFS.has(__key_ref))
    return json({ error: "invalid_key_ref" }, 400);

  const apiKey = process.env[__key_ref];
  if (!apiKey) return json({ error: "key_not_configured" }, 500);

  const route = SERVICE_ROUTES[service];
  if (!route && !customUrl) return json({ error: "unknown_service" }, 400);

  let upstreamUrl;
  const headers = new Headers({ "Content-Type": "application/json" });

  if (service === "custom" && customUrl) {
    upstreamUrl = new URL(customUrl);
    headers.set("Authorization", `Bearer ${apiKey}`);
  } else if (service === "social") {
    upstreamUrl = new URL(route.base + route.defaultPath);
    headers.set("Authorization", `Bearer ${apiKey}`);
    Object.entries(rest).forEach(([k, v]) => upstreamUrl.searchParams.set(k, v));
  } else {
    upstreamUrl = new URL(route.base + route.defaultPath);
    if (route.keyParam) upstreamUrl.searchParams.set(route.keyParam, apiKey);
    Object.entries(rest).forEach(([k, v]) => upstreamUrl.searchParams.set(k, v));
  }

  try {
    const upstream = await fetch(upstreamUrl.toString(), { method: "GET", headers });
    const data = await upstream.json();
    const clean = JSON.parse(
      JSON.stringify(data).replace(new RegExp(apiKey, "g"), "[REDACTED]")
    );
    return json(clean, upstream.status);
  } catch (err) {
    return json({ error: "upstream_failed", detail: err.message }, 502);
  }
}

function json(data, status = 200) {
  return new Response(JSON.stringify(data), {
    status,
    headers: {
      "Content-Type": "application/json",
      "X-Powered-By": "GenLayer Studio",
    },
  });
}
```

---

## File 6 — `pages/index.js`

```jsx
import Head from "next/head";
import { useState } from "react";
import styles from "../styles/studio.module.css";

const LIBRARIES = [
  {
    id: "weather", name: "gl-weather", icon: "☁",
    color: "#E1F5EE", textColor: "#0F6E56",
    description: "Live weather, forecasts, and climate data.",
    methods: ["get_current()", "get_forecast()", "get_temp()", "is_raining()"],
    keyRef: "WEATHER_KEY", file: "gl_weather.py",
  },
  {
    id: "pricefeed", name: "gl-pricefeed", icon: "$",
    color: "#E6F1FB", textColor: "#185FA5",
    description: "Token prices, forex, and commodity feeds.",
    methods: ["get_price()", "get_prices()", "above_threshold()", "get_market_cap()"],
    keyRef: "COINGECKO_KEY", file: "gl_pricefeed.py",
  },
  {
    id: "social", name: "gl-social", icon: "✦",
    color: "#FAEEDA", textColor: "#854F0B",
    description: "Posts, sentiment, and follower counts from X and Reddit.",
    methods: ["search_posts()", "sentiment()", "keyword_trending()", "get_post_count()"],
    keyRef: "TWITTER_BEARER", file: "gl_social.py",
  },
  {
    id: "http", name: "gl-http", icon: "⬡",
    color: "#EEEDFE", textColor: "#534AB7",
    description: "Generic HTTP client — any API, keys stay server-side.",
    methods: ["get()", "get_text()", "with_key()", "post()"],
    keyRef: "CUSTOM_API_KEY", file: "gl_http.py",
  },
];

const EXAMPLE_CONTRACT = `from gl_weather import WeatherLib
from gl_pricefeed import PriceFeedLib

PROXY   = "https://your-app.vercel.app/api/proxy"
weather = WeatherLib(proxy_url=PROXY, key_ref="WEATHER_KEY")
prices  = PriceFeedLib(proxy_url=PROXY, key_ref="COINGECKO_KEY")

@gl.contract
class WeatherInsurance:
    threshold_c: float

    def __init__(self, threshold_c: float):
        self.threshold_c = threshold_c

    @gl.public.write
    async def evaluate(self, city: str) -> dict:
        temp      = await weather.get_temp(city)
        eth_price = await prices.get_price("ethereum", "usd")
        payout    = (100 / eth_price) if temp < self.threshold_c else 0
        return {"payout_eth": payout, "temp": temp}`;

export default function Studio() {
  const [activeTab, setActiveTab] = useState("libraries");
  const [selectedLib, setSelectedLib] = useState(null);
  const [proxyTest, setProxyTest] = useState(null);
  const [testing, setTesting] = useState(false);

  async function testProxy(service, keyRef) {
    setTesting(true);
    setProxyTest(null);
    try {
      const params = new URLSearchParams({ service, q: "London",
        ids: "ethereum", vs_currencies: "usd", __key_ref: keyRef });
      const res = await fetch(`/api/proxy?${params}`);
      const data = await res.json();
      setProxyTest({ ok: res.ok, status: res.status, data });
    } catch (e) {
      setProxyTest({ ok: false, error: e.message });
    } finally { setTesting(false); }
  }

  const TABS = [
    { id: "libraries", label: "Libraries", icon: "◈" },
    { id: "vault",     label: "API Vault",  icon: "⌁" },
    { id: "playground",label: "Playground", icon: "❯" },
    { id: "deploy",    label: "Deploy",     icon: "⬡" },
  ];

  return (
    <>
      <Head>
        <title>GenLayer Studio</title>
        <meta name="description" content="Intelligent Contract development studio" />
      </Head>
      <div className={styles.layout}>
        <aside className={styles.sidebar}>
          <div className={styles.brand}>
            <span className={styles.dot} />
            <div>
              <div className={styles.brandName}>GenLayer Studio</div>
              <div className={styles.brandSub}>dev · v0.1.0</div>
            </div>
          </div>
          <nav className={styles.nav}>
            <div className={styles.navSection}>workspace</div>
            {TABS.map(({ id, label, icon }) => (
              <button key={id}
                className={`${styles.navItem} ${activeTab === id ? styles.navActive : ""}`}
                onClick={() => setActiveTab(id)}>
                <span className={styles.navIcon}>{icon}</span>{label}
              </button>
            ))}
            <div className={styles.navSection}>resources</div>
            <a href="https://docs.genlayer.com" target="_blank" rel="noreferrer"
              className={styles.navItem}>
              <span className={styles.navIcon}>≡</span>Docs ↗
            </a>
          </nav>
        </aside>

        <main className={styles.main}>
          <header className={styles.topbar}>
            <h1 className={styles.topbarTitle}>
              {{ libraries:"Contract Libraries", vault:"API Vault",
                 playground:"Playground", deploy:"Deploy" }[activeTab]}
            </h1>
            <a href="https://vercel.com/new" target="_blank" rel="noreferrer"
              className={styles.btnPrimary}>Deploy to Vercel ↗</a>
          </header>

          <div className={styles.content}>

            {activeTab === "libraries" && (
              <div>
                <p className={styles.sectionDesc}>
                  Drop-in Python libraries for Intelligent Contracts. Keys resolved server-side.
                </p>
                <div className={styles.libGrid}>
                  {LIBRARIES.map((lib) => (
                    <div key={lib.id}
                      className={`${styles.libCard} ${selectedLib?.id === lib.id ? styles.libCardActive : ""}`}
                      onClick={() => setSelectedLib(lib)}>
                      <div className={styles.libHeader}>
                        <div className={styles.libIcon}
                          style={{ background: lib.color, color: lib.textColor }}>
                          {lib.icon}
                        </div>
                        <span className={styles.libName}>{lib.name}</span>
                      </div>
                      <p className={styles.libDesc}>{lib.description}</p>
                      <div className={styles.methodList}>
                        {lib.methods.map((m) => (
                          <code key={m} className={styles.methodTag}>{m}</code>
                        ))}
                      </div>
                    </div>
                  ))}
                </div>
                {selectedLib && (
                  <div className={styles.libDetail}>
                    <div className={styles.libDetailHeader}>
                      <span className={styles.libDetailName}>{selectedLib.name}</span>
                      <button className={styles.btnSecondary}
                        onClick={() => testProxy(selectedLib.id, selectedLib.keyRef)}
                        disabled={testing}>
                        {testing ? "Testing…" : "Test proxy connection"}
                      </button>
                    </div>
                    {proxyTest && (
                      <div className={`${styles.proxyResult} ${proxyTest.ok ? styles.proxyOk : styles.proxyErr}`}>
                        <span>Status {proxyTest.status}</span>
                        <code>{JSON.stringify(proxyTest.data || proxyTest.error, null, 2).slice(0, 200)}…</code>
                      </div>
                    )}
                    <p className={styles.installHint}>
                      Copy <code>lib/contracts/{selectedLib.file}</code> into your project.
                      Set <code>{selectedLib.keyRef}</code> in Vercel env vars.
                    </p>
                  </div>
                )}
              </div>
            )}

            {activeTab === "vault" && (
              <div>
                <p className={styles.sectionDesc}>
                  Keys stored as Vercel env vars, injected by the edge proxy at request time.
                  Never appear in contract code or transaction data.
                </p>
                <div className={styles.vaultTable}>
                  <div className={styles.vaultHeader}>
                    <span>Key reference</span><span>Service</span>
                    <span>Status</span><span>Action</span>
                  </div>
                  {[
                    { ref: "WEATHER_KEY",    service: "OpenWeatherMap", status: "active" },
                    { ref: "COINGECKO_KEY",  service: "CoinGecko",      status: "active" },
                    { ref: "TWITTER_BEARER", service: "X API v2",        status: "expiring" },
                    { ref: "CUSTOM_API_KEY", service: "Custom",          status: "not_set" },
                  ].map(({ ref, service, status }) => (
                    <div key={ref} className={styles.vaultRow}>
                      <code className={styles.vaultRef}>{ref}</code>
                      <span className={styles.vaultService}>{service}</span>
                      <span className={`${styles.badge} ${styles[`badge_${status}`]}`}>
                        {status.replace("_", " ")}
                      </span>
                      <button className={styles.btnSecondary}>
                        {status === "not_set" ? "Add key" : "Rotate"}
                      </button>
                    </div>
                  ))}
                </div>
                <div className={styles.codeBlock}>
                  <div className={styles.codeLabel}>Proxy pattern</div>
                  <pre>{`// __key_ref resolved here, never sent to client
const apiKey = process.env[params.__key_ref];
const upstream = await fetch(buildUrl(service, apiKey, rest));
return json(await upstream.json());`}</pre>
                </div>
              </div>
            )}

            {activeTab === "playground" && (
              <div>
                <p className={styles.sectionDesc}>Write and preview Intelligent Contract code.</p>
                <div className={styles.codeLabel}>Contract editor</div>
                <textarea className={styles.editor} defaultValue={EXAMPLE_CONTRACT} spellCheck={false} />
                <div className={styles.playgroundActions}>
                  <button className={styles.btnPrimary}>Validate syntax</button>
                  <button className={styles.btnSecondary}>Generate tests</button>
                  <button className={styles.btnSecondary}>Deploy to localnet</button>
                </div>
              </div>
            )}

            {activeTab === "deploy" && (
              <div>
                <p className={styles.sectionDesc}>
                  Deploy the Studio to Vercel and contracts to GenLayer testnet.
                </p>
                <div className={styles.deployGrid}>
                  {[
                    { title: "1 — Fork & clone",
                      code: "git clone https://github.com/your-org/genlayer-studio\ncd genlayer-studio && npm install" },
                    { title: "2 — Set env vars",
                      code: "WEATHER_KEY=sk_live_...\nCOINGECKO_KEY=cg_demo_...\nTWITTER_BEARER=AAAA...\nNEXT_PUBLIC_GL_RPC=http://localhost:4000" },
                    { title: "3 — Push to Vercel",
                      code: "npx vercel\n# or connect GitHub repo\n# in the Vercel dashboard" },
                    { title: "4 — Deploy contract",
                      code: "genlayer deploy WeatherInsurance.py \\\n  --network testnet \\\n  --constructor-args 5.0" },
                  ].map(({ title, code }) => (
                    <div key={title} className={styles.deployCard}>
                      <div className={styles.deployCardTitle}>{title}</div>
                      <pre className={styles.deployCode}>{code}</pre>
                    </div>
                  ))}
                </div>
              </div>
            )}

          </div>
        </main>
      </div>
    </>
  );
}
```

---

## File 7 — `styles/globals.css`

```css
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
body { background: #0e0e10; color: #e8e8e0; }
a { color: inherit; }
```

---

## File 8 — `styles/studio.module.css`

```css
.layout {
  display: grid;
  grid-template-columns: 220px 1fr;
  min-height: 100vh;
  font-family: 'Courier New', Courier, monospace;
  background: #0e0e10;
  color: #e8e8e0;
}
.sidebar {
  background: #111113;
  border-right: 0.5px solid #2a2a2e;
  display: flex;
  flex-direction: column;
}
.brand {
  display: flex; align-items: center; gap: 10px;
  padding: 20px 16px;
  border-bottom: 0.5px solid #2a2a2e;
}
.dot {
  width: 8px; height: 8px; border-radius: 50%;
  background: #00C896; flex-shrink: 0;
}
.brandName { font-size: 12px; font-weight: 500; color: #e8e8e0; letter-spacing: .06em; }
.brandSub  { font-size: 10px; color: #666; margin-top: 1px; }
.nav { padding: 12px 0; flex: 1; }
.navSection {
  font-size: 9px; color: #444; letter-spacing: .1em;
  text-transform: uppercase; padding: 14px 16px 5px;
}
.navItem {
  display: flex; align-items: center; gap: 8px;
  width: 100%; padding: 7px 16px;
  font-size: 12px; font-family: inherit;
  color: #888; background: none;
  border: none; border-left: 2px solid transparent;
  cursor: pointer; text-decoration: none; text-align: left;
  transition: color .12s, background .12s;
}
.navItem:hover { color: #e8e8e0; background: #181820; }
.navActive { color: #e8e8e0 !important; border-left-color: #00C896; background: #181820 !important; }
.navIcon { width: 16px; opacity: .7; font-size: 13px; }
.main { display: flex; flex-direction: column; background: #0e0e10; }
.topbar {
  display: flex; align-items: center; justify-content: space-between;
  padding: 14px 24px; border-bottom: 0.5px solid #2a2a2e;
  background: #111113;
}
.topbarTitle { font-size: 13px; font-weight: 500; color: #e8e8e0; }
.content { flex: 1; padding: 24px; overflow: auto; }
.sectionDesc { font-size: 12px; color: #888; line-height: 1.7; margin-bottom: 20px; max-width: 640px; }
.btnPrimary {
  font-size: 11px; padding: 6px 14px; border-radius: 6px;
  background: #00C896; color: #0e0e10; border: none;
  cursor: pointer; font-family: inherit; font-weight: 500; text-decoration: none;
}
.btnPrimary:hover { opacity: .88; }
.btnSecondary {
  font-size: 11px; padding: 6px 14px; border-radius: 6px;
  background: none; color: #888; border: 0.5px solid #2a2a2e;
  cursor: pointer; font-family: inherit;
}
.btnSecondary:hover { color: #e8e8e0; border-color: #444; }
.btnSecondary:disabled { opacity: .5; cursor: default; }
.libGrid { display: grid; grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); gap: 12px; margin-bottom: 20px; }
.libCard { border: 0.5px solid #2a2a2e; border-radius: 8px; padding: 14px; cursor: pointer; background: #111113; transition: border-color .15s; }
.libCard:hover { border-color: #444; }
.libCardActive { border-color: #00C896 !important; }
.libHeader { display: flex; align-items: center; gap: 8px; margin-bottom: 8px; }
.libIcon { width: 28px; height: 28px; border-radius: 6px; display: flex; align-items: center; justify-content: center; font-size: 14px; }
.libName { font-size: 12px; font-weight: 500; color: #e8e8e0; }
.libDesc { font-size: 11px; color: #888; line-height: 1.5; margin-bottom: 10px; }
.methodList { display: flex; flex-wrap: wrap; gap: 4px; }
.methodTag { font-size: 10px; padding: 2px 7px; border-radius: 4px; background: #181820; color: #888; border: 0.5px solid #2a2a2e; font-family: inherit; }
.libDetail { border: 0.5px solid #2a2a2e; border-radius: 8px; padding: 16px; background: #111113; margin-top: 4px; }
.libDetailHeader { display: flex; align-items: center; justify-content: space-between; margin-bottom: 12px; }
.libDetailName { font-size: 13px; font-weight: 500; color: #00C896; }
.proxyResult { padding: 10px; border-radius: 6px; margin-bottom: 10px; font-size: 11px; }
.proxyResult code { display: block; margin-top: 4px; color: #888; white-space: pre-wrap; }
.proxyOk { border: 0.5px solid #00C89640; background: #00C89610; }
.proxyErr { border: 0.5px solid #E24B4A40; background: #E24B4A10; }
.installHint { font-size: 11px; color: #888; }
.installHint code { color: #00C896; }
.vaultTable { border: 0.5px solid #2a2a2e; border-radius: 8px; overflow: hidden; margin-bottom: 20px; }
.vaultHeader { display: grid; grid-template-columns: 2fr 1.5fr 1fr 1fr; padding: 8px 14px; font-size: 10px; color: #444; letter-spacing: .06em; text-transform: uppercase; border-bottom: 0.5px solid #2a2a2e; background: #111113; }
.vaultRow { display: grid; grid-template-columns: 2fr 1.5fr 1fr 1fr; align-items: center; padding: 10px 14px; border-bottom: 0.5px solid #1a1a1e; gap: 8px; }
.vaultRow:last-child { border-bottom: none; }
.vaultRef { font-size: 12px; color: #00C896; font-family: inherit; }
.vaultService { font-size: 11px; color: #888; }
.badge { font-size: 10px; padding: 2px 8px; border-radius: 4px; }
.badge_active   { background: #00C89620; color: #00C896; }
.badge_expiring { background: #F59E0B20; color: #F59E0B; }
.badge_not_set  { background: #2a2a2e; color: #666; }
.codeBlock { background: #0a0a0c; border: 0.5px solid #2a2a2e; border-radius: 8px; padding: 14px; font-size: 11px; line-height: 1.7; }
.codeLabel { font-size: 10px; color: #444; letter-spacing: .06em; text-transform: uppercase; margin-bottom: 8px; }
.codeBlock pre { color: #888; white-space: pre-wrap; margin: 0; font-family: inherit; }
.editor { width: 100%; min-height: 280px; background: #0a0a0c; border: 0.5px solid #2a2a2e; border-radius: 8px; padding: 14px; font-family: 'Courier New', monospace; font-size: 12px; line-height: 1.7; color: #c8c8c0; resize: vertical; margin-bottom: 12px; }
.editor:focus { outline: none; border-color: #00C89640; }
.playgroundActions { display: flex; gap: 8px; flex-wrap: wrap; }
.deployGrid { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; }
.deployCard { border: 0.5px solid #2a2a2e; border-radius: 8px; padding: 16px; background: #111113; }
.deployCardTitle { font-size: 11px; font-weight: 500; color: #e8e8e0; margin-bottom: 10px; }
.deployCode { font-family: 'Courier New', monospace; font-size: 11px; color: #888; white-space: pre-wrap; line-height: 1.7; margin: 0; }
@media (max-width: 700px) {
  .layout { grid-template-columns: 1fr; }
  .sidebar { display: none; }
  .deployGrid, .libGrid { grid-template-columns: 1fr; }
}
```

---

## File 9 — `lib/contracts/gl_weather.py`

```python
"""
gl-weather — GenLayer Intelligent Contract Library
Weather data via proxy (API key stays server-side).
"""
import json
import gl

class WeatherLib:
    def __init__(self, proxy_url: str = None, key_ref: str = "WEATHER_KEY"):
        self._proxy = proxy_url or "https://genlayer-studio.vercel.app/api/proxy"
        self._key_ref = key_ref

    async def get_current(self, city: str) -> dict:
        url = (f"{self._proxy}?service=weather&q={city}"
               f"&units=metric&__key_ref={self._key_ref}")
        return json.loads(await gl.get_webpage(url, mode="text"))

    async def get_forecast(self, city: str, days: int = 5) -> dict:
        url = (f"{self._proxy}?service=forecast&q={city}"
               f"&cnt={days * 8}&units=metric&__key_ref={self._key_ref}")
        return json.loads(await gl.get_webpage(url, mode="text"))

    async def get_temp(self, city: str) -> float:
        return (await self.get_current(city))["main"]["temp"]

    async def get_humidity(self, city: str) -> float:
        return (await self.get_current(city))["main"]["humidity"]

    async def is_raining(self, city: str) -> bool:
        data = await self.get_current(city)
        condition = data.get("weather", [{}])[0].get("main", "").lower()
        return "rain" in condition or "drizzle" in condition
```

---

## File 10 — `lib/contracts/gl_pricefeed.py`

```python
"""
gl-pricefeed — GenLayer Intelligent Contract Library
Token prices, forex, and market cap via CoinGecko proxy.
"""
import json
import gl

class PriceFeedLib:
    def __init__(self, proxy_url: str = None, key_ref: str = "COINGECKO_KEY"):
        self._proxy = proxy_url or "https://genlayer-studio.vercel.app/api/proxy"
        self._key_ref = key_ref

    async def get_price(self, token_id: str, vs_currency: str = "usd") -> float:
        url = (f"{self._proxy}?service=pricefeed&ids={token_id}"
               f"&vs_currencies={vs_currency}&__key_ref={self._key_ref}")
        data = json.loads(await gl.get_webpage(url, mode="text"))
        return data[token_id][vs_currency]

    async def get_prices(self, token_ids: list, vs_currency: str = "usd") -> dict:
        ids = ",".join(token_ids)
        url = (f"{self._proxy}?service=pricefeed&ids={ids}"
               f"&vs_currencies={vs_currency}&__key_ref={self._key_ref}")
        data = json.loads(await gl.get_webpage(url, mode="text"))
        return {t: data[t][vs_currency] for t in token_ids if t in data}

    async def get_market_cap(self, token_id: str, vs_currency: str = "usd") -> float:
        url = (f"{self._proxy}?service=pricefeed&ids={token_id}"
               f"&vs_currencies={vs_currency}&include_market_cap=true"
               f"&__key_ref={self._key_ref}")
        data = json.loads(await gl.get_webpage(url, mode="text"))
        return data[token_id].get(f"{vs_currency}_market_cap", 0)

    async def above_threshold(self, token_id: str, threshold: float,
                               vs_currency: str = "usd") -> bool:
        return (await self.get_price(token_id, vs_currency)) > threshold
```

---

## File 11 — `lib/contracts/gl_social.py`

```python
"""
gl-social — GenLayer Intelligent Contract Library
Posts, sentiment, and trending via X API v2 proxy.
"""
import json
import gl

class SocialLib:
    def __init__(self, proxy_url: str = None, key_ref: str = "TWITTER_BEARER"):
        self._proxy = proxy_url or "https://genlayer-studio.vercel.app/api/proxy"
        self._key_ref = key_ref

    async def search_posts(self, query: str, max_results: int = 10) -> list:
        url = (f"{self._proxy}?service=social&query={query}"
               f"&max_results={max_results}"
               f"&tweet.fields=created_at,author_id,public_metrics"
               f"&__key_ref={self._key_ref}")
        data = json.loads(await gl.get_webpage(url, mode="text"))
        return data.get("data", [])

    async def get_post_count(self, query: str) -> int:
        return len(await self.search_posts(query, max_results=100))

    async def keyword_trending(self, keyword: str, threshold: int = 50) -> bool:
        return (await self.get_post_count(keyword)) >= threshold

    async def simple_sentiment(self, query: str, max_results: int = 20) -> dict:
        posts = await self.search_posts(query, max_results)
        positive_words = {"good", "great", "bullish", "moon", "win", "up", "buy"}
        negative_words = {"bad", "crash", "bearish", "dump", "loss", "down", "sell"}
        pos = neg = neu = 0
        for post in posts:
            words = set(post.get("text", "").lower().split())
            if words & positive_words: pos += 1
            elif words & negative_words: neg += 1
            else: neu += 1
        total = pos + neg + neu or 1
        return {"positive": pos, "negative": neg, "neutral": neu,
                "score": round((pos - neg) / total, 4)}
```

---

## File 12 — `lib/contracts/WeatherInsurance.py`

Full worked example contract combining all three libraries.

```python
"""
WeatherInsurance.py — Example GenLayer Intelligent Contract
Parametric insurance: pays out ETH if temperature drops below threshold.
Validators independently fetch data and reach MAJORITY consensus.
"""
import gl
from gl_weather   import WeatherLib
from gl_pricefeed import PriceFeedLib
from gl_social    import SocialLib

PROXY   = "https://your-app.vercel.app/api/proxy"
weather = WeatherLib(proxy_url=PROXY, key_ref="WEATHER_KEY")
prices  = PriceFeedLib(proxy_url=PROXY, key_ref="COINGECKO_KEY")
social  = SocialLib(proxy_url=PROXY, key_ref="TWITTER_BEARER")


@gl.contract
class WeatherInsurance:
    city:          str
    threshold_c:   float
    premium_usd:   float
    policyholder:  str
    active:        bool

    def __init__(self, city: str, threshold_c: float, premium_usd: float):
        self.city         = city
        self.threshold_c  = threshold_c
        self.premium_usd  = premium_usd
        self.policyholder = gl.message.sender_address
        self.active       = True

    @gl.public.view
    async def current_risk_assessment(self) -> dict:
        temp      = await weather.get_temp(self.city)
        eth_price = await prices.get_price("ethereum", "usd")
        sentiment = await social.simple_sentiment(f"weather {self.city}")
        return {
            "city": self.city,
            "current_temp_c": temp,
            "threshold_c": self.threshold_c,
            "at_risk": temp < self.threshold_c,
            "eth_price_usd": eth_price,
            "social_sentiment_score": sentiment["score"],
        }

    @gl.public.write
    async def evaluate_and_pay(self) -> dict:
        if not self.active:
            return {"status": "inactive"}
        temp      = await weather.get_temp(self.city)
        eth_price = await prices.get_price("ethereum", "usd")
        if temp >= self.threshold_c:
            return {"status": "no_payout", "temp": temp}
        payout_eth  = self.premium_usd / eth_price
        self.active = False
        return {
            "status":      "payout_approved",
            "payout_eth":  round(payout_eth, 6),
            "temp":        temp,
            "eth_price":   eth_price,
            "recipient":   self.policyholder,
        }

    @gl.public.view
    def get_policy(self) -> dict:
        return {
            "city":         self.city,
            "threshold_c":  self.threshold_c,
            "premium_usd":  self.premium_usd,
            "policyholder": self.policyholder,
            "active":       self.active,
        }
```

---

## Setup & deploy

```bash
# 1. Install
npm install

# 2. Configure
cp .env.example .env.local
# Fill in your API keys

# 3. Dev server
npm run dev        # http://localhost:3000

# 4. Deploy
npx vercel         # follow prompts, add env vars in dashboard
```

## How the proxy security works

```
GenLayer Contract        Vercel Edge Proxy           Upstream API
────────────────         ─────────────────           ────────────
?__key_ref=          →   process.env[key_ref]    →   api.openweathermap.org
  WEATHER_KEY             inject real key              ?appid=sk_live_…
                     ←   return JSON only         ←   { temp: 12.3 }
```

Raw keys never appear in contract bytecode, transaction logs, or the browser.

## License

MIT
