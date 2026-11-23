# Cloudflare Worker Setup: Real-Time Chat + Live Feed Proxy

The Worker at `https://dnf.definitelynotpropaganda.workers.dev` (`dnp-chat-bot` in
`wrangler.jsonc`) handles two jobs:

1. Relaying chat messages to and from the Matrix room
2. Fetching RSS feeds for the News Feed and homepage server-side, so the
   browser never has to fight CORS or depend on flaky third-party proxy
   services (allorigins, corsproxy, r.jina.ai, etc. come and go and are why
   feed sources sometimes show as "unavailable")

## Worker Code

Paste this into the Worker (Cloudflare dashboard → your worker → **Edit code**):

```javascript
const MATRIX_HOME_SERVER = 'https://matrix.org';
const MATRIX_ROOM_ID = '!wqmkZHnfQmDMgBKkDl:matrix.org';

// Only these hosts can be requested through the feed proxy — keeps the
// Worker from being abused as an open proxy for arbitrary URLs.
const ALLOWED_FEED_HOSTS = [
  'aljazeera.com', 'www.aljazeera.com',
  'trtworld.com', 'www.trtworld.com',
  'aa.com.tr', 'www.aa.com.tr',
  'middleeasteye.net', 'www.middleeasteye.net',
  '972mag.com', 'www.972mag.com',
  'cgtn.com', 'www.cgtn.com',
  'presstv.ir', 'www.presstv.ir',

  // US / Western outlets — added so the Live Feed can show their reporting
  // side-by-side with the independent sources above (flagged on the page).
  'feeds.npr.org',
  'rss.nytimes.com',
  'feeds.washingtonpost.com',
  'rss.cnn.com',
];

export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);

    // Handle CORS
    if (request.method === 'OPTIONS') {
      return new Response(null, {
        headers: {
          'Access-Control-Allow-Origin': '*',
          'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
          'Access-Control-Allow-Headers': 'Content-Type',
        },
      });
    }

    // Fetch messages from Matrix room
    if (request.method === 'GET' && url.searchParams.get('action') === 'getMessages') {
      const limit = parseInt(url.searchParams.get('limit')) || 10;
      return handleGetMessages(limit, env);
    }

    // Proxy an RSS/Atom feed (server-side fetch — no CORS issues)
    if (request.method === 'GET' && url.searchParams.get('action') === 'fetchFeed') {
      return handleFetchFeed(url.searchParams.get('url'));
    }

    // Post message to Matrix
    if (request.method === 'POST') {
      const body = await request.json();
      return handlePostMessage(body, env);
    }

    return new Response('Not Found', { status: 404 });
  },
};

async function handleGetMessages(limit, env) {
  try {
    const token = env.MATRIX_BOT_TOKEN;
    if (!token) {
      throw new Error('MATRIX_BOT_TOKEN not configured');
    }

    // Fetch messages from Matrix room
    const response = await fetch(
      `${MATRIX_HOME_SERVER}/_matrix/client/v3/rooms/${encodeURIComponent(MATRIX_ROOM_ID)}/messages?dir=b&limit=${limit}`,
      {
        method: 'GET',
        headers: {
          'Authorization': `Bearer ${token}`,
        },
      }
    );

    if (!response.ok) {
      throw new Error(`Matrix API error: ${response.status}`);
    }

    const data = await response.json();
    const messages = data.chunk
      .filter(event => event.type === 'm.room.message' && event.content.msgtype === 'm.text')
      .reverse() // Oldest first
      .map(event => ({
        event_id: event.event_id,
        sender: event.sender,
        sender_name: event.sender.replace('@', '').split(':')[0],
        body: event.content.body,
        origin_server_ts: event.origin_server_ts,
      }));

    return new Response(JSON.stringify({ messages }), {
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*',
      },
    });
  } catch (error) {
    return new Response(JSON.stringify({ error: error.message }), {
      status: 500,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*',
      },
    });
  }
}

async function handleFetchFeed(feedUrl) {
  const corsHeaders = { 'Access-Control-Allow-Origin': '*' };

  if (!feedUrl) {
    return new Response(JSON.stringify({ error: 'Missing url parameter' }), {
      status: 400,
      headers: { 'Content-Type': 'application/json', ...corsHeaders },
    });
  }

  let parsed;
  try {
    parsed = new URL(feedUrl);
  } catch {
    return new Response(JSON.stringify({ error: 'Invalid url' }), {
      status: 400,
      headers: { 'Content-Type': 'application/json', ...corsHeaders },
    });
  }

  if (!ALLOWED_FEED_HOSTS.includes(parsed.hostname)) {
    return new Response(JSON.stringify({ error: `Host not allowed: ${parsed.hostname}` }), {
      status: 403,
      headers: { 'Content-Type': 'application/json', ...corsHeaders },
    });
  }

  try {
    // Fetching server-side avoids CORS entirely, and a normal browser-like
    // User-Agent gets past the basic bot-blocking that defeats public proxies.
    const response = await fetch(feedUrl, {
      headers: {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0 Safari/537.36',
        'Accept': 'application/rss+xml, application/xml, text/xml, */*',
      },
    });

    if (!response.ok) {
      throw new Error(`Upstream HTTP ${response.status}`);
    }

    const text = await response.text();
    return new Response(text, {
      headers: {
        'Content-Type': 'application/xml; charset=utf-8',
        'Cache-Control': 'public, max-age=180',
        ...corsHeaders,
      },
    });
  } catch (error) {
    return new Response(JSON.stringify({ error: error.message }), {
      status: 502,
      headers: { 'Content-Type': 'application/json', ...corsHeaders },
    });
  }
}

async function handlePostMessage(body, env) {
  const { message, handle } = body;

  if (!message || !handle) {
    return new Response(JSON.stringify({ error: 'Missing message or handle' }), {
      status: 400,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*',
      },
    });
  }

  try {
    const token = env.MATRIX_BOT_TOKEN;
    if (!token) {
      throw new Error('MATRIX_BOT_TOKEN not configured');
    }

    // Post message to Matrix as bot
    const response = await fetch(
      `${MATRIX_HOME_SERVER}/_matrix/client/v3/rooms/${encodeURIComponent(MATRIX_ROOM_ID)}/send/m.room.message`,
      {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          msgtype: 'm.text',
          body: message,
        }),
      }
    );

    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.error || `HTTP ${response.status}`);
    }

    return new Response(JSON.stringify({ success: true }), {
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*',
      },
    });
  } catch (error) {
    return new Response(JSON.stringify({ error: error.message }), {
      status: 500,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*',
      },
    });
  }
}
```

## Setup Steps

1. **Get a Matrix bot token** (first-time setup only)
   - Create a Matrix account for the bot (or use an existing one) and log in
     with Element
   - Go to **Settings → Help & About**, scroll to **Advanced**, and copy the
     access token
   - Make sure the bot account has joined the room in `MATRIX_ROOM_ID`

2. **Deploy the Worker**
   - Go to your Cloudflare Workers dashboard → your worker → **Edit code**
   - Replace the code with the version above, then **Save and Deploy**
   - Store the token as an encrypted secret named `MATRIX_BOT_TOKEN`, either
     in the dashboard (**Settings → Variables and Secrets → Add**, type
     *Secret*) or with `npx wrangler secret put MATRIX_BOT_TOKEN`
   - Don't put the token in `wrangler.jsonc`: that file is committed to this
     public repo
   - If the token is already set, redeploying leaves it in place

3. **Verify chat**
   - `curl "https://dnf.definitelynotpropaganda.workers.dev?action=getMessages&limit=3"`
   - Should return JSON with the room's most recent messages

4. **Verify the feed proxy**
   - `curl "https://dnf.definitelynotpropaganda.workers.dev?action=fetchFeed&url=https://www.aljazeera.com/xml/rss/all.xml"`
   - Should return raw RSS/XML text starting with `<?xml ...`

5. **If you add more news sources later**, add their domain(s) to the
   `ALLOWED_FEED_HOSTS` list at the top of the Worker code, or the proxy
   will reject them with "Host not allowed".

## Notes

- The feed proxy only allows fetching from the specific news domains listed
  in `ALLOWED_FEED_HOSTS` — this stops anyone from using your Worker as a
  general-purpose CORS proxy for arbitrary sites.
- Responses are cached at the edge for 3 minutes (`Cache-Control: public,
  max-age=180`) to cut down on repeat requests to the same feed.
- `feed.html` tries this Worker endpoint first and falls back to the old
  third-party proxy chain if the Worker is unreachable, so nothing breaks
  while you're updating it.
- Chat messages are fetched newest first, then reversed so the page shows
  them oldest first. The bot posts under its own Matrix user ID.
- The Worker handles CORS itself (the `OPTIONS` preflight branch).
