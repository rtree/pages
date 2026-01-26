---
title: "Building an AI-Powered Omnichannel Contact Center with Next.js, Twilio, and Vertex AI"
date: 2026-01-26
classes: wide
---

# Metatel: AI-Powered Omnichannel Contact Center

**Metatel** is a production-ready omnichannel contact center system that integrates **phone calls**, **SMS**, and **LINE messaging** with AI-powered responses using **Google Vertex AI (Gemini)**. Built with **Next.js** and deployed on **Google Cloud Run**, it provides a unified customer communication platform with real-time admin dashboard.

> **📞 Try it yourself!** You can experience the AI-powered IVR system by calling our main line:
>
> <a href="tel:+16176827774" style="display: inline-block; background: linear-gradient(135deg, #007bff, #0056b3); color: white; font-weight: bold; padding: 0.6em 1.2em; border-radius: 12px; text-decoration: none; box-shadow: 0 4px 15px rgba(0,123,255,0.4);">
> 📱 +1-617-682-7774
> </a>

---

## Overview

1. **Unified Customer Identity**
   - Customers are identified across channels (phone, SMS, LINE) using a unified user ID system
   - Phone numbers and LINE IDs are linked to create a single customer profile
   - Conversation history is shared across all channels for contextual AI responses

2. **AI-Powered Responses with Vertex AI Gemini**
   - Uses Google's Gemini 3 Flash model for intelligent, context-aware responses
   - Adapts communication style based on customer intent (quick answers vs. empathetic listening)
   - Automatically extracts and registers phone numbers from conversations

3. **Multi-Channel Support**
   - **Phone (Twilio)**: IVR system with AI greeting, call transfer, and voicemail
   - **SMS (Twilio)**: Automated AI responses to incoming text messages
   - **LINE**: Full chatbot integration with typing indicators and read receipts
   - **BYOC (Bring Your Own Carrier)**: Support for SIP trunking with existing phone numbers

4. **Real-Time Admin Dashboard**
   - Google OAuth authentication for secure access
   - Live conversation view across all channels
   - AI-generated customer profiles and analytics
   - 5-second auto-refresh for real-time updates

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Customer Touchpoints                          │
├─────────────────┬─────────────────┬─────────────────┬──────────────────┤
│   Phone Call    │      SMS        │      LINE       │   BYOC/SIP       │
│   +1-617-xxx    │   +1-617-xxx    │  @company_bot   │  Your Number     │
└────────┬────────┴────────┬────────┴────────┬────────┴────────┬─────────┘
         │                 │                 │                 │
         ▼                 ▼                 ▼                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        Twilio / LINE Platform                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │ Voice API   │  │ SMS API     │  │ Messaging   │  │ BYOC Trunk  │    │
│  │ TwiML       │  │ Webhook     │  │ API         │  │ SIP INVITE  │    │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘    │
└─────────┼────────────────┼────────────────┼────────────────┼───────────┘
          │                │                │                │
          ▼                ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     Next.js Application (Cloud Run)                     │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                        API Routes (App Router)                    │  │
│  │  /api/voice/*      - Phone call handling (TwiML)                 │  │
│  │  /api/sms          - SMS webhook & AI response                   │  │
│  │  /api/line-webhook - LINE message handling                       │  │
│  │  /api/byoc/voice/* - BYOC SIP trunk handling                     │  │
│  │  /api/admin/*      - Admin API (user list, details)              │  │
│  │  /api/auth/*       - Google OAuth flow                           │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                         Shared Libraries                          │  │
│  │  lib/voice-handler.ts  - Unified voice call logic                │  │
│  │  lib/gemini.ts         - Vertex AI Gemini integration            │  │
│  │  lib/firestore.ts      - Database operations                     │  │
│  │  lib/twilio.ts         - Twilio client & helpers                 │  │
│  │  lib/emergency-guard.ts - Safety guards (blacklist/whitelist)    │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
          │                │                │
          ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         Google Cloud Platform                           │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐         │
│  │   Firestore     │  │   Vertex AI     │  │   Cloud Run     │         │
│  │   - users       │  │   Gemini 3      │  │   Container     │         │
│  │   - events      │  │   Flash         │  │   Deployment    │         │
│  │   - phone-links │  │                 │  │                 │         │
│  │   - line-links  │  │                 │  │                 │         │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Database Design

### Unified User Model

The system uses a **unified user identity** approach where customers are identified by a single internal ID, regardless of how they contact you.

```
users/{auto-id}
├── lineUserId: string | null      # LINE user ID (if connected)
├── displayName: string | null     # Display name from LINE
├── phoneNumbers: string[]         # All associated phone numbers
├── createdAt: timestamp
└── updatedAt: timestamp

phone-links/{normalizedPhone}      # Reverse lookup: phone → user
└── userId: string

line-links/{lineUserId}            # Reverse lookup: LINE → user  
└── userId: string

events/{auto-id}                   # All interactions (unified history)
├── userId: string                 # Internal user ID
├── type: 'line_message' | 'phone_call' | 'sms'
├── timestamp: timestamp
├── phoneNumber: string | null
└── content: {
│     role?: 'user' | 'model'      # For messages
│     text?: string
│     direction?: 'inbound' | 'outbound'  # For calls
│   }
```

This design enables:
- **Cross-channel history**: When a customer calls after messaging on LINE, the AI knows the full context
- **Automatic linking**: Phone numbers mentioned in LINE chat are automatically linked to the user
- **Efficient queries**: Composite indexes on `(userId, timestamp)` for fast history retrieval

---

## AI Response System

### Context-Aware Conversations

The Gemini AI adapts its response style based on customer intent:

```typescript
// lib/gemini.ts - System prompt excerpt

const SYSTEM_PROMPT = `You are the LINE customer support for Metaproxy technologies.

【Reading the Room】
Adapt your response based on customer intent:

■ Quick Resolution Type (just stating requests)
→ 1-2 sentences max. "Got it!" and done.

■ Needs Listening Type (troubled, seeking advice)  
→ Listen carefully. Show empathy. Ask one follow-up question.

【How to Identify】
Quick Type:
- "Please do X" - direct requests
- Just sends phone number
- Short, to-the-point messages

Listening Type:
- "I'm having trouble with..."
- Explains situation in detail
- "What should I do about...?"
`;
```

### Phone Number Extraction

The AI automatically detects and registers phone numbers from conversations:

```typescript
export function extractPhoneNumber(text: string): string | null {
  // Japanese formats: 090-1234-5678, 09012345678, +81-90-1234-5678
  const patterns = [
    /(?:\+81[-\s]?)?[0-9]{2,4}[-\s]?[0-9]{2,4}[-\s]?[0-9]{3,4}/g,
    /0[0-9]{9,10}/g,
  ];
  
  for (const pattern of patterns) {
    const match = text.match(pattern);
    if (match) {
      return match[0].replace(/[-\s]/g, '');
    }
  }
  return null;
}
```

---

## Voice Call Flow

### Inbound Call Handling

```
Customer calls → Twilio webhook → /api/voice/route.ts
                                        │
                    ┌───────────────────┴───────────────────┐
                    ▼                                       ▼
            Business Hours?                          After Hours
                    │                                       │
                    ▼                                       ▼
            AI Greeting                              Voicemail
            "Hello! Press 1 for                      Recording
             sales, 2 for support..."                     │
                    │                                       ▼
                    ▼                                   Slack
            /api/voice/gather                        Notification
            (DTMF input)
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
    Press 1     Press 2     Press 9
    Transfer    Transfer    Emergency
    to Sales    to Support  Transfer
        │           │           │
        ▼           ▼           ▼
    /api/voice/dial-status
    (Call result handling)
```

### TwiML Response Generation

```typescript
// lib/voice-handler.ts

export async function handleVoiceCall(
  params: { from: string; to: string; sipCallId?: string; byocTrunkSid?: string },
  options: VoiceHandlerOptions = {}
): Promise<string> {
  const response = createVoiceResponse();
  
  // Check business hours
  if (!isBusinessHours()) {
    response.say({ language: 'ja-JP', voice: 'Polly.Mizuki' },
      'Thank you for calling. We are currently closed...');
    response.record({ /* voicemail settings */ });
    return response.toString();
  }
  
  // Business hours: AI greeting with IVR
  response.say({ language: 'ja-JP', voice: 'Polly.Mizuki' },
    'Thank you for calling Metaproxy technologies...');
  
  response.gather({
    numDigits: 1,
    action: `${publicUrl}${options.gatherUrl}`,
    timeout: 10,
  }).say({ language: 'ja-JP', voice: 'Polly.Mizuki' },
    'Press 1 for new inquiries, 2 for existing projects...');
  
  return response.toString();
}
```

---

## Safety Guards

### Emergency Number Protection

The system implements **dual-layer protection** against accidental calls/SMS to emergency numbers:

```typescript
// lib/emergency-guard.ts

// Layer 1: Blacklist (absolute block)
const BLACKLIST = new Set([
  '911', '110', '119', '118',  // US & Japan emergency
  '112', '999', '000',         // International emergency
]);

// Layer 2: Whitelist (only these patterns allowed)
const WHITELIST_PATTERNS = [
  /^\+81[0-9]{9,10}$/,  // Japan numbers only
];

// Layer 3: Short number block
if (normalized.length <= 6) {
  return { blocked: true, reason: 'Too short' };
}

// For SMS: Additional US number block (A2P 10DLC compliance)
if (normalized.startsWith('+1')) {
  return { blocked: true, reason: 'US numbers blocked - A2P 10DLC not registered' };
}
```

---

## Admin Dashboard

### Real-Time Conversation View

The admin dashboard provides a Discord-like interface for monitoring all customer interactions:

```typescript
// app/admin/page.tsx

// Auto-refresh every 5 seconds
useEffect(() => {
  const refreshInterval = setInterval(() => {
    refreshUsersQuietly();
    if (selectedUser) {
      refreshUserDetailQuietly(selectedUser.id);
    }
  }, 5000);
  return () => clearInterval(refreshInterval);
}, [user, selectedUser]);
```

### Features

- **Unified Timeline**: See phone calls, SMS, and LINE messages in chronological order
- **Channel Badges**: Visual indicators (📞 Phone, 📱 SMS, 💬 LINE) for each interaction
- **AI Customer Profiles**: Automatically generated summaries of customer situation, interests
- **Contact Statistics**: Call count, message count, last interaction dates

---

## Deployment

### Docker Configuration

```dockerfile
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]
```

### Environment Variables

```bash
# Twilio
TWILIO_ACCOUNT_SID=ACxxxx
TWILIO_AUTH_TOKEN=xxxx
TWILIO_PHONE_NUMBER=+1617xxxxxxx

# LINE
LINE_CHANNEL_SECRET=xxxx
LINE_CHANNEL_ACCESS_TOKEN=xxxx

# Google Cloud
GCP_PROJECT_ID=your-project
GEMINI_MODEL=gemini-3-flash-preview

# Application
PUBLIC_URL=https://your-app.run.app
FORWARD_PHONE_NUMBER=+81xxxxxxxxxx
ADMIN_EMAILS=admin@example.com
```

### Google Cloud Run Deployment

```bash
# Build and push
gcloud builds submit --tag gcr.io/PROJECT_ID/metatel

# Deploy
gcloud run deploy metatel \
  --image gcr.io/PROJECT_ID/metatel \
  --platform managed \
  --region asia-northeast1 \
  --allow-unauthenticated
```

---

## Webhook Configuration

### Twilio Console

| Channel | Webhook URL | Method |
|---------|-------------|--------|
| Voice (A call comes in) | `https://your-app.run.app/api/voice` | HTTP POST |
| SMS (A message comes in) | `https://your-app.run.app/api/sms` | HTTP POST |
| BYOC Trunk | `https://your-app.run.app/api/byoc/voice` | HTTP POST |

### LINE Developers Console

| Setting | Value |
|---------|-------|
| Webhook URL | `https://your-app.run.app/api/line-webhook` |
| Use webhook | Enabled |
| Auto-reply | Disabled |

---

## Key Features Summary

| Feature | Technology | Description |
|---------|------------|-------------|
| Voice IVR | Twilio TwiML | Multi-level menu with DTMF input |
| SMS Bot | Twilio + Gemini | AI-powered text responses |
| LINE Bot | LINE Messaging API + Gemini | Full chatbot with typing indicators |
| BYOC | Twilio SIP Trunking | Use your existing phone numbers |
| Unified Identity | Firestore | Cross-channel customer profiles |
| AI Responses | Vertex AI Gemini 3 | Context-aware, style-adaptive replies |
| Admin Dashboard | Next.js + React | Real-time conversation monitoring |
| Safety Guards | Blacklist/Whitelist | Emergency number protection |

---

## Conclusion

Metatel demonstrates how to build a production-ready omnichannel contact center using modern cloud technologies:

- **Next.js App Router** for serverless API routes
- **Twilio** for voice and SMS capabilities
- **LINE Messaging API** for chat integration
- **Google Vertex AI Gemini** for intelligent, contextual responses
- **Firestore** for real-time data synchronization
- **Cloud Run** for scalable, containerized deployment

The unified user identity model ensures customers receive consistent, contextual service regardless of which channel they use to reach you.

---

## References

- [Twilio Voice TwiML](https://www.twilio.com/docs/voice/twiml)
- [LINE Messaging API](https://developers.line.biz/en/docs/messaging-api/)
- [Google Vertex AI](https://cloud.google.com/vertex-ai/docs)
- [Next.js App Router](https://nextjs.org/docs/app)
- [Google Cloud Run](https://cloud.google.com/run/docs)
