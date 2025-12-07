# Migration Guide: Replacing the Request Library

## Overview

This guide outlines the recommended approach to migrate jira-connector away from the deprecated `request` library to a modern, secure HTTP client.

## Why Migrate?

The `request` library is:
- **Deprecated** since 2020
- **Unmaintained** - no security patches
- **Vulnerable** - has known security issues (SSRF, etc.)
- **Outdated** - doesn't support modern JavaScript patterns (Promises, async/await)

## Recommended Approach: Axios

We recommend migrating to `axios` because it:
- ✅ Most popular HTTP client (50M+ weekly downloads)
- ✅ Promise-based (supports async/await)
- ✅ Actively maintained with regular updates
- ✅ Better error handling
- ✅ Built-in request/response interceptors
- ✅ Automatic JSON transformation
- ✅ Supports OAuth via interceptors
- ✅ Better TypeScript support

## Migration Steps

### Phase 1: Preparation

1. **Add axios as a dependency**
   ```bash
   npm install axios
   ```

2. **Create adapter layer** (for gradual migration)
   ```javascript
   // lib/http-adapter.js
   const axios = require('axios');
   
   class HttpAdapter {
       constructor(config) {
           this.client = axios.create({
               baseURL: config.baseURL,
               timeout: config.timeout || 30000,
               headers: config.headers || {}
           });
       }
       
       async request(options) {
           try {
               const response = await this.client.request(options);
               return {
                   statusCode: response.status,
                   body: response.data,
                   headers: response.headers
               };
           } catch (error) {
               throw this.transformError(error);
           }
       }
       
       transformError(error) {
           if (error.response) {
               return {
                   statusCode: error.response.status,
                   body: error.response.data,
                   message: error.message
               };
           }
           return error;
       }
   }
   
   module.exports = HttpAdapter;
   ```

### Phase 2: Update Core Client

**Current implementation** (index.js, lines 228-243):
```javascript
this.makeRequest = function (options, callback, successString) {
    if (this.oauthConfig) {
        options.oauth = this.oauthConfig;
    } else if (this.basic_auth) {
        options.auth = this.basic_auth;
    }
    request(options, function (err, response, body) {
        if (err || response.statusCode.toString()[0] != 2) {
            return callback(err ? err : body);
        }

        if (typeof body == 'string') body = JSON.parse(body);

        return callback(null, successString ? successString : body);
    });
};
```

**New implementation with axios**:
```javascript
const axios = require('axios');
const crypto = require('crypto');

this.makeRequest = function (options, callback, successString) {
    const axiosConfig = {
        method: options.method,
        url: options.uri,
        data: options.body,
        params: options.qs,
        headers: options.headers || {},
        maxRedirects: options.followAllRedirects ? 5 : 0
    };

    // Handle authentication
    if (this.oauthConfig) {
        axiosConfig.headers['Authorization'] = this.generateOAuthHeader(
            options.method,
            options.uri,
            this.oauthConfig
        );
    } else if (this.basic_auth) {
        axiosConfig.auth = {
            username: this.basic_auth.user,
            password: this.basic_auth.pass
        };
    }

    axios(axiosConfig)
        .then(response => {
            callback(null, successString || response.data);
        })
        .catch(error => {
            if (error.response) {
                // Server responded with error status
                callback(error.response.data || error.message);
            } else if (error.request) {
                // Request made but no response
                callback(new Error('No response from server'));
            } else {
                // Error setting up request
                callback(error);
            }
        });
};

// Helper method for OAuth header generation
this.generateOAuthHeader = function(method, url, oauthConfig) {
    // Use oauth-1.0a library for proper OAuth signing
    const OAuth = require('oauth-1.0a');
    const oauth = OAuth({
        consumer: {
            key: oauthConfig.consumer_key,
            secret: oauthConfig.private_key  // Jira uses RSA private key here
        },
        signature_method: 'RSA-SHA1',
        hash_function(base_string, key) {
            // RSA-SHA1 signing for Jira OAuth
            return crypto
                .createSign('RSA-SHA1')
                .update(base_string)
                .sign(key, 'base64');
        }
    });

    const token = {
        key: oauthConfig.token,
        secret: oauthConfig.token_secret
    };

    const authHeader = oauth.toHeader(oauth.authorize({url, method}, token));
    return authHeader.Authorization;  // Extract Authorization header value
};
```

### Phase 3: Add Promise Support (Backward Compatible)

```javascript
// Add promisified version alongside callback version
this.makeRequestAsync = function (options, successString) {
    return new Promise((resolve, reject) => {
        this.makeRequest(options, (err, result) => {
            if (err) reject(err);
            else resolve(result);
        }, successString);
    });
};
```

### Phase 4: Update OAuth Utilities

Replace `oauth` package with `oauth-1.0a`:

```javascript
// lib/oauth_util.js
const OAuth = require('oauth-1.0a');
const crypto = require('crypto');
const axios = require('axios');

exports.getAuthorizeURL = async function (config) {
    const oauth = createOAuthInstance(config);
    
    const requestData = {
        url: buildUrl(config, '/plugins/servlet/oauth/request-token'),
        method: 'POST'
    };

    const authHeader = oauth.toHeader(oauth.authorize(requestData));

    try {
        const response = await axios.post(requestData.url, null, { 
            headers: authHeader,
            responseType: 'text'  // OAuth responses are URL-encoded text
        });
        // Parse URL-encoded OAuth response
        const params = new URLSearchParams(response.data);
        const token = params.get('oauth_token');
        const token_secret = params.get('oauth_token_secret');
        
        return {
            url: buildUrl(config, '/plugins/servlet/oauth/authorize') + '?oauth_token=' + token,
            token,
            token_secret
        };
    } catch (error) {
        throw new Error('Failed to get OAuth token: ' + error.message);
    }
};

function createOAuthInstance(config) {
    return OAuth({
        consumer: {
            key: config.oauth.consumer_key,
            secret: config.oauth.private_key
        },
        signature_method: 'RSA-SHA1',
        hash_function(base_string, key) {
            return crypto
                .createSign('RSA-SHA1')
                .update(base_string)
                .sign(key, 'base64');
        }
    });
}
```

### Phase 5: Update Package Dependencies

**package.json changes**:
```json
{
  "dependencies": {
    "axios": "^1.6.0",
    "oauth-1.0a": "^2.2.6"
  }
}
```

Remove:
```json
{
  "dependencies": {
    "oauth": "^0.9.12",
    "request": "^2.51.0"
  }
}
```

### Phase 6: Add Modern Features

#### 6.1 Add Timeout Configuration
```javascript
var JiraClient = module.exports = function (config) {
    // ... existing code ...
    this.timeout = config.timeout || 30000; // 30 second default
    
    this.axiosInstance = axios.create({
        timeout: this.timeout,
        maxRedirects: 5
    });
};
```

#### 6.2 Add Request Interceptors
```javascript
this.axiosInstance.interceptors.request.use(
    config => {
        // Add custom headers, logging, etc.
        return config;
    },
    error => Promise.reject(error)
);
```

#### 6.3 Add Response Interceptors
```javascript
this.axiosInstance.interceptors.response.use(
    response => response,
    error => {
        // Add custom error handling, retry logic, etc.
        if (error.response?.status === 401) {
            // Handle authentication errors
        }
        return Promise.reject(error);
    }
);
```

### Phase 7: Testing Strategy

1. **Unit Tests**
   ```javascript
   // tests/client.test.js
   const JiraClient = require('../index');
   const nock = require('nock');
   
   describe('JiraClient', () => {
       it('should make basic auth request', async () => {
           nock('https://test.atlassian.net')
               .get('/rest/api/2/issue/TEST-1')
               .reply(200, { key: 'TEST-1' });
           
           const client = new JiraClient({
               host: 'test.atlassian.net',
               basic_auth: {
                   username: 'user',
                   password: 'pass'
               }
           });
           
           // Test with new promise API
           const issue = await client.issue.getIssueAsync({
               issueKey: 'TEST-1'
           });
           
           expect(issue.key).toBe('TEST-1');
       });
   });
   ```

2. **Integration Tests**
   - Test against real Jira instance (or mock server)
   - Verify OAuth flow
   - Test all API endpoints

3. **Backward Compatibility Tests**
   - Ensure callback API still works
   - Test with existing user code

## Breaking Changes

This migration introduces some breaking changes:

1. **Minimum Node.js Version**: May require Node.js 12+ for axios
2. **Error Format**: Error objects structure may differ slightly
3. **Dependencies**: Removes `request` and `oauth` packages
4. **OAuth Library**: Changes from `oauth` to `oauth-1.0a`

## Migration Timeline (Suggested)

| Phase | Duration | Tasks |
|-------|----------|-------|
| 1 | Week 1 | Add axios, create adapter layer |
| 2 | Week 2 | Update core client, maintain backward compatibility |
| 3 | Week 3 | Add promise support to all API methods |
| 4 | Week 3-4 | Update OAuth utilities |
| 5 | Week 4 | Update dependencies, remove request |
| 6 | Week 5 | Add modern features (timeouts, interceptors) |
| 7 | Week 6-8 | Comprehensive testing |

## Testing Checklist

- [ ] All existing API methods work with callbacks
- [ ] All API methods have promise versions
- [ ] Basic authentication works
- [ ] OAuth authentication works
- [ ] Error handling works correctly
- [ ] All 42 API modules tested
- [ ] Integration tests pass
- [ ] Performance benchmarks acceptable
- [ ] Memory usage acceptable
- [ ] No dependency vulnerabilities

## Rollout Strategy

1. **Alpha Release** (v0.2.0-alpha)
   - Internal testing only
   - Feedback from maintainers

2. **Beta Release** (v0.2.0-beta)
   - Public beta testing
   - Gather user feedback
   - Fix issues

3. **Release Candidate** (v0.2.0-rc)
   - Final testing
   - Documentation updates
   - Migration guide for users

4. **Stable Release** (v0.2.0)
   - Full release
   - Deprecation notice for v0.1.x
   - Support both versions for 6 months

## Example User Migration

**Before (v0.1.x with callbacks)**:
```javascript
const JiraClient = require('jira-connector');

const jira = new JiraClient({
    host: 'mycompany.atlassian.net',
    basic_auth: {
        username: 'user',
        password: 'pass'
    }
});

jira.issue.getIssue({
    issueKey: 'TEST-1'
}, function(error, issue) {
    if (error) {
        console.error(error);
    } else {
        console.log(issue.fields.summary);
    }
});
```

**After (v0.2.0 with async/await)**:
```javascript
const JiraClient = require('jira-connector');

const jira = new JiraClient({
    host: 'mycompany.atlassian.net',
    basic_auth: {
        username: 'user',
        password: 'pass'
    }
});

// Option 1: Still use callbacks (backward compatible)
jira.issue.getIssue({
    issueKey: 'TEST-1'
}, function(error, issue) {
    if (error) {
        console.error(error);
    } else {
        console.log(issue.fields.summary);
    }
});

// Option 2: Use promises with async/await (new)
try {
    const issue = await jira.issue.getIssue({
        issueKey: 'TEST-1'
    });
    console.log(issue.fields.summary);
} catch (error) {
    console.error(error);
}
```

## Resources

- [Axios Documentation](https://axios-http.com/)
- [oauth-1.0a Documentation](https://github.com/ddo/oauth-1.0a)
- [Atlassian OAuth Documentation](https://developer.atlassian.com/cloud/jira/platform/oauth/)
- [Node.js Promises Guide](https://nodejs.org/en/docs/guides/promises/)

## Support

For questions or issues during migration:
1. Open an issue on GitHub
2. Check existing migration issues
3. Join the discussion in the migration planning issue

---

**Document Version**: 1.0  
**Last Updated**: December 7, 2025
