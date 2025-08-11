## Build a Serverless Web Application using Generative AI

A full‑stack React + AWS Amplify Gen 2 application that lets users authenticate, enter ingredients, and receive an AI‑generated recipe. The app uses Amplify for auth and hosting, AppSync for the API, Lambda for server logic, and Amazon Bedrock for foundation models.

### What you’ll get
- **Frontend**: React + TypeScript + Vite 5, Amplify UI for Auth.
- **Auth**: Amazon Cognito via Amplify `defineAuth` with email verification (custom subject/body).
- **API**: Amplify Data (AppSync GraphQL) + custom resolver to call a Lambda.
- **AI**: Lambda integrates with Amazon Bedrock to generate recipes from ingredients.
- **Hosting/CI**: Amplify Hosting with `amplify.yml` for backend + frontend build.

---

## Architecture
- Users sign up/sign in with Cognito (Amplify Auth).
- Frontend calls API using Amplify Data client (`generateClient`).
- AppSync resolves a custom query (e.g., `askBedrock`) to a Lambda function.
- Lambda calls Amazon Bedrock and returns the generated recipe text.

Services: Amplify Hosting → AppSync → Lambda → Bedrock. Auth via Cognito.

---

## Prerequisites
- Node.js 18 (this repo pins Vite to a Node 18‑compatible version).
- An AWS account with permission to use Amplify, Cognito, AppSync, Lambda, S3, and Bedrock.
- AWS credentials locally (via SSO or access keys). See Setup below.

---

## Project structure
```
ai-recipe-generator/
  amplify/                # Amplify Gen 2 backend (auth, data, build entry)
    auth/resource.ts      # Cognito login/verification configuration
    data/resource.ts      # Data schema + authorization modes
    backend.ts            # Registers resources with Amplify backend
  src/
    App.tsx               # UI + calls Amplify Data client query
  amplify.yml             # Amplify Hosting build steps (backend + frontend)
  package.json            # Scripts and dependencies
```

---

## Local development (first‑time setup)

1) Install dependencies
```bash
npm ci
```

2) Configure an AWS profile (recommended: SSO)
```bash
npx ampx configure profile
```
- Choose browser sign‑in (or your org SSO), pick account/role/region, name it (e.g., `amplify-sandbox`).

3) Start a one‑time sandbox deploy (creates cloud resources in an isolated dev space)
```bash
npx ampx sandbox --profile amplify-sandbox --once
```
This command also writes the app’s configuration file `amplify_outputs.*` into your project directory.

4) Run the app locally
```bash
npm run dev
```
Open the printed local URL. Create an account, verify email, and try generating a recipe.

---

## About amplify_outputs.json (common error and fix)
If you see a Vite error like:
- “Failed to resolve import '../amplify_outputs.json' ... Does the file exist?”

It means the app has not yet written Amplify outputs.

Fix:
- Run one of the following from your project root:
```bash
# Single deployment, then exit
npx ampx sandbox --profile <your-profile> --once

# Or watch for changes (long‑running)
npx ampx sandbox --profile <your-profile>
```
- After success, ensure `amplify_outputs.json` (or `amplify_outputs.ts/mjs` if you chose another format) is present at the project root and the import path in `src/App.tsx` is `../amplify_outputs.json`.

Tip: You can choose the outputs format with `--outputs-format json|mjs|ts` if you prefer. Update the import accordingly.

---

## Deployment with Amplify Hosting (CI/CD)
1) Push your code to GitHub.
2) In Amplify Console, connect the repo/branch. This repo includes an `amplify.yml` that:
   - Installs backend deps and runs `npx ampx pipeline-deploy`.
   - Builds the frontend using Vite and deploys `dist/`.

Node version compatibility is handled by using Vite 5 (works on Node 18). If you upgrade Vite to 7+, ensure the build image runs Node 20+ or pin your own runtime.

---

## Auth configuration
`amplify/auth/resource.ts` uses `defineAuth` with custom email verification:
- Subject: “Welcome to the AI-Powered Recipe Generator!”
- Body includes a generated code via `createCode()`.

Users can sign up with email, receive a code, then sign in. The frontend imports Amplify UI styles and calls `Amplify.configure(outputs)` using the generated `amplify_outputs.json`.

---

## Data/API and AI integration
- `amplify/data/resource.ts` declares the Data schema and authorization modes.
- A custom query (referenced in the UI as `amplifyClient.queries.askBedrock`) is resolved by a Lambda that invokes Amazon Bedrock to generate recipe text.
- The Data client is created via:
```ts
import { generateClient } from "aws-amplify/data";
const client = generateClient<Schema>({ authMode: "userPool" });
```

---

## Scripts
- `npm run dev` – start Vite dev server.
- `npm run build` – type‑check + Vite build (output in `dist/`).
- `npm run preview` – preview built app.

---

## Troubleshooting
- “Unknown argument: --yes” with `ampx sandbox`:
  - The `--yes` flag isn’t supported. Use `npx ampx sandbox --once` or `npx ampx sandbox`.
- “Failed to load default AWS credentials”:
  - Configure a profile: `npx ampx configure profile`, then pass `--profile <name>`.
  - Or export env vars in PowerShell session:
    ```powershell
    $env:AWS_ACCESS_KEY_ID="..."
    $env:AWS_SECRET_ACCESS_KEY="..."
    $env:AWS_REGION="us-east-1"
    ```
- Amplify Hosting build fails due to Node version:
  - This repo uses Vite 5 to stay compatible with Node 18 build images. If you upgrade, ensure Node 20+ in your build image.
- Amplify backend command not found in CI:
  - `amplify.yml` includes backend `preBuild: npm ci`; ensure it’s present so `npx ampx` is available.

---

## Cleanup
To delete the sandbox resources when you’re done developing:
```bash
npx ampx sandbox delete --profile <your-profile>
```