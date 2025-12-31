# TypeScript Monorepo POC - Complete Setup Guide

## 🎯 Architecture Overview

This monorepo demonstrates a type-safe API development pattern with:
- **packages/backend-shared**: Zod schemas & TypeScript types (single source of truth)
- - **packages/sdk**: Typed API client with runtime validation
  - - **backend**: Example API implementation
    - - **frontend**: Next.js app consuming the SDK
      - - **contracts**: JSON fixtures for regression testing
       
        - ## 📦 Prerequisites
       
        - ```bash
          node >= 18.0.0
          pnpm >= 8.0.0
          ```

          ## 🚀 Quick Start

          ```bash
          # Clone repository
          git clone https://github.com/redact22/typescript-monorepo-poc.git
          cd typescript-monorepo-poc

          # Install dependencies
          pnpm install

          # Build all packages
          pnpm build

          # Run development mode
          pnpm dev
          ```

          ## 📁 Complete File Structure

          ```
          typescript-monorepo-poc/
          ├── packages/
          │   ├── backend-shared/
          │   │   ├── src/
          │   │   │   ├── schemas/
          │   │   │   │   ├── user.schema.ts
          │   │   │   │   └── index.ts
          │   │   │   └── index.ts
          │   │   ├── package.json
          │   │   └── tsconfig.json
          │   ├── sdk/
          │   │   ├── src/
          │   │   │   └── index.ts
          │   │   ├── package.json
          │   │   └── tsconfig.json
          ├── backend/
          │   ├── src/
          │   │   └── index.ts
          │   ├── package.json
          │   └── tsconfig.json
          ├── frontend/
          │   ├── app/
          │   │   └── page.tsx
          │   ├── package.json
          │   └── tsconfig.json
          ├── contracts/
          │   ├── requests/
          │   │   └── auth.json
          │   └── responses/
          │       └── auth.json
          ├── package.json
          ├── pnpm-workspace.yaml
          └── turbo.json
          ```

          ## 🔧 Configuration Files

          ### `pnpm-workspace.yaml`
          ```yaml
          packages:
            - 'packages/*'
            - 'backend'
            - 'frontend'
          ```

          ### `turbo.json`
          ```json
          {
            "$schema": "https://turbo.build/schema.json",
            "globalDependencies": ["**/.env.*local"],
            "pipeline": {
              "build": {
                "dependsOn": ["^build"],
                "outputs": ["dist/**", ".next/**"]
              },
              "dev": {
                "cache": false,
                "persistent": true
              },
              "lint": {},
              "test": {
                "dependsOn": ["build"]
              }
            }
          }
          ```

          ## 📝 Package Implementations

          ### packages/backend-shared/package.json
          ```json
          {
            "name": "@monorepo/backend-shared",
            "version": "1.0.0",
            "main": "./dist/index.js",
            "types": "./dist/index.d.ts",
            "scripts": {
              "build": "tsc",
              "dev": "tsc --watch"
            },
            "dependencies": {
              "zod": "^3.22.4"
            },
            "devDependencies": {
              "typescript": "^5.3.3"
            }
          }
          ```

          ### packages/backend-shared/src/schemas/user.schema.ts
          ```typescript
          import { z } from 'zod';

          export const UserSchema = z.object({
            id: z.string().uuid(),
            email: z.string().email(),
            name: z.string(),
            role: z.enum(['admin', 'user', 'guest']),
            createdAt: z.string().datetime()
          });

          export const LoginRequestSchema = z.object({
            email: z.string().email(),
            password: z.string().min(8)
          });

          export const LoginResponseSchema = z.object({
            token: z.string(),
            user: UserSchema
          });

          export type User = z.infer<typeof UserSchema>;
          export type LoginRequest = z.infer<typeof LoginRequestSchema>;
          export type LoginResponse = z.infer<typeof LoginResponseSchema>;
          ```

          ### packages/backend-shared/src/index.ts
          ```typescript
          export * from './schemas/user.schema';
          ```

          ### packages/sdk/package.json
          ```json
          {
            "name": "@monorepo/sdk",
            "version": "1.0.0",
            "main": "./dist/index.js",
            "types": "./dist/index.d.ts",
            "scripts": {
              "build": "tsc",
              "dev": "tsc --watch"
            },
            "dependencies": {
              "@monorepo/backend-shared": "workspace:*",
              "zod": "^3.22.4"
            },
            "devDependencies": {
              "typescript": "^5.3.3"
            }
          }
          ```

          ### packages/sdk/src/index.ts
          ```typescript
          import {
            User,
            UserSchema,
            LoginRequest,
            LoginRequestSchema,
            LoginResponse,
            LoginResponseSchema
          } from '@monorepo/backend-shared';

          export class ApiClient {
            private baseURL: string;
            private token?: string;

            constructor(baseURL: string, token?: string) {
              this.baseURL = baseURL;
              this.token = token;
            }

            private async request<T>(
              endpoint: string,
              options: RequestInit = {},
              schema: any
            ): Promise<T> {
              const headers: HeadersInit = {
                'Content-Type': 'application/json',
                ...(this.token && { Authorization: `Bearer ${this.token}` }),
                ...options.headers
              };

              const response = await fetch(`${this.baseURL}${endpoint}`, {
                ...options,
                headers
              });

              if (!response.ok) {
                throw new Error(`API Error: ${response.statusText}`);
              }

              const data = await response.json();

              // Runtime validation with Zod
              return schema.parse(data);
            }

            async login(credentials: LoginRequest): Promise<LoginResponse> {
              // Validate request
              LoginRequestSchema.parse(credentials);

              // Make API call with response validation
              return this.request<LoginResponse>('/auth/login', {
                method: 'POST',
                body: JSON.stringify(credentials)
              }, LoginResponseSchema);
            }

            async getUser(userId: string): Promise<User> {
              return this.request<User>(`/users/${userId}`, {}, UserSchema);
            }

            setToken(token: string) {
              this.token = token;
            }
          }

          export { User, LoginRequest, LoginResponse };
          ```

          ### backend/package.json
          ```json
          {
            "name": "@monorepo/backend",
            "version": "1.0.0",
            "scripts": {
              "dev": "tsx watch src/index.ts",
              "build": "tsc"
            },
            "dependencies": {
              "@monorepo/backend-shared": "workspace:*",
              "express": "^4.18.2"
            },
            "devDependencies": {
              "@types/express": "^4.17.21",
              "tsx": "^4.7.0",
              "typescript": "^5.3.3"
            }
          }
          ```

          ### backend/src/index.ts
          ```typescript
          import express from 'express';
          import { LoginRequestSchema, LoginResponseSchema } from '@monorepo/backend-shared';

          const app = express();
          app.use(express.json());

          app.post('/auth/login', (req, res) => {
            try {
              // Validate incoming request with shared schema
              const credentials = LoginRequestSchema.parse(req.body);

              // Business logic here
              const response = {
                token: 'mock-jwt-token',
                user: {
                  id: '123e4567-e89b-12d3-a456-426614174000',
                  email: credentials.email,
                  name: 'John Doe',
                  role: 'user' as const,
                  createdAt: new Date().toISOString()
                }
              };

              // Validate response before sending
              const validatedResponse = LoginResponseSchema.parse(response);
              res.json(validatedResponse);
            } catch (error) {
              res.status(400).json({ error: 'Invalid request' });
            }
          });

          app.listen(3001, () => {
            console.log('Backend running on http://localhost:3001');
          });
          ```

          ### frontend/package.json
          ```json
          {
            "name": "@monorepo/frontend",
            "version": "1.0.0",
            "scripts": {
              "dev": "next dev",
              "build": "next build",
              "start": "next start"
            },
            "dependencies": {
              "@monorepo/sdk": "workspace:*",
              "next": "^14.0.4",
              "react": "^18.2.0",
              "react-dom": "^18.2.0"
            },
            "devDependencies": {
              "@types/react": "^18.2.45",
              "typescript": "^5.3.3"
            }
          }
          ```

          ### frontend/app/page.tsx
          ```typescript
          'use client';

          import { useState } from 'react';
          import { ApiClient } from '@monorepo/sdk';

          const api = new ApiClient('http://localhost:3001');

          export default function Home() {
            const [email, setEmail] = useState('');
            const [password, setPassword] = useState('');
            const [response, setResponse] = useState<any>(null);
            const [error, setError] = useState<string | null>(null);

            const handleLogin = async (e: React.FormEvent) => {
              e.preventDefault();
              setError(null);

              try {
                // SDK automatically validates request and response
                const result = await api.login({ email, password });
                setResponse(result);
              } catch (err: any) {
                setError(err.message);
              }
            };

            return (
              <main style={{ padding: '2rem' }}>
                <h1>Monorepo POC - Type-Safe API Demo</h1>

                <form onSubmit={handleLogin} style={{ marginTop: '2rem' }}>
                  <div>
                    <input
                      type="email"
                      placeholder="Email"
                      value={email}
                      onChange={(e) => setEmail(e.target.value)}
                      style={{ padding: '0.5rem', marginRight: '0.5rem' }}
                    />
                    <input
                      type="password"
                      placeholder="Password (min 8 chars)"
                      value={password}
                      onChange={(e) => setPassword(e.target.value)}
                      style={{ padding: '0.5rem', marginRight: '0.5rem' }}
                    />
                    <button type="submit" style={{ padding: '0.5rem 1rem' }}>
                      Login
                    </button>
                  </div>
                </form>

                {error && (
                  <div style={{ color: 'red', marginTop: '1rem' }}>
                    Error: {error}
                  </div>
                )}

                {response && (
                  <div style={{ marginTop: '2rem' }}>
                    <h2>Response:</h2>
                    <pre>{JSON.stringify(response, null, 2)}</pre>
                  </div>
                )}
              </main>
            );
          }
          ```

          ### contracts/requests/auth.json
          ```json
          {
            "login": {
              "email": "test@example.com",
              "password": "password123"
            }
          }
          ```

          ### contracts/responses/auth.json
          ```json
          {
            "login": {
              "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
              "user": {
                "id": "123e4567-e89b-12d3-a456-426614174000",
                "email": "test@example.com",
                "name": "Test User",
                "role": "user",
                "createdAt": "2024-01-01T00:00:00Z"
              }
            }
          }
          ```

          ## 🧪 Contract Testing

          Create a test file to validate contracts:

          ### contracts/test.ts
          ```typescript
          import * as fs from 'fs';
          import { LoginRequestSchema, LoginResponseSchema } from '@monorepo/backend-shared';

          const authRequests = JSON.parse(fs.readFileSync('./requests/auth.json', 'utf8'));
          const authResponses = JSON.parse(fs.readFileSync('./responses/auth.json', 'utf8'));

          // Validate request fixtures
          console.log('Validating request contracts...');
          LoginRequestSchema.parse(authRequests.login);
          console.log('✓ Login request valid');

          // Validate response fixtures
          console.log('Validating response contracts...');
          LoginResponseSchema.parse(authResponses.login);
          console.log('✓ Login response valid');

          console.log('\\n✅ All contracts valid!');
          ```

          ## 🔄 CI/CD Integration

          Add to your CI pipeline:

          ```yaml
          # .github/workflows/ci.yml
          name: CI
          on: [push, pull_request]

          jobs:
            contract-tests:
              runs-on: ubuntu-latest
              steps:
                - uses: actions/checkout@v3
                - uses: pnpm/action-setup@v2
                  with:
                    version: 8
                - uses: actions/setup-node@v3
                  with:
                    node-version: 18
                    cache: 'pnpm'
                - run: pnpm install
                - run: pnpm build
                - run: pnpm test
          ```

          ## 🎓 Key Benefits

          1. **Type Safety**: Compile-time errors if frontend/backend drift
          2. 2. **Runtime Validation**: Zod catches invalid API responses at runtime
             3. 3. **Single Source of Truth**: All types defined once in backend-shared
                4. 4. **Contract Testing**: JSON fixtures prevent breaking changes
                   5. 5. **Developer Experience**: Autocomplete for all API calls
                     
                      6. ## 🚀 Next Steps
                     
                      7. 1. Add more schemas to `backend-shared`
                         2. 2. Expand SDK with additional endpoints
                            3. 3. Implement error handling middleware
                               4. 4. Add authentication/authorization layers
                                  5. 5. Set up contract regression tests in CI
                                    
                                     6. ---
                                    
                                     7. **Repository**: https://github.com/redact22/typescript-monorepo-poc
                                     8. **Created**: 2025
                                     9. 
