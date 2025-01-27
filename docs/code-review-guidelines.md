# TWORG Code Review Guidelines

## Core Principles

1. **Be Constructive**: Focus on improving code quality, not criticizing the developer
2. **Be Specific**: Provide concrete examples and suggestions
3. **Be Timely**: Review code promptly to maintain development velocity
4. **Be Thorough**: Consider all aspects of the code change

## Code Review Checklist

### 1. Code Quality

#### Readability
```javascript
// ❌ DON'T: Unclear variable names and magic numbers
const x = arr.filter(i => i > 7);

// ✅ DO: Clear naming and documented constants
const MINIMUM_ORDER_QUANTITY = 7;
const validOrders = orders.filter(order => order.quantity > MINIMUM_ORDER_QUANTITY);
```

#### Error Handling
```javascript
// ❌ DON'T: Swallow errors silently
try {
  processData(data);
} catch (e) {}

// ✅ DO: Proper error handling and logging
try {
  await processData(data);
} catch (error) {
  logger.error('Data processing failed', {
    error: error.message,
    data: data.id,
    timestamp: new Date()
  });
  throw new ProcessingError('Failed to process data', { cause: error });
}
```

### 2. Security

#### Input Validation
```javascript
// ❌ DON'T: Trust user input
app.post('/api/users', (req, res) => {
  db.query(`SELECT * FROM users WHERE id = ${req.body.id}`);
});

// ✅ DO: Validate and sanitize input
app.post('/api/users', async (req, res) => {
  const schema = Joi.object({
    id: Joi.number().integer().required()
  });

  try {
    const { id } = await schema.validateAsync(req.body);
    const user = await db.query('SELECT * FROM users WHERE id = ?', [id]);
    res.json(user);
  } catch (error) {
    res.status(400).json({ error: 'Invalid input' });
  }
});
```

### 3. Performance

#### Database Queries
```javascript
// ❌ DON'T: N+1 queries
async function getUsersWithPosts() {
  const users = await User.findAll();
  for (const user of users) {
    user.posts = await Post.findAll({ where: { userId: user.id }});
  }
  return users;
}

// ✅ DO: Efficient querying
async function getUsersWithPosts() {
  return User.findAll({
    include: [{
      model: Post,
      attributes: ['id', 'title', 'content']
    }],
    where: { active: true }
  });
}
```

### 4. Testing

#### Test Coverage
```javascript
// ❌ DON'T: Incomplete test cases
describe('calculateTotal', () => {
  it('should calculate total correctly', () => {
    expect(calculateTotal(10, 2)).toBe(12);
  });
});

// ✅ DO: Comprehensive test cases
describe('calculateTotal', () => {
  it('should calculate total with tax correctly', () => {
    expect(calculateTotal(10, 0.2)).toBe(12);
  });

  it('should handle zero values', () => {
    expect(calculateTotal(0, 0.2)).toBe(0);
  });

  it('should throw error for negative values', () => {
    expect(() => calculateTotal(-10, 0.2)).toThrow('Amount cannot be negative');
  });

  it('should handle floating point precision', () => {
    expect(calculateTotal(10.99, 0.2)).toBeCloseTo(13.19);
  });
});
```

## Review Process

### 1. Before Review

- [ ] Run automated checks (linting, tests)
- [ ] Review the associated ticket/issue
- [ ] Check branch naming convention
- [ ] Verify commit message format

### 2. During Review

#### Code Structure
```javascript
// ❌ DON'T: Monolithic functions
function processOrder(order) {
  // 200 lines of mixed responsibilities
}

// ✅ DO: Single Responsibility Principle
class OrderProcessor {
  async validate(order) {
    // Validation logic
  }

  async calculateTotals(order) {
    // Price calculation
  }

  async processPayment(order) {
    // Payment processing
  }

  async sendConfirmation(order) {
    // Email notification
  }
}
```

#### Documentation
```javascript
// ❌ DON'T: Missing or unclear documentation
function process(d) {
  // ...
}

// ✅ DO: Clear documentation with examples
/**
 * Processes a customer order and returns the order confirmation
 * @param {Order} order - The order object to process
 * @returns {Promise<OrderConfirmation>} The processed order confirmation
 * @throws {ValidationError} If the order is invalid
 * @example
 * const order = new Order({ items: [...], customer: {...} });
 * const confirmation = await processOrder(order);
 */
async function processOrder(order) {
  // ...
}
```

### 3. After Review

#### Feedback Template
```markdown
### Overview
- [ ] Code follows TWORG standards
- [ ] Tests are comprehensive
- [ ] Documentation is complete

### Suggestions
1. Consider using the repository pattern for data access
2. Add error boundary for component
3. Include performance metrics logging

### Questions
1. How does this handle concurrent requests?
2. What's the fallback for service unavailability?

### Required Changes
1. Add input validation
2. Implement retry mechanism
3. Update API documentation
```

## Best Practices

### 1. Pull Request Size
- Aim for < 400 lines of code
- Break large changes into smaller PRs
- Each PR should represent a single logical change

### 2. Review Comments

#### Constructive Feedback
```javascript
// ❌ DON'T: Unhelpful feedback
// This is wrong

// ✅ DO: Constructive feedback with explanation
// Consider using a connection pool here to improve database performance
// Example:
const pool = new Pool({
  max: 20,
  idleTimeoutMillis: 30000
});
```

### 3. Security Checklist

- [ ] Authentication checks
- [ ] Authorization validation
- [ ] Input sanitization
- [ ] Secure communication
- [ ] Proper error handling
- [ ] Secrets management
- [ ] Audit logging

## Review Tools

### 1. Automated Checks
```javascript
// Example GitHub Action workflow
name: Code Review Checks

on: [pull_request]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Run linter
        run: npm run lint
      - name: Run tests
        run: npm run test
      - name: Check coverage
        run: npm run coverage
```

### 2. Code Quality Metrics
- Cyclomatic complexity
- Code duplication
- Test coverage
- Dependencies audit

## Communication Guidelines

### 1. Review Comments
```markdown
#### Good Comment
The current implementation might face race conditions under high load. Consider using a distributed lock:

```javascript
const lock = await redisLock.acquire('order-' + orderId);
try {
  await processOrder(orderId);
} finally {
  await lock.release();
}
```

#### Bad Comment
This code is bad and needs to be rewritten.
```

### 2. Response Template
```markdown
Thank you for the review! I've addressed the feedback:

1. ✅ Added input validation
2. ✅ Implemented connection pooling
3. 🤔 Need clarification on the suggested retry mechanism
   - What retry intervals would be appropriate?
   - Should we implement exponential backoff?
```

## Contact

### Code Review Team
- Lead Reviewer: reviews@tworg.com
- Architecture Team: architecture@tworg.com
- Security Team: security-review@tworg.com

