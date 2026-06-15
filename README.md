# NexusCore_v6_Architecture-
Architecture for NexusCore v6
scripts/
├── deploy-production.sh          ← Main deployment script
├── docker-compose.prod.yml       ← Production Docker Compose
└── .env.production.example
version: '3.8'

services:
  # ===================== FRONTEND =====================
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    image: nexuscore-frontend:latest
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - NEXT_PUBLIC_BACKEND_URL=${NEXT_PUBLIC_BACKEND_URL}
      - NEXT_PUBLIC_HELIUS_RPC_URL=${NEXT_PUBLIC_HELIUS_RPC_URL}
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '1'
          memory: 512M
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ===================== TYPESCRIPT BACKEND =====================
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    image: nexuscore-backend:latest
    restart: unless-stopped
    ports:
      - "4000:4000"
    environment:
      - NODE_ENV=production
      - PORT=4000
      - REDIS_URL=redis://redis:6379
      - SQUADS_MULTISIG=${SQUADS_MULTISIG}
      - HELIUS_API_KEY=${HELIUS_API_KEY}
    depends_on:
      - redis
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1.5'
          memory: 1G
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ===================== RUST HELIUS MICROSERVICE =====================
  helius-service:
    build:
      context: ./rust-helius-service
      dockerfile: Dockerfile
    image: nexuscore-helius-service:latest
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - HELIUS_API_KEY=${HELIUS_API_KEY}
      - LUMINEX_MINT=EyCMRsiSxbLRspptLHNqqMQG8HB2oTZSPWRyWJqXpump
      - TREASURY_TOKEN_ACCOUNT=${TREASURY_TOKEN_ACCOUNT}
      - TREASURY_AUTHORITY=${TREASURY_AUTHORITY}
      - SQUADS_MULTISIG=${SQUADS_MULTISIG}
      - SQUADS_VAULT=${SQUADS_VAULT}
    depends_on:
      - redis
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '2'
          memory: 1.5G
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ===================== REDIS =====================
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes --maxmemory 512mb --maxmemory-policy allkeys-lru
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 1G

  # ===================== MONITORING =====================
  prometheus:
    image: prom/prometheus:v2.53
    restart: unless-stopped
    volumes:
      - ./prometheus:/etc/prometheus
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:11.1
    restart: unless-stopped
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD}
    volumes:
      - ./grafana:/etc/grafana/provisioning
      - grafana_data:/var/lib/grafana
    ports:
      - "3001:3000"

volumes:
  redis_data:
  prometheus_data:
  grafana_data:
#!/bin/bash
set -e

echo "🚀 Starting NexusCore Production Deployment"

# Load environment
if [ -f .env.production ]; then
  export $(grep -v '^#' .env.production | xargs)
else
  echo "❌ .env.production file not found!"
  exit 1
fi

echo "📦 Building all services..."

# Build Frontend
docker build -t nexuscore-frontend:latest ./frontend

# Build TypeScript Backend
docker build -t nexuscore-backend:latest ./backend

# Build Rust Helius Microservice
docker build -t nexuscore-helius-service:latest ./rust-helius-service

echo "✅ All images built successfully"

# Deploy with Docker Compose
echo "🚢 Deploying to production..."

docker compose -f docker/docker-compose.prod.yml down --remove-orphans

docker compose -f docker/docker-compose.prod.yml up -d \
  --scale frontend=2 \
  --scale backend=3 \
  --scale helius-service=2

echo "⏳ Waiting for services to be healthy..."
sleep 15

# Health checks
echo "🔍 Running health checks..."

services=("frontend:3000" "backend:4000" "helius-service:8080")

for service in "${services[@]}"; do
  name=$(echo $service | cut -d: -f1)
  port=$(echo $service | cut -d: -f2)
  
  if curl -sSf http://localhost:$port/health > /dev/null 2>&1; then
    echo "✅ $name is healthy"
  else
    echo "❌ $name health check failed"
  fi
done

echo ""
echo "🎉 NexusCore Production Deployment Complete!"
echo ""
echo "Services running:"
echo "  - Frontend:        http://localhost:3000"
echo "  - Backend:         http://localhost:4000"
echo "  - Helius Service:  http://localhost:8080"
echo "  - Grafana:         http://localhost:3001"
echo ""
echo "To scale further:"
echo "  docker compose -f docker/docker-compose.prod.yml up -d --scale backend=5"
chmod +x scripts/deploy-production.sh
# 1. Copy environment template
cp .env.production.example .env.production
nano .env.production          # Fill in real values

# 2. Deploy everything
./scripts/deploy-production.sh
# Scale individual services
docker compose -f docker/docker-compose.prod.yml up -d --scale backend=5
docker compose -f docker/docker-compose.prod.yml up -d --scale helius-service=3
docker compose -f docker/docker-compose.prod.yml up -d --scale frontend=4

# View running services
docker compose -f docker/docker-compose.prod.yml ps

# View logs
docker compose -f docker/docker-compose.prod.yml logs -f backend
backend/
├── src/
│   ├── index.ts                    # Main Express server
│   ├── config/
│   │   └── env.ts                  # Environment validation
│   ├── routes/
│   │   ├── auth.ts
│   │   ├── trade.ts
│   │   ├── launch.ts
│   │   ├── withdraw.ts             # Squads proposal creation
│   │   ├── monitoring.ts
│   │   └── webhooks.ts
│   ├── middleware/
│   │   ├── auth.ts
│   │   ├── rateLimiter.ts
│   │   └── errorHandler.ts
│   ├── services/
│   │   ├── squads.ts               # Squads Multisig logic
│   │   ├── helius.ts               # Calls to Rust microservice
│   │   ├── redis.ts
│   │   └── pqc.ts                  # Post-Quantum helpers
│   ├── utils/
│   │   ├── logger.ts
│   │   ├── metrics.ts
│   │   └── create-luminex-withdrawal.ts
│   ├── workers/
│   │   └── stream-consumer.ts      # Redis Streams consumer
│   └── types/
│       └── index.ts
├── package.json
├── tsconfig.json
├── Dockerfile
└── .env.example
import express from 'express';
import helmet from 'helmet';
import cors from 'cors';
import { rateLimiter } from './middleware/rateLimiter';
import { errorHandler } from './middleware/errorHandler';
import withdrawRouter from './routes/withdraw';
import tradeRouter from './routes/trade';
import launchRouter from './routes/launch';

const app = express();

app.use(helmet());
app.use(cors());
app.use(express.json({ limit: '1mb' }));
app.use(rateLimiter);

app.use('/api/withdraw', withdrawRouter);
app.use('/api/trade', tradeRouter);
app.use('/api/launch', launchRouter);

app.use(errorHandler);

const PORT = process.env.PORT || 4000;
app.listen(PORT, () => {
  console.log(`🚀 NexusCore Backend running on port ${PORT}`);
});
import express from 'express';
import { createLuminexWithdrawalProposal } from '../utils/create-luminex-withdrawal';

const router = express.Router();

router.post('/luminex', async (req, res, next) => {
  try {
    const { destination, amount } = req.body;

    // Call Rust microservice
    const rustResponse = await fetch('http://helius-service:8080/withdraw/luminex', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ destination, amount }),
    });

    const instructionData = await rustResponse.json();

    if (!instructionData.success) {
      return res.status(400).json(instructionData);
    }

    // Create real Squads proposal
    const proposalIx = await createLuminexWithdrawalProposal(
      instructionData,
      req.body.creator // wallet address
    );

    res.json({
      success: true,
      message: 'Squads withdrawal proposal created',
      proposal: proposalIx,
    });
  } catch (error) {
    next(error);
  }
});

export default router;
import { z } from 'zod';

const envSchema = z.object({
  PORT: z.string().default('4000'),
  REDIS_URL: z.string(),
  SQUADS_MULTISIG: z.string(),
  HELIUS_API_KEY: z.string(),
  NEXT_PUBLIC_SOLANA_RPC: z.string(),
});

export const env = envSchema.parse(process.env);
import { Request, Response, NextFunction } from 'express';

export const errorHandler = (
  err: any,
  req: Request,
  res: Response,
  next: NextFunction
) => {
  console.error(err);

  const status = err.status || 500;
  res.status(status).json({
    success: false,
    error: err.message || 'Internal Server Error',
  });
};
backend/
├── src/
│   ├── index.ts
│   ├── config/
│   │   └── env.ts
│   ├── routes/
│   │   ├── withdraw.ts
│   │   ├── trade.ts
│   │   ├── launch.ts
│   │   └── monitoring.ts
│   ├── middleware/
│   │   ├── errorHandler.ts
│   │   └── rateLimiter.ts
│   ├── services/
│   │   └── squads.ts
│   ├── utils/
│   │   └── create-luminex-withdrawal.ts
│   ├── workers/
│   │   └── stream-consumer.ts
│   └── types/
│       └── index.ts
├── package.json
├── tsconfig.json
├── Dockerfile
└── .env.example
{
  "name": "nexuscore-backend",
  "version": "1.0.0",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "worker": "tsx src/workers/stream-consumer.ts"
  },
  "dependencies": {
    "@sqds/sdk": "^2.0.0",
    "@solana/web3.js": "^1.95.0",
    "@solana/spl-token": "^0.4.0",
    "express": "^4.19.2",
    "helmet": "^7.1.0",
    "cors": "^2.8.5",
    "dotenv": "^16.4.5",
    "zod": "^3.23.8",
    "ioredis": "^5.4.1",
    "pino": "^9.2.0"
  },
  "devDependencies": {
    "typescript": "^5.4.5",
    "tsx": "^4.7.0",
    "@types/express": "^4.17.21",
    "@types/node": "^20.12.12"
  }
}
import express from 'express';
import helmet from 'helmet';
import cors from 'cors';
import dotenv from 'dotenv';
import { errorHandler } from './middleware/errorHandler';
import { rateLimiter } from './middleware/rateLimiter';
import withdrawRouter from './routes/withdraw';
import tradeRouter from './routes/trade';
import launchRouter from './routes/launch';
import monitoringRouter from './routes/monitoring';

dotenv.config();

const app = express();

app.use(helmet());
app.use(cors());
app.use(express.json({ limit: '1mb' }));
app.use(rateLimiter);

app.use('/api/withdraw', withdrawRouter);
app.use('/api/trade', tradeRouter);
app.use('/api/launch', launchRouter);
app.use('/api/monitoring', monitoringRouter);

app.use(errorHandler);

const PORT = process.env.PORT || 4000;
app.listen(PORT, () => {
  console.log(`🚀 NexusCore Backend running on port ${PORT}`);
});
import { z } from 'zod';

const envSchema = z.object({
  PORT: z.string().default('4000'),
  REDIS_URL: z.string(),
  SQUADS_MULTISIG: z.string(),
  HELIUS_API_KEY: z.string(),
  NEXT_PUBLIC_SOLANA_RPC: z.string().url(),
});

export const env = envSchema.parse(process.env);
import express from 'express';
import { createLuminexWithdrawalProposal } from '../utils/create-luminex-withdrawal';

const router = express.Router();

router.post('/luminex', async (req, res, next) => {
  try {
    const { destination, amount, creator } = req.body;

    const rustRes = await fetch('http://helius-service:8080/withdraw/luminex', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ destination, amount }),
    });

    const instructionData = await rustRes.json();

    if (!instructionData.success) {
      return res.status(400).json(instructionData);
    }

    const proposal = await createLuminexWithdrawalProposal(instructionData, creator);

    res.json({
      success: true,
      message: 'Squads withdrawal proposal created',
      proposal,
    });
  } catch (error) {
    next(error);
  }
});

export default router;
import { Connection, PublicKey, TransactionInstruction } from '@solana/web3.js';
import { Multisig } from '@sqds/sdk';

interface RustInstruction {
  program_id: string;
  accounts: Array<{ pubkey: string; is_signer: boolean; is_writable: boolean }>;
  data: string;
}

export async function createLuminexWithdrawalProposal(
  rustInstruction: RustInstruction,
  creatorAddress: string
) {
  const connection = new Connection(process.env.NEXT_PUBLIC_SOLANA_RPC!);
  const multisigPda = new PublicKey(process.env.SQUADS_MULTISIG!);
  const creator = new PublicKey(creatorAddress);

  const instruction = new TransactionInstruction({
    programId: new PublicKey(rustInstruction.program_id),
    keys: rustInstruction.accounts.map(acc => ({
      pubkey: new PublicKey(acc.pubkey),
      isSigner: acc.is_signer,
      isWritable: acc.is_writable,
    })),
    data: Buffer.from(rustInstruction.data, 'base64'),
  });

  const multisig = await Multisig.fromAccountAddress(connection, multisigPda);

  return await multisig.createTransaction({
    multisigPda,
    transactionIndex: await multisig.getNextTransactionIndex(),
    creator,
    instructions: [instruction],
  });
}
import { Request, Response, NextFunction } from 'express';

export const errorHandler = (err: any, req: Request, res: Response, next: NextFunction) => {
  console.error(err);
  res.status(err.status || 500).json({
    success: false,
    error: err.message || 'Internal Server Error',
  });
};
import rateLimit from 'express-rate-limit';

export const rateLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  message: { success: false, error: 'Too many requests' },
});
import Redis from 'ioredis';
import { createConsumerGroup, readFromConsumerGroup, acknowledgeMessage } from '../utils/redis';

const redis = new Redis(process.env.REDIS_URL!);

export async function startStreamConsumer() {
  await createConsumerGroup('events:trades', 'trade-processors');
  await createConsumerGroup('events:launches', 'launch-processors');

  while (true) {
    const messages = await readFromConsumerGroup('events:trades', 'trade-processors', 'worker-1');

    for (const msg of messages) {
      console.log('Processing trade event:', msg.data);
      await acknowledgeMessage('events:trades', 'trade-processors', msg.id);
    }
  }
}

startStreamConsumer();
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build
EXPOSE 4000
CMD ["node", "dist/index.js"]
PORT=4000
REDIS_URL=redis://redis:6379
SQUADS_MULTISIG=your_squads_multisig_pda
HELIUS_API_KEY=your_helius_key
NEXT_PUBLIC_SOLANA_RPC=https://api.mainnet-beta.solana.com
import Redis from 'ioredis';

// ===================== REDIS CLIENT =====================
const redis = new Redis(process.env.REDIS_URL || 'redis://localhost:6379', {
  maxRetriesPerRequest: 3,
  enableReadyCheck: true,
});

redis.on('connect', () => console.log('✅ Connected to Redis'));
redis.on('error', (err) => console.error('❌ Redis Error:', err));

// ===================== PUB/SUB =====================
export const pub = new Redis(process.env.REDIS_URL || 'redis://localhost:6379');
export const sub = new Redis(process.env.REDIS_URL || 'redis://localhost:6379');

// ===================== REDIS STREAMS =====================
export interface StreamMessage {
  id: string;
  data: Record<string, string>;
}

/**
 * Add event to a Redis Stream
 */
export async function addToStream(stream: string, data: Record<string, string>): Promise<string> {
  try {
    const id = await redis.xadd(stream, '*', ...Object.entries(data).flat());
    return id;
  } catch (error) {
    console.error(`[Redis Stream] Failed to add to stream ${stream}`, error);
    throw error;
  }
}

/**
 * Create a Consumer Group (safe - won't error if already exists)
 */
export async function createConsumerGroup(stream: string, group: string): Promise<void> {
  try {
    await redis.xgroup('CREATE', stream, group, '$', 'MKSTREAM');
  } catch (error: any) {
    if (!error.message.includes('BUSYGROUP')) {
      console.error(`[Redis Stream] Failed to create consumer group ${group}`, error);
      throw error;
    }
  }
}

/**
 * Read from a Consumer Group
 */
export async function readFromConsumerGroup(
  stream: string,
  group: string,
  consumer: string,
  count = 10,
  blockMs = 5000
): Promise<StreamMessage[]> {
  try {
    const results = await redis.xreadgroup(
      'GROUP', group, consumer,
      'COUNT', count,
      'BLOCK', blockMs,
      'STREAMS', stream, '>'
    );

    if (!results || results.length === 0) return [];

    return results[0][1].map(([id, fields]: any) => ({
      id: id as string,
      data: Object.fromEntries(
        fields.reduce((acc: any[], curr: any, i: number) => {
          if (i % 2 === 0) acc.push([curr, fields[i + 1]]);
          return acc;
        }, [])
      ),
    }));
  } catch (error) {
    console.error(`[Redis Stream] Failed to read from consumer group`, error);
    return [];
  }
}

/**
 * Acknowledge a message (mark as processed)
 */
export async function acknowledgeMessage(
  stream: string,
  group: string,
  messageId: string
): Promise<void> {
  try {
    await redis.xack(stream, group, messageId);
  } catch (error) {
    console.error(`[Redis Stream] Failed to acknowledge message ${messageId}`, error);
  }
}

// ===================== CACHING =====================
export async function getCache<T>(key: string): Promise<T | null> {
  try {
    const data = await redis.get(key);
    return data ? JSON.parse(data) : null;
  } catch (error) {
    console.error(`[Redis] Failed to get cache for key: ${key}`, error);
    return null;
  }
}

export async function setCache(key: string, value: any, ttlSeconds = 60): Promise<void> {
  try {
    await redis.setex(key, ttlSeconds, JSON.stringify(value));
  } catch (error) {
    console.error(`[Redis] Failed to set cache for key: ${key}`, error);
  }
}

export async function invalidateCacheByPattern(pattern: string): Promise<void> {
  try {
    const keys = await redis.keys(pattern);
    if (keys.length >
import { 
  createConsumerGroup, 
  readFromConsumerGroup, 
  acknowledgeMessage 
} from '../utils/redis';

// ... rest of the worker code
