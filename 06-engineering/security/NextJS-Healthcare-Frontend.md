---
type: domain-knowledge
date: "2026-02-27"
tags:
  - engineering
  - frontend
  - nextjs
  - security
  - hipaa
  - module-b
  - module-f
category: engineering
confidence: high
sources: []
---

# Next.js 15 Healthcare Frontend Architecture

This document defines the frontend architecture for [[HEALTHCARE_OS_MASTERPLAN]] using Next.js 15 with the App Router. It covers PHI safety through Server Components, HIPAA-aligned security patterns, authentication flows, real-time features, accessibility, and the full project structure. It is tightly coupled with [[System-Architecture-Overview]], [[HIPAA-Deep-Dive]], and [[ADR-004-fastapi-backend-architecture]].

---

## 1. Why Next.js 15 for Healthcare

Next.js 15 is the strongest choice for a healthcare frontend for three structural reasons:

**Server Components keep PHI off the client.** React Server Components (RSC) execute exclusively on the server. When a Server Component fetches patient demographics, lab results, or medication lists, the raw JSON from the FHIR API never appears in the client-side JavaScript bundle. The browser receives only the rendered HTML. This is not a convention -- it is an architectural guarantee enforced by the React runtime. For HIPAA compliance, this means PHI exposure surface is reduced to the server process and the TLS-encrypted HTML response.

**App Router supports complex clinical layouts.** Healthcare workflows demand multi-pane layouts: a patient chart alongside a note editor, a scheduling grid with a patient quick-view modal, a dashboard with parallel data streams. The App Router's route groups, parallel routes, intercepting routes, and nested layouts map directly onto these requirements without resorting to client-side routing hacks.

**Built-in security primitives.** Next.js 15 provides middleware for request-level auth checks, encrypted cookies via `next/headers`, Content Security Policy through `next.config.js` headers, and Server Actions with automatic encrypted references. These are not third-party add-ons; they are first-class framework features that align with the defense-in-depth approach described in [[HIPAA-Deep-Dive]].

---

## 2. Server Components for PHI Safety

This is the most critical architectural decision in the entire frontend. **PHI must only exist inside Server Components.** Violating this rule means PHI leaks into the client JS bundle, which is cached, logged, and inspectable by browser dev tools -- a HIPAA violation.

### The Rule

| Component Type | Can Access PHI? | Can Use Hooks? | Shipped to Client? |
|---|---|---|---|
| Server Component (default) | Yes | No | No (HTML only) |
| Client Component (`"use client"`) | **Never** | Yes | Yes |
| Server Action (`"use server"`) | Yes (mutations) | No | No (RPC endpoint) |

### CORRECT Pattern: Server Component Fetches PHI, Renders HTML

```tsx
// app/(provider)/patients/[id]/page.tsx
// This is a Server Component by default -- no "use client" directive.

import { getPatientById } from "@/lib/fhir/patient";
import { verifySession } from "@/lib/auth/session";
import { PatientBanner } from "@/components/clinical/PatientBanner";
import { MedicationList } from "@/components/clinical/MedicationList";

export default async function PatientChartPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const session = await verifySession();
  if (!session || !session.user.roles.includes("provider")) {
    redirect("/auth/login");
  }

  const { id } = await params;
  const patient = await getPatientById(id, session.accessToken);

  // PHI is fetched and rendered entirely on the server.
  // The client receives only the HTML output.
  return (
    <div className="flex flex-col gap-6 p-6">
      <PatientBanner patient={patient} />
      <MedicationList patientId={id} accessToken={session.accessToken} />
    </div>
  );
}
```

```tsx
// components/clinical/PatientBanner.tsx
// Server Component -- no "use client" directive.

import type { Patient } from "@/types/fhir";

interface PatientBannerProps {
  patient: Patient;
}

export function PatientBanner({ patient }: PatientBannerProps) {
  const name = `${patient.name?.[0]?.given?.join(" ")} ${patient.name?.[0]?.family}`;
  const dob = patient.birthDate;
  const mrn = patient.identifier?.find(
    (i) => i.type?.coding?.[0]?.code === "MR"
  )?.value;

  return (
    <header className="flex items-center gap-4 rounded-lg border border-neutral-200 bg-white p-4 shadow-sm">
      <div>
        <h1 className="text-xl font-semibold text-neutral-900">{name}</h1>
        <p className="text-sm text-neutral-600">
          DOB: {dob} | MRN: {mrn} | Sex: {patient.gender}
        </p>
      </div>
    </header>
  );
}
```

### INCORRECT Pattern: PHI Leaks to Client Bundle

```tsx
// BAD -- DO NOT DO THIS
"use client"; // This directive ships everything to the client

import { useEffect, useState } from "react";

export function PatientBannerClient({ patientId }: { patientId: string }) {
  const [patient, setPatient] = useState(null);

  useEffect(() => {
    // PHI is fetched client-side. The JSON response is in browser memory,
    // visible in DevTools Network tab, and cached by the browser.
    fetch(`/api/patients/${patientId}`)
      .then((r) => r.json())
      .then(setPatient);
  }, [patientId]);

  // The patient object is now in the client JS bundle.
  return <div>{patient?.name}</div>;
}
```

### Server Actions for PHI Mutations

When a provider updates a patient's allergy list or adds a medication, use Server Actions. These are RPC-style functions that execute on the server and are referenced by the client only through encrypted, opaque identifiers.

```tsx
// app/(provider)/patients/[id]/actions.ts
"use server";

import { verifySession } from "@/lib/auth/session";
import { updateAllergies } from "@/lib/fhir/allergy";
import { revalidatePath } from "next/cache";

export async function addAllergyAction(formData: FormData) {
  const session = await verifySession();
  if (!session || !session.user.roles.includes("provider")) {
    throw new Error("Unauthorized");
  }

  const patientId = formData.get("patientId") as string;
  const substance = formData.get("substance") as string;
  const severity = formData.get("severity") as string;

  await updateAllergies(patientId, { substance, severity }, session.accessToken);

  revalidatePath(`/patients/${patientId}`);
}
```

### Client Components: Interactivity Only, No PHI

Client Components are used exclusively for interactive UI elements: buttons, form inputs, drag-drop interactions, modals, and animation. They receive only the minimum data needed for display -- identifiers or pre-rendered content, never raw PHI.

```tsx
// components/ui/AllergyFormDialog.tsx
"use client";

import { useActionState } from "react";
import { addAllergyAction } from "@/app/(provider)/patients/[id]/actions";

export function AllergyFormDialog({ patientId }: { patientId: string }) {
  const [state, formAction, isPending] = useActionState(addAllergyAction, null);

  return (
    <form action={formAction}>
      <input type="hidden" name="patientId" value={patientId} />
      <label htmlFor="substance">Substance</label>
      <input id="substance" name="substance" type="text" required />
      <label htmlFor="severity">Severity</label>
      <select id="severity" name="severity">
        <option value="mild">Mild</option>
        <option value="moderate">Moderate</option>
        <option value="severe">Severe</option>
      </select>
      <button type="submit" disabled={isPending}>
        {isPending ? "Saving..." : "Add Allergy"}
      </button>
    </form>
  );
}
```

---

## 3. Authentication Flow

Authentication integrates SMART on FHIR OAuth2 (for EHR-launched contexts) and a custom session layer (for standalone access). The full auth model is documented in [[HIPAA-Deep-Dive]].

### SMART on FHIR OAuth2 Integration

```tsx
// app/auth/smart/callback/route.ts
import { NextRequest, NextResponse } from "next/server";
import { exchangeCodeForToken } from "@/lib/auth/smart-on-fhir";
import { createSession } from "@/lib/auth/session";

export async function GET(request: NextRequest) {
  const code = request.nextUrl.searchParams.get("code");
  const state = request.nextUrl.searchParams.get("state");

  if (!code || !state) {
    return NextResponse.redirect(new URL("/auth/error", request.url));
  }

  const tokenResponse = await exchangeCodeForToken(code, state);

  await createSession({
    userId: tokenResponse.patient || tokenResponse.sub,
    accessToken: tokenResponse.access_token,
    refreshToken: tokenResponse.refresh_token,
    expiresAt: Date.now() + tokenResponse.expires_in * 1000,
    fhirBaseUrl: tokenResponse.fhir_base_url,
    roles: tokenResponse.scope.split(" "),
  });

  return NextResponse.redirect(new URL("/dashboard", request.url));
}
```

### Middleware for Route Protection

The middleware runs on every request before any page renders. It checks session validity and enforces role-based access.

```tsx
// middleware.ts
import { NextRequest, NextResponse } from "next/server";
import { decrypt } from "@/lib/auth/session";
import { cookies } from "next/headers";

const ROLE_ROUTES: Record<string, string[]> = {
  "/provider": ["provider", "admin"],
  "/patient": ["patient", "admin"],
  "/admin": ["admin"],
  "/billing": ["billing_specialist", "admin"],
};

const PUBLIC_ROUTES = ["/auth/login", "/auth/smart/launch", "/auth/smart/callback"];

export default async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // Allow public routes
  if (PUBLIC_ROUTES.some((route) => pathname.startsWith(route))) {
    return NextResponse.next();
  }

  // Validate session
  const cookieStore = await cookies();
  const sessionCookie = cookieStore.get("session")?.value;
  if (!sessionCookie) {
    return NextResponse.redirect(new URL("/auth/login", request.url));
  }

  const session = await decrypt(sessionCookie);
  if (!session || session.expiresAt < Date.now()) {
    return NextResponse.redirect(new URL("/auth/login", request.url));
  }

  // Check token refresh
  if (session.expiresAt - Date.now() < 5 * 60 * 1000) {
    // Token expires in < 5 minutes; trigger refresh
    const response = NextResponse.next();
    response.headers.set("x-trigger-token-refresh", "true");
    return response;
  }

  // Enforce role-based access
  for (const [routePrefix, allowedRoles] of Object.entries(ROLE_ROUTES)) {
    if (pathname.startsWith(routePrefix)) {
      const hasRole = session.roles.some((role: string) =>
        allowedRoles.includes(role)
      );
      if (!hasRole) {
        return NextResponse.redirect(new URL("/unauthorized", request.url));
      }
    }
  }

  // Apply CSP nonce
  const nonce = Buffer.from(crypto.randomUUID()).toString("base64");
  const cspHeader = buildCSPHeader(nonce);

  const requestHeaders = new Headers(request.headers);
  requestHeaders.set("x-nonce", nonce);
  requestHeaders.set("Content-Security-Policy", cspHeader);

  const response = NextResponse.next({ request: { headers: requestHeaders } });
  response.headers.set("Content-Security-Policy", cspHeader);

  return response;
}

function buildCSPHeader(nonce: string): string {
  const isDev = process.env.NODE_ENV === "development";
  return `
    default-src 'self';
    script-src 'self' 'nonce-${nonce}' 'strict-dynamic'${isDev ? " 'unsafe-eval'" : ""};
    style-src 'self' 'nonce-${nonce}';
    img-src 'self' blob: data:;
    font-src 'self';
    connect-src 'self' ${process.env.FHIR_BASE_URL || ""} ${process.env.API_BASE_URL || ""};
    object-src 'none';
    base-uri 'self';
    form-action 'self';
    frame-ancestors 'none';
    upgrade-insecure-requests;
  `
    .replace(/\s{2,}/g, " ")
    .trim();
}

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico).*)"],
};
```

### Session Management with Encrypted Cookies

```tsx
// lib/auth/session.ts
import { SignJWT, jwtVerify } from "jose";
import { cookies } from "next/headers";

const SECRET_KEY = new TextEncoder().encode(process.env.SESSION_SECRET);

export interface SessionPayload {
  userId: string;
  accessToken: string;
  refreshToken: string;
  expiresAt: number;
  fhirBaseUrl: string;
  roles: string[];
}

export async function createSession(payload: SessionPayload) {
  const jwt = await new SignJWT(payload as unknown as Record<string, unknown>)
    .setProtectedHeader({ alg: "HS256" })
    .setIssuedAt()
    .setExpirationTime("8h")
    .sign(SECRET_KEY);

  const cookieStore = await cookies();
  cookieStore.set("session", jwt, {
    httpOnly: true,
    secure: true,
    sameSite: "lax",
    maxAge: 8 * 60 * 60, // 8 hours
    path: "/",
  });
}

export async function decrypt(token: string): Promise<SessionPayload | null> {
  try {
    const { payload } = await jwtVerify(token, SECRET_KEY);
    return payload as unknown as SessionPayload;
  } catch {
    return null;
  }
}

export async function deleteSession() {
  const cookieStore = await cookies();
  cookieStore.delete("session");
}
```

### Token Refresh Strategy

The refresh strategy is proactive: middleware detects tokens within 5 minutes of expiry and flags the response. A client-side listener in the root layout triggers a silent refresh via a Server Action.

```tsx
// app/(provider)/layout.tsx
import { TokenRefreshProvider } from "@/components/auth/TokenRefreshProvider";

export default function ProviderLayout({ children }: { children: React.ReactNode }) {
  return (
    <TokenRefreshProvider>
      <div className="flex h-screen">
        <Sidebar />
        <main className="flex-1 overflow-auto">{children}</main>
      </div>
    </TokenRefreshProvider>
  );
}
```

---

## 4. App Router Architecture for Healthcare OS

### Route Groups

Route groups organize the application by user role without affecting the URL structure. Each group has its own layout with role-specific navigation, sidebar, and styling.

```
app/
├── (auth)/
│   ├── layout.tsx              # Minimal layout (no sidebar)
│   ├── login/page.tsx
│   └── smart/
│       ├── launch/route.ts
│       └── callback/route.ts
├── (provider)/
│   ├── layout.tsx              # Provider shell: sidebar, patient search
│   ├── dashboard/page.tsx
│   ├── patients/
│   │   ├── [id]/
│   │   │   ├── page.tsx        # Patient chart
│   │   │   ├── encounters/page.tsx
│   │   │   ├── medications/page.tsx
│   │   │   ├── labs/page.tsx
│   │   │   └── notes/
│   │   │       ├── page.tsx
│   │   │       └── [noteId]/page.tsx
│   │   └── page.tsx            # Patient list
│   ├── schedule/page.tsx
│   └── inbox/page.tsx
├── (patient)/
│   ├── layout.tsx              # Patient portal shell
│   ├── dashboard/page.tsx
│   ├── records/page.tsx
│   ├── messages/page.tsx
│   └── appointments/page.tsx
├── (admin)/
│   ├── layout.tsx              # Admin shell
│   ├── users/page.tsx
│   ├── settings/page.tsx
│   └── audit-log/page.tsx
├── (billing)/
│   ├── layout.tsx              # Billing shell
│   ├── dashboard/page.tsx
│   ├── claims/page.tsx
│   ├── prior-auth/page.tsx
│   └── denials/page.tsx
├── layout.tsx                  # Root layout
├── page.tsx                    # Redirect to role-based dashboard
└── middleware.ts
```

### Parallel Routes for Split-Screen Workflows

Providers frequently need a patient chart visible alongside a note editor. Parallel routes render both simultaneously within the same layout.

```tsx
// app/(provider)/workspace/layout.tsx
export default function WorkspaceLayout({
  children,
  chart,
  editor,
}: {
  children: React.ReactNode;
  chart: React.ReactNode;
  editor: React.ReactNode;
}) {
  return (
    <div className="flex h-full">
      <div className="w-1/2 overflow-auto border-r">{chart}</div>
      <div className="w-1/2 overflow-auto">{editor}</div>
    </div>
  );
}
```

The `@chart` and `@editor` slots are defined as subdirectories:

```
app/(provider)/workspace/
├── layout.tsx
├── @chart/
│   └── page.tsx          # Patient chart pane
├── @editor/
│   └── page.tsx          # Note editor pane
└── page.tsx              # Default/fallback
```

### Intercepting Routes for Modals

Patient quick-view modals intercept navigation to the full chart, showing a summary overlay instead.

```
app/(provider)/
├── patients/
│   └── page.tsx                   # Patient list
├── (.)patients/[id]/
│   └── page.tsx                   # Modal quick-view (intercepted)
```

### Loading and Error Boundaries

Every route segment has its own loading and error states. Clinical data loading must provide clear feedback, not blank screens.

```tsx
// app/(provider)/patients/[id]/loading.tsx
export default function PatientChartLoading() {
  return (
    <div className="animate-pulse space-y-4 p-6">
      <div className="h-20 rounded-lg bg-neutral-200" />
      <div className="h-64 rounded-lg bg-neutral-200" />
      <div className="h-48 rounded-lg bg-neutral-200" />
    </div>
  );
}

// app/(provider)/patients/[id]/error.tsx
"use client";

export default function PatientChartError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div className="flex flex-col items-center gap-4 p-12" role="alert">
      <h2 className="text-lg font-semibold text-red-700">
        Unable to load patient chart
      </h2>
      <p className="text-sm text-neutral-600">
        Error reference: {error.digest}
      </p>
      <button
        onClick={reset}
        className="rounded-md bg-neutral-900 px-4 py-2 text-sm text-white"
      >
        Retry
      </button>
    </div>
  );
}
```

---

## 5. Key Pages/Views to Build

| Page | Route | Description |
|---|---|---|
| Provider Dashboard | `/(provider)/dashboard` | Today's schedule, pending tasks, unread messages, recent results |
| Patient Chart | `/(provider)/patients/[id]` | Demographics, conditions, medications, allergies, encounters, labs |
| Clinical Documentation | `/(provider)/workspace` | Split-screen: patient context + note editor with audio recording and AI-assisted coding suggestions |
| Scheduling Grid | `/(provider)/schedule` | Day/week/month views, drag-drop appointment management |
| Prior Auth Tracker | `/(billing)/prior-auth` | Kanban board: submitted, in-review, approved, denied, appealed |
| Revenue Cycle Dashboard | `/(billing)/dashboard` | Claims pipeline, denial rates, A/R aging, payment trends |
| Patient Portal | `/(patient)/dashboard` | Upcoming appointments, recent results, secure messages, medication refill requests |
| Admin Console | `/(admin)/users` | User management, role assignment, practice settings, audit log viewer |

---

## 6. Real-Time Features

### Server-Sent Events vs WebSocket

For healthcare, SSE is preferred over WebSocket for most use cases:

| Criterion | SSE | WebSocket |
|---|---|---|
| Direction | Server to client (unidirectional) | Bidirectional |
| Reconnection | Automatic (built-in) | Manual |
| HTTP/2 multiplexing | Yes | No |
| Proxy/firewall compatibility | Excellent | Problematic in some hospital networks |
| Use case fit | Notifications, results, status updates | Real-time chat, collaborative editing |

Hospital networks often have strict proxy configurations that interfere with WebSocket upgrades. SSE works over standard HTTP and reconnects automatically.

### SSE Implementation

```tsx
// app/api/events/route.ts
import { verifySession } from "@/lib/auth/session";

export async function GET() {
  const session = await verifySession();
  if (!session) {
    return new Response("Unauthorized", { status: 401 });
  }

  const stream = new ReadableStream({
    start(controller) {
      const encoder = new TextEncoder();

      const sendEvent = (event: string, data: unknown) => {
        controller.enqueue(
          encoder.encode(`event: ${event}\ndata: ${JSON.stringify(data)}\n\n`)
        );
      };

      // Subscribe to relevant events for this user
      const unsubscribe = subscribeToUserEvents(session.userId, sendEvent);

      // Heartbeat every 30 seconds
      const heartbeat = setInterval(() => {
        controller.enqueue(encoder.encode(": heartbeat\n\n"));
      }, 30_000);

      // Cleanup on close
      const cleanup = () => {
        clearInterval(heartbeat);
        unsubscribe();
      };

      controller.close = cleanup;
    },
  });

  return new Response(stream, {
    headers: {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache, no-transform",
      Connection: "keep-alive",
    },
  });
}
```

### React Query for Server State Management

React Query (TanStack Query) handles caching, revalidation, and optimistic updates for server state. It pairs with Server Components: initial data is fetched server-side, then React Query manages client-side freshness.

```tsx
// components/providers/QueryProvider.tsx
"use client";

import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { useState } from "react";

export function QueryProvider({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 60 * 1000, // 1 minute
            refetchOnWindowFocus: true,
          },
        },
      })
  );

  return (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  );
}
```

### Optimistic Updates

```tsx
// hooks/useMedicationMutation.ts
"use client";

import { useMutation, useQueryClient } from "@tanstack/react-query";

export function useMedicationMutation(patientId: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (newMed: MedicationRequest) => {
      const res = await fetch(`/api/patients/${patientId}/medications`, {
        method: "POST",
        body: JSON.stringify(newMed),
      });
      return res.json();
    },
    onMutate: async (newMed) => {
      await queryClient.cancelQueries({ queryKey: ["medications", patientId] });
      const previous = queryClient.getQueryData(["medications", patientId]);
      queryClient.setQueryData(["medications", patientId], (old: any[]) => [
        ...old,
        { ...newMed, id: "temp-" + Date.now(), status: "pending" },
      ]);
      return { previous };
    },
    onError: (_err, _newMed, context) => {
      queryClient.setQueryData(["medications", patientId], context?.previous);
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ["medications", patientId] });
    },
  });
}
```

---

## 7. Accessibility (WCAG 2.1 AA)

Healthcare applications serve an older demographic and clinical users under time pressure. Accessibility is not optional -- it is a functional requirement.

### Requirements

- **Color contrast**: Minimum 4.5:1 for normal text, 3:1 for large text. Clinical status indicators must not rely on color alone (use icons + text labels).
- **Keyboard navigation**: Every interactive element must be reachable and operable via keyboard. Focus management must be predictable -- modal opens get focus, modal closes return focus.
- **Screen readers**: Clinical data tables must use proper `<th>`, `scope`, and `aria-label` attributes. FHIR resource displays must have semantic headings.
- **Focus indicators**: Visible focus rings on all interactive elements. Default browser outlines are insufficient.

### Component Library: Radix UI + Tailwind CSS

Radix UI provides unstyled, accessible primitives (Dialog, DropdownMenu, Tabs, Select, Toast) with correct ARIA attributes, keyboard handling, and focus management built in. Tailwind CSS handles visual styling without conflicting with Radix's accessibility semantics.

```tsx
// components/ui/Dialog.tsx
"use client";

import * as RadixDialog from "@radix-ui/react-dialog";

interface DialogProps {
  trigger: React.ReactNode;
  title: string;
  description?: string;
  children: React.ReactNode;
}

export function Dialog({ trigger, title, description, children }: DialogProps) {
  return (
    <RadixDialog.Root>
      <RadixDialog.Trigger asChild>{trigger}</RadixDialog.Trigger>
      <RadixDialog.Portal>
        <RadixDialog.Overlay className="fixed inset-0 bg-black/50" />
        <RadixDialog.Content className="fixed left-1/2 top-1/2 w-full max-w-lg -translate-x-1/2 -translate-y-1/2 rounded-lg bg-white p-6 shadow-xl focus:outline-none">
          <RadixDialog.Title className="text-lg font-semibold">
            {title}
          </RadixDialog.Title>
          {description && (
            <RadixDialog.Description className="mt-1 text-sm text-neutral-600">
              {description}
            </RadixDialog.Description>
          )}
          <div className="mt-4">{children}</div>
          <RadixDialog.Close asChild>
            <button
              className="absolute right-4 top-4 text-neutral-400 hover:text-neutral-600"
              aria-label="Close"
            >
              X
            </button>
          </RadixDialog.Close>
        </RadixDialog.Content>
      </RadixDialog.Portal>
    </RadixDialog.Root>
  );
}
```

---

## 8. Component Architecture

### Design System Components

The healthcare design system is organized in three tiers:

**Tier 1 -- Primitives** (Radix UI wrappers):
`Button`, `Input`, `Select`, `Checkbox`, `Dialog`, `DropdownMenu`, `Tabs`, `Toast`, `Tooltip`, `Badge`, `Avatar`, `Separator`

**Tier 2 -- Composed Components**:
`DataTable`, `FormField`, `SearchCombobox`, `DatePicker`, `StatusIndicator`, `Sidebar`, `TopBar`, `CommandPalette`, `LoadingSkeleton`

**Tier 3 -- Clinical Components** (FHIR-aware):
`PatientBanner`, `MedicationList`, `AllergyList`, `ConditionList`, `EncounterTimeline`, `LabResultPanel`, `VitalSignsChart`, `ClinicalNote`, `ProblemList`, `ImmunizationRecord`

### FHIR Resource Display Components

```tsx
// components/clinical/MedicationList.tsx
// Server Component -- fetches from FHIR server, renders on server.

import { getMedicationsByPatient } from "@/lib/fhir/medication";
import type { MedicationRequest } from "@/types/fhir";

interface MedicationListProps {
  patientId: string;
  accessToken: string;
}

export async function MedicationList({ patientId, accessToken }: MedicationListProps) {
  const medications = await getMedicationsByPatient(patientId, accessToken);

  if (medications.length === 0) {
    return <p className="text-sm text-neutral-500">No active medications.</p>;
  }

  return (
    <section aria-labelledby="medications-heading">
      <h2 id="medications-heading" className="text-lg font-semibold mb-3">
        Active Medications
      </h2>
      <table className="w-full text-sm" role="table">
        <thead>
          <tr className="border-b text-left text-neutral-600">
            <th scope="col" className="pb-2">Medication</th>
            <th scope="col" className="pb-2">Dosage</th>
            <th scope="col" className="pb-2">Frequency</th>
            <th scope="col" className="pb-2">Status</th>
          </tr>
        </thead>
        <tbody>
          {medications.map((med: MedicationRequest) => (
            <tr key={med.id} className="border-b">
              <td className="py-2 font-medium">
                {med.medicationCodeableConcept?.text}
              </td>
              <td className="py-2">
                {med.dosageInstruction?.[0]?.doseAndRate?.[0]?.doseQuantity?.value}{" "}
                {med.dosageInstruction?.[0]?.doseAndRate?.[0]?.doseQuantity?.unit}
              </td>
              <td className="py-2">
                {med.dosageInstruction?.[0]?.timing?.code?.text}
              </td>
              <td className="py-2">
                <StatusBadge status={med.status} />
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </section>
  );
}

function StatusBadge({ status }: { status: string }) {
  const styles: Record<string, string> = {
    active: "bg-green-100 text-green-800",
    completed: "bg-neutral-100 text-neutral-600",
    "entered-in-error": "bg-red-100 text-red-800",
    stopped: "bg-amber-100 text-amber-800",
  };

  return (
    <span
      className={`inline-flex items-center rounded-full px-2 py-0.5 text-xs font-medium ${
        styles[status] || styles.active
      }`}
    >
      {status}
    </span>
  );
}
```

### Form Components: Zod + React Hook Form

```tsx
// components/forms/PatientSearchForm.tsx
"use client";

import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

const searchSchema = z.object({
  query: z.string().min(2, "Enter at least 2 characters"),
  searchType: z.enum(["name", "mrn", "dob"]),
});

type SearchFormData = z.infer<typeof searchSchema>;

interface PatientSearchFormProps {
  onSearch: (data: SearchFormData) => void;
}

export function PatientSearchForm({ onSearch }: PatientSearchFormProps) {
  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<SearchFormData>({
    resolver: zodResolver(searchSchema),
    defaultValues: { searchType: "name" },
  });

  return (
    <form onSubmit={handleSubmit(onSearch)} className="flex gap-2" role="search">
      <select
        {...register("searchType")}
        className="rounded-md border px-3 py-2 text-sm"
        aria-label="Search type"
      >
        <option value="name">Name</option>
        <option value="mrn">MRN</option>
        <option value="dob">Date of Birth</option>
      </select>
      <div className="flex-1">
        <input
          {...register("query")}
          className="w-full rounded-md border px-3 py-2 text-sm"
          placeholder="Search patients..."
          aria-label="Search patients"
          aria-invalid={!!errors.query}
          aria-describedby={errors.query ? "search-error" : undefined}
        />
        {errors.query && (
          <p id="search-error" className="mt-1 text-xs text-red-600" role="alert">
            {errors.query.message}
          </p>
        )}
      </div>
      <button
        type="submit"
        className="rounded-md bg-neutral-900 px-4 py-2 text-sm text-white"
      >
        Search
      </button>
    </form>
  );
}
```

---

## 9. Performance Patterns

### Streaming SSR for Large Patient Charts

Patient charts can contain hundreds of encounters, lab results, and clinical notes. Streaming SSR with Suspense boundaries ensures the page is interactive before all data loads.

```tsx
// app/(provider)/patients/[id]/page.tsx
import { Suspense } from "react";
import { PatientBanner } from "@/components/clinical/PatientBanner";
import { MedicationList } from "@/components/clinical/MedicationList";
import { LabResultPanel } from "@/components/clinical/LabResultPanel";
import { EncounterTimeline } from "@/components/clinical/EncounterTimeline";

export default async function PatientChartPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  const session = await verifySession();
  const patient = await getPatientById(id, session.accessToken);

  return (
    <div className="space-y-6 p-6">
      {/* Banner loads immediately -- critical patient identification */}
      <PatientBanner patient={patient} />

      {/* Sections stream in as data arrives */}
      <div className="grid grid-cols-1 gap-6 lg:grid-cols-2">
        <Suspense fallback={<SectionSkeleton title="Medications" />}>
          <MedicationList patientId={id} accessToken={session.accessToken} />
        </Suspense>

        <Suspense fallback={<SectionSkeleton title="Lab Results" />}>
          <LabResultPanel patientId={id} accessToken={session.accessToken} />
        </Suspense>
      </div>

      <Suspense fallback={<SectionSkeleton title="Encounters" />}>
        <EncounterTimeline patientId={id} accessToken={session.accessToken} />
      </Suspense>
    </div>
  );
}
```

### Image Optimization for Clinical Images

Clinical images (X-rays, dermatology photos, wound images) require special handling. They must not be cached in public CDNs, and EXIF metadata must be stripped.

```tsx
// next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: "https",
        hostname: process.env.FHIR_IMAGE_HOST || "images.clinic.internal",
      },
    ],
    // Clinical images must not be cached in shared caches
    minimumCacheTTL: 0,
  },
};

export default nextConfig;
```

### Bundle Analysis

```json
// package.json (partial)
{
  "scripts": {
    "analyze": "ANALYZE=true next build",
    "build": "next build",
    "lint": "next lint"
  }
}
```

Use `@next/bundle-analyzer` to identify client bundle bloat. The target: total first-load JS under 100kB. Clinical components should remain Server Components to stay out of the client bundle entirely.

---

## 10. Testing Strategy

### Unit Tests: Vitest + React Testing Library

```tsx
// __tests__/components/PatientBanner.test.tsx
import { describe, it, expect } from "vitest";
import { render, screen } from "@testing-library/react";
import { PatientBanner } from "@/components/clinical/PatientBanner";

const mockPatient = {
  resourceType: "Patient",
  id: "123",
  name: [{ given: ["Jane"], family: "Doe" }],
  birthDate: "1985-03-15",
  gender: "female",
  identifier: [
    {
      type: { coding: [{ code: "MR" }] },
      value: "MRN-001234",
    },
  ],
};

describe("PatientBanner", () => {
  it("renders patient name and demographics", () => {
    render(<PatientBanner patient={mockPatient} />);

    expect(screen.getByText("Jane Doe")).toBeInTheDocument();
    expect(screen.getByText(/MRN-001234/)).toBeInTheDocument();
    expect(screen.getByText(/1985-03-15/)).toBeInTheDocument();
  });

  it("handles missing name gracefully", () => {
    const patientNoName = { ...mockPatient, name: [] };
    render(<PatientBanner patient={patientNoName} />);
    // Should not throw
  });
});
```

### E2E Tests: Playwright for Clinical Workflows

```tsx
// e2e/clinical-workflow.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Provider Clinical Workflow", () => {
  test.beforeEach(async ({ page }) => {
    // Login as provider
    await page.goto("/auth/login");
    await page.getByLabel("Email").fill("provider@clinic.test");
    await page.getByLabel("Password").fill("test-password");
    await page.getByRole("button", { name: "Sign in" }).click();
    await expect(page).toHaveURL("/dashboard");
  });

  test("can search and open patient chart", async ({ page }) => {
    await page.getByRole("search").getByLabel("Search patients").fill("Jane Doe");
    await page.getByRole("button", { name: "Search" }).click();
    await page.getByRole("link", { name: /Jane Doe/ }).click();

    await expect(page.getByRole("heading", { name: "Jane Doe" })).toBeVisible();
    await expect(page.getByText("Active Medications")).toBeVisible();
  });

  test("can add an allergy", async ({ page }) => {
    await page.goto("/patients/123");
    await page.getByRole("button", { name: "Add Allergy" }).click();

    await page.getByLabel("Substance").fill("Penicillin");
    await page.getByLabel("Severity").selectOption("severe");
    await page.getByRole("button", { name: "Add Allergy" }).click();

    await expect(page.getByText("Penicillin")).toBeVisible();
  });
});
```

### Accessibility Testing: axe-core

```tsx
// e2e/accessibility.spec.ts
import { test, expect } from "@playwright/test";
import AxeBuilder from "@axe-core/playwright";

test.describe("Accessibility", () => {
  test("patient chart has no accessibility violations", async ({ page }) => {
    await loginAsProvider(page);
    await page.goto("/patients/123");

    const results = await new AxeBuilder({ page })
      .withTags(["wcag2a", "wcag2aa"])
      .analyze();

    expect(results.violations).toEqual([]);
  });

  test("dashboard has no accessibility violations", async ({ page }) => {
    await loginAsProvider(page);
    await page.goto("/dashboard");

    const results = await new AxeBuilder({ page })
      .withTags(["wcag2a", "wcag2aa"])
      .analyze();

    expect(results.violations).toEqual([]);
  });
});
```

---

## 11. Security Headers and CSP

### next.config.ts Security Configuration

```tsx
// next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  poweredBy: false, // Remove X-Powered-By header
  async headers() {
    return [
      {
        source: "/(.*)",
        headers: [
          {
            key: "Strict-Transport-Security",
            value: "max-age=63072000; includeSubDomains; preload",
          },
          {
            key: "X-Frame-Options",
            value: "DENY",
          },
          {
            key: "X-Content-Type-Options",
            value: "nosniff",
          },
          {
            key: "Referrer-Policy",
            value: "strict-origin-when-cross-origin",
          },
          {
            key: "Permissions-Policy",
            value:
              "camera=(self), microphone=(self), geolocation=(), payment=(), usb=()",
          },
          {
            key: "X-DNS-Prefetch-Control",
            value: "on",
          },
        ],
      },
    ];
  },
};

export default nextConfig;
```

Note that `camera=(self)` and `microphone=(self)` are explicitly allowed because the clinical documentation workspace uses audio recording for dictation. All other sensitive APIs are denied. The CSP header is applied dynamically in middleware (see Section 3) to support per-request nonces.

### Healthcare-Specific CSP Considerations

- `connect-src` must include the FHIR server base URL and the backend API URL. Do not use wildcards.
- `img-src` includes `blob:` and `data:` for clinical image rendering and inline SVG icons.
- `frame-ancestors 'none'` prevents clickjacking -- the app must never be embedded in an iframe.
- `form-action 'self'` prevents form data exfiltration.
- `upgrade-insecure-requests` forces HTTPS for all subresources.

---

## 12. Project Structure

```
healthcare-frontend/
├── app/
│   ├── (auth)/
│   │   ├── layout.tsx
│   │   ├── login/
│   │   │   └── page.tsx
│   │   └── smart/
│   │       ├── launch/route.ts
│   │       └── callback/route.ts
│   ├── (provider)/
│   │   ├── layout.tsx
│   │   ├── dashboard/
│   │   │   ├── page.tsx
│   │   │   └── loading.tsx
│   │   ├── patients/
│   │   │   ├── page.tsx
│   │   │   └── [id]/
│   │   │       ├── page.tsx
│   │   │       ├── loading.tsx
│   │   │       ├── error.tsx
│   │   │       ├── actions.ts
│   │   │       ├── encounters/page.tsx
│   │   │       ├── medications/page.tsx
│   │   │       ├── labs/page.tsx
│   │   │       └── notes/
│   │   │           ├── page.tsx
│   │   │           └── [noteId]/page.tsx
│   │   ├── workspace/
│   │   │   ├── layout.tsx
│   │   │   ├── @chart/page.tsx
│   │   │   └── @editor/page.tsx
│   │   ├── schedule/page.tsx
│   │   └── inbox/page.tsx
│   ├── (patient)/
│   │   ├── layout.tsx
│   │   ├── dashboard/page.tsx
│   │   ├── records/page.tsx
│   │   ├── messages/page.tsx
│   │   └── appointments/page.tsx
│   ├── (admin)/
│   │   ├── layout.tsx
│   │   ├── users/page.tsx
│   │   ├── settings/page.tsx
│   │   └── audit-log/page.tsx
│   ├── (billing)/
│   │   ├── layout.tsx
│   │   ├── dashboard/page.tsx
│   │   ├── claims/page.tsx
│   │   ├── prior-auth/page.tsx
│   │   └── denials/page.tsx
│   ├── api/
│   │   └── events/route.ts         # SSE endpoint
│   ├── layout.tsx                   # Root layout
│   ├── page.tsx                     # Root redirect
│   └── globals.css
├── components/
│   ├── ui/                          # Tier 1: Primitives
│   │   ├── Button.tsx
│   │   ├── Input.tsx
│   │   ├── Select.tsx
│   │   ├── Dialog.tsx
│   │   ├── DropdownMenu.tsx
│   │   ├── Tabs.tsx
│   │   ├── Toast.tsx
│   │   ├── Badge.tsx
│   │   └── Tooltip.tsx
│   ├── composed/                    # Tier 2: Composed
│   │   ├── DataTable.tsx
│   │   ├── FormField.tsx
│   │   ├── SearchCombobox.tsx
│   │   ├── DatePicker.tsx
│   │   ├── StatusIndicator.tsx
│   │   ├── Sidebar.tsx
│   │   ├── TopBar.tsx
│   │   └── CommandPalette.tsx
│   ├── clinical/                    # Tier 3: FHIR-aware
│   │   ├── PatientBanner.tsx
│   │   ├── MedicationList.tsx
│   │   ├── AllergyList.tsx
│   │   ├── ConditionList.tsx
│   │   ├── EncounterTimeline.tsx
│   │   ├── LabResultPanel.tsx
│   │   ├── VitalSignsChart.tsx
│   │   ├── ClinicalNote.tsx
│   │   ├── ProblemList.tsx
│   │   └── ImmunizationRecord.tsx
│   ├── forms/                       # Form components
│   │   ├── PatientSearchForm.tsx
│   │   ├── AllergyFormDialog.tsx
│   │   ├── MedicationOrderForm.tsx
│   │   └── EncounterNoteForm.tsx
│   ├── charts/                      # Analytics visualizations
│   │   ├── ClaimsPipelineChart.tsx
│   │   ├── DenialRateChart.tsx
│   │   └── ARAgingChart.tsx
│   ├── auth/
│   │   └── TokenRefreshProvider.tsx
│   └── providers/
│       └── QueryProvider.tsx
├── lib/
│   ├── auth/
│   │   ├── session.ts               # JWT session management
│   │   ├── smart-on-fhir.ts         # SMART on FHIR OAuth2
│   │   └── dal.ts                   # Data access layer (verifySession)
│   ├── fhir/
│   │   ├── client.ts                # FHIR API client
│   │   ├── patient.ts               # Patient resource operations
│   │   ├── medication.ts            # Medication resource operations
│   │   ├── allergy.ts               # AllergyIntolerance operations
│   │   ├── encounter.ts             # Encounter operations
│   │   └── observation.ts           # Observation (labs/vitals)
│   └── utils/
│       ├── audit.ts                 # Audit log helper
│       ├── formatting.ts            # Date, name, unit formatters
│       └── fhir-helpers.ts          # FHIR resource utilities
├── hooks/
│   ├── useSSE.ts                    # Server-Sent Events hook
│   ├── useMedicationMutation.ts
│   └── usePatientSearch.ts
├── types/
│   ├── fhir.ts                      # FHIR R4 resource types
│   ├── session.ts                   # Session types
│   └── api.ts                       # API response types
├── __tests__/
│   ├── components/
│   │   ├── PatientBanner.test.tsx
│   │   ├── MedicationList.test.tsx
│   │   └── AllergyFormDialog.test.tsx
│   ├── lib/
│   │   ├── session.test.ts
│   │   └── fhir-client.test.ts
│   └── e2e/
│       ├── clinical-workflow.spec.ts
│       └── accessibility.spec.ts
├── public/
│   └── favicon.ico
├── middleware.ts
├── next.config.ts
├── tailwind.config.ts
├── tsconfig.json
├── vitest.config.ts
├── playwright.config.ts
├── package.json
└── .env.example
```

### .env.example

```bash
# FHIR Server
FHIR_BASE_URL=
FHIR_IMAGE_HOST=

# Backend API
API_BASE_URL=

# Auth
SESSION_SECRET=
SMART_CLIENT_ID=
SMART_CLIENT_SECRET=
SMART_REDIRECT_URI=

# Feature Flags
ENABLE_AUDIO_DICTATION=
ENABLE_AI_CODING_SUGGESTIONS=
```

---

## Cross-References

- **Backend API architecture**: [[ADR-004-fastapi-backend-architecture]]
- **HIPAA compliance details**: [[HIPAA-Deep-Dive]]
- **System-wide architecture**: [[System-Architecture-Overview]]
- **Overall product vision**: [[HEALTHCARE_OS_MASTERPLAN]]

---

## Summary of Key Principles

1. **PHI stays on the server.** Server Components fetch and render PHI. Client Components handle interactivity. This is the single most important architectural rule.
2. **Middleware enforces auth on every request.** No page renders without a valid, role-checked session.
3. **CSP with per-request nonces.** Every script must be explicitly authorized. No `unsafe-inline` in production.
4. **Streaming SSR for clinical data.** Suspense boundaries ensure progressive loading of large patient charts.
5. **SSE over WebSocket.** Hospital network compatibility and automatic reconnection make SSE the better default.
6. **Accessibility is a requirement, not a feature.** WCAG 2.1 AA compliance is tested in CI with axe-core.
7. **Three-tier component architecture.** Primitives, composed, and clinical components maintain separation of concerns and enable consistent design.
8. **Type safety end-to-end.** Zod validates at boundaries, TypeScript enforces at compile time, FHIR types ensure clinical data correctness.
