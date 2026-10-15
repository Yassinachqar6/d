# StudentSlides AI

StudentSlides AI is a web-first presentation workspace for students with Supabase authentication, persistent presentations, AI generation, an in-browser editor, and PowerPoint export.

## Stack

- React 19, Vite, and TypeScript
- Express and TypeScript API entry point
- Supabase Auth and PostgreSQL-backed persistence
- Lucide React for interface icons
- CSS design tokens and responsive component styling

## Local development

```bash
npm install
copy .env.example .env
npm run dev
```

The client runs on `http://localhost:5173` and the API runs on `http://localhost:3001`. The API health check is available at `/api/health`.

Set `VITE_API_URL`, `VITE_SUPABASE_URL`, and `VITE_SUPABASE_ANON_KEY` in the frontend environment. Set `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` only in the server/deployment environment; the service-role key must never be exposed to Vite or browser code.

## Authentication and saved presentations

Authentication uses Supabase Auth with browser session persistence. The dashboard is protected, and presentation list, save, rename, and delete operations require a verified bearer session on the server. Every database operation is filtered by the authenticated user ID and the PostgreSQL table has RLS policies for select, insert, update, and delete.

To configure production authentication:

1. Create a Supabase project and enable Email authentication.
2. Apply [supabase/schema.sql](./supabase/schema.sql) in the Supabase SQL editor.
3. Configure `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`, `SUPABASE_URL`, and `SUPABASE_SERVICE_ROLE_KEY` in the deployment environment.
4. Keep `SUPABASE_SERVICE_ROLE_KEY` server-only. It is used only after the API verifies the user's access token.

When these credentials are absent, the UI shows setup instructions and saved-presentation APIs return an explicit configuration error. Production authentication is not verified until a real Supabase project and account are configured.

## Vercel deployment

[vercel.json](./vercel.json) configures the Vite frontend and the Express serverless function at [api/index.ts](./api/index.ts). The API and frontend can run in the same Vercel project, so leave `VITE_API_URL` empty and browser requests use the relative `/api` path.

1. Import the repository into Vercel. Use the repository root as the project root.
2. Keep the default Vercel build settings, which are also declared in `vercel.json`:
   - Framework preset: **Vite**
   - Build command: `npm run build`
   - Output directory: `dist`
   - Install command: `npm install` or the detected default
3. Configure these Vercel **Production** environment variables:

```env
VITE_API_URL=
VITE_SUPABASE_URL=https://your-project.supabase.co
VITE_SUPABASE_ANON_KEY=your-supabase-publishable-key
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_SERVICE_ROLE_KEY=server-only-service-role-key
AI_PROVIDER=gemini
GEMINI_API_KEY=server-only-gemini-key
GEMINI_MODEL=your-supported-gemini-model
MAX_SLIDES=20
GENERATION_QUOTA_PER_HOUR=3
REQUIRE_AUTH_FOR_GENERATION=true
GEMINI_TIMEOUT_MS=25000
GEMINI_MAX_OUTPUT_TOKENS=7000
IMAGE_SEARCH_LIMIT=3
UNSPLASH_ACCESS_KEY=optional-server-only-image-key
CORS_ORIGINS=
FRONTEND_URL=
```

`VITE_*` values are bundled into browser code. `SUPABASE_SERVICE_ROLE_KEY`, `GEMINI_API_KEY`, and `UNSPLASH_ACCESS_KEY` must not use the `VITE_` prefix and must never be configured as exposed frontend values.

4. Apply [supabase/schema.sql](./supabase/schema.sql) in Supabase and configure Email Auth.
5. Add the Vercel production URL to Supabase Auth's site URL and redirect URL settings.
6. Redeploy after environment variables are saved.
7. Test `/api/health`, sign-up, login, generation, dashboard reload, editor save, rename, delete, and PowerPoint export.

The repository intentionally does not contain deployment credentials and no Vercel deployment is claimed from this workspace.

Vercel's Express integration invokes [api/index.ts](./api/index.ts) as a Node.js function. The function has a configured maximum duration of 300 seconds in [vercel.json](./vercel.json), matching the current Vercel Hobby maximum reported in Vercel's Functions limits documentation. Gemini requests are already capped at 25 seconds and PowerPoint export uses temporary invocation storage only; durable data remains in Supabase. Vercel function filesystems are not durable between invocations.

Free-tier constraints matter for this application: Vercel Hobby is intended for personal, non-commercial use and has monthly usage/build/bandwidth/function limits. Supabase Free has limited database/storage/egress quotas and may pause inactive projects. Gemini and image-provider quotas are separate from hosting quotas. Treat the configured generation quota as a safety cap, not an availability guarantee. The existing [render.yaml](./render.yaml) remains available for a separate API deployment, but it is not needed when using the same Vercel project.

## Current scope

The landing page, responsive design system, creation wizard, Supabase authentication, protected dashboard, persistent presentation CRUD, Gemini provider boundary, validated generation endpoint, image retrieval, editor, PowerPoint export, and deployment configuration are in place.

## AI generation

The browser calls the configured API origin at `/api/presentations/generate`; the Gemini key is read exclusively by the Express server. Configure a Gemini model that is enabled for your Google AI project:

```env
AI_PROVIDER=gemini
GEMINI_API_KEY=your-server-only-key
GEMINI_MODEL=your-enabled-model
GENERATION_QUOTA_PER_HOUR=5
```

The API applies input validation, a per-IP hourly quota, a timeout, an output-token cap, and one strict repair attempt when Gemini returns invalid JSON. It returns explicit errors for missing configuration, rejected credentials, rate limits, quota exhaustion, and timeouts. It never silently changes providers.

For local pipeline tests without credentials, use:

```env
AI_PROVIDER=mock
```

Mock responses are labeled `provider: "mock"` and are not evidence of successful Gemini generation. To persist authenticated presentations, apply [supabase/schema.sql](./supabase/schema.sql) and configure `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY`. The service role key must remain server-side.
