# Cloudflare Worker Setup for Real-Time Chat

Your Worker at `https://icy-art-12c8.definitelynotpropaganda.workers.dev` needs to be updated to support fetching messages from Matrix.

## Update Your Worker Code

Replace your Cloudflare Worker code with this:

```javascript
const MATRIX_HOME_SERVER = 'https://matrix.org';
const MATRIX_ROOM_ID = '!bxksvGChRQwBgnuARN:matrix.org';

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

1. **Get Matrix Bot Token**
   - Create a Matrix bot account at https://matrix.to (or use an existing one)
   - Log in and go to Settings → Help & About
   - Scroll down and create an access token

2. **Deploy to Cloudflare Workers**
   - Go to your Cloudflare Workers dashboard
   - Update your worker code with the above (or create a new one)
   - Add the Matrix bot token as an environment variable:
     - In wrangler.toml: `MATRIX_BOT_TOKEN = "your_token_here"`
     - Or via the Cloudflare dashboard: Environment Variables → Add Variable

3. **Verify**
   - Test with: `curl "https://your-worker.workers.dev?action=getMessages&limit=3"`
   - Should return JSON with recent messages from the room

## Notes

- The bot posts messages with its own user ID (you might want to customize how it displays handles)
- Messages are fetched in reverse chronological order (newest first) then reversed to display oldest first
- The worker handles CORS automatically
