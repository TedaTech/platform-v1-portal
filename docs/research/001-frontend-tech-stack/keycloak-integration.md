# Keycloak Authentication Integration

## Three Approaches Evaluated

| Approach | Works With | Effort | Branding Consistency |
|----------|-----------|--------|---------------------|
| Keycloakify | React (best), Svelte, Angular | 1-3 days | High (same component library) |
| FreeMarker Templates | Any framework | 3-5 days | Medium (CSS-only sharing) |
| Custom Login Frontend | Any framework | 5-10 days | Highest (fully custom) |

---

## 1. Keycloakify

**Version:** v11.15.0 (Feb 2025) | **GitHub:** 2,258 stars | **npm:** ~14K weekly downloads | **Author:** Joseph Garrone (also created oidc-spa)

### How It Works
React/Svelte/Angular components are compiled into a Keycloak theme JAR. The JAR is deployed to Keycloak's `providers/` directory. Keycloak renders the themed pages using the compiled components instead of default FreeMarker templates.

### Framework Support
- **React**: most complete. Full support for login themes, Single-Page account theme (v3), and admin UI
- **Svelte**: login themes fully supported. Account themes use Multi-Page (v1 fork) with some extra effort. No admin UI support.
- **Angular**: similar to Svelte support level
- **Vue**: NOT supported

### Themeable Pages
Supports the most common user-facing Keycloak pages. For pages not included, you can eject and customize. Key supported pages:
- Login, Registration, Register User Profile
- Login Update Password, Terms
- Info, Error pages
- WebAuthn pages (via Keycloak's built-in flow)
- OTP pages

For a complete list, run: `npx -p keycloakify download-builtin-keycloak-theme`

### CSS Customization
- Dual-class system: `kc` classes (selectors) + `pf-` classes (PatternFly defaults)
- Can disable PatternFly defaults per-page with `doUseDefaultCss={false}`
- Full component override via `npx keycloakify eject-page`
- CSS custom properties not explicitly documented but standard CSS works

### Account Themes
- **Single-Page (v3)**: Keycloak 25+ default. Real Keycloak code, React-only. Adds i18n-next + react-router-dom + PatternFly deps.
- **Multi-Page (v1 fork)**: Keycloakify-maintained. Works with React/Svelte/Angular. Storybook support. No extra deps. Compatible with all Keycloak versions.

### Keycloak Version Compatibility
Keycloakify v11 supports Keycloak 19+. The EDP Keycloak Operator is at v1.32.0 — need to verify the Keycloak version deployed by Cozystack for compatibility.

### Build & Deployment
1. Build: `npx keycloakify build` → produces a theme JAR
2. Deploy options:
   - Custom Keycloak Docker image: `COPY theme.jar /opt/keycloak/providers/`
   - Init container: mount JAR into providers volume
   - ConfigMap/Secret mount (less common)
3. EDP Operator: `KeycloakRealm` CRD has `spec.themes.loginTheme` field to reference the theme name

### Development Workflow
- Storybook integration for local preview (Multi-Page account theme)
- `npx keycloakify start-keycloak` for local Keycloak with live theme reload
- Hot reload during development

### Effort Estimate
- Basic branded login (colors, logo, fonts): **1-2 days**
- Full custom login + registration + password reset: **2-3 days**
- Account theme customization: **+1-2 days**

---

## 2. FreeMarker Templates

### How It Works
Standard Keycloak theming. Create a theme directory structure (`login/`, `account/`, `email/`) with FreeMarker `.ftl` templates and CSS/JS resources. Package as a JAR or mount into the Keycloak container.

### Theme Structure
```
mytheme/
  login/
    theme.properties
    resources/
      css/login.css
      img/logo.png
    login.ftl
    register.ftl
    ...
  account/
    theme.properties
    ...
  email/
    ...
```

### Customization Levels
1. **CSS-only**: Override `theme.properties` to point to custom CSS. Keep default FreeMarker templates. Fastest but limited to color/font/spacing changes.
2. **Template override**: Copy and modify individual `.ftl` files. Full control over markup. More effort, harder to maintain across Keycloak upgrades.

### Design Token Sharing
- Create a shared CSS custom properties file used by both portal and Keycloak theme
- Keycloak theme references these via standard CSS `var(--color-primary)` etc.
- Requires manual sync — no build-time integration between portal and theme

### Development Workflow
- Preview requires running Keycloak locally with theme mounted
- No hot reload — restart Keycloak to see changes (or use `--spi-theme-static-max-age=-1`)
- No Storybook equivalent

### WebAuthn/Passkey
FreeMarker templates for WebAuthn are available in Keycloak's base theme. Can be copied and customized.

### Deployment
Same as Keycloakify: JAR in providers, custom Docker image, init container, or volume mount.

### Effort Estimate
- CSS-only branding: **1-2 days**
- Full template overrides for login + registration: **3-5 days**
- Ongoing maintenance when Keycloak upgrades: **moderate risk** (templates may change)

---

## 3. Custom Login Frontend (Keycloak REST API)

### How It Works
Build login/registration pages entirely in the portal SPA. Call Keycloak's REST APIs directly for authentication.

### Keycloak APIs
- **Token endpoint**: `POST /realms/{realm}/protocol/openid-connect/token` with `grant_type=password` (Resource Owner Password Grant)
- **Authorization Code + PKCE**: redirect-based flow, but the redirect target could be a custom page
- **Admin REST API**: for user management operations

### Security Concerns
- **Resource Owner Password Grant is deprecated** in OAuth 2.1 and discouraged by Keycloak
- Custom frontend must handle all security edge cases: CSRF, token storage, session management, brute force protection
- Social login (GitHub, Google) requires redirect to IdP — cannot be handled by custom frontend without iframe tricks
- **WebAuthn/Passkey**: browser WebAuthn API calls must happen on the Keycloak domain for credential registration. Custom frontend on a different domain cannot register passkeys for Keycloak.

### Viable Approach
Instead of calling Keycloak APIs directly, implement the **Authorization Code Flow with PKCE** but customize the redirect experience:
1. Portal redirects to Keycloak login page (which is themed via Keycloakify or FreeMarker)
2. After auth, Keycloak redirects back to portal with auth code
3. Portal exchanges code for tokens

This is effectively approach 1 or 2 with a redirect, which is the accepted best practice.

### Effort Estimate
- Full custom implementation: **5-10 days** (not recommended)
- Auth Code + PKCE with themed redirect: **1-2 days** (recommended, but this is really approach 1 or 2)

---

## EDP Keycloak Operator Integration

**Version:** 1.32.0 | **Operator Hub:** [edp-keycloak-operator](https://operatorhub.io/operator/edp-keycloak-operator)

### Relevant CRDs
| CRD | Purpose | Theme-Relevant |
|-----|---------|----------------|
| Keycloak | Connection to Keycloak instance | No |
| KeycloakRealm | Realm management | **Yes** — `spec.themes.loginTheme`, `accountTheme`, `emailTheme` |
| KeycloakClient | OIDC client config | No (but portal will need a client) |
| KeycloakAuthFlow | Auth flow customization | Relevant for WebAuthn/passkey flows |
| KeycloakRealmIdentityProvider | GitHub/Google IDPs | Relevant for social login |

### Theme Deployment Pattern
1. Build theme JAR (via Keycloakify or manually)
2. Build custom Keycloak Docker image that includes the JAR in `/opt/keycloak/providers/`
3. Deploy custom image via Cozystack Keycloak deployment
4. Set `spec.themes.loginTheme: "mytheme"` in `KeycloakRealm` CR

### Open Question
Need to verify how Cozystack deploys Keycloak — whether it uses the EDP Operator directly or has its own Keycloak packaging. This determines the exact theme deployment mechanism.

---

## Recommendation

**Use Keycloakify (React)** as the primary approach:

1. **Best DX**: React component development with Storybook, hot reload
2. **Best branding consistency**: use the same component library (shadcn/ui) in portal and login theme
3. **Maintained**: active development (v11.15.0), 14K weekly downloads
4. **Supports all needed pages**: login, registration, password reset, WebAuthn, OTP
5. **oidc-spa integration**: same author, designed to work together

This reinforces React as the frontend framework choice since Keycloakify has the most complete React support. Vue is NOT supported by Keycloakify.

**Fallback**: FreeMarker CSS-only theming if Keycloakify has compatibility issues with the Cozystack-deployed Keycloak version.
