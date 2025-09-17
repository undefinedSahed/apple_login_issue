### Documentation: Understanding the "PKCE code_verifier cookie was missing" Error in NextAuth.js with Apple Provider

This documentation explains the error observed in your logs (`[next-auth][error][OAUTH_CALLBACK_ERROR] PKCE code_verifier cookie was missing`), its underlying causes, potential reasons specific to your setup, and step-by-step fixes. This is based on official NextAuth.js documentation, GitHub discussions/issues, Stack Overflow threads, and common troubleshooting patterns for OAuth/PKCE flows with the Apple provider. Note that NextAuth.js (now evolving into Auth.js) uses PKCE for secure authorization code flows, which is mandatory for Apple's Sign in with Apple service.

#### 1. What is Happening?
- **Error Overview**: The error occurs during the OAuth callback phase (`/api/auth/callback/apple`) after the user authenticates with Apple (at `appleid.apple.com`). NextAuth.js expects a `code_verifier` value stored in a cookie (named `next-auth.pkce.code_verifier` or `__Secure-next-auth.pkce.code_verifier` in production) to verify the authorization code exchanged with Apple. This cookie is set earlier during the sign-in initiation (`/api/auth/signin/apple`).
  
- **PKCE Flow Breakdown**:
  1. **Initiation**: When the user clicks "Sign in with Apple," NextAuth.js generates a `code_verifier` (a random secret) and a `code_challenge` (a hashed version of the verifier using S256 method). The `code_verifier` is stored in a cookie, and the `code_challenge` is sent to Apple's authorization endpoint.
  2. **Redirect to Apple**: The browser redirects to `appleid.apple.com`, where the user logs in.
  3. **Callback**: Apple redirects back to your callback URL (`https://olivestocks.com/api/auth/callback/apple`) via a POST request (due to `response_mode: "form_post"` in your config). NextAuth.js retrieves the `code_verifier` from the cookie to complete the token exchange.
  4. **Error Trigger**: If the cookie is missing during this callback, the exchange fails, resulting in `OAuthCallbackError`. Your logs show this happening consistently, redirecting to `/api/auth/error?error=OAuthCallback` and eventually to `/login` with an error.

- **Why PKCE?**: PKCE (Proof Key for Code Exchange, per RFC 7636) prevents authorization code interception attacks. It's enabled in your config via `code_challenge_method: "S256"` and is required for Apple's OAuth implementation. The cookie expires after 15 minutes by default, but expiration isn't the issue here based on your logs (it's created but not present on callback).

- **Observed in Your Logs**:
  - Cookie is created successfully: `[next-auth][debug][CREATE_PKCECODEVERIFIER] { value: 'rJTJQfArZphBEXUHWcTw6kXIZoAjVPmdT-BqdXAoOUg', maxAge: 900 }`.
  - But missing on callback: `PKCE code_verifier cookie was missing`.
  - This leads to a 302 redirect loop to error pages, preventing successful login.

#### 2. Potential Reasons for the Error
Based on your code and logs, here are the most likely causes (prioritized by relevance):

- **SameSite Cookie Policy Mismatch** (Most Likely in Your Case):
  - Your config uses `sameSite: "lax"` for the `pkceCodeVerifier` cookie.
  - Apple's callback uses a cross-site POST request (from `appleid.apple.com` to `olivestocks.com`). Browsers with `SameSite=Lax` block cookies on cross-site POSTs (only allowing them on top-level GET navigations). This prevents the cookie from being sent back.
  - `SameSite=Lax` is a secure default but incompatible with OAuth callbacks involving cross-origin POSTs like Apple's `form_post`.

- **Secure Flag and HTTPS Mismatch**:
  - In production (`NODE_ENV="production"`), your cookie uses `secure: true` and `__Secure-` prefix, which requires HTTPS. If there's any HTTP fallback or proxy issue, the cookie won't set.
  - Browser privacy features (e.g., Safari's ITP or Chrome's third-party cookie blocking) may restrict secure cookies in cross-site contexts.

- **Domain/URI Mismatches**:
  - Redirect URI mismatch: Your callback is `https://olivestocks.com/api/auth/callback/apple`, but if Apple sees a different URI (e.g., due to port forwarding, subdomains like `www.`, or local testing IPs like `192.168.x.x`), the flow breaks.
  - Local vs. Production: If testing locally (e.g., `localhost:3000` vs. public IP), cookies may not persist across domains.

- **Browser or Network Issues**:
  - Cookies blocked by browser extensions, incognito mode, or strict privacy settings.
  - Network proxies, CDNs, or load balancers stripping/modifying cookies.
  - Cookie overwrite: Other app middleware or auth flows interfering.

- **Configuration Gaps in NextAuth.js**:
  - Missing explicit `cookies` config for PKCE in some setups, though you have it.
  - Outdated NextAuth.js version: Bugs in older versions (pre-v4.24) affected PKCE handling.
  - Apple-Specific: Apple's flow requires `response_mode: "form_post"`, which relies on POST, exacerbating SameSite issues.

- **Other Rare Causes**:
  - State mismatch (related error in logs sometimes), cookie expiration (unlikely here), or invalid client secret generation.

#### 3. How to Fix It
Follow these steps in order. Test after each change by clearing browser cookies/cache and retrying the Apple login flow. Enable `debug: true` in NextAuth.js for detailed logs.

##### Step 1: Update Cookie Configuration (Primary Fix)
Change `sameSite: "lax"` to `"none"` for cross-site compatibility. Make it conditional on environment to avoid issues in development.

Updated `cookies` section in `src/app/api/auth/[...nextauth]/route.ts`:

```typescript
cookies: {
  sessionToken: {
    name: `${
      process.env.NODE_ENV === "production"
        ? "__Secure-next-auth.session-token"
        : "next-auth.session-token"
    }`,
    options: {
      httpOnly: true,
      sameSite: process.env.NODE_ENV === "production" ? "none" : "lax",  // Change to "none" in production
      path: "/",
      secure: process.env.NODE_ENV === "production",
    },
  },
  pkceCodeVerifier: {
    name: `${
      process.env.NODE_ENV === "production"
        ? "__Secure-next-auth.pkce.code_verifier"
        : "next-auth.pkce.code_verifier"
    }`,
    options: {
      httpOnly: true,
      sameSite: process.env.NODE_ENV === "production" ? "none" : "lax",  // Key change here
      path: "/",
      secure: process.env.NODE_ENV === "production",
    },
  },
},
```

- **Why?**: `SameSite=None` allows the cookie on cross-site requests (including POST), but requires `secure: true` (which you have in production).
- Also add `csrfToken` if not present (some fixes include it for completeness):

```typescript
csrfToken: {
  name: `${
    process.env.NODE_ENV === "production"
      ? "__Host-next-auth.csrf-token"
      : "next-auth.csrf-token"
  }`,
  options: {
    httpOnly: true,
    sameSite: process.env.NODE_ENV === "production" ? "none" : "lax",
    path: "/",
    secure: process.env.NODE_ENV === "production",
  },
},
```



##### Step 2: Verify Environment and Deployment
- **Set NODE_ENV Correctly**: Ensure `.env` has `NODE_ENV=production` for your live site. Redeploy.
- **Use HTTPS Everywhere**: Confirm `olivestocks.com` is fully HTTPS. Use tools like SSL Labs to check.
- **Host on a Public Platform**: If port forwarding/local IP is involved, deploy to Vercel/Netlify. Set `trustHost: true` in NextAuth config:

```typescript
const handler = NextAuth({
  trustHost: true,  // Add this
  // ... rest of config
});
```

- **Update NextAuth.js**: Run `npm update next-auth` to get the latest version (v4.24+ recommended as of 2025).

##### Step 3: Validate Apple Configuration
- **Apple Developer Portal**:
  - Log in to https://developer.apple.com/account/resources/identifiers/.
  - For your Service ID (`com.olivestocks.web.login`), ensure the redirect URI exactly matches `https://olivestocks.com/api/auth/callback/apple` (no trailing slash, exact protocol/domain).
  - Add any variants (e.g., with `www.`) if needed.
- **Client Secret**: Your `generateAppleClientSecret` looks correct, but regenerate and test if expired (valid for 180 days).

##### Step 4: Debugging and Testing
- **Browser Tools**: In Chrome DevTools > Application > Cookies, check if `__Secure-next-auth.pkce.code_verifier` sets after sign-in initiation and is sent on callback.
- **Network Logs**: Verify `Set-Cookie` header on `/signin/apple` and cookie presence on `/callback/apple`.
- **Test Locally**: Set `NODE_ENV=development`, use `http://localhost:3000`, and add local redirect URI to Apple portal. Set `secure: false`.
- **Alternative: Disable PKCE Temporarily**: Remove `code_challenge_method: "S256"` to test (not for production, as Apple requires it).
- **If Persists**: Check for middleware conflicts in Next.js (e.g., `middleware.ts`). Search GitHub for similar issues or open a new one at https://github.com/nextauthjs/next-auth/issues.

##### Expected Outcome
After fixes, the callback should succeed, logging the user in and populating the JWT/session with Apple data. If issues remain, share updated logs or browser cookie screenshots for further diagnosis.
