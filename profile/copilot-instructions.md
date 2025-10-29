# TrueLocal Development Guide

## Architecture Overview

**Multi-tenant microservices platform** connecting vendors with customers. Core services communicate via HTTP REST APIs, with shared PostgreSQL database and Redis for caching/queues.

### Service Boundaries
- **thetruelocal-backend** (Port 3000) - Core API: auth, vendors, products, orders, rewards
- **thetruelocal-web** (Port 3001) - Customer Next.js 15 app (App Router)
- **thetruelocal-dealer-portal** (Port 3002) - Vendor Next.js management portal
- **thetruelocal-admin** (Port 3003) - Admin Next.js dashboard
- **Specialized Services** - payments (3004), search (3005), notifications (3006), rewards (3007)

## Backend Development Patterns

### Layer Architecture (Route → Controller → Service)
```typescript
// Routes define endpoints with Swagger docs
router.post('/profile', authenticateVendor, vendorController.updateProfile.bind(vendorController));

// Controllers handle HTTP concerns only
async updateProfile(req: Request, res: Response, next: NextFunction) {
  const result = await vendorService.updateProfile(req.user!.vendorId, req.body);
  res.json(success(result, 'Profile updated'));
}

// Services contain ALL business logic
export class VendorService {
  async updateProfile(vendorId: string, data: UpdateVendorDto) {
    // Validation, database operations, webhooks
  }
}
```

**Critical**: Always bind controller methods when registering routes: `.bind(vendorController)`

### Authentication Patterns
- **Customer/Admin**: JWT via `authenticate` middleware → `req.user` (userId, email, role)
- **Vendors**: JWT via `authenticateVendor` → `req.vendor` (vendorId, email)
- **Partner APIs**: API key via `authenticateApiKey` → `req.store` (storeId, scopes)

Check `src/middleware/auth.ts` for all auth strategies.

### Error Handling
Use custom error classes that errorHandler middleware auto-converts to JSON:
```typescript
throw new UnauthorizedError('Invalid credentials');
throw new NotFoundError('Product not found');
throw new ValidationError('Email required');
```

Never manually send error responses—let middleware handle via `next(error)`.

### Response Format (Standardized)
```typescript
import { ResponseBuilder } from '~src/utils/response';

// Success
ResponseBuilder.success(res, data, 'Message', 200, { page, total });

// Error (auto-handled by middleware)
throw new BadRequestError('Invalid input');
```

All responses follow `{ success, data?, message?, error?, meta? }` structure.

### Database Access
- **ORM**: Prisma Client (`prisma`) from `config/database`
- **Schema Location**: `prisma/schema.prisma` (19+ tables)
- **Migrations**: `npx prisma migrate dev` (Docker: `docker-compose exec backend npx prisma migrate dev`)
- **Always** use transactions for multi-table operations:
```typescript
await prisma.$transaction(async (tx) => {
  await tx.order.create({ data: orderData });
  await tx.inventory.update({ where: { id }, data: { stock: { decrement: qty }}});
});
```

### Redis & Caching
- **Import**: `redis` from `config/redis` (for caching), `bullRedis` for queues
- **Pattern**: Use `CacheService` class for namespaced keys with TTL
- **BullMQ**: Webhook queue at `jobs/webhook.queue.ts` for async event processing

### Background Jobs
Webhooks are queued via BullMQ to Partner APIs on events:
```typescript
await this.webhookQueue.triggerEvent(storeId, 'INVENTORY_LOW', { productId, stock });
```

Check `src/jobs/webhook.queue.ts` for queue implementation.

## Frontend Development Patterns (Next.js)

### API Communication
Use `lib/api.ts` helpers—**never** direct fetch:
```typescript
import { apiRequest, fetchWithAuth } from '@/lib/api';

// Auto-handles token refresh on 401
const data = await apiRequest<Product[]>('/api/products');
```

### State Management
- **Global Auth**: Zustand store at `store/authStore.ts` (token, user, login/logout)
- **Server Data**: TanStack Query for caching/refetching
- **Forms**: React Hook Form + Zod validation

### Authentication Flow (OTP-based)
1. POST `/api/auth/send-otp` with email
2. POST `/api/auth/verify-otp` with email + OTP → returns JWT
3. Store token in Zustand + cookies
4. Middleware (`middleware.ts`) protects routes by verifying JWT

**Vendor Portal**: Same flow but uses `authenticateVendor` backend middleware.

## Docker Development Workflow

### Full Stack Start
```bash
# Root directory
docker-compose up -d  # Starts all 8+ services + Postgres + Redis
docker-compose logs -f backend  # Watch backend logs
```

### Single Service Development
```bash
cd thetruelocal-backend
npm run dev  # Local dev with hot reload
# Ensure Postgres/Redis running: docker-compose up postgres redis
```

### Database Commands
```bash
# Backend migrations
docker-compose exec backend npx prisma migrate dev
docker-compose exec backend npx prisma studio  # GUI at :5555

# Reset DB (⚠️ destroys data)
docker-compose down -v && docker-compose up -d
```

## Key Developer Commands

### Backend Testing
```bash
npm run test           # Jest unit tests
npm run test:coverage  # With coverage report
npm run seed           # Seed test data + get API key
```

**Test Pattern**: Tests live in `tests/` directory, mirroring `src/` structure.

### Swagger API Docs
- Auto-generated from JSDoc `@swagger` comments in route files
- Access at: http://localhost:3000/api/docs
- **Always document** new endpoints with Swagger annotations

## Module Organization

### Backend Modules (`src/modules/`)
Feature-specific modules with full route/controller/service/middleware stacks:
- `admin/` - Admin-specific APIs
- `dealers/`, `users/`, `orders/`, `webhooks/` - Domain modules

Use modules for **complex features** that need isolation. Standard CRUD uses flat `routes/`, `controllers/`, `services/`.

### Path Aliases
Backend uses `~src/` prefix:
```typescript
import { prisma } from '~src/config/database';
import { logger } from '~src/utils/logger';
```

Frontend uses `@/` prefix:
```typescript
import { Button } from '@/components/ui/button';
```

## Data Flow Example: Order Creation

1. **Frontend** → POST `/api/orders` with cart items
2. **Backend Route** → `authenticate` middleware verifies JWT
3. **Controller** → Parses request, calls `orderService.create()`
4. **Service** → 
   - Validates inventory via `productsService`
   - Creates Order + OrderItems + VendorOrders in transaction
   - Queues webhook: `webhookQueue.triggerEvent('ORDER_CREATED')`
   - Calls rewards API: POST `http://rewards:3007/api/rewards/earn`
5. **Background Worker** → Sends webhook to vendor URLs
6. **Response** → Returns order via `ResponseBuilder.created()`

## Environment Variables (Critical)

Required `.env` values for backend:
```bash
DATABASE_URL=postgresql://truelocal:truelocal_dev@postgres:5432/truelocal_db
REDIS_URL=redis://redis:6379
JWT_SECRET=change-in-production
SMTP_HOST=mailpit  # Local testing
STRIPE_SECRET_KEY=sk_test_...
```

Frontend apps need `NEXT_PUBLIC_API_URL=http://localhost:3000`

## Common Gotchas

1. **Prisma Changes**: Always run `npx prisma generate` after schema edits before `migrate dev`
2. **Controller Methods**: Must `.bind()` when registering or `this` breaks
3. **Vendor vs Customer**: Different auth middleware, don't mix in same route
4. **API Keys**: Hashed in DB—use `hashApiKey()` from `utils/helpers` for lookups
5. **Redis Connection**: Two clients required—`redis` (caching) vs `bullRedis` (queues) due to BullMQ needs

## Rewards System Integration

Multi-tier gamification with points, badges, challenges. See `REWARDS_TECHNICAL_GUIDE.md`.

**Earning Points**: Backend calls `POST /api/rewards/earn` after order completion.  
**Redemption**: Frontend calls `POST /api/rewards/redeem` during checkout.

Rewards service is **separate microservice** with own Prisma schema at `thetruelocal-rewards/prisma/`.

## Debug Commands

```bash
# Check service health
curl http://localhost:3000/health

# Test with Partner API key
curl -H "X-API-Key: tlk_xxx" http://localhost:3000/api/v1/partners/products

# View Redis keys
docker-compose exec redis redis-cli KEYS "*"

# Watch database queries
docker-compose exec backend npx prisma studio
```

## When Extending the System

- **New endpoint**: Route (Swagger) → Controller → Service → Update this doc's example
- **New entity**: Update `schema.prisma` → `migrate dev` → Generate client
- **New microservice**: Copy structure from existing, add to `docker-compose.yml`
- **Frontend form**: Use shadcn/ui components + React Hook Form + Zod schema

## AI Development Guidelines

### Code Style & Communication
- **Never use emojis** in code, comments, commit messages, or documentation unless explicitly requested
- **Never reference AI agents** (e.g., "Copilot", "AI assistant", "agent") in code comments or commit messages
- Write professional, human-style commit messages and documentation

### Git Workflow
- **Never commit directly to `main` branch**
- Always create feature branches: `feature/description`, `fix/bug-name`, `refactor/component`
- Use descriptive branch names that reflect the work being done
- Commit messages should be clear and follow conventional commits when possible

### Best Practices
- Keep code changes focused and atomic
- Update tests when modifying functionality
- Run `npm run test` before proposing changes
- Ensure Prisma migrations are generated for schema changes
- Document breaking changes in commit messages
