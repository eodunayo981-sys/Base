# Project: base-yapdrop-starter (Next.js + Tailwind + OpenAI + Base wallet)

This repository is a starter for a Yapdrop-style app tailored to Base (Coinbase L2). It includes a Next.js frontend, a minimal API route that proxies to OpenAI (stub), and wallet-connect support to Base via wagmi and Coinbase Wallet SDK. Use this as a starting point and replace the API-key stubs before deploying.

---

## Quick start (local)

1. Create project folder and copy files from this document into the corresponding paths.
2. `npm install`
3. Create a `.env.local` in project root with the following variables:

```
OPENAI_API_KEY=sk-...    # server-side OpenAI key
NEXT_PUBLIC_BASE_RPC=https://base-mainnet.infura.io/v3/<YOUR_INFURA_KEY>  # optional
NEXT_PUBLIC_APP_NAME=BaseYapdrop
```

4. `npm run dev`

---

## package.json

```json
{
  "name": "base-yapdrop-starter",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  },
  "dependencies": {
    "next": "14.0.0",
    "react": "18.2.0",
    "react-dom": "18.2.0",
    "axios": "1.4.0",
    "tailwindcss": "3.4.0",
    "autoprefixer": "10.4.14",
    "postcss": "8.4.23",
    "wagmi": "1.5.0",
    "@wagmi/cli": "0.3.0",
    "@web3modal/react": "2.6.0",
    "@web3modal/ethereum": "2.3.0",
    "@coinbase/wallet-sdk": "4.0.0"
  }
}
```

---

## tailwind.config.js

```js
module.exports = {
  content: ["./pages/**/*.{js,ts,jsx,tsx}", "./components/**/*.{js,ts,jsx,tsx}"],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

## postcss.config.js

```js
module.exports = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
}
```

---

## pages/_app.tsx

```tsx
import '../styles/globals.css'
import type { AppProps } from 'next/app'

export default function App({ Component, pageProps }: AppProps) {
  return <Component {...pageProps} />
}
```

---

## styles/globals.css

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

html, body, #__next {
  height: 100%;
}

body {
  background: linear-gradient(180deg, #0f172a 0%, #071124 100%);
  color: #e6f0ff;
}
```

---

## pages/index.tsx

```tsx
import { useState } from 'react'
import axios from 'axios'
import TweetGenerator from '../components/TweetGenerator'

export default function Home() {
  return (
    <main className="min-h-screen p-8 max-w-4xl mx-auto">
      <header className="mb-8">
        <h1 className="text-4xl font-bold">BaseYapdrop — generate content for Base</h1>
        <p className="mt-2 text-slate-300">Write announcements, mint copy, and developer snippets tailored to Base (Coinbase L2).</p>
      </header>

      <section>
        <TweetGenerator />
      </section>

      <footer className="mt-12 text-sm text-slate-400">Built with ❤️ for Base. Remember to set your OpenAI API key in server env.</footer>
    </main>
  )
}
```

---

## components/TweetGenerator.tsx

```tsx
import { useState } from 'react'
import axios from 'axios'

export default function TweetGenerator(){
  const [idea, setIdea] = useState('')
  const [mode, setMode] = useState<'tweet'|'thread'|'mint'|'dev'>('tweet')
  const [loading, setLoading] = useState(false)
  const [result, setResult] = useState('')

  async function generate(){
    setLoading(true)
    setResult('')
    try{
      const res = await axios.post('/api/generate', { idea, mode })
      setResult(res.data.text)
    }catch(err:any){
      setResult('Error generating text. Check server logs.')
    }finally{ setLoading(false) }
  }

  return (
    <div className="bg-slate-800/40 p-6 rounded-2xl">
      <label className="block mb-2">Describe your idea for a Base post</label>
      <textarea value={idea} onChange={e=>setIdea(e.target.value)} className="w-full p-3 rounded-lg bg-slate-900/50" rows={4} />

      <div className="flex gap-2 mt-4">
        <select value={mode} onChange={e=>setMode(e.target.value as any)} className="p-2 rounded">
          <option value="tweet">Tweet</option>
          <option value="thread">Thread</option>
          <option value="mint">Mint announcement</option>
          <option value="dev">Developer snippet</option>
        </select>
        <button onClick={generate} className="ml-auto rounded bg-indigo-600 px-4 py-2">{loading? 'Generating...' : 'Generate'}</button>
      </div>

      <div className="mt-6">
        <h3 className="mb-2">Output</h3>
        <div className="p-4 rounded bg-slate-900/60 min-h-[120px] whitespace-pre-wrap">{result || <span className="text-slate-400">Nothing yet — click Generate.</span>}</div>
      </div>
    </div>
  )
}
```

---

## pages/api/generate.ts

```ts
import type { NextApiRequest, NextApiResponse } from 'next'
import axios from 'axios'

export default async function handler(req: NextApiRequest, res: NextApiResponse){
  if(req.method !== 'POST') return res.status(405).end()
  const { idea, mode } = req.body

  if(!process.env.OPENAI_API_KEY) return res.status(500).json({ error: 'Missing OPENAI_API_KEY' })

  const prompt = buildPrompt(idea, mode)

  try{
    const openaiRes = await axios.post('https://api.openai.com/v1/chat/completions', {
      model: 'gpt-4o-mini',
      messages: [{ role: 'user', content: prompt }],
      max_tokens: 400,
      temperature: 0.8
    }, {
      headers: {
        Authorization: `Bearer ${process.env.OPENAI_API_KEY}`,
        'Content-Type': 'application/json'
      }
    })

    const text = openaiRes.data.choices?.[0]?.message?.content ?? ''
    res.status(200).json({ text })
  }catch(err:any){
    console.error(err.response?.data || err.message)
    res.status(500).json({ error: 'Generation failed' })
  }
}

function buildPrompt(idea:string, mode:string){
  const baseTone = `You are an expert community manager writing for Base (Coinbase's Layer 2). Use casual, friendly, and slightly enthusiastic tone. Keep posts concise and include a short call-to-action.`
  if(mode === 'tweet'){
    return `${baseTone}\n\nWrite a single tweet (<=280 chars) from this idea:\n${idea}`
  }
  if(mode === 'thread'){
    return `${baseTone}\n\nWrite a 5-part thread (each part 1-2 short sentences) about:\n${idea}`
  }
  if(mode === 'mint'){
    return `${baseTone}\n\nWrite a mint announcement (title + 2-3 short bullet points + CTA) for:\n${idea}`
  }
  return `${baseTone}\n\nWrite a short developer-oriented blurb or code snippet for:\n${idea}`
}
```

---

## Wallet & Base integration (notes)

This starter intentionally keeps wallet integration out of the minimal example code. Recommended packages and approach:

- Use `wagmi` + `@web3modal/react` for wallet connection UI.
- Configure chains to include Base mainnet and testnet (Base Goerli). Base RPCs can be provided by Infura/Alchemy.
- For Coinbase Wallet deep integration, consider `@coinbase/wallet-sdk`.

Example wagmi snippet (not included in main starter):

```ts
import { configureChains, createClient, WagmiConfig } from 'wagmi'
import { baseGoerli } from 'viem/chains' // example

const { chains, provider } = configureChains([baseGoerli], [jsonRpcProvider({ rpc: () => ({ http: process.env.NEXT_PUBLIC_BASE_RPC }) })])
const client = createClient({ autoConnect: true, provider })
```

---

## Security & deployment notes

- Never expose `OPENAI_API_KEY` in client-side code. Use server-side env variables and proxy through API routes.
- Rate-limit or require signup on `/api/generate` to avoid abuse and unexpected charges.
- For production, add caching and input sanitization.

---

## Next steps I can help with

- Add wallet connect UI (wagmi + web3modal) and onchain save/drafts.
- Improve prompt engineering with templates per project type (NFT, airdrop, bridge guide).
- Add server-side rate limiting and Stripe payments for premium usage.
- Deploy to Vercel with production environment variables.


---

# End of starter project
