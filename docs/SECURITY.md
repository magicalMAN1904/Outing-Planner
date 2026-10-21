# Engineering Case Study: Building Security Into a Scheduling Backend

[← Back to Outing Planner](../README.md)

## The problem

A group scheduling service stores personal information and lets users interact with others. Securing only the login form would not be enough: each request must distinguish the logged-in user from another user's data, protect state-changing actions, and avoid exposing external-calendar credentials.

## Security and supporting infrastructure

| Layer | Approach | Why it matters |
| --- | --- | --- |
| Password storage | BCrypt hashes | Passwords are not stored as plaintext |
| Login state | Server-side sessions in JDBC-backed storage | Authentication state is validated on the server |
| Session rotation | Existing session invalidated at login/registration | Reduces session-fixation exposure |
| Route protection | Centralized route authorization and ownership validation | Rejects mismatched user claims on owner-scoped routes |
| CSRF | Token checks for protected state-changing routes | Protects session-authenticated actions |
| SQL access | Prepared statements via JDBC | Separates values from SQL structure |
| Resource management | HikariCP connection pooling | Supports consistent connection handling; pooling itself is not an authorization control |
| Request throttling | In-memory per-IP rate limiting | Adds a layer of abuse resistance for the current single-instance setup |
| Provider credentials | Encrypted Google OAuth tokens | Limits exposure of persisted token material |

## Authorization is more than authentication

An authenticated user is not automatically authorized to modify another user's information. The backend distinguishes anonymous, shared, and owner-scoped operations. Some routes derive the actor from the session instead of trusting a user ID supplied by a client.

The application distinguishes **401** (not authenticated) from **403** (authenticated but forbidden). Receiving a forbidden response does not silently log the user out.

## Browser-session considerations

Sessions are persisted on the backend and restored after page refresh. Authentication rotates the session identifier and CSRF token. A session cookie is HttpOnly and uses SameSite=Lax for compatibility with OAuth redirects. Its Secure flag is environment-controlled and must be enabled for any HTTPS production deployment.

## Validation and test strategy

The backend is tested with JUnit and Spring MockMvc, including routes interacting with database state. Regression coverage includes multi-session isolation, ownership boundaries, and Google integration trust boundaries. These tests are an important control, not proof that the application is vulnerability-free.

## Trade-offs and next steps

The current in-memory per-IP rate limiter is suitable to the present single-instance setup, not a distributed deployment. The application is still pre-MVP and should not be described as production-hardened. Security testing, deployment configuration, and remaining edge cases must continue to evolve with the product.

## Engineering takeaway

Reliable security requires consistent boundaries between browser requests, authenticated sessions, ownership decisions, database access, and external providers. Outing Planner uses separate controls at each boundary rather than treating authentication as a one-time check.

*Verified against the development main branch on October 20, 2026. This is a public engineering overview, not a security audit or a disclosure of private configuration. Existing test coverage was inspected; no new application test run was performed for this documentation update.*
