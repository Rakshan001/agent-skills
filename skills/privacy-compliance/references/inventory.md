# Inventory recipes: finding personal data in a codebase

Search, read the hits, and record each finding as a data-map row with `file:line`. These
patterns find candidates; reading the code decides. Adapt paths and file types to the stack.
Record "none found" (with the search used) for any category that comes back empty.

## 1. Schema and migrations

```bash
# Column names that usually hold personal data (SQL migrations, ORM schemas, Prisma, Drizzle,
# Django models, Rails schema.rb, TypeORM entities)
grep -rniE "\b(email|e_mail|name|first_name|last_name|display_name|handle|username|phone|mobile|address|city|country|location|time_?zone|dob|birth|age|gender|pronouns|avatar|image|photo|ip|ip_address|ip_hash|user_agent|ua|device|machine_id|fingerprint|referr?er|lat|lng|latitude|longitude|company|website|bio|github_login|oauth|provider_account|last_seen|last_login)\b" \
  --include=*.sql --include=*.prisma --include=*.ts --include=*.tsx --include=*.js --include=*.jsx \
  --include=*.py --include=*.rb --include=*.go --include=*.java --include=*.kt --include=*.php --include=*.cs \
  . 2>/dev/null | grep -v node_modules | grep -viE "test|spec|fixture" | head -300
```

Also read: every JSON/JSONB column (free-form `detail`, `metadata`, `payload` columns often carry
emails and IPs), every free-text column (comments, feedback, bios, issue bodies), and every table
of events, logs or answers.

Postgres, against a dev database (read-only):

```sql
-- Columns whose names suggest personal data, across all tables
SELECT table_name, column_name, data_type
FROM information_schema.columns
WHERE table_schema = 'public'
  AND column_name ~* '(email|name|phone|address|ip|agent|device|avatar|image|birth|location|referr|login|seen)'
ORDER BY table_name, column_name;
```

## 2. Forms and APIs

```bash
grep -rnE "<input|<textarea|<select|useForm|zod\.object|z\.object|FormData|request\.json\(\)|req\.body|request\.POST|params\.permit" src/ app/ 2>/dev/null | head -200
```

For each form: which fields, which handler, which table. Note required vs optional fields
(notices must say whether data is required and what happens if it is not given).

## 3. Cookies and browser storage

```bash
grep -rnE "cookies\(\)\.set|setCookie|Set-Cookie|document\.cookie|res\.cookie|response\.set_cookie|cookie:|cookieName|maxAge|max_age|expires" src/ app/ 2>/dev/null
grep -rnE "localStorage|sessionStorage|indexedDB|caches\.open|serviceWorker" src/ app/ public/ 2>/dev/null
```

Also list the cookies the auth library and any framework set by default (session, CSRF,
cookie-cache, state/PKCE cookies). Then confirm in a real browser: open the app, sign in, and
read the cookie jar and storage. Record name, purpose, lifetime, path, first/third party, and
whether it is strictly necessary.

## 4. Logs, events, error tracking

```bash
grep -rnE "console\.(log|info|warn|error)|logger\.|log\.(info|warn|error)|captureException|Sentry\.|winston|pino|structlog|logging\.getLogger" src/ app/ 2>/dev/null | head -200
grep -rniE "x-forwarded-for|x-real-ip|remoteAddress|request\.ip|cf-connecting-ip|user-agent" src/ app/ 2>/dev/null
```

Note what each log line contains, where it goes (stdout to the host, a log drain, an error
tracker), and the retention the host or provider applies by default. Check audit/activity tables
and staff-action tables separately: they are often append-only.

## 5. Analytics, third-party scripts, embeds

```bash
grep -rnE "<script|next/script|gtag|googletagmanager|analytics|posthog|mixpanel|segment|plausible|umami|hotjar|clarity|fbq|pixel|intercom|crisp|hubspot|recaptcha|turnstile|fonts\.googleapis|youtube|vimeo|<iframe" src/ app/ public/ 2>/dev/null
grep -rnE "Content-Security-Policy|connect-src|img-src|script-src|frame-src" src/ app/ middleware.* next.config.* 2>/dev/null
```

Every host in the CSP is a place the browser may send data. Avatar and image hosts receive the
visitor's IP and referrer too.

## 6. Processors

```bash
# Dependencies that are usually processors
grep -nE "\"(@vercel|@neondatabase|@supabase|firebase|@aws-sdk|@google-cloud|cloudinary|uploadthing|resend|@sendgrid|nodemailer|postmark|mailgun|stripe|razorpay|@sentry|posthog|openai|@anthropic-ai|algolia|meilisearch|typesense|pusher|ably|twilio)" package.json */package.json 2>/dev/null
# Hosts and secrets names hint at the rest (read names only, never values)
grep -rhoE "^[A-Z][A-Z0-9_]*(URL|HOST|DSN|ENDPOINT|REGION|BUCKET)\b" --include="*.example" . 2>/dev/null | sort -u
```

For each processor record: what data it receives, its region, whether a DPA is signed (ask the
user), and its transfer mechanism (DPF certification or SCCs).

## 7. Uploads and object storage

```bash
grep -rnE "upload|putObject|PutObjectCommand|createPresignedPost|signedUrl|cloudinary\.uploader|multer|formidable|blob\(" src/ app/ 2>/dev/null | head -100
```

Record the bucket or folder layout. Erasure needs a way to find a user's objects: a per-user
prefix, or a table mapping objects to owners. Content-addressed blob stores shared across users
need garbage collection, or a deletion promise cannot be kept.

## 8. Email

```bash
grep -rnE "sendEmail|sendMail|emails\.send|resend\.|sgMail|transporter\.sendMail|List-Unsubscribe|unsubscribe" src/ app/ 2>/dev/null
```

Classify each template as transactional (sign-in link, receipt, security alert) or
promotional/digest. Record whether the provider stores contacts, logs or suppression lists.

## 9. Immutable and append-only data

```bash
grep -rniE "create (or replace )?(trigger|rule|policy)|before (update|delete)|do instead nothing|append[_-]?only|immutable|raise exception|prevent_(update|delete)|no_delete|readonly" \
  --include=*.sql --include=*.py --include=*.rb --include=*.ts --include=*.js . 2>/dev/null | grep -v node_modules
# Covers migrations/, */migrations/ (Django), supabase/migrations/, drizzle/, prisma/, db/
```

Then run the foreign-key and trigger queries in `erasure-patterns.md` §1.

## 10. Hashes that are not as anonymous as they look

```bash
grep -rnE "createHash\(['\"](sha1|sha256|md5)['\"]\)\.update\([^)]*(ip|email|addr)" . 2>/dev/null | grep -v node_modules
grep -rnE "hashlib\.(sha1|sha256|md5)\([^)]*(ip|email)" . 2>/dev/null
grep -rnE "crypto\.subtle\.digest|Digest::SHA|sha256\(|md5\(|digest\(" . 2>/dev/null | grep -v node_modules | grep -iE "\bip\b|_ip|ip_|remote_?addr|email"   # SQL and other runtimes
```

An unsalted hash of an IPv4 address or a known email is reversible by enumeration. Replace with a
keyed hash (HMAC with a server-held secret) and give it a retention period.
