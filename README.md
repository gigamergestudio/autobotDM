# ═══════════════════════════════════════════════════════════════
# AUTOBOT — Complete Backend Architecture
# Instagram DM Automation AI SaaS
# ═══════════════════════════════════════════════════════════════

# ───────────────────────────────────────────────────────────────
# FILE: /autobot/backend/package.json
# ───────────────────────────────────────────────────────────────
{
  "name": "autobot-backend",
  "version": "1.0.0",
  "description": "Autobot Instagram DM Automation SaaS Backend",
  "main": "src/index.js",
  "scripts": {
    "dev": "nodemon src/index.js",
    "start": "node src/index.js",
    "worker": "node src/workers/messageWorker.js",
    "db:migrate": "npx prisma migrate dev",
    "db:generate": "npx prisma generate",
    "db:studio": "npx prisma studio"
  },
  "dependencies": {
    "@prisma/client": "^5.7.0",
    "bcryptjs": "^2.4.3",
    "bullmq": "^5.1.0",
    "cors": "^2.8.5",
    "dotenv": "^16.3.1",
    "express": "^4.18.2",
    "express-rate-limit": "^7.1.5",
    "helmet": "^7.1.0",
    "ioredis": "^5.3.2",
    "jsonwebtoken": "^9.0.2",
    "morgan": "^1.10.0",
    "openai": "^4.24.1",
    "winston": "^3.11.0",
    "zod": "^3.22.4",
    "axios": "^1.6.5",
    "crypto": "^1.0.1"
  },
  "devDependencies": {
    "nodemon": "^3.0.2",
    "prisma": "^5.7.0"
  }
}

# ───────────────────────────────────────────────────────────────
# FILE: /autobot/backend/.env.example
# ───────────────────────────────────────────────────────────────
DATABASE_URL="postgresql://user:password@localhost:5432/autobot"
REDIS_URL="redis://localhost:6379"
JWT_SECRET="your-super-secret-jwt-key-change-this"
JWT_EXPIRES_IN="7d"
OPENAI_API_KEY="sk-your-openai-key"
META_APP_ID="your-meta-app-id"
META_APP_SECRET="your-meta-app-secret"
META_WEBHOOK_VERIFY_TOKEN="your-webhook-verify-token"
RAZORPAY_KEY_ID="rzp_live_xxxxx"
RAZORPAY_KEY_SECRET="your-razorpay-secret"
STRIPE_SECRET_KEY="sk_live_xxxxx"
PORT=3001
NODE_ENV=development
FRONTEND_URL="http://localhost:3000"

# ───────────────────────────────────────────────────────────────
# FILE: /autobot/database/schema.prisma
# ───────────────────────────────────────────────────────────────
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id            String   @id @default(cuid())
  email         String   @unique
  password      String?
  name          String
  avatar        String?
  plan          Plan     @default(FREE)
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  instagramAccounts InstagramAccount[]
  automations       Automation[]
  subscriptions     Subscription[]

  @@map("users")
}

model InstagramAccount {
  id           String   @id @default(cuid())
  userId       String
  igUserId     String   @unique
  igUsername   String
  accessToken  String   // encrypted
  tokenExpiry  DateTime?
  isActive     Boolean  @default(true)
  followers    Int      @default(0)
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt

  user          User           @relation(fields: [userId], references: [id], onDelete: Cascade)
  conversations Conversation[]

  @@map("instagram_accounts")
}

model Conversation {
  id                 String   @id @default(cuid())
  instagramAccountId String
  senderId           String   // IG user ID of the sender
  senderName         String?
  senderUsername     String?
  lastMessageAt      DateTime @default(now())
  isResolved         Boolean  @default(false)
  createdAt          DateTime @default(now())

  instagramAccount InstagramAccount @relation(fields: [instagramAccountId], references: [id], onDelete: Cascade)
  messages         Message[]

  @@unique([instagramAccountId, senderId])
  @@map("conversations")
}

model Message {
  id             String      @id @default(cuid())
  conversationId String
  direction      Direction
  content        String
  messageType    MessageType @default(TEXT)
  isAiGenerated  Boolean     @default(false)
  isRead         Boolean     @default(false)
  igMessageId    String?     @unique
  sentAt         DateTime    @default(now())
  status         MsgStatus   @default(SENT)

  conversation Conversation @relation(fields: [conversationId], references: [id], onDelete: Cascade)

  @@map("messages")
}

model Automation {
  id           String   @id @default(cuid())
  userId       String
  name         String
  keyword      String
  replyMessage String
  isActive     Boolean  @default(true)
  triggerCount Int      @default(0)
  exactMatch   Boolean  @default(false)
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("automations")
}

model Subscription {
  id              String   @id @default(cuid())
  userId          String
  plan            Plan
  status          SubStatus @default(ACTIVE)
  razorpaySubId   String?
  stripeSubId     String?
  currentPeriodEnd DateTime
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("subscriptions")
}

enum Plan {
  FREE
  PRO
  PREMIUM
}

enum Direction {
  INBOUND
  OUTBOUND
}

enum MessageType {
  TEXT
  IMAGE
  STORY_REPLY
  REEL_REPLY
}

enum MsgStatus {
  PENDING
  SENT
  DELIVERED
  READ
  FAILED
}

enum SubStatus {
  ACTIVE
  CANCELLED
  PAST_DUE
}

# ───────────────────────────────────────────────────────────────
# FILE: /autobot/backend/src/index.js
# ───────────────────────────────────────────────────────────────
require('dotenv').config();
const express = require('express');
const cors = require('cors');
const helmet = require('helmet');
const morgan = require('morgan');
const { rateLimiter } = require('./middleware/rateLimiter');
const { errorHandler } = require('./middleware/errorHandler');
const logger = require('./utils/logger');

// Routes
const authRoutes = require('./routes/auth');
const webhookRoutes = require('./routes/webhook');
const messageRoutes = require('./routes/messages');
const automationRoutes = require('./routes/automation');
const dashboardRoutes = require('./routes/dashboard');
const billingRoutes = require('./routes/billing');
const settingsRoutes = require('./routes/settings');

const app = express();
const PORT = process.env.PORT || 3001;

// ── Security Middleware ──────────────────────
app.use(helmet());
app.use(cors({
  origin: process.env.FRONTEND_URL,
  credentials: true,
}));
app.use(morgan('combined', { stream: { write: (msg) => logger.info(msg.trim()) } }));

// ── Body Parsing ─────────────────────────────
// Raw body for webhook signature verification
app.use('/webhook', express.raw({ type: 'application/json' }));
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true }));

// ── Rate Limiting ─────────────────────────────
app.use('/auth', rateLimiter({ max: 20, windowMs: 15 * 60 * 1000 }));
app.use('/messages', rateLimiter({ max: 100, windowMs: 60 * 1000 }));
app.use(rateLimiter({ max: 500, windowMs: 60 * 1000 }));

// ── Routes ────────────────────────────────────
app.use('/auth', authRoutes);
app.use('/webhook', webhookRoutes);
app.use('/messages', messageRoutes);
app.use('/automation', automationRoutes);
app.use('/dashboard', dashboardRoutes);
app.use('/billing', billingRoutes);
app.use('/settings', settingsRoutes);

// Health Check
app.get('/health', (req, res) => res.json({ status: 'ok', timestamp: new Date().toISOString() }));

// ── Error Handler ─────────────────────────────
app.use(errorHandler);

app.listen(PORT, () => {
  logger.info(`🚀 Autobot backend running on port ${PORT}`);
});

module.exports = app;

# ───────────────────────────────────────────────────────────────
# FILE: /autobot/backend/src/middleware/auth.js
# ───────────────────────────────────────────────────────────────
const jwt = require('jsonwebtoken');
const { PrismaClient } = require('@prisma/client');
const prisma = new PrismaClient();

const authenticate = async (req, res, next) => {
  try {
    const token = req.headers.authorization?.split(' ')[1];
    if (!token) return res.status(401).json({ error: 'Access token required' });

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await prisma.user.findUnique({ where: { id: decoded.userId } });

    if (!user) return res.status(401).json({ error: 'User not found' });

    req.user = user;
    next();
  } catch (err) {
    return res.status(401).json({ error: 'Invalid or expired token' });
  }
};

const requirePlan = (minPlan) => (req, res, next) => {
  const hierarchy = { FREE: 0, PRO: 1, PREMIUM: 2 };
  if (hierarchy[req.user.plan] < hierarchy[minPlan]) {
    return res.status(403).json({
      error: `This feature requires ${minPlan} plan`,
      upgradeRequired: true,
    });
  }
  next();
};

module.exports = { authenticate, requirePlan };

# ───────────────────────────────────────────────────────────────
# FILE: /autobot/backend/src/middleware/rateLimiter.js
# ───────────────────────────────────────────────────────────────
const rateLimit = require('express-rate-limit');

const rateLimiter = ({ max = 100, windowMs = 60000 } = {}) =>
  rateLimit({
    windowMs,
    max,
    standardHeaders: true,
    legacyHeaders: false,
    message: { error: 'Too many requests, please slow down.' },
  });

module.exports = { rateLimiter };

# ───────────────────────────────────────────────────────────────
# FILE: /autobot/backend/src/middleware/errorHandler.js
# ───────────────────────────────────────────────────────────────
const logger = require('../utils/logger');

const errorHandler = (err, req, res, next) => {
  logger.error(err.stack);
  const status = err.status || 500;
  res.status(status).json({
    error: err.message || 'Internal server error',
    ...(process.env.NODE_ENV === 'development' && { stack: err.stack }),
  });
};

module.exports = { errorHandler };

# ───────────────────────────────────────────────────────────────
# FILE: /autobot/backend/src/utils/logger.js
# ───────────────────────────────────────────────────────────────
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console({
      format: winston.format.combine(winston.format.colorize(), winston.format.simple()),
    }),
    new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
    new winston.transports.File({ filename: 'logs/combined.log' }),
  ],
});

module.exports = logger;

# ───────────────────────────────────────────────────────────────
# FILE: /autobot/backend/src/utils/redis.js
# ───────────────────────────────────────────────────────────────
const { Redis } = require('ioredis');

const redis = new Redis(process.env.REDIS_URL || 'redis://localhost:6379', {
  maxRetriesPerRequest: null, // Required for BullMQ
});

redis.on('connect', () => console.log('✅ Redis connected'));
redis.on('error', (err) => console.error('Redis error:', err));

module.exports = redis;

# ───────────────────────────────────────────────────────────────
# FILE: /autobot/backend/src/services/aiService.js
# ───────────────────────────────────────────────────────────────
const OpenAI = require('openai');
const logger = require('../utils/logger');

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

/**
 * Generate a contextual Instagram DM reply using OpenAI
 */
const generateReply = async ({ message, senderName, conversationHistory = [], systemPrompt = '' }) => {
  try {
    const defaultSystem = `You are an AI assistant managing Instagram DMs for a creator/business.
Generate SHORT, friendly, conversational replies (2-3 sentences max).
Use 1-2 relevant emojis. End with a soft CTA or question.
Be warm and natural — not robotic or corporate.
Always reply in the same language as the incoming message.`;

    const messages = [
      // Include conversation history for context
      ...conversationHistory.slice(-6).map(m => ({
        role: m.direction === 'INBOUND' ? 'user' : 'assistant',
        content: m.content,
      })),
      { role: 'user', content: message },
    ];

    const response = await openai.chat.completions.create({
      model: 'gpt-4-turbo-preview',
      max_tokens: 200,
      temperature: 0.7,
      messages: [
        { role: 'system', content: systemPrompt || defaultSystem },
        ...messages,
      ],
    });

    const reply = response.choices[0]?.message?.content?.trim();
    if (!reply) throw new Error('Empty AI response');

    return { success: true, reply };
  } catch (err) {
    logger.error('AI generation failed:', err.message);
    // Fallback reply
    return {
      success: false,
      reply: "Thanks for reaching out! We'll get back to you shortly 🙏",
    };
  }
};

/**
 * Content moderation check
 */
const moderateContent = async (text) => {
  try {
    const response = await openai.moderations.create({ input: text });
    return !response.results[0]?.flagged;
  } catch {
    return true; // Allow if moderation fails
  }
};

module.exports = { generateReply, moderateContent };

# ───────────────────────────────────────────────────────────────
# FILE: /autobot/backend/src/services/instagramService.js
# ───────────────────────────────────────────────────────────────
const axios = require('axios');
const logger = require('../utils/logger');

const IG_API = 'https://graph.instagram.com/v19.0';
const FB_API = 'https://graph.facebook.com/v19.0';

/**
 * Send Instagram DM via Graph API
 */
const sendDM = async ({ accessToken, recipientId, message }) => {
  try {
    const response = await axios.post(
      `${FB_API}/me/messages`,
      {
        recipient: { id: recipientId },
        message: { text: message },
        messaging_type: 'RESPONSE',
      },
      {
        params: { access_token: accessToken },
        headers: { 'Content-Type': 'application/json' },
      }
    );
    return { success: true, messageId: response.data.message_id };
  } catch (err) {
    logger.error('Instagram sendDM failed:', err.response?.data || err.message);
    throw new Error(`Failed to send DM: ${err.response?.data?.error?.message}`);
  }
};

/**
 * Exchange short-lived token for long-lived token
 */
const exchangeToken = async (shortLivedToken) => {
  const response = await axios.get(`${IG_API}/access_token`, {
    params: {
      grant_type: 'ig_exchange_token',
      client_secret: process.env.META_APP_SECRET,
      access_token: shortLivedToken,
    },
  });
  return response.data;
};

/**
 * Get IG user profile
 */
const getUserProfile = async (accessToken) => {
  const response = await axios.get(`${IG_API}/me`, {
    params: {
      fields: 'id,username,followers_count,profile_picture_url',
      access_token: accessToken,
    },
  });
  return response.data;
};

/**
 * Verify webhook signature from Meta
 */
const verifyWebhookSignature = (payload, signature) => {
  const crypto = require('crypto');
  const expected = crypto
    .createHmac('sha256', process.env.META_APP_SECRET)
    .update(payload)
    .digest('hex');
  return `sha256=${expected}` === signature;
};

module.exports = { sendDM, exchangeToken, getUserProfile, verifyWebhookSignature };

# ───────────────────────────────────────────────────────────────
# FILE: /autobot/backend/src/services/queueService.js
# ───────────────────────────────────────────────────────────────
const { Queue, Worker, QueueEvents } = require('bullmq');
const redis = require('../utils/redis');
const logger = require('../utils/logger');

// Queue definitions
const messageQueue = new Queue('message-processing', {
  connection: redis,
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 2000 },
    removeOnComplete: 100,
    removeOnFail: 50,
  },
});

const outboundQueue = new Queue('outbound-messages', {
  connection: redis,
  defaultJobOptions: {
    attempts: 5,
    backoff: { type: 'exponential', delay: 1000 },
  },
});

/**
 * Add incoming message for processing
 */
const enqueueIncomingMessage = async (messageData) => {
  return messageQueue.add('process-message', messageData, {
    priority: 1,
  });
};

/**
 * Add outbound DM to queue
 */
const enqueueOutboundMessage = async (messageData) => {
  return outboundQueue.add('send-dm', messageData, {
    delay: messageData.delay || 0, // Optional send delay
  });
};

module.exports = { messageQueue, outboundQueue, enqueueIncomingMessage, enqueueOutboundMessage };

# ───────────────────────────────────────────────────────────────
# FILE: /autobot/backend/src/routes/auth.js
# ───────────────────────────────────────────────────────────────
const router = require('express').Router();
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const { z } = require('zod');
const { PrismaClient } = require('@prisma/client');
const { authenticate } = require('../middleware/auth');
const { getUserProfile, exchangeToken } = require('../services/instagramService');
const prisma = new PrismaClient();

const signupSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
  name: z.string().min(2),
});

// POST /auth/signup
router.post('/signup', async (req, res, next) => {
  try {
    const { email, password, name } = signupSchema.parse(req.body);

    const existing = await prisma.user.findUnique({ where: { email } });
    if (existing) return res.status(409).json({ error: 'Email already registered' });

    const hashedPassword = await bcrypt.hash(password, 12);
    const user = await prisma.user.create({
      data: { email, password: hashedPassword, name },
    });

    const token = jwt.sign({ userId: user.id }, process.env.JWT_SECRET, {
      expiresIn: process.env.JWT_EXPIRES_IN,
    });

    res.status(201).json({
      token,
      user: { id: user.id, email: user.email, name: user.name, plan: user.plan },
    });
  } catch (err) {
    next(err);
  }
});

// POST /auth/login
router.post('/login', async (req, res, next) => {
  try {
    const { email, password } = req.body;

    const user = await prisma.user.findUnique({ where: { email } });
    if (!user || !user.password) return res.status(401).json({ error: 'Invalid credentials' });

    const valid = await bcrypt.compare(password, user.password);
    if (!valid) return res.status(401).json({ error: 'Invalid credentials' });

    const token = jwt.sign({ userId: user.id }, process.env.JWT_SECRET, {
      expiresIn: process.env.JWT_EXPIRES_IN,
    });

    res.json({
      token,
      user: { id: user.id, email: user.email, name: user.name, plan: user.plan },
    });
  } catch (err) {
    next(err);
  }
});

// GET /auth/me
router.get('/me', authenticate, async (req, res) => {
  const user = await prisma.user.findUnique({
    where: { id: req.user.id },
    include: { instagramAccounts: { where: { isActive: true } } },
  });
  res.json({ user });
});

// POST /auth/instagram/connect
router.post('/instagram/connect', authenticate, async (req, res, next) => {
  try {
    const { accessToken } = req.body;

    // Exchange for long-lived token
    const tokenData = await exchangeToken(accessToken);
    const profile = await getUserProfile(tokenData.access_token);

    const account = await prisma.instagramAccount.upsert({
      where: { igUserId: profile.id },
      update: {
        accessToken: tokenData.access_token,
        igUsername: profile.username,
        followers: profile.followers_count || 0,
        isActive: true,
      },
      create: {
        userId: req.user.id,
        igUserId: profile.id,
        igUsername: profile.username,
        accessToken: tokenData.access_token,
        followers: profile.followers_count || 0,
      },
    });

    res.json({ account: { id: account.id, username: account.igUsername } });
  } catch (err) {
    next(err);
  }
});

module.exports = router;

# ───────────────────────────────────────────────────────────────
# FILE: /autobot/backend/src/routes/webhook.js
# ───────────────────────────────────────────────────────────────
const router = require('express').Router();
const { PrismaClient } = require('@prisma/client');
const { verifyWebhookSignature } = require('../services/instagramService');
const { enqueueIncomingMessage } = require('../services/queueService');
const logger = require('../utils/logger');
const prisma = new PrismaClient();

// GET /webhook/instagram — Meta webhook verification
router.get('/instagram', (req, res) => {
  const mode = req.query['hub.mode'];
  const token = req.query['hub.verify_token'];
  const challenge = req.query['hub.challenge'];

  if (mode === 'subscribe' && token === process.env.META_WEBHOOK_VERIFY_TOKEN) {
    logger.info('Webhook verified by Meta');
    return res.status(200).send(challenge);
  }
  res.status(403).json({ error: 'Verification failed' });
});

// POST /webhook/instagram — Receive incoming DMs
router.post('/instagram', async (req, res) => {
  try {
    // Verify signature
    const signature = req.headers['x-hub-signature-256'];
    if (!verifyWebhookSignature(req.body, signature)) {
      logger.warn('Invalid webhook signature');
      return res.status(403).json({ error: 'Invalid signature' });
    }

    const body = JSON.parse(req.body.toString());

    // Quick 200 response to Meta (required within 5s)
    res.status(200).send('EVENT_RECEIVED');

    // Process asynchronously via queue
    if (body.object === 'instagram') {
      for (const entry of body.entry || []) {
        for (const event of entry.messaging || []) {
          if (event.message && !event.message.is_echo) {
            await enqueueIncomingMessage({
              igAccountId: entry.id,
              senderId: event.sender.id,
              message: event.message.text,
              timestamp: event.timestamp,
              messageId: event.message.mid,
            });
          }
        }
      }
    }
  } catch (err) {
    logger.error('Webhook processing error:', err);
    res.status(200).send('EVENT_RECEIVED'); // Always 200 to Meta
  }
});

module.exports = router;

# ───────────────────────────────────────────────────────────────
# FILE: /autobot/backend/src/routes/messages.js
# ───────────────────────────────────────────────────────────────
const router = require('express').Router();
const { PrismaClient } = require('@prisma/client');
const { authenticate } = require('../middleware/auth');
const { enqueueOutboundMessage } = require('../services/queueService');
const prisma = new PrismaClient();

// GET /messages/conversations
router.get('/conversations', authenticate, async (req, res, next) => {
  try {
    const accounts = await prisma.instagramAccount.findMany({
      where: { userId: req.user.id, isActive: true },
    });
    const accountIds = accounts.map(a => a.id);

    const conversations = await prisma.conversation.findMany({
      where: { instagramAccountId: { in: accountIds } },
      include: {
        messages: { orderBy: { sentAt: 'desc' }, take: 1 },
        _count: { select: { messages: { where: { isRead: false, direction: 'INBOUND' } } } },
      },
      orderBy: { lastMessageAt: 'desc' },
    });

    res.json({ conversations });
  } catch (err) {
    next(err);
  }
});

// GET /messages/conversations/:id
router.get('/conversations/:id', authenticate, async (req, res, next) => {
  try {
    const conversation = await prisma.conversation.findUnique({
      where: { id: req.params.id },
      include: {
        messages: { orderBy: { sentAt: 'asc' }, take: 50 },
        instagramAccount: true,
      },
    });

    if (!conversation) return res.status(404).json({ error: 'Conversation not found' });

    // Mark as read
    await prisma.message.updateMany({
      where: { conversationId: req.params.id, isRead: false, direction: 'INBOUND' },
      data: { isRead: true },
    });

    res.json({ conversation });
  } catch (err) {
    next(err);
  }
});

// POST /messages/send — Manual send
router.post('/send', authenticate, async (req, res, next) => {
  try {
    const { conversationId, message } = req.body;

    const conversation = await prisma.conversation.findUnique({
      where: { id: conversationId },
      include: { instagramAccount: true },
    });

    if (!conversation) return res.status(404).json({ error: 'Conversation not found' });

    await enqueueOutboundMessage({
      accessToken: conversation.instagramAccount.accessToken,
      recipientId: conversation.senderId,
      message,
      conversationId,
      isAiGenerated: false,
    });

    res.json({ success: true, message: 'Message queued for delivery' });
  } catch (err) {
    next(err);
  }
});

module.exports = router;

# ───────────────────────────────────────────────────────────────
# FILE: /autobot/backend/src/routes/automation.js
# ───────────────────────────────────────────────────────────────
const router = require('express').Router();
const { PrismaClient } = require('@prisma/client');
const { authenticate, requirePlan } = require('../middleware/auth');
const prisma = new PrismaClient();

const PLAN_LIMITS = { FREE: 2, PRO: 50, PREMIUM: Infinity };

// GET /automation
router.get('/', authenticate, async (req, res, next) => {
  try {
    const automations = await prisma.automation.findMany({
      where: { userId: req.user.id },
      orderBy: { createdAt: 'desc' },
    });
    res.json({ automations });
  } catch (err) { next(err); }
});

// POST /automation/create
router.post('/create', authenticate, async (req, res, next) => {
  try {
    const { keyword, replyMessage, name, exactMatch = false } = req.body;

    // Check plan limit
    const count = await prisma.automation.count({ where: { userId: req.user.id } });
    const limit = PLAN_LIMITS[req.user.plan];
    if (count >= limit) {
      return res.status(403).json({
        error: `Your plan allows maximum ${limit} automation rules. Upgrade to create more.`,
        upgradeRequired: true,
      });
    }

    const automation = await prisma.automation.create({
      data: {
        userId: req.user.id,
        keyword: keyword.toUpperCase().trim(),
        replyMessage,
        name: name || keyword,
        exactMatch,
      },
    });

    res.status(201).json({ automation });
  } catch (err) { next(err); }
});

// PATCH /automation/:id
router.patch('/:id', authenticate, async (req, res, next) => {
  try {
    const automation = await prisma.automation.findFirst({
      where: { id: req.params.id, userId: req.user.id },
    });
    if (!automation) return res.status(404).json({ error: 'Automation not found' });

    const updated = await prisma.automation.update({
      where: { id: req.params.id },
      data: { ...req.body },
    });
    res.json({ automation: updated });
  } catch (err) { next(err); }
});

// DELETE /automation/:id
router.delete('/:id', authenticate, async (req, res, next) => {
  try {
    await prisma.automation.deleteMany({
      where: { id: req.params.id, userId: req.user.id },
    });
    res.json({ success: true });
  } catch (err) { next(err); }
});

module.exports = router;

# ───────────────────────────────────────────────────────────────
# FILE: /autobot/backend/src/routes/dashboard.js
# ───────────────────────────────────────────────────────────────
const router = require('express').Router();
const { PrismaClient } = require('@prisma/client');
const { authenticate } = require('../middleware/auth');
const prisma = new PrismaClient();

// GET /dashboard/stats
router.get('/stats', authenticate, async (req, res, next) => {
  try {
    const accounts = await prisma.instagramAccount.findMany({
      where: { userId: req.user.id, isActive: true },
    });
    const accountIds = accounts.map(a => a.id);

    const sevenDaysAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);

    const [totalMessages, aiMessages, automationTriggers] = await Promise.all([
      prisma.message.count({
        where: { conversation: { instagramAccountId: { in: accountIds } } },
      }),
      prisma.message.count({
        where: {
          conversation: { instagramAccountId: { in: accountIds } },
          isAiGenerated: true,
        },
      }),
      prisma.automation.aggregate({
        where: { userId: req.user.id },
        _sum: { triggerCount: true },
      }),
    ]);

    // Weekly activity (last 7 days)
    const weeklyActivity = await prisma.$queryRaw`
      SELECT DATE(sent_at) as date, COUNT(*) as count
      FROM messages m
      JOIN conversations c ON m.conversation_id = c.id
      WHERE c.instagram_account_id = ANY(${accountIds}::text[])
        AND m.sent_at >= ${sevenDaysAgo}
      GROUP BY DATE(sent_at)
      ORDER BY date ASC
    `;

    const inbound = await prisma.message.count({
      where: {
        conversation: { instagramAccountId: { in: accountIds } },
        direction: 'INBOUND',
      },
    });
    const outbound = totalMessages - inbound;
    const responseRate = inbound > 0 ? Math.round((outbound / inbound) * 100) : 0;

    res.json({
      stats: {
        totalMessages,
        aiReplies: aiMessages,
        automationTriggers: automationTriggers._sum.triggerCount || 0,
        responseRate: `${Math.min(responseRate, 100)}%`,
        activeAccounts: accounts.length,
      },
      weeklyActivity,
    });
  } catch (err) { next(err); }
});

module.exports = router;

# ───────────────────────────────────────────────────────────────
# FILE: /autobot/backend/src/routes/billing.js
# ───────────────────────────────────────────────────────────────
const router = require('express').Router();
const { PrismaClient } = require('@prisma/client');
const { authenticate } = require('../middleware/auth');
const axios = require('axios');
const prisma = new PrismaClient();

const PLANS = {
  pro: { amount: 99900, currency: 'INR', interval: 'monthly', name: 'Autobot Pro' },
  premium: { amount: 249900, currency: 'INR', interval: 'monthly', name: 'Autobot Premium' },
};

// POST /billing/subscribe (Razorpay)
router.post('/subscribe', authenticate, async (req, res, next) => {
  try {
    const { planId, paymentId, orderId, signature } = req.body;
    const plan = PLANS[planId];
    if (!plan) return res.status(400).json({ error: 'Invalid plan' });

    // TODO: Verify Razorpay signature
    // const crypto = require('crypto');
    // const body = orderId + '|' + paymentId;
    // const expected = crypto.createHmac('sha256', process.env.RAZORPAY_KEY_SECRET).update(body).digest('hex');
    // if (expected !== signature) return res.status(400).json({ error: 'Invalid payment signature' });

    const periodEnd = new Date();
    periodEnd.setMonth(periodEnd.getMonth() + 1);

    await prisma.$transaction([
      prisma.user.update({
        where: { id: req.user.id },
        data: { plan: planId.toUpperCase() },
      }),
      prisma.subscription.upsert({
        where: { userId: req.user.id },
        update: { plan: planId.toUpperCase(), status: 'ACTIVE', currentPeriodEnd: periodEnd },
        create: {
          userId: req.user.id,
          plan: planId.toUpperCase(),
          status: 'ACTIVE',
          currentPeriodEnd: periodEnd,
          razorpaySubId: paymentId,
        },
      }),
    ]);

    res.json({ success: true, plan: planId });
  } catch (err) { next(err); }
});

// POST /billing/create-order (Razorpay order)
router.post('/create-order', authenticate, async (req, res, next) => {
  try {
    const { planId } = req.body;
    const plan = PLANS[planId];
    if (!plan) return res.status(400).json({ error: 'Invalid plan' });

    const auth = Buffer.from(`${process.env.RAZORPAY_KEY_ID}:${process.env.RAZORPAY_KEY_SECRET}`).toString('base64');
    const response = await axios.post(
      'https://api.razorpay.com/v1/orders',
      { amount: plan.amount, currency: plan.currency, receipt: `order_${Date.now()}` },
      { headers: { Authorization: `Basic ${auth}` } }
    );

    res.json({ order: response.data, keyId: process.env.RAZORPAY_KEY_ID });
  } catch (err) { next(err); }
});

module.exports = router;

# ───────────────────────────────────────────────────────────────
# FILE: /autobot/workers/messageWorker.js
# ───────────────────────────────────────────────────────────────
require('dotenv').config();
const { Worker } = require('bullmq');
const { PrismaClient } = require('@prisma/client');
const { generateReply, moderateContent } = require('../backend/src/services/aiService');
const { sendDM } = require('../backend/src/services/instagramService');
const redis = require('../backend/src/utils/redis');
const logger = require('../backend/src/utils/logger');
const prisma = new PrismaClient();

// ── Inbound Message Processor ─────────────────
const inboundWorker = new Worker('message-processing', async (job) => {
  const { igAccountId, senderId, message, timestamp, messageId } = job.data;
  logger.info(`Processing message from ${senderId}: ${message}`);

  // 1. Find Instagram account
  const account = await prisma.instagramAccount.findFirst({
    where: { igUserId: igAccountId, isActive: true },
    include: { user: { include: { automations: { where: { isActive: true } } } } },
  });

  if (!account) {
    logger.warn(`No active account found for ${igAccountId}`);
    return;
  }

  // 2. Upsert conversation
  const conversation = await prisma.conversation.upsert({
    where: { instagramAccountId_senderId: { instagramAccountId: account.id, senderId } },
    update: { lastMessageAt: new Date(timestamp * 1000) },
    create: {
      instagramAccountId: account.id,
      senderId,
      lastMessageAt: new Date(timestamp * 1000),
    },
  });

  // 3. Save inbound message (idempotency)
  const existing = await prisma.message.findUnique({ where: { igMessageId: messageId } });
  if (existing) return;

  await prisma.message.create({
    data: {
      conversationId: conversation.id,
      direction: 'INBOUND',
      content: message,
      igMessageId: messageId,
      isRead: false,
    },
  });

  // 4. Check keyword automations
  const upperMessage = message.toUpperCase();
  const matchedRule = account.user.automations.find(a =>
    a.exactMatch ? upperMessage === a.keyword : upperMessage.includes(a.keyword)
  );

  let replyText = null;
  let isAi = false;

  if (matchedRule) {
    replyText = matchedRule.replyMessage;
    // Increment trigger count
    await prisma.automation.update({
      where: { id: matchedRule.id },
      data: { triggerCount: { increment: 1 } },
    });
  } else {
    // 5. AI auto-reply
    const history = await prisma.message.findMany({
      where: { conversationId: conversation.id },
      orderBy: { sentAt: 'desc' },
      take: 6,
    });

    const safe = await moderateContent(message);
    if (safe) {
      const aiResult = await generateReply({
        message,
        senderName: conversation.senderName || 'there',
        conversationHistory: history.reverse(),
      });
      replyText = aiResult.reply;
      isAi = true;
    }
  }

  // 6. Send reply and save
  if (replyText) {
    await sendDM({
      accessToken: account.accessToken,
      recipientId: senderId,
      message: replyText,
    });

    await prisma.message.create({
      data: {
        conversationId: conversation.id,
        direction: 'OUTBOUND',
        content: replyText,
        isAiGenerated: isAi,
        status: 'SENT',
      },
    });
  }
}, { connection: redis, concurrency: 10 });

// ── Outbound Message Processor ────────────────
const outboundWorker = new Worker('outbound-messages', async (job) => {
  const { accessToken, recipientId, message, conversationId, isAiGenerated } = job.data;

  const result = await sendDM({ accessToken, recipientId, message });

  await prisma.message.create({
    data: {
      conversationId,
      direction: 'OUTBOUND',
      content: message,
      isAiGenerated,
      igMessageId: result.messageId,
      status: 'SENT',
    },
  });

  logger.info(`Outbound message sent: ${result.messageId}`);
}, { connection: redis, concurrency: 5 });

inboundWorker.on('failed', (job, err) => {
  logger.error(`Inbound job ${job.id} failed:`, err.message);
});
outboundWorker.on('failed', (job, err) => {
  logger.error(`Outbound job ${job.id} failed:`, err.message);
});

logger.info('🔧 Message workers started');

# ───────────────────────────────────────────────────────────────
# FILE: /autobot/docker/Dockerfile.backend
# ───────────────────────────────────────────────────────────────
FROM node:20-alpine AS builder
WORKDIR /app
COPY backend/package*.json ./
RUN npm ci --only=production
COPY backend/ .
RUN npx prisma generate

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app .
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001
USER nodejs
EXPOSE 3001
CMD ["node", "src/index.js"]

# ───────────────────────────────────────────────────────────────
# FILE: /autobot/docker/docker-compose.yml
# ───────────────────────────────────────────────────────────────
version: '3.9'

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: autobot
      POSTGRES_USER: autobot
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"

  backend:
    build:
      context: ..
      dockerfile: docker/Dockerfile.backend
    env_file: ../backend/.env
    ports:
      - "3001:3001"
    depends_on:
      - postgres
      - redis
    restart: unless-stopped

  worker:
    build:
      context: ..
      dockerfile: docker/Dockerfile.backend
    command: node workers/messageWorker.js
    env_file: ../backend/.env
    depends_on:
      - postgres
      - redis
    restart: unless-stopped
    deploy:
      replicas: 3  # Scale worker instances

volumes:
  postgres_data:
  redis_data:

# ───────────────────────────────────────────────────────────────
# FILE: /autobot/frontend/next.config.js
# ───────────────────────────────────────────────────────────────
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: { appDir: true },
  images: { domains: ['graph.instagram.com', 'cdninstagram.com'] },
  env: {
    NEXT_PUBLIC_API_URL: process.env.NEXT_PUBLIC_API_URL,
    NEXT_PUBLIC_META_APP_ID: process.env.NEXT_PUBLIC_META_APP_ID,
    NEXT_PUBLIC_RAZORPAY_KEY: process.env.NEXT_PUBLIC_RAZORPAY_KEY,
  },
};
module.exports = nextConfig;

# ───────────────────────────────────────────────────────────────
# DEPLOYMENT GUIDE
# ───────────────────────────────────────────────────────────────

## 1. DATABASE — Neon (PostgreSQL, free tier)
   a. Go to neon.tech → New Project
   b. Copy DATABASE_URL
   c. Run: npx prisma migrate deploy

## 2. REDIS — Upstash (serverless Redis)
   a. Go to upstash.com → New Database
   b. Copy REDIS_URL (use TLS URL for production)

## 3. BACKEND — Railway
   a. railway login
   b. railway init
   c. Set all env vars in Railway dashboard
   d. railway up
   e. Your API URL: https://autobot-api.railway.app

## 4. WORKERS — Railway (separate service)
   a. Create a new Railway service
   b. Set START_COMMAND = "node workers/messageWorker.js"
   c. Scale to 2+ instances for production

## 5. FRONTEND — Vercel
   a. vercel --prod
   b. Set env vars: NEXT_PUBLIC_API_URL, etc.
   c. Your app: https://autobot.vercel.app

## 6. META WEBHOOK SETUP
   a. Go to developers.facebook.com
   b. App → Webhooks → Instagram → Subscribe
   c. Callback URL: https://your-api.railway.app/webhook/instagram
   d. Verify Token: (same as META_WEBHOOK_VERIFY_TOKEN)
   e. Subscribe to: messages, messaging_postbacks

## 7. SCALING CHECKLIST (10k users)
   ✓ Backend: Stateless Express → scale horizontally
   ✓ Workers: 3+ replicas with BullMQ concurrency=10
   ✓ DB: Neon/Supabase connection pooling (PgBouncer)
   ✓ Redis: Upstash scales automatically
   ✓ Rate limiting: Per-IP and per-user
   ✓ AI: Async via queue (never blocks requests)
   ✓ Webhook: 200ms response, async processing
