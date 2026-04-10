// /api/claude.js
// Vercel serverless proxy for the Anthropic Messages API.
//
// Why this exists:
//   The browser must NEVER hold the Anthropic API key.
//   The frontend posts the same shape it would send to Anthropic directly;
//   this function forwards it with the key attached from ANTHROPIC_API_KEY.
//
// Env vars required (set these in Vercel → Project → Settings → Environment Variables):
//   ANTHROPIC_API_KEY   — your sk-ant-... key

export const config = {
  runtime: 'nodejs',
  maxDuration: 60
};

export default async function handler(req, res) {
  // Basic CORS (only needed if you ever call this from another origin)
  res.setHeader('Access-Control-Allow-Origin', '*');
  res.setHeader('Access-Control-Allow-Methods', 'POST, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type');

  if (req.method === 'OPTIONS') {
    return res.status(204).end();
  }

  if (req.method !== 'POST') {
    return res.status(405).json({ error: { message: 'Method not allowed' } });
  }

  const apiKey = process.env.ANTHROPIC_API_KEY;
  if (!apiKey) {
    return res.status(500).json({
      error: {
        message: 'ANTHROPIC_API_KEY is not configured. Add it in Vercel → Project → Settings → Environment Variables, then redeploy.'
      }
    });
  }

  // Accept the body whether Vercel has parsed it or not
  let body = req.body;
  if (typeof body === 'string') {
    try { body = JSON.parse(body); } catch { body = {}; }
  }
  body = body || {};

  // Sensible defaults, allow frontend overrides
  const payload = {
    model: body.model || 'claude-sonnet-4-5',
    max_tokens: Math.min(Math.max(parseInt(body.max_tokens, 10) || 1500, 1), 8192),
    messages: Array.isArray(body.messages) ? body.messages : [],
    ...(body.system ? { system: body.system } : {}),
    ...(body.temperature != null ? { temperature: body.temperature } : {})
  };

  if (payload.messages.length === 0) {
    return res.status(400).json({ error: { message: 'messages array is required' } });
  }

  try {
    const upstream = await fetch('https://api.anthropic.com/v1/messages', {
      method: 'POST',
      headers: {
        'content-type': 'application/json',
        'x-api-key': apiKey,
        'anthropic-version': '2023-06-01'
      },
      body: JSON.stringify(payload)
    });

    const data = await upstream.json();

    if (!upstream.ok) {
      // Forward Anthropic's status & error payload so the UI shows the real reason
      return res.status(upstream.status).json(data);
    }

    return res.status(200).json(data);
  } catch (err) {
    return res.status(502).json({
      error: {
        message: 'Upstream request to Anthropic failed: ' + (err?.message || String(err))
      }
    });
  }
}
