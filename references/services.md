# Services Pattern

Services live in `src/services/` (global) or `<topic>/services/` (topic-scoped). They are plain TypeScript classes — no Stencil decorators, no JSX. Access is always through a **singleton factory** (`getXxxService()`), never via direct instantiation.

---

## Global Service Template — Service + Factory in one file

```ts
// src/services/api.service.ts

// ==================== Types ====================

/** Abstraction over storage backends (localStorage, sessionStorage, in-memory for tests) */
export interface TokenStorage {
  getItem(key: string): string | null;
  setItem(key: string, value: string): void;
  removeItem(key: string): void;
}

interface ApiServiceConfig {
  baseUrl?: string;
  getToken?: () => Promise<string | null>; // injected at setup — avoids circular imports
}

// ==================== Service Class ====================

/**
 * Base HTTP client — handles auth token injection and error handling.
 * Used as the foundation for all topic-scoped services.
 *
 * Token injection is handled via a `getToken` callback passed in the config,
 * keeping ApiService decoupled from AuthService (no circular imports).
 */
export class ApiService {
  private readonly baseUrl: string;
  private readonly getToken: (() => Promise<string | null>) | undefined;

  constructor(config: ApiServiceConfig = {}) {
    this.baseUrl = config.baseUrl ?? '/api';
    this.getToken = config.getToken;
  }

  private async getHeaders(): Promise<Record<string, string>> {
    const token = this.getToken ? await this.getToken() : null;
    return {
      'Content-Type': 'application/json',
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
    };
  }

  private async request<T>(method: string, path: string, body?: unknown): Promise<T> {
    const response = await fetch(`${this.baseUrl}${path}`, {
      method,
      headers: await this.getHeaders(),
      ...(body ? { body: JSON.stringify(body) } : {}),
    });
    if (!response.ok) throw new Error(`API error ${response.status}: ${path}`);
    return response.json();
  }

  async get<T>(path: string): Promise<T> {
    return this.request<T>('GET', path);
  }

  async post<T>(path: string, body?: unknown): Promise<T> {
    return this.request<T>('POST', path, body);
  }

  async put<T>(path: string, body?: unknown): Promise<T> {
    return this.request<T>('PUT', path, body);
  }

  async delete<T>(path: string): Promise<T> {
    return this.request<T>('DELETE', path);
  }
}

// ==================== Singleton Factory ====================

let instance: ApiService | null = null;

export const getApiService = (config?: ApiServiceConfig): ApiService => {
  if (!instance) {
    instance = new ApiService(config);
  }
  return instance;
};

// For tests: reset the singleton
export const resetApiService = (): void => {
  instance = null;
};
```

---

## Wiring Services at App Start

Initialize the `ApiService` with the token callback once in the root component — this connects Auth and Api without circular imports:

```ts
// src/app/app-root.tsx
import { getApiService } from '../services/api.service';
import { getAuthService } from '../services/auth.service';

@Component({ tag: 'app-root', shadow: false })
export class AppRoot implements ComponentInterface {
  componentWillLoad() {
    // Wire token injection once — ApiService stays decoupled from AuthService
    getApiService({
      getToken: () => getAuthService().ensureValidToken(),
    });
  }
}
```

> This must happen **before** any topic service calls `getApiService()`. Since `componentWillLoad` of the root component runs first, this is guaranteed.

---

## AuthService — Types & Config

```ts
// src/services/auth.service.ts

import { TokenStorage } from './api.service';

interface AuthServiceConfig {
  storage?: TokenStorage;     // defaults to localStorage
  refreshEndpoint?: string;   // defaults to '/api/auth/refresh'
  tokenExpiryBuffer?: number; // seconds before expiry to trigger refresh, default 30
}
```

---

## Rules

- Service class and factory always in **one file**
- Factory name: `getXxxService()` — always this pattern
- Singleton created on first call, reused after
- Always provide `resetXxxService()` for tests
- Constructor takes an optional `config` object — enables dependency injection in tests
- Services are **not Stencil components** — no `@Component`, no JSX
- Always call `getXxxService()` **locally inside methods** — never store as a class field or pass as `@Prop`

---

## Using a Service in a Component

```ts
import { Component, ComponentInterface, State, h } from '@stencil/core';
import { getAuthService } from '../../services/auth.service';

@Component({ tag: 'my-page', styleUrl: 'my-page.scss', shadow: false })
export class MyPage implements ComponentInterface {

  @State() user: User | null = null;

  async componentWillLoad() {
    // Always call locally — never store as this.authService
    this.user = getAuthService().getUser();
  }

  private async handleLogout() {
    getAuthService().logout();
    const router = document.querySelector('ion-router');
    await router.push('/login', 'root');
  }
}
```

---

## AuthService — Method Reference

### Token Storage
| Method | Description |
|--------|-------------|
| `getAccessToken()` | Access token from storage |
| `getRefreshToken()` | Refresh token from storage |
| `setTokens(access, refresh?)` | Persist tokens |
| `clearTokens()` | Remove all auth data |

### User Storage
| Method | Description |
|--------|-------------|
| `getUser()` | User object from storage (JSON-parsed) |
| `setUser(user)` | Persist user object |

### Auth State
| Method | Description |
|--------|-------------|
| `isAuthenticated()` | Token present and not expired |
| `isAdmin()` | Has `ADMIN` or `SUPERADMIN` role |
| `isSuperAdmin()` | Has `SUPERADMIN` role |
| `hasRole(role)` | Has a specific role |
| `getRoles()` | All roles from JWT payload |

### Token Utils
| Method | Description |
|--------|-------------|
| `getTokenPayload(token)` | Decode JWT payload |
| `isTokenExpired(token)` | Expired including buffer time |
| `ensureValidToken()` | Returns valid token — auto-refreshes if expired |
| `refreshAccessToken()` | Explicit refresh — deduplicates concurrent requests |

### Logout
| Method | Description |
|--------|-------------|
| `logout()` | Clear all auth data |

---

## ensureValidToken() — standard pattern for Guards

```ts
// Internal flow:
async ensureValidToken(): Promise<string | null> {
  const token = this.getAccessToken();
  if (!token) return null;                           // no token at all
  if (!this.isTokenExpired(token)) return token;     // still valid
  const refreshed = await this.refreshAccessToken(); // try refresh
  return refreshed ? this.getAccessToken() : null;
}
```

Concurrent refresh requests are deduplicated — even if multiple guards call `ensureValidToken()` simultaneously, only **one** HTTP request goes out.

---

## Topic-Scoped Services — use global services as base

Topic services (`chat/services/chat.service.ts`) handle their own domain and use global services like `ApiService` as the HTTP base. Same factory pattern, but inside the topic folder.

```ts
// src/chat/services/chat.service.ts
import { getApiService } from '../../services/api.service';
import { ChatMessage, ChatRoom } from '../types/chat.types';

/**
 * Chat domain service — rooms and messages.
 * Uses ApiService for HTTP, which handles auth token injection.
 */
export class ChatService {
  async getRooms(): Promise<ChatRoom[]> {
    return getApiService().get<ChatRoom[]>('/chat/rooms');
  }

  async getMessages(roomId: string): Promise<ChatMessage[]> {
    return getApiService().get<ChatMessage[]>(`/chat/rooms/${roomId}/messages`);
  }

  async sendMessage(roomId: string, text: string): Promise<ChatMessage> {
    return getApiService().post<ChatMessage>(`/chat/rooms/${roomId}/messages`, { text });
  }
}

// Singleton Factory — same convention as global services
let instance: ChatService | null = null;

export const getChatService = (): ChatService => {
  if (!instance) {
    instance = new ChatService();
  }
  return instance;
};

export const resetChatService = (): void => {
  instance = null;
};
```

### Dependency direction

```
src/chat/services/chat.service.ts    →  src/services/api.service.ts
src/user/services/user.service.ts    →  src/services/api.service.ts
                                        src/services/auth.service.ts

src/guards/auth.guard.ts             →  src/services/auth.service.ts
```

> Topic services import **only** from `src/services/` — never from other topic folders.
> Global services import **nothing** from topic folders — dependency always flows top-down.
