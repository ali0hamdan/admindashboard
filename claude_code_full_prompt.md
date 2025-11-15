# CLAUDE CODE: Multi-Tenant WhatsApp AI Bot Admin Dashboard
## Full-Stack SaaS Platform - Autonomous Development Guide

---

## 🎯 MISSION

Build a complete multi-tenant SaaS platform that allows you to sell WhatsApp AI bot management dashboards to multiple businesses. Each business gets their own isolated admin dashboard with complete data separation.

---

## 📋 EXECUTION STRATEGY

Work through **6 PHASES** systematically. After each phase, verify functionality before proceeding.

1. **Phase 1:** Project Setup & Multi-Tenant Database
2. **Phase 2:** Backend API & Super Admin System
3. **Phase 3:** Business Authentication & Admin Management
4. **Phase 4:** Frontend - Super Admin Dashboard
5. **Phase 5:** Frontend - Business Admin Dashboard
6. **Phase 6:** N8N Integration & Testing

---

## 🗄️ PHASE 1: PROJECT SETUP & MULTI-TENANT DATABASE

### Objective
Initialize the project structure and create a multi-tenant database schema with complete data isolation.

### Tasks

#### 1.1 Create Project Structure

```bash
mkdir whatsapp-admin-saas
cd whatsapp-admin-saas
mkdir backend frontend
git init
```

#### 1.2 Backend Initialization

```bash
cd backend
npm init -y
npm install @prisma/client bcrypt cors dotenv express express-rate-limit express-validator jsonwebtoken nodemailer
npm install -D @types/bcrypt @types/cors @types/express @types/jsonwebtoken @types/node @types/nodemailer prisma tsx typescript
npx tsc --init
npx prisma init
```

#### 1.3 Update tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

#### 1.4 Create Multi-Tenant Database Schema

Create `prisma/schema.prisma`:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Business {
  id                String    @id @default(uuid())
  slug              String    @unique
  businessName      String
  ownerName         String
  ownerEmail        String    @unique
  ownerPhone        String
  whatsappNumber    String?
  n8nWebhookSecret  String    @default(uuid())
  
  plan              String    @default("free")
  status            String    @default("active")
  monthlyPrice      Float     @default(0)
  subscriptionDate  DateTime  @default(now())
  expiresAt         DateTime?
  
  messageCount      Int       @default(0)
  messageLimit      Int       @default(1000)
  
  logoUrl           String?
  primaryColor      String    @default("#3B82F6")
  
  createdAt         DateTime  @default(now())
  updatedAt         DateTime  @updatedAt
  
  admins            Admin[]
  customers         Customer[]
  systemSettings    SystemSettings[]
  
  @@index([slug])
  @@index([ownerEmail])
}

model SuperAdmin {
  id          String   @id @default(uuid())
  username    String   @unique
  email       String   @unique
  password    String
  fullName    String
  createdAt   DateTime @default(now())
  
  @@index([username])
  @@index([email])
}

model Admin {
  id          String   @id @default(uuid())
  businessId  String
  business    Business @relation(fields: [businessId], references: [id], onDelete: Cascade)
  
  adminId     String   @unique
  fullName    String
  firstName   String
  lastName    String
  phoneNumber String
  email       String
  password    String
  isOwner     Boolean  @default(false)
  pushToken   String?
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  createdBy   String?
  
  @@index([adminId])
  @@index([email])
  @@index([businessId])
  @@unique([businessId, email])
}

model Customer {
  id              String         @id @default(uuid())
  businessId      String
  business        Business       @relation(fields: [businessId], references: [id], onDelete: Cascade)
  
  phoneNumber     String
  whatsappName    String?
  firstContact    DateTime       @default(now())
  lastContact     DateTime       @updatedAt
  totalMessages   Int            @default(0)
  conversations   Conversation[]
  
  @@unique([businessId, phoneNumber])
  @@index([businessId])
  @@index([phoneNumber])
}

model Conversation {
  id                 String    @id @default(uuid())
  customerId         String
  customer           Customer  @relation(fields: [customerId], references: [id], onDelete: Cascade)
  startedAt          DateTime  @default(now())
  endedAt            DateTime?
  satisfactionRating Int?
  liked              Boolean?
  totalMessages      Int       @default(0)
  chatLogs           ChatLog[]
  
  @@index([customerId])
  @@index([startedAt])
}

model ChatLog {
  id              String       @id @default(uuid())
  conversationId  String
  conversation    Conversation @relation(fields: [conversationId], references: [id], onDelete: Cascade)
  messageId       String       @unique
  timestamp       DateTime     @default(now())
  sender          String
  messageType     String
  content         String       @db.Text
  metadata        Json?
  
  @@index([conversationId])
  @@index([timestamp])
}

model OTP {
  id         String   @id @default(uuid())
  email      String
  code       String
  purpose    String
  businessId String?
  expiresAt  DateTime
  used       Boolean  @default(false)
  createdAt  DateTime @default(now())
  
  @@index([email, code])
}

model SystemSettings {
  id          String    @id @default(uuid())
  businessId  String?
  business    Business? @relation(fields: [businessId], references: [id], onDelete: Cascade)
  key         String
  value       String
  updatedAt   DateTime  @updatedAt
  
  @@unique([businessId, key])
  @@index([key])
  @@index([businessId])
}

model UsageLog {
  id           String   @id @default(uuid())
  businessId   String
  date         DateTime @default(now())
  messageCount Int      @default(0)
  
  @@unique([businessId, date])
  @@index([businessId, date])
}
```

#### 1.5 Create Environment File

Create `backend/.env`:

```env
NODE_ENV=development
PORT=5000

DATABASE_URL="postgresql://username:password@localhost:5432/whatsapp_admin_saas"

JWT_SECRET=your-super-secret-jwt-key-min-32-characters-long
JWT_EXPIRES_IN=7d

SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASS=your-app-specific-password

FRONTEND_URL=http://localhost:5173

N8N_WEBHOOK_SECRET=your-n8n-webhook-secret
```

#### 1.6 Push Database Schema

```bash
npx prisma db push
npx prisma generate
```

#### 1.7 Update package.json Scripts

Add to `backend/package.json`:

```json
{
  "scripts": {
    "dev": "tsx watch src/server.ts",
    "build": "tsc",
    "start": "node dist/server.js",
    "prisma:studio": "npx prisma studio",
    "prisma:generate": "npx prisma generate",
    "seed": "tsx prisma/seed.ts"
  }
}
```

#### 1.8 Create Super Admin Seed Script

Create `prisma/seed.ts`:

```typescript
import { PrismaClient } from '@prisma/client';
import bcrypt from 'bcrypt';

const prisma = new PrismaClient();

async function main() {
  const password = await bcrypt.hash('ChangeThisPassword123!', 10);
  
  const superAdmin = await prisma.superAdmin.upsert({
    where: { username: 'super_admin' },
    update: {},
    create: {
      username: 'super_admin',
      email: 'admin@yourcompany.com',
      password: password,
      fullName: 'Platform Administrator'
    }
  });
  
  console.log('✅ Super admin created:', superAdmin.username);
}

main()
  .catch((e) => {
    console.error(e);
    process.exit(1);
  })
  .finally(async () => {
    await prisma.$disconnect();
  });
```

Run seed:
```bash
npm run seed
```

### ✅ Phase 1 Checkpoint
- [ ] Backend directory structure created
- [ ] All dependencies installed
- [ ] Prisma schema created and pushed
- [ ] Super admin account seeded
- [ ] Can access database via `npx prisma studio`

---

## 🔧 PHASE 2: BACKEND API & SUPER ADMIN SYSTEM

### Objective
Build complete backend API with multi-tenant support, super admin endpoints, and all business logic.

### Backend Directory Structure

```
backend/src/
├── config/
│   └── database.ts
├── controllers/
│   ├── superAdminController.ts
│   ├── authController.ts
│   ├── adminController.ts
│   ├── customerController.ts
│   ├── chatlogController.ts
│   ├── analyticsController.ts
│   └── n8nController.ts
├── middleware/
│   ├── auth.ts
│   ├── businessContext.ts
│   └── errorHandler.ts
├── routes/
│   ├── superAdminRoutes.ts
│   ├── authRoutes.ts
│   ├── adminRoutes.ts
│   ├── customerRoutes.ts
│   ├── chatlogRoutes.ts
│   ├── analyticsRoutes.ts
│   └── n8nRoutes.ts
├── services/
│   ├── emailService.ts
│   └── otpService.ts
├── utils/
│   ├── generateAdminId.ts
│   └── encryption.ts
├── types/
│   └── index.ts
└── server.ts
```

### 2.1 Utility Functions

#### utils/encryption.ts

```typescript
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';

const SALT_ROUNDS = 10;

export async function hashPassword(password: string): Promise<string> {
  return bcrypt.hash(password, SALT_ROUNDS);
}

export async function comparePassword(password: string, hash: string): Promise<boolean> {
  return bcrypt.compare(password, hash);
}

export function generateToken(payload: any): string {
  return jwt.sign(payload, process.env.JWT_SECRET!, {
    expiresIn: process.env.JWT_EXPIRES_IN || '7d'
  });
}

export function verifyToken(token: string): any {
  return jwt.verify(token, process.env.JWT_SECRET!);
}

export function generateRandomPassword(length: number = 12): string {
  const charset = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%^&*';
  let password = '';
  
  password += 'ABCDEFGHIJKLMNOPQRSTUVWXYZ'[Math.floor(Math.random() * 26)];
  password += 'abcdefghijklmnopqrstuvwxyz'[Math.floor(Math.random() * 26)];
  password += '0123456789'[Math.floor(Math.random() * 10)];
  password += '!@#$%^&*'[Math.floor(Math.random() * 8)];
  
  for (let i = password.length; i < length; i++) {
    password += charset[Math.floor(Math.random() * charset.length)];
  }
  
  return password.split('').sort(() => Math.random() - 0.5).join('');
}
```

#### utils/generateAdminId.ts

```typescript
export function generateAdminId(
  businessSlug: string,
  firstName: string,
  lastName: string
): string {
  const firstInitial = firstName.charAt(0).toUpperCase();
  const lastInitial = lastName.charAt(0).toUpperCase();
  
  const now = new Date();
  const day = String(now.getDate()).padStart(2, '0');
  const month = String(now.getMonth() + 1).padStart(2, '0');
  const year = now.getFullYear();
  const dateStr = `${day}${month}${year}`;
  
  const randomNums = Math.floor(1000 + Math.random() * 9000);
  
  const cleanSlug = businessSlug
    .toUpperCase()
    .replace(/[^A-Z0-9]/g, '')
    .substring(0, 10);
  
  return `${cleanSlug}-${firstInitial}${lastInitial}${dateStr}${randomNums}`;
}
```

### 2.2 Services

#### services/otpService.ts

```typescript
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

export async function generateOTP(email: string, purpose: string, businessId?: string): Promise<string> {
  const code = Math.floor(100000 + Math.random() * 900000).toString();
  const expiresAt = new Date(Date.now() + 10 * 60 * 1000);
  
  await prisma.oTP.deleteMany({
    where: { email, purpose, used: false }
  });
  
  await prisma.oTP.create({
    data: { email, code, purpose, expiresAt, businessId }
  });
  
  return code;
}

export async function verifyOTP(email: string, code: string, purpose: string): Promise<boolean> {
  const otp = await prisma.oTP.findFirst({
    where: {
      email,
      code,
      purpose,
      used: false,
      expiresAt: { gt: new Date() }
    }
  });
  
  if (!otp) return false;
  
  await prisma.oTP.update({
    where: { id: otp.id },
    data: { used: true }
  });
  
  return true;
}
```

#### services/emailService.ts

```typescript
import nodemailer from 'nodemailer';

const transporter = nodemailer.createTransport({
  host: process.env.SMTP_HOST,
  port: parseInt(process.env.SMTP_PORT || '587'),
  secure: false,
  auth: {
    user: process.env.SMTP_USER,
    pass: process.env.SMTP_PASS
  }
});

export async function sendOTPEmail(email: string, code: string, purpose: string): Promise<void> {
  const subjects: Record<string, string> = {
    'signup': 'Your Admin Signup OTP Code',
    'admin_creation': 'OTP to Create New Admin',
    'password_reset': 'Password Reset OTP Code'
  };
  
  const messages: Record<string, string> = {
    'signup': `Your OTP code for admin signup is: <strong>${code}</strong>`,
    'admin_creation': `Your OTP code to create a new admin is: <strong>${code}</strong>`,
    'password_reset': `Your OTP code for password reset is: <strong>${code}</strong>`
  };
  
  await transporter.sendMail({
    from: process.env.SMTP_USER,
    to: email,
    subject: subjects[purpose],
    html: `
      <div style="font-family: Arial, sans-serif; padding: 20px;">
        <h2>WhatsApp Admin Dashboard</h2>
        <p>${messages[purpose]}</p>
        <p>This code will expire in 10 minutes.</p>
        <p>If you didn't request this code, please ignore this email.</p>
      </div>
    `
  });
}

export async function sendBusinessWelcomeEmail(
  email: string,
  businessName: string,
  slug: string,
  tempPassword: string
): Promise<void> {
  await transporter.sendMail({
    from: process.env.SMTP_USER,
    to: email,
    subject: `Welcome to WhatsApp Bot Admin - ${businessName}`,
    html: `
      <div style="font-family: Arial, sans-serif; padding: 20px;">
        <h2>Welcome to WhatsApp Bot Admin Dashboard!</h2>
        <p>Your business account has been created.</p>
        <div style="background: #f5f5f5; padding: 15px; border-radius: 5px; margin: 20px 0;">
          <p><strong>Business:</strong> ${businessName}</p>
          <p><strong>Login URL:</strong> ${process.env.FRONTEND_URL}/login</p>
          <p><strong>Business Slug:</strong> ${slug}</p>
          <p><strong>Temporary Password:</strong> ${tempPassword}</p>
        </div>
        <p><strong>Next Steps:</strong></p>
        <ol>
          <li>Visit the login page</li>
          <li>Enter your business slug: ${slug}</li>
          <li>Complete your first-time setup with the OTP we'll send</li>
        </ol>
        <p>If you have any questions, please contact support.</p>
      </div>
    `
  });
}
```

### 2.3 Middleware

#### middleware/auth.ts

```typescript
import { Request, Response, NextFunction } from 'express';
import { verifyToken } from '../utils/encryption';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

export interface AuthRequest extends Request {
  admin?: any;
  superAdmin?: any;
}

export async function authenticateAdmin(
  req: AuthRequest,
  res: Response,
  next: NextFunction
): Promise<void> {
  try {
    const token = req.headers.authorization?.replace('Bearer ', '');
    
    if (!token) {
      res.status(401).json({ error: 'No token provided' });
      return;
    }
    
    const decoded = verifyToken(token);
    
    const admin = await prisma.admin.findUnique({
      where: { id: decoded.id },
      include: { business: true }
    });
    
    if (!admin) {
      res.status(401).json({ error: 'Invalid token' });
      return;
    }
    
    req.admin = admin;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
}

export async function authenticateSuperAdmin(
  req: AuthRequest,
  res: Response,
  next: NextFunction
): Promise<void> {
  try {
    const token = req.headers.authorization?.replace('Bearer ', '');
    
    if (!token) {
      res.status(401).json({ error: 'No token provided' });
      return;
    }
    
    const decoded = verifyToken(token);
    
    const superAdmin = await prisma.superAdmin.findUnique({
      where: { id: decoded.id }
    });
    
    if (!superAdmin) {
      res.status(401).json({ error: 'Invalid token' });
      return;
    }
    
    req.superAdmin = superAdmin;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
}

export async function requireOwner(
  req: AuthRequest,
  res: Response,
  next: NextFunction
): Promise<void> {
  if (!req.admin?.isOwner) {
    res.status(403).json({ error: 'Only owner can perform this action' });
    return;
  }
  next();
}
```

#### middleware/businessContext.ts

```typescript
import { Response, NextFunction } from 'express';
import { AuthRequest } from './auth';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

export async function injectBusinessContext(
  req: AuthRequest,
  res: Response,
  next: NextFunction
): Promise<void> {
  try {
    if (!req.admin) {
      res.status(401).json({ error: 'Authentication required' });
      return;
    }
    
    const business = await prisma.business.findUnique({
      where: { id: req.admin.businessId }
    });
    
    if (!business) {
      res.status(403).json({ error: 'Business not found' });
      return;
    }
    
    if (business.status !== 'active') {
      res.status(403).json({ 
        error: 'Business account is suspended. Please contact support.' 
      });
      return;
    }
    
    req.admin.business = business;
    next();
  } catch (error) {
    res.status(500).json({ error: 'Failed to load business context' });
  }
}
```

#### middleware/errorHandler.ts

```typescript
import { Request, Response, NextFunction } from 'express';

export function errorHandler(
  err: any,
  req: Request,
  res: Response,
  next: NextFunction
): void {
  console.error('Error:', err);
  
  const status = err.status || 500;
  const message = err.message || 'Internal server error';
  
  res.status(status).json({
    error: message,
    ...(process.env.NODE_ENV === 'development' && { stack: err.stack })
  });
}
```

### 2.4 Controllers

Due to length constraints, I'll provide the key controllers. Create these files:

#### controllers/superAdminController.ts

```typescript
import { Response } from 'express';
import { PrismaClient } from '@prisma/client';
import { AuthRequest } from '../middleware/auth';
import { hashPassword, comparePassword, generateToken, generateRandomPassword } from '../utils/encryption';
import { sendBusinessWelcomeEmail } from '../services/emailService';

const prisma = new PrismaClient();

export async function superAdminLogin(req: AuthRequest, res: Response): Promise<void> {
  try {
    const { username, password } = req.body;
    
    const superAdmin = await prisma.superAdmin.findUnique({ where: { username } });
    if (!superAdmin) {
      res.status(401).json({ error: 'Invalid credentials' });
      return;
    }
    
    const isValid = await comparePassword(password, superAdmin.password);
    if (!isValid) {
      res.status(401).json({ error: 'Invalid credentials' });
      return;
    }
    
    const token = generateToken({ id: superAdmin.id, type: 'super_admin' });
    
    res.json({
      token,
      superAdmin: {
        id: superAdmin.id,
        username: superAdmin.username,
        email: superAdmin.email,
        fullName: superAdmin.fullName
      }
    });
  } catch (error: any) {
    res.status(500).json({ error: error.message });
  }
}

export async function getAllBusinesses(req: AuthRequest, res: Response): Promise<void> {
  try {
    const businesses = await prisma.business.findMany({
      orderBy: { createdAt: 'desc' },
      include: {
        _count: {
          select: {
            admins: true,
            customers: true
          }
        }
      }
    });
    
    res.json(businesses);
  } catch (error: any) {
    res.status(500).json({ error: error.message });
  }
}

export async function createBusiness(req: AuthRequest, res: Response): Promise<void> {
  try {
    const {
      businessName,
      slug,
      ownerName,
      ownerEmail,
      ownerPhone,
      whatsappNumber,
      plan
    } = req.body;
    
    // Check if slug or email already exists
    const existing = await prisma.business.findFirst({
      where: {
        OR: [
          { slug },
          { ownerEmail }
        ]
      }
    });
    
    if (existing) {
      res.status(400).json({ error: 'Business slug or email already exists' });
      return;
    }
    
    // Set message limit based on plan
    const limits: Record<string, number> = {
      free: 1000,
      basic: 10000,
      premium: 50000
    };
    
    const tempPassword = generateRandomPassword(16);
    
    const business = await prisma.business.create({
      data: {
        businessName,
        slug,
        ownerName,
        ownerEmail,
        ownerPhone,
        whatsappNumber,
        plan,
        messageLimit: limits[plan] || 1000
      }
    });
    
    // Send welcome email
    await sendBusinessWelcomeEmail(ownerEmail, businessName, slug, tempPassword);
    
    res.json({
      message: 'Business created successfully',
      business
    });
  } catch (error: any) {
    res.status(500).json({ error: error.message });
  }
}

export async function updateBusiness(req: AuthRequest, res: Response): Promise<void> {
  try {
    const { id } = req.params;
    const updates = req.body;
    
    const business = await prisma.business.update({
      where: { id },
      data: updates
    });
    
    res.json(business);
  } catch (error: any) {
    res.status(500).json({ error: error.message });
  }
}

export async function getPlatformAnalytics(req: AuthRequest, res: Response): Promise<void> {
  try {
    const [
      totalBusinesses,
      activeBusinesses,
      totalMessages,
      totalCustomers
    ] = await Promise.all([
      prisma.business.count(),
      prisma.business.count({ where: { status: 'active' } }),
      prisma.business.aggregate({ _sum: { messageCount: true } }),
      prisma.customer.count()
    ]);
    
    res.json({
      totalBusinesses,
      activeBusinesses,
      totalMessages: totalMessages._sum.messageCount || 0,
      totalCustomers
    });
  } catch (error: any) {
    res.status(500).json({ error: error.message });
  }
}
```

For the remaining controllers (authController, adminController, customerController, chatlogController, analyticsController, n8nController), implement following the same patterns from the multi-tenant SaaS prompt, ensuring all queries include businessId filtering.

### 2.5 Routes

Create route files connecting controllers to endpoints. Example:

#### routes/superAdminRoutes.ts

```typescript
import express from 'express';
import {
  superAdminLogin,
  getAllBusinesses,
  createBusiness,
  updateBusiness,
  getPlatformAnalytics
} from '../controllers/superAdminController';
import { authenticateSuperAdmin } from '../middleware/auth';

const router = express.Router();

router.post('/login', superAdminLogin);
router.get('/businesses', authenticateSuperAdmin, getAllBusinesses);
router.post('/businesses', authenticateSuperAdmin, createBusiness);
router.put('/businesses/:id', authenticateSuperAdmin, updateBusiness);
router.get('/analytics', authenticateSuperAdmin, getPlatformAnalytics);

export default router;
```

Create similar route files for: auth, admin, customer, chatlog, analytics, and n8n.

### 2.6 Server Setup

#### src/server.ts

```typescript
import express from 'express';
import cors from 'cors';
import dotenv from 'dotenv';
import rateLimit from 'express-rate-limit';
import { errorHandler } from './middleware/errorHandler';

import superAdminRoutes from './routes/superAdminRoutes';
import authRoutes from './routes/authRoutes';
import adminRoutes from './routes/adminRoutes';
import customerRoutes from './routes/customerRoutes';
import chatlogRoutes from './routes/chatlogRoutes';
import analyticsRoutes from './routes/analyticsRoutes';
import n8nRoutes from './routes/n8nRoutes';

dotenv.config();

const app = express();
const PORT = process.env.PORT || 5000;

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:5173',
  credentials: true
}));
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true, limit: '10mb' }));

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100
});
app.use('/api/', limiter);

app.get('/health', (req, res) => {
  res.json({ status: 'OK', timestamp: new Date().toISOString() });
});

app.use('/api/super-admin', superAdminRoutes);
app.use('/api/auth', authRoutes);
app.use('/api/admins', adminRoutes);
app.use('/api/customers', customerRoutes);
app.use('/api/chatlogs', chatlogRoutes);
app.use('/api/analytics', analyticsRoutes);
app.use('/api/n8n', n8nRoutes);

app.use(errorHandler);

app.listen(PORT, () => {
  console.log(`🚀 Server running on port ${PORT}`);
  console.log(`📊 Environment: ${process.env.NODE_ENV}`);
  console.log(`🌐 Frontend URL: ${process.env.FRONTEND_URL}`);
});
```

### ✅ Phase 2 Checkpoint
- [ ] All controllers implemented
- [ ] All routes configured
- [ ] Middleware set up
- [ ] Server starts without errors
- [ ] Can test endpoints with API client
- [ ] Super admin can login
- [ ] Can create test business

Test:
```bash
npm run dev
# Visit http://localhost:5000/health
```

---

## 🎨 PHASE 3: BUSINESS AUTHENTICATION & ADMIN MANAGEMENT

This phase implements the business-specific authentication flow.

Key implementations needed:
1. Business owner first-time setup with OTP
2. Admin login with business slug + adminID
3. Password reset flow
4. Admin creation by owner

Follow patterns from Phase 2, implementing authController and adminController fully with multi-tenant support.

### ✅ Phase 3 Checkpoint
- [ ] Business owner can complete first setup
- [ ] Admin login works with business slug
- [ ] Password reset sends OTP to owner
- [ ] Owner can create new admins
- [ ] Admin IDs generated correctly (SLUG-AB15112025XXXX)

---

## 🖥️ PHASE 4 & 5: FRONTEND DEVELOPMENT

### Initialize Frontend

```bash
cd frontend
npm create vite@latest . -- --template react-ts
npm install react-router-dom axios zustand @tanstack/react-query
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
npx shadcn-ui@latest init
```

Install Shadcn components:
```bash
npx shadcn-ui@latest add button input card dialog table toast select tabs badge separator avatar label
```

Build two separate dashboards:
1. Super Admin Dashboard (manage all businesses)
2. Business Admin Dashboard (manage own business)

Follow React best practices with TypeScript, implementing all screens from the multi-tenant SaaS prompt.

### ✅ Phase 4-5 Checkpoint
- [ ] Super admin can login and manage businesses
- [ ] Business admin can login with slug
- [ ] Dashboard shows analytics
- [ ] Chat logs viewable
- [ ] Customer management works
- [ ] Admin management works (owner only)

---

## 🔗 PHASE 6: N8N INTEGRATION & TESTING

Add n8n webhook endpoints that:
1. Identify business by webhook secret
2. Log messages with business context
3. Track satisfaction ratings
4. Enforce message limits

### ✅ Phase 6 Checkpoint
- [ ] N8N webhooks log messages correctly
- [ ] Messages tied to correct business
- [ ] Satisfaction tracking works
- [ ] Message limits enforced
- [ ] Data isolation verified

---

## 🚀 AUTONOMOUS EXECUTION GUIDELINES

As Claude Code:

1. **Work systematically** through each phase
2. **Test after each major component** - run the code, verify it works
3. **Ask for credentials** when needed (database URL, SMTP settings)
4. **Create clean, documented code** with TypeScript types
5. **Handle errors gracefully** with try-catch and proper responses
6. **Never commit secrets** - use .env files
7. **Create a PROGRESS.md** file tracking completed tasks

## 🎯 COMPLETION CRITERIA

Project is complete when:
- [ ] Super admin can manage all businesses
- [ ] Businesses completely isolated (test this!)
- [ ] Business owners can manage their admins
- [ ] Chat logs display correctly
- [ ] Analytics show real data
- [ ] N8N integration works
- [ ] All TypeScript compiles
- [ ] No console errors
- [ ] Deployment ready

---

## 🚀 START COMMAND

"I'm ready to build the multi-tenant WhatsApp Admin SaaS platform. I'll work through all 6 phases systematically, testing as I go. Starting with Phase 1: Project Setup & Multi-Tenant Database. I'll pause when I need database credentials or SMTP settings from you."

Good luck! Build something amazing! 🎉
