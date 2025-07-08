# ai-trading-buddy-simple
a simple trading bot that tell you to do something

## Project Setup
### AI SDK (by Vercel)
<br/>
The AI SDK is the TypeScript toolkit designed to help developers build AI-powered applications and agents with React, Next.js, Vue, Svelte, Node.js

*we will be using react.js vite for typescript, tailwindcss in this project*
```bash
npm create vite@latest
```
*then, do tailwind css*
<a>https://tailwindcss.com/docs/installation/using-vite</a>

*then, install this package*
```bash
npm i ai # for ai sdk
npm install lucide-react # for icons
npm i remark-gfm
npm i react-markdown # for bold characters
npm install express cors body-parser dotenv @google/generative-ai
```
then do this, but you can visit ai-sdk.dev to see more, but we will be working with gemini api in this project
```bash
npm install @ai-sdk/google
# or
pnpm add @ai-sdk/google
# or
yarn add @ai-sdk/google
```

## create a file: .env.local
create an env file
```bash
GEMINI_API_KEY="YOUR_GEMINI_API_KEY"
FINNHUB_API_KEY="YOUR_FINBUD_API_KEY"
```

## CODE:
### investorPage.tsx
```bash
// File: src/pages/InvestorAI.tsx
import { useState, useRef, useEffect } from 'react';
import { Send, Bot } from 'lucide-react';
import { StockCard } from '@/components/StockCard';
import ReactMarkdown from 'react-markdown';

export default function InvestorAI() {
  const [symbols, setSymbols] = useState('');
  const [message, setMessage] = useState('');
  const [quotes, setQuotes] = useState([]);
  const [aiResponse, setAiResponse] = useState('');
  const [loading, setLoading] = useState(false);
  const bottomRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [aiResponse, quotes]);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    if (!symbols.trim() || !message.trim()) return;

    setLoading(true);
    setAiResponse('');
    setQuotes([]);

    const res = await fetch('/api/investor', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        symbols: symbols.split(',').map((s) => s.trim().toUpperCase()),
        message,
      }),
    });

    const data = await res.json();
    setQuotes(data.quotes);
    setAiResponse(data.response);
    setLoading(false);
  }

  return (
    <main className="max-w-3xl mx-auto px-4 py-6">
      <header className="mb-6 text-center">
        <h1 className="text-4xl font-bold text-green-700 flex justify-center items-center gap-2">
          <Bot className="w-8 h-8" /> Investor AI
        </h1>
        <p className="text-gray-500 mt-2 text-sm">Enter stock symbols and ask anything</p>
      </header>

      <form onSubmit={handleSubmit} className="space-y-4 mb-6">
        <input
          type="text"
          value={symbols}
          onChange={(e) => setSymbols(e.target.value)}
          placeholder="Stock symbols (e.g. AAPL, TSLA)"
          className="w-full p-3 rounded border border-gray-300 focus:ring-2 focus:ring-green-500"
        />
        <div className="flex gap-2">
          <input
            type="text"
            value={message}
            onChange={(e) => setMessage(e.target.value)}
            placeholder="Ask Investor AI something..."
            className="flex-1 p-3 rounded border border-gray-300 focus:ring-2 focus:ring-green-500"
          />
          <button
            type="submit"
            disabled={loading}
            className="bg-green-600 hover:bg-green-700 text-white px-4 py-2 rounded flex items-center gap-1"
          >
            <Send className="w-4 h-4" /> Send
          </button>
        </div>
      </form>

      {loading && <p className="text-center text-sm text-gray-500">Analyzing market data...</p>}

      {quotes.length > 0 && (
        <div>
          {quotes.map((q) => (
            <StockCard key={q.symbol} data={q} />
          ))}
        </div>
      )}

      {aiResponse && (
        <div className="prose max-w-none bg-gray-50 p-4 rounded border border-gray-300 mb-6">
          <ReactMarkdown>{aiResponse}</ReactMarkdown>
        </div>
      )}

      <div ref={bottomRef} />
    </main>
  );
}
```

### api route.ts
```bash
// File: src/routes/api/investor.ts
import { getStockQuote } from '@/lib/finhub';
import { generateText } from 'ai';
import { google } from '@ai-sdk/google';
import type { APIRoute } from 'vite-plugin-vercel';

export const POST: APIRoute = async ({ request }) => {
  const { message, symbols } = await request.json();

  const results = await Promise.all(
    symbols.map(async (sym: string) => {
      const data = await getStockQuote(sym);
      return { symbol: sym, ...data };
    })
  );

  const prompt = `
You are an investment AI assistant. User is interested in: ${symbols.join(', ')}

Here is real-time data:
${JSON.stringify(results, null, 2)}

User asks: "${message}"

Respond in **Markdown**, including:
- Summary per stock
- Suggest Buy/Hold/Sell
- Mention price trends
`;

  const { text } = await generateText({
    model: google('gemini-1.5-flash'),
    prompt,
    temperature: 0.7,
  });

  return new Response(JSON.stringify({ response: text, quotes: results }), {
    headers: { 'Content-Type': 'application/json' },
  });
};
```

### lib/finbub.ts
```bash
import axios from 'axios';

const FINNHUB_TOKEN = process.env.FINNHUB_API_KEY;

export async function getStockQuote(symbol: string) {
  const url = `https://finnhub.io/api/v1/quote?symbol=${symbol}&token=${FINNHUB_TOKEN}`;
  const res = await axios.get(url);
  return res.data; // { c: current, h: high, l: low, o: open, pc: prevClose }
}
```

### components/StockCard.tsx
```bash
'use client';

import { MiniChart } from 'react-ts-tradingview-widgets';

type StockData = {
  symbol: string;
  c: number;
  h: number;
  l: number;
  o: number;
  pc: number;
  marketCapitalization?: number;
  peBasicExclExtraTTM?: number;
  epsTTM?: number;
  dividendYieldIndicatedAnnual?: number;
};

export function StockCard({ data }: { data: StockData }) {
  const percentChange = ((data.c - data.pc) / data.pc) * 100;
  const chartData = [data.pc, data.o, data.l, data.h, data.c].map((price, i) => ({
    time: i,
    value: price,
  }));

  return (
    <div className="bg-white rounded-xl shadow p-5 space-y-6">
  <div className="flex justify-between items-center">
    <h2 className="text-xl font-bold">{data.symbol}</h2>
    <span className={`text-sm font-semibold ${percentChange >= 0 ? 'text-green-600' : 'text-red-500'}`}>
      {percentChange.toFixed(2)}%
    </span>
  </div>

  <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
    <div className="grid grid-cols-2 gap-3 text-sm">
      <p><span className="text-gray-500">💰 Current:</span> ${data.c}</p>
      <p><span className="text-gray-500">📈 High:</span> ${data.h}</p>
      <p><span className="text-gray-500">📉 Low:</span> ${data.l}</p>
      <p><span className="text-gray-500">🟢 Open:</span> ${data.o}</p>
      <p><span className="text-gray-500">🔵 Prev Close:</span> ${data.pc}</p>
      <p><span className="text-gray-500">📦 Market Cap:</span> ${data.marketCapitalization?.toFixed(2) ?? '—'}B</p>
      <p><span className="text-gray-500">🧮 P/E Ratio:</span> {data.peBasicExclExtraTTM?.toFixed(2) ?? '—'}</p>
      <p><span className="text-gray-500">💵 EPS:</span> ${data.epsTTM?.toFixed(2) ?? '—'}</p>
      <p><span className="text-gray-500">🪙 Dividend:</span> {data.dividendYieldIndicatedAnnual?.toFixed(2) ?? '—'}%</p>
    </div>

    <div className="mt-4">
      <MiniChart
        symbol={`NASDAQ:${data.symbol}`}
        width="100%"
        height={200}
        colorTheme="light"
      />
    </div>
  </div>
</div>
  );
}
```
