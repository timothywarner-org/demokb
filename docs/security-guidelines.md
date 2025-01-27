# TWORG Security Guidelines

## Authentication & Authorization

### 1. JWT Implementation

```typescript
// ✅ DO: Implement secure JWT handling
class JWTManager {
  private readonly secret: string;
  private readonly algorithm = 'HS256';

  constructor() {
    this.secret = process.env.JWT_SECRET;
    if (!this.secret) {
      throw new Error('JWT_SECRET environment variable is required');
    }
  }

  async generateToken(user: User): Promise<string> {
    const payload = {
      sub: user.id,
      email: user.email,
      roles: user.roles,
      iat: Math.floor(Date.now() / 1000),
      exp: Math.floor(Date.now() / 1000) + (60 * 60) // 1 hour
    };

    return jwt.sign(payload, this.secret, { algorithm: this.algorithm });
  }

  async verifyToken(token: string): Promise<JWTPayload> {
    try {
      return jwt.verify(token, this.secret, { algorithms: [this.algorithm] });
    } catch (error) {
      if (error instanceof jwt.TokenExpiredError) {
        throw new AuthError('Token has expired');
      }
      throw new AuthError('Invalid token');
    }
  }
}
```

### 2. OAuth2 Integration

```typescript
// ✅ DO: Implement OAuth2 with PKCE
class OAuth2Client {
  private readonly clientId: string;
  private readonly redirectUri: string;

  async generateAuthUrl(): Promise<string> {
    const codeVerifier = this.generateCodeVerifier();
    const codeChallenge = await this.generateCodeChallenge(codeVerifier);

    // Store code_verifier in session
    session.set('code_verifier', codeVerifier);

    const params = new URLSearchParams({
      client_id: this.clientId,
      redirect_uri: this.redirectUri,
      response_type: 'code',
      code_challenge: codeChallenge,
      code_challenge_method: 'S256',
      scope: 'openid profile email'
    });

    return `${this.authorizationEndpoint}?${params.toString()}`;
  }

  private generateCodeVerifier(): string {
    return crypto.randomBytes(32)
      .toString('base64')
      .replace(/[^a-zA-Z0-9]/g, '')
      .substring(0, 128);
  }

  private async generateCodeChallenge(verifier: string): Promise<string> {
    const hash = await crypto.subtle.digest('SHA-256',
      new TextEncoder().encode(verifier));
    return btoa(String.fromCharCode(...new Uint8Array(hash)))
      .replace(/[^a-zA-Z0-9]/g, '')
      .replace(/=/g, '');
  }
}
```

## Secure Communication

### 1. API Security

```typescript
// ✅ DO: Implement security headers middleware
const securityHeaders = (req: Request, res: Response, next: NextFunction) => {
  // Prevent clickjacking
  res.setHeader('X-Frame-Options', 'DENY');

  // Enable XSS protection
  res.setHeader('X-XSS-Protection', '1; mode=block');

  // Prevent MIME type sniffing
  res.setHeader('X-Content-Type-Options', 'nosniff');

  // Set strict transport security
  res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');

  // Set content security policy
  res.setHeader('Content-Security-Policy', `
    default-src 'self';
    script-src 'self' 'unsafe-inline' 'unsafe-eval';
    style-src 'self' 'unsafe-inline';
    img-src 'self' data: https:;
    font-src 'self';
    object-src 'none';
    base-uri 'self';
    form-action 'self';
    frame-ancestors 'none';
  `.replace(/\s+/g, ' ').trim());

  next();
};
```

### 2. Data Encryption

```typescript
// ✅ DO: Implement field-level encryption
class FieldEncryption {
  private readonly algorithm = 'aes-256-gcm';
  private readonly keyLength = 32;
  private readonly ivLength = 12;
  private readonly tagLength = 16;

  constructor(private readonly encryptionKey: Buffer) {
    if (encryptionKey.length !== this.keyLength) {
      throw new Error(`Encryption key must be ${this.keyLength} bytes`);
    }
  }

  async encrypt(data: string): Promise<string> {
    const iv = crypto.randomBytes(this.ivLength);
    const cipher = crypto.createCipheriv(this.algorithm, this.encryptionKey, iv);

    const encrypted = Buffer.concat([
      cipher.update(data, 'utf8'),
      cipher.final()
    ]);

    const tag = cipher.getAuthTag();

    // Format: iv:encrypted:tag
    return Buffer.concat([iv, encrypted, tag]).toString('base64');
  }

  async decrypt(encryptedData: string): Promise<string> {
    const buffer = Buffer.from(encryptedData, 'base64');

    const iv = buffer.slice(0, this.ivLength);
    const tag = buffer.slice(-this.tagLength);
    const encrypted = buffer.slice(this.ivLength, -this.tagLength);

    const decipher = crypto.createDecipheriv(this.algorithm, this.encryptionKey, iv);
    decipher.setAuthTag(tag);

    return decipher.update(encrypted) + decipher.final('utf8');
  }
}
```

## Secure Coding Practices

### 1. Input Validation

```typescript
// ✅ DO: Implement comprehensive input validation
class InputValidator {
  static validateEmail(email: string): boolean {
    const emailRegex = /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/;
    return emailRegex.test(email) && email.length <= 254;
  }

  static sanitizeHtml(input: string): string {
    return DOMPurify.sanitize(input, {
      ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a'],
      ALLOWED_ATTR: ['href']
    });
  }

  static validatePassword(password: string): ValidationResult {
    const minLength = 12;
    const hasUpperCase = /[A-Z]/.test(password);
    const hasLowerCase = /[a-z]/.test(password);
    const hasNumbers = /\d/.test(password);
    const hasSpecialChar = /[!@#$%^&*(),.?":{}|<>]/.test(password);

    const errors: string[] = [];

    if (password.length < minLength) {
      errors.push(`Password must be at least ${minLength} characters long`);
    }
    if (!hasUpperCase) errors.push('Password must contain uppercase letters');
    if (!hasLowerCase) errors.push('Password must contain lowercase letters');
    if (!hasNumbers) errors.push('Password must contain numbers');
    if (!hasSpecialChar) errors.push('Password must contain special characters');

    return {
      isValid: errors.length === 0,
      errors
    };
  }
}
```

### 2. SQL Injection Prevention

```typescript
// ✅ DO: Use parameterized queries
class DatabaseService {
  async getUserById(id: string): Promise<User> {
    // ✅ DO: Use parameterized queries
    const query = 'SELECT * FROM users WHERE id = ?';
    const [user] = await this.db.execute(query, [id]);
    return user;
  }

  // ❌ DON'T: Use string concatenation
  async unsafeGetUser(id: string): Promise<User> {
    const query = `SELECT * FROM users WHERE id = '${id}'`; // Vulnerable!
    const [user] = await this.db.query(query);
    return user;
  }
}
```

## Security Monitoring

### 1. Audit Logging

```typescript
// ✅ DO: Implement comprehensive audit logging
class AuditLogger {
  async logSecurityEvent(event: SecurityEvent): Promise<void> {
    const entry = {
      timestamp: new Date().toISOString(),
      eventType: event.type,
      userId: event.userId,
      action: event.action,
      resource: event.resource,
      status: event.status,
      ipAddress: event.ipAddress,
      userAgent: event.userAgent,
      metadata: event.metadata
    };

    // Log to secure audit log storage
    await this.secureStorage.store('security_audit', entry);

    // Alert on high-severity events
    if (this.isHighSeverity(event)) {
      await this.alertSecurityTeam(entry);
    }
  }

  private isHighSeverity(event: SecurityEvent): boolean {
    return [
      'UNAUTHORIZED_ACCESS',
      'SUSPICIOUS_ACTIVITY',
      'MULTIPLE_LOGIN_FAILURES',
      'PRIVILEGE_ESCALATION'
    ].includes(event.type);
  }
}
```

## Security Checklist

### Pre-deployment Security Checks
- [ ] Dependencies are up to date and scanned for vulnerabilities
- [ ] Security headers are properly configured
- [ ] Authentication mechanisms are properly implemented
- [ ] Input validation is in place
- [ ] Sensitive data is encrypted
- [ ] Audit logging is enabled
- [ ] Error handling doesn't expose sensitive information
- [ ] CORS is properly configured
- [ ] Rate limiting is implemented
- [ ] File upload validation is in place

### Regular Security Tasks
- Daily:
  - [ ] Review security logs
  - [ ] Monitor for suspicious activities
  - [ ] Check system health metrics

- Weekly:
  - [ ] Review user access patterns
  - [ ] Scan for vulnerabilities
  - [ ] Update security patches

- Monthly:
  - [ ] Conduct security training
  - [ ] Review security policies
  - [ ] Perform penetration testing

## Contact

### Security Team
- Security Officer: security-officer@tworg.com
- Incident Response: security-911@tworg.com
- Compliance: compliance@tworg.com