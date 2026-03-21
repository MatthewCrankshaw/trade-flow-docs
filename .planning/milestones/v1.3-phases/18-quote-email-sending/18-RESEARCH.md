# Phase 18: Quote Email Sending - Research

**Researched:** 2026-03-21
**Domain:** Email sending (SendGrid), HTML email templates (Maizzle), rich text editing (Tiptap), business settings extension
**Confidence:** HIGH

## Summary

Phase 18 adds the ability for a tradesperson to send a quote to their customer via email. The system already has a working `EmailSenderService` wrapping `@sendgrid/mail`, a `QuoteTokenCreator` for generating secure view links, and a `QuoteTransitionService` that handles DRAFT->SENT transitions. The primary new work is: (1) a `QuoteEmailSender` orchestration service, (2) an HTML email template built with Maizzle, (3) a `SendQuoteDialog` frontend component with rich text editing for the message body, (4) business settings extension for default email template configuration, and (5) a new `POST /v1/business/:businessId/quote/:quoteId/send` API endpoint.

The existing codebase patterns are well-established. The email module already exports `EmailSenderService`, the quote module has transition logic, and the frontend follows shadcn Dialog + RTK Query mutation patterns consistently. The main research questions were around Maizzle's programmatic rendering API and the rich text editor choice for the send dialog.

**Primary recommendation:** Use Maizzle `@maizzle/framework` v5.5.0 for server-side HTML email rendering with Tailwind CSS, Tiptap (`@tiptap/react` v3.20.4 + `@tiptap/starter-kit`) for the message body rich text editor, and simple string replacement for template variables (`{{variableName}}` -> value).

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Full email preview dialog with three editable fields: To (email), Subject (text), Message (rich text with bold/italic/links)
- **D-02:** To field pre-filled from customer record email; Subject pre-filled from default template; Message pre-filled from default template with variables resolved
- **D-03:** If customer has no email on file, dialog opens with empty To field and a warning ("No email on file for this customer") plus a checkbox "Save email to customer record"
- **D-04:** To field is always editable -- tradesperson can send to a different email than the one on file
- **D-05:** Auto-focus lands on the message body textarea when dialog opens
- **D-06:** After successful send: success toast ("Quote sent to john@example.com"), dialog closes, quote status badge updates to Sent, sentAt timestamp appears
- **D-07:** Default email template configured in Business Settings -- new "Quote Email" section with default subject line and default message body
- **D-08:** Template variables: `{{customerName}}`, `{{quoteNumber}}`, `{{jobTitle}}`, `{{businessName}}`, `{{userName}}` -- resolved when the send dialog opens
- **D-09:** If business has not configured a template, use a system-provided default
- **D-10:** Branded card layout: business name header, tradesperson's personal message, prominent "View Quote Online" CTA button, quote summary (number, date, total inc. tax), small "Sent via Trade Flow" footer
- **D-11:** HTML email built with Maizzle using Tailwind CSS -- templates live in trade-flow-api in a dedicated templates/ directory
- **D-12:** "View Quote Online" button links to `/quote/view/{token}` (from Phase 17)
- **D-13:** Re-send uses the same dialog as first send, pre-filled fresh from the default template
- **D-14:** Re-send generates a new token -- previous tokens remain active until 30-day expiry
- **D-15:** Status stays Sent on re-send (SENT->SENT transition already supported)
- **D-16:** Quote status transitions from Draft to Sent automatically when email is successfully sent
- **D-17:** Transition happens server-side after confirmed SendGrid delivery -- not optimistically on the client

### Claude's Discretion
- Maizzle project structure and build configuration within trade-flow-api
- Rich text editor library choice for the message body in the send dialog
- Email template variable resolution approach (server-side vs client-side for preview)
- SendGrid dynamic template vs server-rendered HTML approach
- Quote email sender service class structure
- Business settings UI layout for the template configuration section
- Error handling for SendGrid failures (retry, user feedback)

### Deferred Ideas (OUT OF SCOPE)
- Store sent message per send event for re-send pre-fill
- Email delivery status tracking (bounced, opened)
- Multiple email templates per business
- CC/BCC fields on send dialog
- "Powered by Trade Flow" link in email footer (future branding)
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| DLVR-01 | User can send a quote to a customer via email with a link to view the quote online | New send endpoint + QuoteEmailSender service + Maizzle HTML email template + QuoteTokenCreator integration. PDF attachment excluded (Phase 20). |
| DLVR-02 | User can configure a default quote email template in business settings with variable placeholders | Business entity/DTO extension with quoteEmailSubject + quoteEmailBody fields, business settings UI tab, business update endpoint already exists |
| DLVR-03 | User can review and edit the pre-filled email message in a send dialog before sending, with template variables automatically resolved | SendQuoteDialog component with Tiptap rich text editor, client-side variable resolution for preview |
| DLVR-04 | User can re-send a quote that is already in Sent status | Same send endpoint, SENT->SENT transition already supported in quote-transitions.ts, new token generated per D-14 |
| AUTO-01 | Quote status automatically transitions from Draft to Sent when the user sends the quote | QuoteTransitionService.transition() called server-side after SendGrid success, sentAt timestamp set automatically |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| @sendgrid/mail | 8.1.6 | Email delivery via SendGrid API | Already installed and wrapped in EmailSenderService |
| @maizzle/framework | 5.5.0 | Server-side HTML email rendering with Tailwind CSS | User decision D-11; email-optimized Tailwind with inline CSS output |
| @tiptap/react | 3.20.4 | Rich text editor for message body in send dialog | Headless, composable, TypeScript-first, works with shadcn styling |
| @tiptap/starter-kit | 3.20.4 | Core Tiptap extensions (bold, italic, lists, etc.) | Provides all D-01 formatting needs in one package |
| @tiptap/extension-link | 3.20.4 | Link support for message body | D-01 requires links in rich text |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| @tiptap/extension-placeholder | 3.20.4 | Placeholder text in empty editor | UX polish for message body |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Maizzle | Raw HTML + inline styles manually | Maizzle handles email client quirks (Outlook tables, inline CSS) automatically; raw HTML is error-prone |
| Maizzle | react-email | react-email is React-based (mismatch for NestJS backend); Maizzle is Node.js native |
| Tiptap | Plain textarea | D-01 requires bold/italic/links -- textarea cannot do this |
| Tiptap | minimal-tiptap shadcn component | Pulls in too many dependencies (lowlight, image zoom, etc.); custom minimal setup with starter-kit is lighter |
| Tiptap | Lexical | Tiptap has simpler API for basic formatting; Lexical is more complex for this use case |

**Installation (API):**
```bash
cd trade-flow-api && npm install @maizzle/framework@5.5.0
```

**Installation (UI):**
```bash
cd trade-flow-ui && npm install @tiptap/react@3.20.4 @tiptap/starter-kit@3.20.4 @tiptap/extension-link@3.20.4 @tiptap/extension-placeholder@3.20.4
```

## Architecture Patterns

### Recommended Project Structure (API additions)
```
trade-flow-api/src/
├── email/
│   ├── services/
│   │   ├── email-sender.service.ts          # Existing -- SendGrid wrapper
│   │   └── quote-email-renderer.service.ts  # NEW -- Maizzle template rendering
│   ├── templates/
│   │   └── quote-email.html                 # NEW -- Maizzle HTML email template
│   └── interfaces/
│       └── send-email.dto.ts                # Existing -- extend if needed
├── quote/
│   ├── controllers/
│   │   └── quote.controller.ts              # Existing -- add send endpoint
│   ├── services/
│   │   └── quote-email-sender.service.ts    # NEW -- orchestration service
│   └── requests/
│       └── send-quote.request.ts            # NEW -- request validation
├── business/
│   ├── entities/
│   │   └── business.entity.ts               # EXTEND -- add quoteEmailSubject, quoteEmailBody
│   ├── data-transfer-objects/
│   │   └── business.dto.ts                  # EXTEND -- add quoteEmailSubject, quoteEmailBody
│   └── services/
│       └── business-updater.service.ts      # EXTEND -- permit new fields
```

### Recommended Project Structure (UI additions)
```
trade-flow-ui/src/
├── features/quotes/
│   ├── components/
│   │   ├── SendQuoteDialog.tsx              # NEW -- send email dialog
│   │   └── QuoteActionStrip.tsx             # MODIFY -- wire Send/Re-send to dialog
│   └── api/
│       └── quoteApi.ts                      # EXTEND -- add sendQuote mutation
├── features/business/
│   └── components/
│       └── QuoteEmailSettings.tsx           # NEW -- template config in settings
├── components/
│   └── ui/
│       └── rich-text-editor.tsx             # NEW -- reusable Tiptap wrapper
├── pages/
│   └── SettingsPage.tsx                     # MODIFY -- add "Quote Email" tab
```

### Pattern 1: QuoteEmailSender Orchestration Service
**What:** A service that orchestrates the full send flow: validate inputs -> generate token -> render HTML email -> send via SendGrid -> transition quote status.
**When to use:** Called from the quote controller's send endpoint.
**Example:**
```typescript
// trade-flow-api/src/quote/services/quote-email-sender.service.ts
@Injectable()
export class QuoteEmailSender {
  constructor(
    private readonly quoteRetriever: QuoteRetriever,
    private readonly customerRetriever: CustomerRetriever,
    private readonly businessRetriever: BusinessRetriever,
    private readonly quoteTokenCreator: QuoteTokenCreator,
    private readonly quoteEmailRenderer: QuoteEmailRenderer,
    private readonly emailSenderService: EmailSenderService,
    private readonly quoteTransitionService: QuoteTransitionService,
    private readonly configService: ConfigService,
  ) {}

  public async send(authUser: IUserDto, quoteId: string, request: SendQuoteRequest): Promise<IQuoteDto> {
    // 1. Retrieve quote, customer, business
    // 2. Generate token via quoteTokenCreator.create()
    // 3. Render HTML email via quoteEmailRenderer.render()
    // 4. Send via emailSenderService.sendEmail()
    // 5. Transition status via quoteTransitionService.transition()
    // 6. Return updated quote
  }
}
```

### Pattern 2: Client-Side Variable Resolution for Preview
**What:** Template variables (`{{customerName}}`, etc.) are resolved on the client when the send dialog opens. The resolved text is editable. The server receives the final message text, not a template.
**When to use:** Send dialog pre-fill. The server does NOT re-resolve variables -- it receives the final user-edited message and injects it into the Maizzle HTML layout.
**Why:** The user may edit the message after variables are resolved. Sending the raw template + variables to the server would lose user edits.

### Pattern 3: Maizzle Programmatic Rendering
**What:** Use `@maizzle/framework`'s `render()` API to compile the HTML email template server-side with Tailwind CSS inlined.
**When to use:** In QuoteEmailRenderer service when preparing the email HTML.
**Example:**
```typescript
// trade-flow-api/src/email/services/quote-email-renderer.service.ts
import { render } from '@maizzle/framework';

@Injectable()
export class QuoteEmailRenderer {
  public async render(data: QuoteEmailData): Promise<string> {
    const template = fs.readFileSync(
      path.join(__dirname, '../templates/quote-email.html'), 'utf-8'
    );
    // Inject data into template before Maizzle processes it
    const populated = template
      .replace('{{businessName}}', data.businessName)
      .replace('{{message}}', data.message)
      .replace('{{viewUrl}}', data.viewUrl)
      // ... etc
    const { html } = await render(populated, {
      tailwind: {
        config: { /* email-safe tailwind config */ },
      },
    });
    return html;
  }
}
```

### Pattern 4: Extending Business Entity for Template Fields
**What:** Add optional `quoteEmailSubject` and `quoteEmailBody` fields to the business entity, DTO, and repository mapping.
**When to use:** Business settings template configuration.
**Example:**
```typescript
// Addition to IBusinessDto
quoteEmailSubject?: string;
quoteEmailBody?: string;
```

### Anti-Patterns to Avoid
- **Optimistic status transition on client:** D-17 explicitly requires server-side transition after confirmed SendGrid delivery. Never update status optimistically.
- **SendGrid Dynamic Templates:** The user decision (D-11) specifies Maizzle + Tailwind CSS for email rendering. Do NOT use SendGrid's dynamic template system -- render HTML server-side and send raw HTML via SendGrid.
- **Storing rendered HTML per send:** Out of scope per deferred ideas. The message body from the dialog is injected into the Maizzle layout at send time.
- **Heavy rich text editor:** Do not install a full-featured editor like CKEditor or Quill. Tiptap with starter-kit provides exactly bold/italic/links with minimal bundle size.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| HTML email compatibility | Manual inline CSS, table layouts | Maizzle `@maizzle/framework` | Outlook, Gmail, Yahoo all have different CSS support; Maizzle handles inline styles, responsive tables, and client-specific fixes automatically |
| Rich text editing | Custom contentEditable div | Tiptap with @tiptap/starter-kit | Cursor management, selection handling, HTML serialization are deceptively complex |
| Email delivery | Raw SMTP or fetch to SendGrid API | @sendgrid/mail (already installed) | Handles authentication, rate limiting, error codes |
| Token generation | Custom random string | QuoteTokenCreator (already built) | Cryptographically secure, 30-day expiry, stored in MongoDB |
| Status transitions | Manual status field update | QuoteTransitionService (already built) | Validates allowed transitions, sets timestamps |

**Key insight:** The existing codebase already has 80% of the infrastructure. The new work is primarily orchestration (QuoteEmailSender), one HTML template (Maizzle), and one dialog component (SendQuoteDialog).

## Common Pitfalls

### Pitfall 1: Maizzle ESM Import in NestJS CommonJS
**What goes wrong:** Maizzle v5 is an ESM-only package. NestJS projects typically use CommonJS. Importing with `require()` fails.
**Why it happens:** `@maizzle/framework` v5.x ships as ESM module only.
**How to avoid:** Use dynamic `import()` in the renderer service: `const { render } = await import('@maizzle/framework')`. Or configure the email renderer as a standalone ES module. Alternatively, if the project tsconfig targets ES modules, it works directly.
**Warning signs:** `ERR_REQUIRE_ESM` error at runtime.

### Pitfall 2: Email Client CSS Stripping
**What goes wrong:** CSS classes or `<style>` blocks are stripped by Gmail, Outlook, etc. Email appears unstyled.
**Why it happens:** Email clients remove `<style>` tags and only support inline styles.
**How to avoid:** Maizzle automatically inlines Tailwind CSS utilities as inline styles during the `render()` call. Verify the output HTML has inline styles, not class-based styling.
**Warning signs:** Email looks correct in browser preview but broken in Gmail/Outlook.

### Pitfall 3: SendGrid "from" Address Not Verified
**What goes wrong:** SendGrid rejects the email with a 403 error because the sender address is not verified.
**Why it happens:** SendGrid requires sender identity verification for all "from" addresses.
**How to avoid:** This is already flagged in STATE.md as an ops concern. Use a verified sender (e.g., `noreply@tradeflow.app`) from environment config. Do NOT use the tradesperson's personal email as the "from" address.
**Warning signs:** 403 Forbidden from SendGrid API.

### Pitfall 4: Rich Text HTML Injection in Email
**What goes wrong:** The user's rich text message (from Tiptap) contains HTML that breaks the email layout or enables XSS.
**Why it happens:** Tiptap outputs HTML (`<p>`, `<strong>`, `<a>`) which is injected into the Maizzle template.
**How to avoid:** Sanitize the HTML output from Tiptap before injecting into the email template. Only allow `<p>`, `<br>`, `<strong>`, `<em>`, `<a>` tags. Strip everything else.
**Warning signs:** Broken email layout, unexpected JavaScript execution.

### Pitfall 5: Template Variable Resolution Timing
**What goes wrong:** Variables like `{{customerName}}` appear literally in the email because they were resolved at the wrong stage.
**Why it happens:** Confusion between client-side preview resolution and server-side email rendering.
**How to avoid:** Variables are resolved CLIENT-SIDE when the dialog opens (for preview/editing). The server receives the final edited message text (with variables already resolved). The server only injects the message into the Maizzle HTML layout -- it does NOT re-process template variables in the message body.
**Warning signs:** Literal `{{customerName}}` in sent emails.

### Pitfall 6: Missing Customer Email Handling
**What goes wrong:** Send is attempted with an empty "to" address.
**Why it happens:** Customer has no email on file (email field is `string | null`).
**How to avoid:** D-03 defines the UX: warning message + empty To field. The send button should be disabled until a valid email is entered. Server-side validation must also reject empty "to" field.
**Warning signs:** SendGrid error for invalid recipient.

## Code Examples

### Send Quote Dialog (Frontend Pattern)
```typescript
// trade-flow-ui/src/features/quotes/components/SendQuoteDialog.tsx
// Follows CreateQuoteDialog pattern: Dialog wraps a form component
import { Dialog, DialogContent } from "@/components/ui/dialog";

interface SendQuoteDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  quote: Quote;
  customer: { name: string; email: string | null };
  business: Business;
  userName: string;
}

export function SendQuoteDialog({ open, onOpenChange, ...props }: SendQuoteDialogProps) {
  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent className="sm:max-w-lg">
        {open && <SendQuoteForm onOpenChange={onOpenChange} {...props} />}
      </DialogContent>
    </Dialog>
  );
}
```

### RTK Query Send Mutation
```typescript
// Addition to quoteApi.ts
sendQuote: builder.mutation<
  Quote,
  { businessId: string; quoteId: string; to: string; subject: string; message: string; saveEmail?: boolean }
>({
  query: ({ businessId, quoteId, to, subject, message, saveEmail }) => ({
    url: `/v1/business/${businessId}/quote/${quoteId}/send`,
    method: "POST",
    body: { to, subject, message, saveEmail },
  }),
  transformResponse: (response: StandardResponse<Quote>) => {
    if (response.data && response.data.length > 0) return response.data[0];
    throw new Error("No quote data returned");
  },
  invalidatesTags: (_result, _error, { quoteId }) => [
    { type: "Quote", id: quoteId },
    { type: "Quote", id: "LIST" },
  ],
}),
```

### Send Quote Request (Backend Validation)
```typescript
// trade-flow-api/src/quote/requests/send-quote.request.ts
import { IsEmail, IsString, IsNotEmpty, IsOptional, IsBoolean } from "class-validator";

export class SendQuoteRequest {
  @IsEmail()
  to: string;

  @IsString()
  @IsNotEmpty()
  subject: string;

  @IsString()
  @IsNotEmpty()
  message: string;

  @IsOptional()
  @IsBoolean()
  saveEmail?: boolean;
}
```

### Maizzle Email Template Structure
```html
<!-- trade-flow-api/src/email/templates/quote-email.html -->
<!DOCTYPE html>
<html>
<head>
  <style>
    /* Tailwind utilities will be processed by Maizzle and inlined */
  </style>
</head>
<body class="bg-gray-100">
  <table class="w-full max-w-[600px] mx-auto">
    <tr>
      <td class="bg-white p-8 rounded-lg">
        <!-- Business name header -->
        <h1 class="text-xl font-bold text-gray-900">{{ businessName }}</h1>
        <hr class="my-4 border-gray-200">

        <!-- Tradesperson's personal message (sanitized HTML from Tiptap) -->
        <div class="text-gray-700 text-base leading-relaxed">
          {{{ message }}}
        </div>

        <!-- CTA Button -->
        <table class="w-full my-6">
          <tr>
            <td align="center">
              <a href="{{ viewUrl }}" class="bg-blue-600 text-white px-8 py-3 rounded-md font-semibold no-underline inline-block">
                View Quote Online
              </a>
            </td>
          </tr>
        </table>

        <!-- Quote summary -->
        <table class="w-full bg-gray-50 rounded-md p-4">
          <tr><td class="text-sm text-gray-600">Quote: {{ quoteNumber }}</td></tr>
          <tr><td class="text-sm text-gray-600">Date: {{ quoteDate }}</td></tr>
          <tr><td class="text-sm font-semibold text-gray-900">Total: {{ total }}</td></tr>
        </table>

        <!-- Footer -->
        <p class="text-xs text-gray-400 text-center mt-8">Sent via Trade Flow</p>
      </td>
    </tr>
  </table>
</body>
</html>
```

### Tiptap Minimal Editor Component
```typescript
// trade-flow-ui/src/components/ui/rich-text-editor.tsx
import { useEditor, EditorContent } from "@tiptap/react";
import StarterKit from "@tiptap/starter-kit";
import Link from "@tiptap/extension-link";
import Placeholder from "@tiptap/extension-placeholder";

interface RichTextEditorProps {
  content: string;
  onChange: (html: string) => void;
  placeholder?: string;
  autoFocus?: boolean;
}

export function RichTextEditor({ content, onChange, placeholder, autoFocus }: RichTextEditorProps) {
  const editor = useEditor({
    extensions: [
      StarterKit.configure({ heading: false, codeBlock: false, blockquote: false }),
      Link.configure({ openOnClick: false }),
      Placeholder.configure({ placeholder }),
    ],
    content,
    autofocus: autoFocus ? "end" : false,
    onUpdate: ({ editor }) => onChange(editor.getHTML()),
  });

  return (
    <div className="rounded-md border border-input bg-background">
      {/* Minimal toolbar: Bold, Italic, Link */}
      <div className="flex gap-1 border-b px-2 py-1">
        {/* Toggle buttons using editor.chain().focus().toggleBold().run() etc. */}
      </div>
      <EditorContent editor={editor} className="prose prose-sm max-w-none p-3 min-h-[120px]" />
    </div>
  );
}
```

### Template Variable Resolution (Client-Side)
```typescript
// Utility function for resolving template variables in the send dialog
function resolveTemplateVariables(template: string, variables: Record<string, string>): string {
  return template.replace(/\{\{(\w+)\}\}/g, (match, key) => variables[key] ?? match);
}

// Usage in SendQuoteDialog:
const variables = {
  customerName: customer.name,
  quoteNumber: quote.number,
  jobTitle: quote.jobTitle,
  businessName: business.name,
  userName: user.name ?? user.email,
};
const resolvedSubject = resolveTemplateVariables(defaultSubject, variables);
const resolvedMessage = resolveTemplateVariables(defaultMessage, variables);
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| MJML for email templates | Maizzle (Tailwind-based) | Maizzle v5 (2024) | Tailwind CSS familiarity, same styling approach as frontend |
| SendGrid Dynamic Templates | Server-rendered HTML via Maizzle | N/A (architecture choice) | Full control over template, no vendor lock-in for template design |
| Textarea for message input | Tiptap rich text editor | Tiptap v2->v3 (2024) | Headless architecture allows shadcn-style customization |

**Deprecated/outdated:**
- Maizzle v3 syntax (v5 uses ESM, different render API)
- Tiptap v1 (v3.20 is current, significant API changes from v1)

## Open Questions

1. **Maizzle ESM Compatibility with NestJS**
   - What we know: Maizzle v5.5.0 is ESM-only. NestJS typically uses CommonJS (ts-node with module: "commonjs").
   - What's unclear: Whether dynamic `import()` works cleanly in the NestJS runtime, or if additional tsconfig/package.json configuration is needed.
   - Recommendation: Use dynamic `import()` in the QuoteEmailRenderer service. Test early in implementation. If ESM causes issues, fall back to spawning a separate render script or pre-building the template at build time.

2. **Tiptap HTML Output Sanitization**
   - What we know: Tiptap produces clean HTML for the extensions it supports. StarterKit with limited extensions should produce only safe tags.
   - What's unclear: Whether additional server-side sanitization is needed beyond the limited tag set.
   - Recommendation: Implement a simple server-side allowlist sanitizer (allow `p`, `br`, `strong`, `em`, `a`, `ul`, `ol`, `li` only) as defense-in-depth.

3. **SendGrid "from" Address Configuration**
   - What we know: STATE.md flags this as an ops concern. `EmailSenderService` reads from the `SendEmailDto.from` field.
   - What's unclear: Whether a verified sender identity exists.
   - Recommendation: Use env var `SENDGRID_FROM_EMAIL` (e.g., `noreply@tradeflow.app`). Sender name should be the business name.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Jest 30.2.0 |
| Config file | package.json `jest` section in trade-flow-api |
| Quick run command | `cd trade-flow-api && npx jest --testPathPattern="quote-email" --no-coverage` |
| Full suite command | `cd trade-flow-api && npm run test` |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| DLVR-01 | QuoteEmailSender orchestrates token + render + send + transition | unit | `npx jest quote-email-sender.service.spec.ts -x` | Wave 0 |
| DLVR-01 | QuoteEmailRenderer produces valid HTML from Maizzle template | unit | `npx jest quote-email-renderer.service.spec.ts -x` | Wave 0 |
| DLVR-01 | Quote controller send endpoint validates and delegates | unit | `npx jest quote.controller.spec.ts -x` | Wave 0 |
| DLVR-02 | Business entity/DTO includes quoteEmailSubject and quoteEmailBody | unit | `npx jest business.repository.spec.ts -x` | Existing (extend) |
| DLVR-03 | SendQuoteDialog renders with pre-filled fields | manual-only | N/A -- React component, no existing test framework for UI | N/A |
| DLVR-04 | Re-send generates new token, status stays SENT | unit | `npx jest quote-email-sender.service.spec.ts -x` | Wave 0 |
| AUTO-01 | Status transitions to SENT after successful send | unit | `npx jest quote-email-sender.service.spec.ts -x` | Wave 0 |

### Sampling Rate
- **Per task commit:** `cd trade-flow-api && npx jest --testPathPattern="quote-email|quote-transition" --no-coverage`
- **Per wave merge:** `cd trade-flow-api && npm run test`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `trade-flow-api/src/quote/test/services/quote-email-sender.service.spec.ts` -- covers DLVR-01, DLVR-04, AUTO-01
- [ ] `trade-flow-api/src/email/test/services/quote-email-renderer.service.spec.ts` -- covers DLVR-01 (HTML rendering)
- [ ] `trade-flow-api/src/quote/test/mocks/quote-email.mock-generator.ts` -- shared test fixtures for email sending

## Sources

### Primary (HIGH confidence)
- Existing codebase: `email-sender.service.ts`, `quote-transition.service.ts`, `quote-token-creator.service.ts` -- verified current state
- Existing codebase: `QuoteActionStrip.tsx`, `CreateQuoteDialog.tsx`, `quoteApi.ts` -- verified frontend patterns
- Existing codebase: `business.entity.ts`, `business.dto.ts`, `business-updater.service.ts` -- verified extension points
- npm registry: `@maizzle/framework@5.5.0`, `@tiptap/react@3.20.4`, `@tiptap/starter-kit@3.20.4` -- verified current versions

### Secondary (MEDIUM confidence)
- [Maizzle Introduction](https://maizzle.com/docs/introduction) -- framework overview and capabilities
- [Maizzle Node.js API (v3)](https://v3.maizzle.com/docs/nodejs/) -- programmatic render API (v3 docs; v5 API similar but ESM)
- [Maizzle Starter API](https://github.com/maizzle/starter-api) -- programmatic usage example
- [Tiptap React Docs](https://tiptap.dev/docs/editor/getting-started/install/react) -- React integration
- [Tiptap GitHub](https://github.com/ueberdosis/tiptap) -- framework overview

### Tertiary (LOW confidence)
- Maizzle v5 ESM compatibility with NestJS CommonJS -- not verified with concrete integration test; based on general ESM/CJS interop knowledge

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - all packages verified on npm, existing SendGrid already installed
- Architecture: HIGH - follows established project patterns (controller -> service -> repository), existing code reviewed
- Pitfalls: HIGH - ESM/CJS issue is well-documented, email client rendering is known domain, SendGrid verification flagged in STATE.md
- Maizzle v5 API specifics: MEDIUM - v3 docs verified, v5 render API documented but official docs page returned only CSS

**Research date:** 2026-03-21
**Valid until:** 2026-04-21 (stable domain, package versions pinned)
