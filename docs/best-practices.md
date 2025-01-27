# TWORG Best Practices

## Security Best Practices

### Authentication & Authorization

```javascript
// ✅ DO: Implement proper token validation
const validateJWT = async (token) => {
  try {
    const decoded = await jwt.verify(token, process.env.JWT_SECRET);
    return {
      valid: true,
      payload: decoded
    };
  } catch (error) {
    console.error('Token validation failed:', error.message);
    return {
      valid: false,
      error: error.message
    };
  }
};

// ✅ DO: Use role-based access control
const checkPermission = (user, resource, action) => {
  const userRole = user.role;
  const permissionMatrix = {
    admin: ['read', 'write', 'delete'],
    editor: ['read', 'write'],
    viewer: ['read']
  };

  return permissionMatrix[userRole]?.includes(action) || false;
};
```

## Performance Optimization

### Caching Strategy

```javascript
// ✅ DO: Implement efficient caching
class Cache {
  constructor(ttlSeconds = 3600) {
    this.cache = new Map();
    this.ttl = ttlSeconds * 1000;
  }

  set(key, value) {
    const expires = Date.now() + this.ttl;
    this.cache.set(key, { value, expires });
  }

  get(key) {
    const item = this.cache.get(key);
    if (!item) return null;
    if (Date.now() > item.expires) {
      this.cache.delete(key);
      return null;
    }
    return item.value;
  }
}

// Example usage
const dataCache = new Cache(1800); // 30 minutes TTL
```

## API Design

### RESTful Endpoints

```javascript
// ✅ DO: Use consistent API response format
const apiResponse = (data, status = 200, message = 'Success') => {
  return {
    status,
    message,
    data,
    timestamp: new Date().toISOString()
  };
};

// Example API endpoint
app.get('/api/users/:id', async (req, res) => {
  try {
    const user = await UserService.findById(req.params.id);
    if (!user) {
      return res.status(404).json(
        apiResponse(null, 404, 'User not found')
      );
    }
    return res.json(apiResponse(user));
  } catch (error) {
    return res.status(500).json(
      apiResponse(null, 500, 'Internal server error')
    );
  }
});
```

## Database Practices

### Query Optimization

```javascript
// ✅ DO: Use efficient database queries
const getUserWithPosts = async (userId) => {
  // Better: Single query with join
  const result = await db.query(`
    SELECT u.*, p.id as post_id, p.title
    FROM users u
    LEFT JOIN posts p ON u.id = p.user_id
    WHERE u.id = ?
  `, [userId]);

  // ❌ DON'T: N+1 query problem
  // const user = await db.query('SELECT * FROM users WHERE id = ?', [userId]);
  // const posts = await db.query('SELECT * FROM posts WHERE user_id = ?', [userId]);
};
```

## Error Handling

### Centralized Error Management

```javascript
// ✅ DO: Create custom error types
class ApplicationError extends Error {
  constructor(message, status = 500, code = 'INTERNAL_ERROR') {
    super(message);
    this.status = status;
    this.code = code;
  }
}

class ValidationError extends ApplicationError {
  constructor(message) {
    super(message, 400, 'VALIDATION_ERROR');
  }
}

// Example middleware
const errorHandler = (err, req, res, next) => {
  console.error('Error:', {
    message: err.message,
    stack: err.stack,
    code: err.code
  });

  if (err instanceof ApplicationError) {
    return res.status(err.status).json({
      error: {
        message: err.message,
        code: err.code
      }
    });
  }

  return res.status(500).json({
    error: {
      message: 'Internal Server Error',
      code: 'INTERNAL_ERROR'
    }
  });
};
```

## Testing Practices

### Integration Testing

```javascript
// ✅ DO: Write comprehensive integration tests
describe('User Service Integration', () => {
  let testUser;

  beforeAll(async () => {
    await setupTestDatabase();
  });

  beforeEach(async () => {
    testUser = await createTestUser();
  });

  afterEach(async () => {
    await cleanupTestData();
  });

  it('should create user and associated profile', async () => {
    const userData = {
      email: 'test@tworg.com',
      name: 'Test User',
      profile: {
        bio: 'Test bio',
        location: 'Test City'
      }
    };

    const result = await UserService.createWithProfile(userData);
    expect(result.email).toBe(userData.email);
    expect(result.profile.bio).toBe(userData.profile.bio);
  });
});
```

## Logging Best Practices

```javascript
// ✅ DO: Implement structured logging
const logger = {
  info: (message, meta = {}) => {
    console.log(JSON.stringify({
      level: 'info',
      message,
      timestamp: new Date().toISOString(),
      ...meta
    }));
  },
  error: (message, error, meta = {}) => {
    console.error(JSON.stringify({
      level: 'error',
      message,
      error: {
        message: error.message,
        stack: error.stack
      },
      timestamp: new Date().toISOString(),
      ...meta
    }));
  }
};

// Example usage
logger.info('User login successful', {
  userId: 'user123',
  loginMethod: 'oauth'
});
```

