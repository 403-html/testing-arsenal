# testing-arsenal

This repository contains technical guides and patterns for implementing effective automated testing practices, focusing on language/framework-agnostic (_examples will be in TS and Playwright tho_) concepts applicable across various testing frameworks.

## Table of Contents

- [testing-arsenal](#testing-arsenal)
  - [Table of Contents](#table-of-contents)
  - [Authentication strategies](#authentication-strategies)
    - [Google IAP (Identity-Aware Proxy)](#google-iap-identity-aware-proxy)
  - [Test Data Management](#test-data-management)
    - [External Services (Email, SMS, OTP)](#external-services-email-sms-otp)
    - [Centralized Fixtures](#centralized-fixtures)
  - [API Interaction](#api-interaction)
    - [Encapsulate API calls](#encapsulate-api-calls)
    - [Native Fetch and Error Handling](#native-fetch-and-error-handling)
  - [Code Reusability](#code-reusability)
    - [Centralized Selectors](#centralized-selectors)
    - [Centralized Utilities](#centralized-utilities)
  - [Test Data Generation](#test-data-generation)
    - [Builder Pattern](#builder-pattern)
  - [Contributing](#contributing)
  - [License](#license)

## Authentication strategies

### Google IAP (Identity-Aware Proxy)

Here's high-level overview of authenticating to applications behind Identity-Aware Proxy (IAP). While IAP effectively secures Google Cloud-hosted applications, testing these protected endpoints requires specific steps. Below is a streamlined authentication process with clear designation of responsibilities (workflow/SRE/DevOps or test code/developers/QA).

1. **Service Account:**  Configure a Service Account in Google Cloud and link it with a Workload Identity Provider. (SRE/DevOps)
2. **OAuth Client ID:** Obtain the OAuth Client ID (ID Token Audience) for your specific environment. (Workflow)
3. **Token Retrieval:** Authenticate to Google Cloud using the Client ID and retrieve an `id_token` (JWT). (Workflow)
4. **IAP Token Usage:** (Test Code)
    - **API Calls:** Include the `Proxy-Authorization` header with `Bearer <IAP token>`.
    - **Web Pages:** Route requests through a proxy that adds the `Proxy-Authorization` header.  The proxy should pass responses back to the client without modification.

Example code for retrieving an IAP token in test:

```typescript
// assuming you have IAP token from previous steps

(async () => {
  const iapToken = process.env.IAP_TOKEN;
  const response = await fetch('https://my-iap-protected-app.com/api/v1/resource', {
    method: 'GET',
    headers: {
      'Proxy-Authorization': `Bearer ${iapToken}`,
    },
  });
})();
```

## Test Data Management

### External Services (Email, SMS, OTP)

When testing applications that rely on external services (email message, SMS, OTP code), it's crucial to manage test data effectively. Here are some strategies to consider:

- **Prioritize API requests:** Use APIs to retrieve email messages (like Gmail API) or SMS messages (Twilio API) to ensure test data is available when needed.
- **Use dedicated test accounts:** Create dedicated email addresses or phone numbers for testing purposes. This ensures that test data is isolated from production data.
- **Automate data cleanup:** Implement cleanup scripts to remove test data after tests are complete. This ensures that test data does not accumulate over time.
- **Use mock services:** Consider using mock services for email (like InBucket) or SMS (like MockSMS) to simulate external services without relying on real data.

### Centralized Fixtures

Store test data in a centralized location (like a database or JSON file) and load it into tests as needed. This approach ensures that test data is consistent across tests and can be easily updated when needed. Avoid hardcoding values within tests.

## API Interaction

### Encapsulate API calls

Encapsulate API calls in a separate module or class to promote reusability and maintainability and simplifies adding common functionalities like authentication headers or retry logic.

Example:

```typescript
// api-client.ts
export class ApiClient {
  constructor(private baseUrl: string) {}

  async get(path: string): Promise<Response> {
    return this.makeRequest('GET', path);
  }

  // ...other methods (POST, PUT, DELETE, etc.)

  private async makeRequest(method: string, path: string, body?: any): Promise<Response> {
    const url = `${this.baseUrl}${path}`;
    const headers = {
      // add common headers here (like Authorization, Content-Type or Proxy-Authorization)
    };

    try {
      const response = await fetch(url, { method, headers, body: JSON.stringify(body) });
      if (!response.ok) {
        throw new Error(`API request failed: ${response.status} ${response.statusText}`);
      }
      return response;
    } catch (error) {
      // handle errors (logging, retry logic, etc)
      throw error; // re-throw after handling
    }
  }
}


// test.ts
import { ApiClient } from './api-client';

const apiClient = new ApiClient('https://api.example.com'); // side note: in Playwright for example, you can put it in a global fixture

test('fetch users', async () => {
  const response = await apiClient.get('/users');
  const users = await response.json();
  // ...assertions
});
```

### Native Fetch and Error Handling

When using the native API for making HTTP requests (like `fetch` in JavaScript), it's important to handle errors properly. Your custom error handling also allows you to add additional logic like retrying requests, logging errors, or displaying user-friendly messages.

Here's an example of how to handle errors when using JS's `fetch`:

```typescript
try {
  const response = await fetch('https://api.example.com/data');
  if (!response.ok) {
    throw new Error(`Request failed: ${response.status} ${response.statusText}`);
  }
  const data = await response.json();
  // process data
} catch (error) {
  console.error('An error occurred:', error.message);
}
```

## Code Reusability

When writing automated tests, it's important to maximize code reusability to reduce duplication and make tests easier to maintain.

### Centralized Selectors

Store element selectors (like elements ids names) in a centralized location (like a JSON file) and load them into tests as needed. This approach ensures that selectors are consistent across tests and can be easily updated when needed. Avoid hardcoding selectors within tests. This also allow developers to put in app's codebase and QA can use it in their tests.

Example:

```json
// selectors.json
{
  "login": {
    "username": "username",
    "password": "password",
    "submit": "submit"
  },
  "dashboard": {
    "welcomeMessage": "welcome-message",
    "logoutButton": "logout"
  }
}
```

```typescript
// test.ts
import selectors from './selectors.json';

// edit how you load the selectors based on your test framework (like Playwright, Cypress, etc. I'll use Playwright here)
const select = (page, selector) => page.locator(`[data-testid=${selector}]`);

test('login', async ({ page }) => {
  await page.goto('https://example.com/login');
  await select(page, selectors.login.username).fill('myusername');
  await select(page, selectors.login.password).fill('mypassword');
  await select(page, selectors.login.submit).click();
  expect(await select(page, selectors.dashboard.welcomeMessage).innerText()).toBe('Welcome, user!');
});
```

### Centralized Utilities

Create a dedicated project or library for shared test utilities (like helper functions, custom matchers). This promotes code reuse and reduces maintenance effort. Publish the library as a package (npm, private registry) for easy integration into different projects (like unit tests, API tests, E2E tests etc).

Example:

```typescript
// different-repo -> shared-utils
// shared-utils/index.ts
export function customMatcher(received, expected) {
  // custom logic
  return received === expected;
}

// test.ts
import { customMatcher } from 'shared-utils';

test('custom matcher', () => {
  expect(customMatcher(1, 1)).toBe(true);
});
```

## Test Data Generation

### Builder Pattern

Use the builder pattern to create complex test data objects in a flexible and readable manner. This is particularly useful when dealing with optional or conditional properties. Your existing example is a good implementation. Ensure the builder interacts with your encapsulated API client for creating resources.

Example:

```typescript
// user-builder.ts
import { User } from './types';
import { ApiClient } from './api-client';

export class UserBuilder {
  private apiClient = new ApiClient('https://api.example.com');
  private user: Partial<User> = {};

  withName(name: string): UserBuilder {
    this.user.name = name;
    return this;
  }

  withEmail(email: string): UserBuilder {
    this.user.email = email;
    return this;
  }
  
  withAdminRole(): UserBuilder {
    this.user.role = 'admin';
    return this;
  }

  async build(): User {
    const data = await apiClient.post('/users', this.user);
    return data;
  }
}

// test.ts
import { UserBuilder } from './user-builder';

import { login, canSeeAdminPanel } from './test-helpers';

test('default user', () => {
  test('should not be able to see admin panel', async ({ page }) => {
    const user = new UserBuilder().withName('John Doe').withEmail('test@example.com').build();
    await login(user.
    expect(await canSeeAdminPanel()).toBe(false);
  });
});
```

## Contributing

If you have any suggestions or improvements, feel free to open an issue or submit a pull request. Your feedback is highly appreciated!

## License

This repository is licensed under the MIT License. See [LICENSE](LICENSE) for more information.

<!-- Done, thanks for reading! 🚀 -->