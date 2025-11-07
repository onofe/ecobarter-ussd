# System Architecture - EcoBarter 2.0 USSD Platform

## Overview
EcoBarter 2.0 is built on a lightweight, stateful architecture designed to handle high volumes of concurrent USSD sessions. The system prioritizes reliability, low latency, and simplicity to serve users on basic mobile devices across areas with limited infrastructure. This architecture transforms the original EcoBarter app into an accessible solution that works within Nigeria's infrastructure constraints.

## Technology Stack

### Frontend (USSD Interface)
- **Platform**: USSD (Unstructured Supplementary Service Data)
- **Gateway**: Africa's Talking USSD API
- **Protocol**: Session-based interactive menus
- **Response Format**: Plain text menus (160 characters per screen)
- **Session Timeout**: 180 seconds (industry standard)

### Backend
- **Runtime**: Node.js v18+
- **Framework**: Express.js v4.18
- **API Type**: RESTful API with USSD webhook endpoints
- **Session Storage**: Redis v7.0 (for USSD session state management)
- **Authentication**: Session-based user identification via phone numbers
- **Key Libraries**: 
  - `africastalking` v0.6+ - Official SDK for USSD and SMS integration
  - `mongoose` v7.0+ - MongoDB object modeling
  - `express-session` v1.17+ - Session middleware
  - `redis` v4.6+ - Redis client for Node.js
  - `dotenv` v16.0+ - Environment variable management
  - `joi` v17.9+ - Input validation

### Database
- **Type**: NoSQL (Document-based)
- **Database**: MongoDB v6.0+
- **ORM**: Mongoose v7.0+
- **Hosting**: MongoDB Atlas (cloud-hosted)
- **Collections**: 
  - Users - User profiles and account information
  - Transactions - Recycling reports and reward redemptions
  - Pickups - Scheduled waste collections
  - CollectionPoints - Drop-off locations database
  - Prices - Current recyclable material rates

### Infrastructure
- **Application Hosting**: Railway (primary) / Heroku (alternative)
- **Database Hosting**: MongoDB Atlas (Free/Shared Tier for MVP)
- **Cache Layer**: Redis Cloud (Free tier: 30MB)
- **USSD/SMS Gateway**: Africa's Talking
- **Monitoring**: Winston (logging) + Sentry (error tracking)
- **Version Control**: GitHub

## Architecture Diagram
```
[User's Phone - Any Mobile Network]
         ↓
[Telco USSD Gateway (MTN/Glo/Airtel/9mobile)]
         ↓
[Africa's Talking USSD API]
         ↓ (HTTP POST Webhook)
[Express.js Backend on Railway/Heroku]
    ↓                    ↓
[Redis Cache]     [MongoDB Atlas]
(Sessions)        (User Data, Transactions)
         ↓
[Africa's Talking SMS API]
         ↓
[User's Phone - SMS Confirmation]
```

## Component Communication

### User → Backend (USSD Request Flow)
1. User dials USSD code (e.g., `*347*123#`) from any phone
2. Telco (MTN/Glo/Airtel/9mobile) routes request to Africa's Talking
3. Africa's Talking sends HTTP POST request to our webhook endpoint: `/api/ussd`
4. Express backend receives: `{ sessionId, phoneNumber, serviceCode, text }`
5. Backend queries Redis for existing session state
6. Backend processes user input based on current menu level
7. Backend updates session state in Redis
8. Backend queries MongoDB for user data, prices, or pickup info (if needed)
9. Backend returns USSD response: `{ text: "Menu options...", action: "CON" }` (CON = continue, END = terminate)
10. Africa's Talking displays menu on user's phone

**Response Time Target**: Under 2 seconds (to prevent session timeout)

### Backend → Database (Data Persistence)
- **Connection**: Mongoose establishes persistent connection pool (max 10 connections for MVP)
- **Queries**: All database operations use indexed fields for O(log n) lookups
  - Primary indexes: `phoneNumber`, `userId`, `sessionId`
- **Writes**: Atomic operations prevent race conditions (e.g., concurrent reward redemptions)
- **Reads**: Frequently accessed data (prices, collection points) cached in Redis for 1 hour

### Backend → Redis (Session Management)
- **Session Creation**: New USSD session stored with 5-minute TTL (time-to-live)
- **Session Data**: `{ phoneNumber, currentStep, tempData, timestamp }`
- **Session Updates**: Each user action updates Redis state
- **Automatic Cleanup**: Expired sessions auto-deleted by Redis TTL mechanism
- **Key Pattern**: `ussd:session:{phoneNumber}:{sessionId}`

### Backend → SMS Service (Notifications)
- **Trigger Events**: 
  - Pickup scheduled → SMS with agent details & ETA
  - Points earned → SMS confirmation with new balance
  - Airtime redeemed → SMS with recharge PIN
- **Async Processing**: SMS requests added to internal queue (don't block USSD response)
- **Delivery**: Africa's Talking SMS API handles delivery to telcos
- **Retry Logic**: Failed SMS retried up to 3 times with exponential backoff

## Data Models

### User Model
```javascript
{
  _id: ObjectId,
  phoneNumber: String,           // Unique, indexed (e.g., "08012345678")
  name: String,                   // User's full name
  language: String,               // Enum: ['en', 'ha', 'yo', 'ig'] - Default: 'en'
  points: Number,                 // Current reward points balance - Default: 0
  totalRecycled: Number,          // Total kg recycled (lifetime) - Default: 0
  location: {
    state: String,                // Nigerian state
    lga: String                   // Local Government Area
  },
  joinedAt: Date,                 // Registration timestamp
  lastActive: Date,               // Last USSD session timestamp
  isActive: Boolean,              // Account status - Default: true
  createdAt: Date,
  updatedAt: Date
}
```

### Transaction Model
```javascript
{
  _id: ObjectId,
  userId: ObjectId,               // Reference to User model
  phoneNumber: String,            // Denormalized for quick lookups
  type: String,                   // Enum: ['recycle', 'redeem', 'referral']
  
  // For recycle transactions
  materialType: String,           // E.g., "plastic", "metal", "paper", "glass"
  quantity: Number,               // Weight in kg
  pointsEarned: Number,           // Points awarded
  
  // For redeem transactions
  redeemAction: String,           // E.g., "airtime", "cash_withdrawal"
  pointsSpent: Number,            // Points deducted
  redeemValue: Number,            // Cash equivalent (in Naira)
  
  pickupId: ObjectId,             // Reference to Pickup (if applicable)
  status: String,                 // Enum: ['pending', 'verified', 'completed', 'cancelled']
  verifiedBy: ObjectId,           // Admin/Agent who verified (if applicable)
  verifiedAt: Date,
  
  timestamp: Date,
  createdAt: Date,
  updatedAt: Date
}
```

### Pickup Model
```javascript
{
  _id: ObjectId,
  userId: ObjectId,               // Reference to User model
  phoneNumber: String,
  pickupAddress: String,          // User-provided address
  location: {
    state: String,
    lga: String
  },
  scheduledDate: Date,            // Requested pickup date
  scheduledTime: String,          // E.g., "Morning (9AM-12PM)", "Afternoon (12PM-4PM)"
  
  assignedAgent: {
    name: String,
    phone: String,
    agentId: ObjectId
  },
  
  materials: [{
    type: String,                 // Material type
    estimatedQuantity: Number     // User's estimate in kg
  }],
  
  status: String,                 // Enum: ['pending', 'assigned', 'in_progress', 'completed', 'cancelled']
  actualQuantity: Number,         // Verified weight after pickup
  pointsAwarded: Number,          // Points given to user
  
  notes: String,                  // Special instructions from user
  completedAt: Date,
  createdAt: Date,
  updatedAt: Date
}
```

### CollectionPoint Model
```javascript
{
  _id: ObjectId,
  name: String,                   // E.g., "Yaba Collection Center"
  address: String,                // Full address
  state: String,
  lga: String,
  phoneNumber: String,            // Contact number
  
  materialsAccepted: [String],    // E.g., ["plastic", "metal", "paper"]
  
  coordinates: {
    latitude: Number,
    longitude: Number
  },
  
  operatingHours: {
    weekdays: String,             // E.g., "8AM - 6PM"
    weekends: String              // E.g., "10AM - 4PM"
  },
  
  isActive: Boolean,              // Default: true
  createdAt: Date,
  updatedAt: Date
}
```

### Price Model
```javascript
{
  _id: ObjectId,
  materialType: String,           // E.g., "plastic_bottles", "aluminum_cans"
  pricePerKg: Number,             // Current rate in Naira per kg
  pointsPerKg: Number,            // Points awarded per kg
  
  effectiveFrom: Date,            // When this price takes effect
  isActive: Boolean,              // Current pricing - Default: true
  
  updatedBy: ObjectId,            // Admin who set price
  createdAt: Date,
  updatedAt: Date
}
```

## Security Considerations

### Authentication & Authorization
- **Phone Number Verification**: Leverages telco infrastructure (USSD requests come with verified phone numbers)
- **Session Validation**: Each USSD request validated against Redis session store
- **No Passwords**: Phone number acts as identifier (secured by SIM ownership)

### Input Validation
- All USSD inputs validated using Joi schemas before processing
- Sanitization prevents injection attacks (e.g., NoSQL injection in MongoDB queries)
- Quantity inputs limited to reasonable ranges (0.1kg - 1000kg)

### API Security
- **Webhook Authentication**: Africa's Talking requests verified using API key
- **Rate Limiting**: 100 requests/minute per phone number (prevents abuse)
- **CORS**: Configured to accept only Africa's Talking IP ranges
- **HTTPS Only**: All communications encrypted (TLS 1.3)

### Data Protection
- **Database**: MongoDB Atlas connection uses SSL/TLS encryption
- **Session Data**: Redis stores only temporary navigation state (no sensitive data)
- **PII Handling**: Phone numbers stored hashed for analytics, plain for operations
- **Compliance**: NDPR (Nigeria Data Protection Regulation) compliant

### Financial Security
- **Transaction Atomicity**: MongoDB transactions ensure points aren't double-spent
- **Redemption Limits**: Daily withdrawal limits prevent fraud (₦5,000/day for MVP)
- **Audit Trail**: All transactions logged with timestamps and verification status

## Scalability Approach

### Current MVP Scale (0-5,000 users)
- **Single Railway/Heroku instance** (512MB RAM, 1 CPU)
- **MongoDB Atlas Free Tier** (512MB storage, shared cluster)
- **Redis Cloud Free Tier** (30MB memory)
- **Africa's Talking Pay-as-you-go** pricing

### Growth Strategy (5,000-50,000 users)
1. **Horizontal Scaling**: Add more Railway/Heroku dynos with load balancer
2. **Database Optimization**: 
   - Upgrade to MongoDB Atlas dedicated cluster (M10)
   - Implement connection pooling (50 connections)
   - Add compound indexes for frequent queries
3. **Caching Strategy**:
   - Cache collection points and prices for 1 hour
   - Cache frequently accessed user profiles
   - Redis upgrade to 100MB memory

### Performance Optimizations
- **Database Indexes**: phoneNumber, userId, sessionId, status fields indexed
- **Query Optimization**: Use projection to fetch only required fields
- **Session TTL**: Auto-expire inactive sessions after 5 minutes
- **Async Operations**: SMS and non-critical logging processed asynchronously
- **CDN (Future)**: Serve static content (if web dashboard added) via CDN

## Technical Feasibility

### Why This Stack Works for Nigeria

**1. USSD Accessibility**
- Works on 100% of mobile phones (feature phones + smartphones)
- No internet/data costs for users
- 2G network compatible (important for rural areas)
- Proven technology used by banks, telcos, betting platforms

**2. Node.js Performance**
- Event-driven architecture perfect for I/O-heavy USSD operations
- Single-threaded but non-blocking (handles thousands of concurrent sessions)
- Fast startup times reduce serverless cold-start issues on Railway/Heroku

**3. MongoDB Flexibility**
- Schema evolution without downtime (add new material types, reward structures)
- Document model matches our nested data (users → transactions → pickups)
- Horizontal scaling path as data grows

**4. Africa's Talking Reliability**
- 99.9% uptime SLA
- Direct partnerships with Nigerian telcos
- Handles session timeouts and network disconnections gracefully
- Local support team in Lagos

### Potential Challenges & Mitigation

**Challenge 1: USSD Session Timeouts (180 seconds)**
- **Impact**: Users disconnected if navigation takes too long
- **Mitigation**: 
  - Shallow menu hierarchies (max 3 levels deep)
  - Quick database queries (<500ms)
  - Redis caching for frequently accessed data
  - Clear, concise menu text

**Challenge 2: Limited Screen Space (160 characters/screen)**
- **Impact**: Complex information hard to convey
- **Mitigation**:
  - Use abbreviations (e.g., "Pts" instead of "Points")
  - Pagination for long lists (collection points, transaction history)
  - SMS fallback for detailed information

**Challenge 3: Network Latency in Rural Areas**
- **Impact**: Slow responses may frustrate users
- **Mitigation**:
  - Optimize database queries (indexed fields, projections)
  - Cache static data (prices, collection points) in Redis
  - Async processing for non-critical operations (SMS, logging)
  - Graceful degradation (system works even if SMS fails)

**Challenge 4: Concurrent Request Handling**
- **Impact**: Multiple users hitting system simultaneously
- **Mitigation**:
  - Node.js non-blocking I/O handles concurrency naturally
  - MongoDB connection pooling prevents bottlenecks
  - Redis provides fast session lookups (sub-millisecond)
  - Horizontal scaling ready for growth

### MVP Scope & Development Timeline

**Phase 1: Core USSD Platform (Week 1-2)**
- User registration via USSD
- Check recyclable prices
- Request waste pickup
- View points balance

**Phase 2: Rewards & Transactions (Week 3-4)**
- Redeem points for airtime
- Cash withdrawal to mobile money
- Transaction history
- Multi-language support (English, Hausa, Yoruba, Igbo)

**Phase 3: Admin & Verification (Week 5-6)**
- Admin dashboard for pickup verification
- Agent assignment system
- Analytics and reporting
- SMS notification system

**Testing Strategy**
- Africa's Talking Sandbox environment for USSD testing
- ngrok for local webhook testing
- Unit tests with Jest (80% coverage target)
- Integration tests with Supertest
- Load testing with Artillery (simulate 100 concurrent users)

## API Endpoints

### USSD Webhook
```
POST /api/ussd
Headers: { Content-Type: "application/x-www-form-urlencoded" }
Body: { 
  sessionId: String,      // Unique session identifier
  serviceCode: String,    // USSD code dialed (e.g., "*347*123#")
  phoneNumber: String,    // User's phone number
  text: String            // User's input history (e.g., "1*2*500")
}
Response: { 
  text: String,           // Menu to display
  action: "CON" | "END"   // CON = continue, END = terminate session
}
```

### Admin API Endpoints (Future)
```
GET /api/admin/users              - List all users with filters
GET /api/admin/pickups/pending    - View pending pickup verifications
POST /api/admin/pickup/:id/verify - Verify and award points for pickup
GET /api/admin/stats              - Dashboard analytics
POST /api/admin/prices/update     - Update recyclable material prices
GET /api/admin/transactions       - View all transactions
```

### Internal Endpoints (Backend Only)
```
POST /api/internal/sms            - Queue SMS notification
GET /api/internal/health          - Health check endpoint
GET /api/internal/metrics         - Prometheus metrics (future monitoring)
```

## Development Workflow

### Local Development Setup
1. Clone repository from GitHub
2. Install dependencies: `npm install`
3. Configure `.env` file with:
   - Africa's Talking API credentials (Sandbox)
   - MongoDB connection string (local or Atlas)
   - Redis connection URL (local or Redis Cloud)
4. Start Redis: `redis-server`
5. Start app: `npm run dev` (uses nodemon for auto-restart)
6. Expose localhost with ngrok: `ngrok http 3000`
7. Configure ngrok URL in Africa's Talking dashboard

### Testing Approach
- **Unit Tests**: Jest for business logic, helpers, validators
- **Integration Tests**: Supertest for API endpoint testing
- **USSD Flow Tests**: Simulate complete user journeys
- **Load Tests**: Artillery to simulate concurrent users
- **Manual Testing**: Real USSD testing on sandbox before production

### CI/CD Pipeline (GitHub Actions)
```yaml
1. Push code to GitHub
2. GitHub Actions runs:
   - Lint code (ESLint)
   - Run unit tests (Jest)
   - Run integration tests
   - Build Docker image (optional)
3. If tests pass → Deploy to Railway staging
4. Manual approval → Deploy to Railway production
```

### Monitoring & Logging
- **Logging**: Winston writes structured logs to files + console
- **Error Tracking**: Sentry captures exceptions with context
- **Uptime Monitoring**: UptimeRobot pings health endpoint every 5 minutes
- **Metrics**: Track USSD response times, session completion rates, error rates

## Success Metrics

### Technical KPIs
- **Response Time**: 95% of USSD requests under 2 seconds
- **Uptime**: 99.5% availability (allowing for 3.6 hours downtime/month)
- **Session Completion**: 80% of USSD sessions reach intended action
- **Error Rate**: Less than 2% of requests result in errors

### Business KPIs
- **User Acquisition**: 5,000 active users within 3 months
- **Engagement**: 60% of users return within 7 days
- **Transaction Volume**: 500+ pickups completed per month
- **Reward Redemptions**: 70% of users redeem points within 30 days

---