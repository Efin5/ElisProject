/*
Personal Sports Profile — Single-file React app (App.jsx)

What this file contains:
- A complete React component (default export) you can drop into a Create React App / Vite + React project.
- Tailwind-ready markup (uses Tailwind utility classes).
- UI: top sports selector, center area with three search bars (Team / Player / League), example cards for schedules & stats.
- Fetch stubs that call a backend proxy at /api/espn? (ESPN doesn't offer a free public JSON API; see server instructions below)

How to use / deploy (quick guide):
1) Frontend
   - Create a new repo on GitHub and push a React app (Vite or CRA). Replace src/App.jsx with this file.
   - Install dependencies: react, react-dom, tailwindcss setup (or use a utility CDN in index.html for quick demo).
   - Deploy the frontend on GitHub Pages, Netlify, or Vercel.

2) Backend proxy (required)
   - ESPN pages are not a public JSON API and often block CORS. To get schedule & stats from ESPN, run a small server-side proxy that scrapes ESPN pages or uses an authorized data provider.
   - Example Node/Express proxy (quick & minimal) is included below in comments — it uses node-fetch + cheerio to scrape HTML and return JSON. Run this on Heroku, Render, Vercel Serverless Functions, or your VPS.
   - IMPORTANT: Scraping ESPN may violate their Terms of Use. Consider using a licensed sports data API (SportRadar, TheRundown, APIsports, etc.) for production.

3) How the frontend talks to backend
   - Frontend calls endpoints such as:
       GET /api/espn?sport=nhl&type=schedule&query=...
       GET /api/espn?sport=nfl&type=player&query=Tom+Brady
     The backend is responsible for fetching ESPN (or other) data, transforming into JSON, and returning.

--- Example Node proxy (minimal) ---
// server.js (example, run with: node server.js)
// npm i express node-fetch cheerio cors

/*
const express = require('express');
const fetch = require('node-fetch');
const cheerio = require('cheerio');
const cors = require('cors');
const app = express();
app.use(cors());

app.get('/api/espn', async (req, res) => {
  const sport = (req.query.sport || 'nhl').toLowerCase();
  const type = req.query.type || 'schedule';
  const query = req.query.query || '';

  try {
    // Example: fetch ESPN scoreboard for NFL
    // Mapping sport to ESPN path is necessary. This is minimal and will need expansion.
    let url;
    if (type === 'schedule') {
      // Example scoreboard page (date sensitive)
      url = `https://www.espn.com/${sport}/scoreboard`;
    } else if (type === 'player') {
      url = `https://www.espn.com/search/results?q=${encodeURIComponent(query)}`;
    } else {
      url = `https://www.espn.com/${sport}/`;
    }

    const r = await fetch(url, { headers: { 'User-Agent': 'Mozilla/5.0' } });
    const html = await r.text();
    const $ = cheerio.load(html);

    // Minimal parsing example (very likely needs adaptation per page)
    const items = [];
    $('.scoreboard').each((i, el) => {
      // parse scoreboard entries
      items.push($(el).text().trim());
    });

    res.json({ ok: true, sport, type, url, items: items.slice(0, 20) });
  } catch (err) {
    res.status(500).json({ ok: false, error: err.message });
  }
});

app.listen(process.env.PORT || 3001, () => console.log('Proxy listening'));
*/

/*
Notes & recommendations:
- For production use, prefer an official sports data provider. ESPN scraping will be brittle and may be blocked.
- Cache responses server-side to avoid excessive requests.
- Obey ESPN's robots.txt and Terms of Service.

If you want, I can generate the server code and a GitHub Actions workflow to automatically deploy both the frontend (to GitHub Pages) and the backend (to Render or Railway). Tell me which provider you'd like and I will scaffold it.
*/

import React, { useState, useEffect } from 'react';

const SPORTS = [
  { id: 'nhl', label: 'NHL - Hockey' },
  { id: 'nfl', label: 'NFL - Football' },
  { id: 'mlb', label: 'MLB - Baseball' },
  { id: 'f1', label: 'F1 - Racing' },
  { id: 'soccer', label: 'Soccer' },
  { id: 'golf', label: 'Golf' },
  { id: 'pll', label: 'PLL - Lacrosse' },
  { id: 'college-football', label: 'College Football' },
  { id: 'college-hockey', label: 'College Hockey' },
  { id: 'college-lacrosse', label: 'College Lacrosse' },
  { id: 'college-baseball', label: 'College Baseball' },
  { id: 'college-soccer', label: 'College Soccer' }
];

function SportButton({ sport, active, onClick }) {
  return (
    <button
      onClick={() => onClick(sport.id)}
      className={`px-3 py-1 rounded-md whitespace-nowrap mr-2 mb-2 text-sm border ${
        active ? 'bg-slate-700 text-white' : 'bg-white text-slate-700'
      }`}
    >
      {sport.label}
    </button>
  );
}

function SearchBar({ placeholder, value, onChange, onEnter }) {
  return (
    <input
      value={value}
      onChange={e => onChange(e.target.value)}
      onKeyDown={e => { if (e.key === 'Enter') onEnter(); }}
      placeholder={placeholder}
      className="w-full md:w-1/3 p-3 rounded-md border border-gray-300 focus:outline-none focus:ring-2 focus:ring-slate-300"
    />
  );
}

export default function App() {
  const [selectedSport, setSelectedSport] = useState('nhl');
  const [teamQuery, setTeamQuery] = useState('');
  const [playerQuery, setPlayerQuery] = useState('');
  const [leagueQuery, setLeagueQuery] = useState('');

  const [loading, setLoading] = useState(false);
  const [schedule, setSchedule] = useState(null);
  const [stats, setStats] = useState(null);
  const [message, setMessage] = useState('Select a sport and run a search (Team / Player / League)');

  useEffect(() => {
    // On sport change, clear results
    setSchedule(null);
    setStats(null);
    setMessage('Select a sport and run a search (Team / Player / League)');
  }, [selectedSport]);

  async function fetchFromProxy(type, query) {
    setLoading(true);
    setMessage('Fetching from backend proxy...');
    try {
      const q = query ? `&query=${encodeURIComponent(query)}` : '';
      const resp = await fetch(`/api/espn?sport=${encodeURIComponent(selectedSport)}&type=${encodeURIComponent(type)}${q}`);
      const data = await resp.json();
      setLoading(false);
      return data;
    } catch (err) {
      setLoading(false);
      setMessage('Error contacting backend: ' + err.message);
      return null;
    }
  }

  async function searchTeam() {
    setMessage('Searching team...');
    const data = await fetchFromProxy('team', teamQuery);
    if (!data) return;
    // Backend should return a normalized JSON: { schedule: [...], stats: [...] }
    setSchedule(data.schedule || data.items || []);
    setStats(data.stats || []);
    setMessage(data.ok ? 'Results loaded' : 'No results');
  }

  async function searchPlayer() {
    setMessage('Searching player...');
    const data = await fetchFromProxy('player', playerQuery);
    if (!data) return;
    setSchedule(data.schedule || []);
    setStats(data.stats || data.items || []);
    setMessage(data.ok ? 'Results loaded' : 'No results');
  }

  async function searchLeague() {
    setMessage('Searching league...');
    const data = await fetchFromProxy('league', leagueQuery || selectedSport);
    if (!data) return;
    setSchedule(data.schedule || data.items || []);
    setStats(data.leaderboards || []);
    setMessage(data.ok ? 'Results loaded' : 'No results');
  }

  return (
    <div className="min-h-screen bg-gradient-to-b from-slate-50 to-white text-slate-800 p-6">
      <header className="max-w-6xl mx-auto mb-6">
        <div className="flex items-center justify-between">
          <div>
            <h1 className="text-2xl font-bold">My Sports Profile</h1>
            <p className="text-sm text-slate-500">Schedules & stats across pro + college sports (ESPN data via proxy)</p>
          </div>
          <div className="text-sm text-slate-600">GitHub: your-username / personal-profile</div>
        </div>

        <div className="mt-4 flex flex-wrap">
          {SPORTS.map(s => (
            <SportButton key={s.id} sport={s} active={s.id === selectedSport} onClick={setSelectedSport} />
          ))}
        </div>
      </header>

      <main className="max-w-6xl mx-auto">
        <section className="bg-white p-6 rounded-2xl shadow-sm">
          <h2 className="text-lg font-semibold mb-4">Search {SPORTS.find(s=>s.id===selectedSport)?.label}</h2>

          <div className="flex flex-col md:flex-row md:space-x-4 space-y-3 md:space-y-0 mb-4">
            <SearchBar placeholder="Search Team (press Enter or click)" value={teamQuery} onChange={setTeamQuery} onEnter={searchTeam} />
            <SearchBar placeholder="Search Player (press Enter or click)" value={playerQuery} onChange={setPlayerQuery} onEnter={searchPlayer} />
            <SearchBar placeholder="Search League (press Enter or click)" value={leagueQuery} onChange={setLeagueQuery} onEnter={searchLeague} />
          </div>

          <div className="flex space-x-2">
            <button onClick={searchTeam} className="px-4 py-2 rounded-md bg-slate-800 text-white">Team</button>
            <button onClick={searchPlayer} className="px-4 py-2 rounded-md bg-slate-700 text-white">Player</button>
            <button onClick={searchLeague} className="px-4 py-2 rounded-md bg-slate-600 text-white">League</button>
          </div>

          <div className="mt-4 text-sm text-slate-500">{message}</div>
        </section>

        <section className="mt-6 grid grid-cols-1 md:grid-cols-2 gap-4">
          <div className="bg-white p-6 rounded-xl shadow-sm">
            <h3 className="font-semibold mb-3">Schedule / Scoreboard</h3>
            {loading ? (
              <div>Loading scoreboard...</div>
            ) : !schedule || schedule.length === 0 ? (
              <div className="text-sm text-slate-400">No schedule loaded. Try searching a team or league.</div>
            ) : (
              <ul className="space-y-3">
                {schedule.map((s, idx) => (
                  <li key={idx} className="p-3 border rounded-md bg-slate-50">
                    <div className="text-sm font-medium">{s.title || s.name || s}</div>
                    <div className="text-xs text-slate-500">{s.detail || s.description || ''}</div>
                  </li>
                ))}
              </ul>
            )}
          </div>

          <div className="bg-white p-6 rounded-xl shadow-sm">
            <h3 className="font-semibold mb-3">Player / Team Stats</h3>
            {loading ? (
              <div>Loading stats...</div>
            ) : !stats || stats.length === 0 ? (
              <div className="text-sm text-slate-400">No stats loaded. Try searching a player or team.</div>
            ) : (
              <ul className="space-y-3">
                {stats.map((p, idx) => (
                  <li key={idx} className="p-3 border rounded-md">
                    <div className="flex justify-between">
                      <div>
                        <div className="text-sm font-medium">{p.name || p.player || JSON.stringify(p).slice(0,40)}</div>
                        <div className="text-xs text-slate-500">{p.team || p.position || ''}</div>
                      </div>
                      <div className="text-sm text-slate-600">{p.statline || p.stats || ''}</div>
                    </div>
                  </li>
                ))}
              </ul>
            )}
          </div>
        </section>

        <section className="mt-6 bg-white p-6 rounded-xl shadow-sm">
          <h3 className="font-semibold mb-3">Notes & tips</h3>
          <ul className="list-disc pl-5 text-sm text-slate-600">
            <li>Front-end calls a backend proxy to fetch ESPN or other data sources. This app includes fetch stubs to <code>/api/espn</code>.</li>
            <li>For stable data and production readiness, use a licensed sports data API (SportRadar, ESPN API partners, APIsports, etc.).</li>
            <li>Cache responses server-side and respect rate limits and terms of service.</li>
          </ul>
        </section>

      </main>

      <footer className="max-w-6xl mx-auto mt-8 text-center text-xs text-slate-400">
        Built with ❤️ — replace placeholder URLs and backend with your own. If you'd like, I can scaffold the Node proxy and GitHub Actions files next.
      </footer>
    </div>
  );
}
