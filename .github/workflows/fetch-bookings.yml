// scripts/fetch-bookings.js
// Requires Node.js 20+ (uses built-in fetch and Headers.getSetCookie())
// No npm packages needed.
//
// Fetches bookings for all resources for the current week + 2 weeks ahead,
// then writes data/bookings.json to the repo root.

'use strict';

const fs   = require('fs');
const path = require('path');

const BASE_URL = 'https://nynashamn.interbookfri.se';

const RESOURCE_MAP = {
  viahallen: { id: 108, name: 'Via' },
  svanis:    { id: 34,  name: 'Svanis' },
  tallbacka: { id: 85,  name: 'Tallbacka' },
  kyrkis:    { id: 55,  name: 'Kyrkis' }
  /* ── Inactive resources (IDs confirmed) ──────────────────
  grondal:   { id: 25,  name: 'Gröndal' },
  gronnere:  { id: 28,  name: 'Gröndal Nedre' },
  sunnerby:  { id: 57,  name: 'Sunnerby' },
  kvarndans: { id: 109, name: 'Kvarnis Dans' },
  kvarnis:   { id: 111, name: 'Kvarnis' },
  kallberga: { id: 5,   name: 'Källberga' },
  vika:      { id: 8,   name: 'Vika' },
  vansta:    { id: 65,  name: 'Vansta' }
  ─────────────────────────────────────────────────────── */
};

/* ── Date helpers ───────────────────────────────────────── */

function getUtcRangeForLocalDate(dateString) {
  const [y, m, d] = dateString.split('-').map(Number);
  return {
    startIso: new Date(Date.UTC(y, m - 1, d - 1, 21, 0, 0)).toISOString(),  // prev-day 21:00 UTC
    endIso:   new Date(Date.UTC(y, m - 1, d,     23, 0, 0)).toISOString()   // curr-day 23:00 UTC
  };
}

function shiftDate(ds, n) {
  const d = new Date(ds + 'T12:00:00Z');
  d.setUTCDate(d.getUTCDate() + n);
  return d.toISOString().slice(0, 10);
}

function getMondayOf(ds) {
  const d = new Date(ds + 'T12:00:00Z');
  const day = d.getUTCDay();  // 0 = Sunday
  d.setUTCDate(d.getUTCDate() + (day === 0 ? -6 : 1 - day));
  return d.toISOString().slice(0, 10);
}

/**
 * Returns YYYY-MM-DD strings from 30 days ago through 60 days ahead (91 days total).
 */
function getFetchDates() {
  const today = new Intl.DateTimeFormat('sv-SE', { timeZone: 'Europe/Stockholm' })
    .format(new Date());
  const start = shiftDate(today, -30);
  return Array.from({ length: 91 }, (_, i) => shiftDate(start, i));
}

function cleanDescription(html) {
  if (!html) return '';
  return html
    .replace(/<br\s*\/?>/gi, '\n')
    .replace(/<[^>]+>/g, '')
    .replace(/\n{3,}/g, '\n\n')
    .trim();
}

/* ── API calls ──────────────────────────────────────────── */

async function getSessionCookie() {
  const resp = await fetch(BASE_URL + '/', {
    headers: { 'Accept': 'text/html,application/xhtml+xml,*/*' },
    redirect: 'follow'
  });
  // getSetCookie() returns each Set-Cookie header as a separate array entry (Node 20+)
  const cookies = resp.headers.getSetCookie?.() ?? [];
  if (cookies.length === 0) {
    // Fallback: try combined header string
    const raw = resp.headers.get('set-cookie') || '';
    return raw.split(/,(?=[^ ])/).map(c => c.split(';')[0].trim()).filter(Boolean).join('; ');
  }
  return cookies.map(c => c.split(';')[0].trim()).join('; ');
}

async function fetchResourceDay(resourceId, dateString, cookieHeader) {
  const range = getUtcRangeForLocalDate(dateString);

  const resp = await fetch(BASE_URL + '/BookingAPI/GetBookingsForSchedule', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Accept':       'application/json, text/plain, */*',
      'Cookie':       cookieHeader,
      'Origin':       BASE_URL,
      'Referer':      BASE_URL + '/#/'
    },
    body: JSON.stringify({
      resources:  [resourceId],
      start:      range.startIso,
      end:        range.endIso,
      isPublic:   true,
      timestamp:  new Date().toISOString()
    }),
    redirect: 'follow'
  });

  const text = await resp.text();
  if (!resp.ok)          throw new Error(`HTTP ${resp.status}: ${text.slice(0, 200)}`);
  if (text.trim()[0] === '<') throw new Error(`Expected JSON, got HTML: ${text.slice(0, 100)}`);

  return JSON.parse(text);
}

async function getBookingsForDay(resourceKey, dateString, cookieHeader) {
  const resource = RESOURCE_MAP[resourceKey];
  try {
    const data   = await fetchResourceDay(resource.id, dateString, cookieHeader);
    const events = (data.events || [])
      .filter(e => e.type === 'normal' && e.status === 'booked' && e.start?.startsWith(dateString))
      .map(e => ({
        start:       e.start.substring(11, 16),
        end:         e.end.substring(11, 16),
        description: cleanDescription(e.description)
      }))
      .sort((a, b) => a.start.localeCompare(b.start));
    return { events, error: '' };
  } catch (err) {
    console.error(`  ERROR ${resourceKey} ${dateString}: ${err.message}`);
    return { events: [], error: `Kunde inte läsa bokningar för ${resource.name}.` };
  }
}

/* ── Main ───────────────────────────────────────────────── */

async function main() {
  console.log('Fetching session cookie...');
  const cookie = await getSessionCookie();
  console.log(`Cookie: ${cookie ? cookie.slice(0, 50) + '…' : '(none — may still work)'}`);

  const dates = getFetchDates();
  const keys  = Object.keys(RESOURCE_MAP);
  console.log(`\nFetching ${dates.length} dates × ${keys.length} resources = ${dates.length * keys.length} calls\n`);

  const result = { generated: new Date().toISOString(), dates: {} };

  for (const date of dates) {
    result.dates[date] = {};
    for (const key of keys) {
      process.stdout.write(`  ${date}  ${key.padEnd(10)}`);
      result.dates[date][key] = await getBookingsForDay(key, date, cookie);
      const n = result.dates[date][key].events.length;
      process.stdout.write(`${n} event(s)\n`);
      await new Promise(r => setTimeout(r, 150));   // polite delay
    }
  }

  const outDir  = path.join(__dirname, '..', 'data');
  const outFile = path.join(outDir, 'bookings.json');
  fs.mkdirSync(outDir, { recursive: true });
  fs.writeFileSync(outFile, JSON.stringify(result, null, 2), 'utf8');
  console.log(`\nWritten → ${outFile}`);
}

main().catch(err => {
  console.error('Fatal:', err);
  process.exit(1);
});
