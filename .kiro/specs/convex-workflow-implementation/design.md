# Design Document

## Overview

The Convex workflow implementation provides a robust, type-safe database layer for the HireWise AI interview preparation platform. The design emphasizes simplicity, performance, and maintainability while supporting real-time updates and scalable data operations. The system manages user accounts, interview sessions, questions/answers, and feedback through a well-structured schema and organized function modules.

## Architecture

### Database Schema Design

The system uses a normalized schema with two primary tables:

1. **users** - Stores user account information with Clerk integration
2. **interviews** - Stores interview sessions with embedded questions, answers, and feedback

### Key Design Decisions

- **Embedded vs. Normalized Data**: Questions and answers are embedded within interview documents to reduce query complexity and improve performance for the primary use case
- **Status Management**: Interview status uses union types for type safety and controlled state transitions
- **Indexing Strategy**: Strategic indexes on common query patterns (user lookups, interview listings)
- **Optional Fields**: Extensive use of optional fields to support flexible data entry workflows

## Components and Interfaces

### Schema Definition (`convex/schema.ts`)

```typescript
export default defineSchema({
  users: defineTable({
    clerkId: v.string(),
    email: v.string(),
    name: v.string(),
    imageUrl: v.optional(v.string()),
    createdAt: v.number(),
  }).index("by_clerk_id", ["clerkId"]),

  interviews: defineTable({
    userId: v.id("users"),
    jobTitle: v.string(),
    jobDescription: v.optional(v.string()),
    resumeUrl: v.optional(v.string()),
    status: v.union(v.literal("draft"), v.literal("in_progress"), v.literal("completed")),
    questions: v.array(v.object({
      question: v.string(),
      answer: v.optional(v.string()),
      order: v.number(),
    })),
    feedback: v.optional(v.object({
      overallRating: v.number(),
      strengths: v.array(v.string()),
      improvements: v.array(v.string()),
      detailedFeedback: v.string(),
    })),
    createdAt: v.number(),
    completedAt: v.optional(v.number()),
  }).index("by_user", ["userId"]).index("by_user_status", ["userId", "status"]),
});
```

### User Management Module (`convex/users.ts`)

**Functions:**
- `createOrGetUser` - Mutation for user creation/retrieval with Clerk integration
- `getUserById` - Query for retrieving user by Convex ID
- `getUserByClerkId` - Query for retrieving user by Clerk ID

**Key Features:**
- Automatic user creation on first login
- Duplicate prevention through Clerk ID indexing
- Timestamp tracking for user creation

### Interview Management Module (`convex/interviews.ts`)

**Core Functions:**
- `createInterview` - Creates new interview session with questions
- `updateInterviewAnswer` - Updates individual question answers
- `bulkUpdateAnswers` - Batch updates multiple answers for performance
- `updateInterviewStatus` - Manages status transitions
- `addFeedback` - Stores comprehensive feedback data

**Query Functions:**
- `getInterview` - Retrieves complete interview data
- `getUserInterviews` - Lists user's interviews with pagination support
- `getInterviewsByStatus` - Filters interviews by status

### Validation and Type Safety

**Reusable Validators:**
```typescript
const questionValidator = v.object({
  question: v.string(),
  answer: v.optional(v.string()),
  order: v.number(),
});

const feedbackValidator = v.object({
  overallRating: v.number(),
  strengths: v.array(v.string()),
  improvements: v.array(v.string()),
  detailedFeedback: v.string(),
});
```

## Data Models

### User Model
- **clerkId**: Unique identifier from Clerk authentication
- **email**: User's email address
- **name**: Display name
- **imageUrl**: Optional profile image URL
- **createdAt**: Timestamp of account creation

### Interview Model
- **userId**: Reference to user who owns the interview
- **jobTitle**: Target job position
- **jobDescription**: Optional detailed job requirements
- **resumeUrl**: Optional uploaded resume file URL
- **status**: Current interview state (draft/in_progress/completed)
- **questions**: Array of question objects with answers
- **feedback**: Optional comprehensive feedback object
- **createdAt**: Interview creation timestamp
- **completedAt**: Optional completion timestamp

### Question Object Structure
- **question**: The interview question text
- **answer**: Optional user's answer
- **order**: Question sequence number

### Feedback Object Structure
- **overallRating**: Numeric rating (1-10 scale)
- **strengths**: Array of identified strengths
- **improvements**: Array of improvement areas
- **detailedFeedback**: Comprehensive feedback text

## Error Handling

### Validation Strategy
- Input validation using Convex validators
- Type safety through TypeScript and Convex schema
- Business logic validation in function handlers

### Authorization Checks
```typescript
async function validateUserOwnership(ctx, userId, resourceUserId) {
  if (userId !== resourceUserId) {
    throw new Error("Unauthorized access");
  }
}
```

### Common Error Scenarios
- User not found during operations
- Interview not found for updates
- Unauthorized access to other users' data
- Invalid status transitions
- Malformed input data

## Testing Strategy

### Unit Testing Approach
- Test individual functions with mock Convex context
- Validate input/output contracts
- Test error conditions and edge cases

### Integration Testing
- Test complete workflows (create user → create interview → add answers → add feedback)
- Validate database constraints and relationships
- Test concurrent operations and data consistency

### Performance Testing
- Query performance with large datasets
- Batch operation efficiency
- Index utilization verification

### Test Data Management
- Automated test data generation
- Cleanup procedures for test isolation
- Mock external dependencies (Clerk, file uploads)

## Performance Considerations

### Indexing Strategy
- Primary indexes on frequently queried fields
- Compound indexes for complex query patterns
- Regular index performance monitoring

### Query Optimization
- Use of pagination for large result sets
- Batch operations for multiple updates
- Efficient filtering and sorting

### Caching Strategy
- Leverage Convex's built-in reactivity for real-time updates
- Client-side caching for frequently accessed data
- Minimize redundant database queries

## Security Considerations

### Authentication Integration
- Seamless Clerk authentication integration
- User session validation
- Secure user identification through Clerk IDs

### Data Access Control
- User-scoped data access
- Ownership validation for all operations
- Prevention of cross-user data access

### Input Sanitization
- Comprehensive input validation
- Protection against injection attacks
- Safe handling of user-generated content

## Migration and Deployment

### Schema Evolution
- Backward-compatible schema changes
- Data migration procedures for breaking changes
- Version management for schema updates

### Deployment Strategy
- Staged rollout of database changes
- Monitoring and rollback procedures
- Performance impact assessment