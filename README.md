# Help n Seek

A full-stack lost-and-found application for the **UC San Diego community**. Students can publish lost or found items, search listings, and contact one another through an inbox. Google Cloud Vision provides image labels to assist item identification.

**Stack:** React 19 · React Router · Node.js · Express 5 · MongoDB/Mongoose · JWT · Cloudinary · Google Cloud Vision · Nodemailer/AWS SES

## Features

- **UCSD account verification:** signup requires an `@ucsd.edu` address and an emailed verification code, with password recovery and bcrypt password hashing.
- **Lost and found listings:** posts include item descriptions, categories, locations, and optional images; owners can resolve or delete their listings.
- **Search and browsing:** authenticated listing and search APIs include pagination, with category and filter interfaces in React.
- **Image identification:** an uploaded photo is sent to Google Cloud Vision label detection, returning labels and the highest-ranked label as an item name.
- **Messaging:** authenticated threads, messages, seen status, and email notification support.
- **Profiles:** account settings, avatars, and notification preferences.

## Architecture

| Component | Responsibility |
| --- | --- |
| `src/pages/`, `src/components/` | React screens, listing workflows, inbox, and settings |
| `backend/server.js` | Express middleware, CORS, route mounting, and startup |
| `backend/routes/` | Authentication, profiles, listings, messaging, uploads, and image identification |
| `backend/models/` | Users, posts, verification codes, threads, and messages |
| `backend/utils/` | Cloudinary uploads and email delivery |
| `backend/config/db.js` | MongoDB connection setup and fallback methods |

The React client calls the Express API. MongoDB stores application data, Cloudinary stores uploaded images, and email services deliver verification and notification messages. Vision requests run on the backend so service-account credentials remain outside the frontend.

## Engineering details

- **Expiring verification data:** verification codes have an expiry timestamp and a MongoDB TTL index; routes also check expiry before accepting a code.
- **Authenticated API access:** middleware accepts JWTs from a bearer header or supported cookies.
- **Bounded list responses:** listing endpoints default to 20 results and cap page size at 50.
- **Listing index:** posts have a compound index on type, resolution status, and creation time.
- **Upload handling:** listing uploads use memory-backed Multer storage, an image MIME-type filter, and a 5 MB limit before streaming to Cloudinary. The separate Vision endpoint has a 10 MB limit.
- **Email delivery options:** SMTP through Nodemailer or AWS SES through its HTTPS API, with SMTP timeout and retry handling.

## Local setup

### 1. Install dependencies

Use a current Node.js LTS release and npm. A TLS-enabled MongoDB instance such as MongoDB Atlas is required by the current connection configuration.

```bash
git clone https://github.com/kylehtet/help-seek.git
cd help-seek
npm install
cd backend
npm install
```

### 2. Configure the backend

Create `backend/.env` with your own service configuration:

```dotenv
PORT=4000
USE_HTTPS=false
FRONTEND_URL=http://localhost:3000
MONGO_URI_SRV=mongodb+srv://<username>:<password>@<cluster>/helpnseek
JWT_ACCESS_SECRET=<long-random-secret>

CLOUDINARY_CLOUD_NAME=<cloud-name>
CLOUDINARY_API_KEY=<api-key>
CLOUDINARY_API_SECRET=<api-secret>

SMTP_HOST=<smtp-host>
SMTP_PORT=587
SMTP_USER=<smtp-username>
SMTP_PASS=<smtp-password>
EMAIL_FROM=<sender-email>
```

Use an email provider configured to send from your chosen sender. Signup and recovery require working email delivery; image uploads require Cloudinary.

For image identification, enable Google Cloud Vision for your project and place its service-account JSON at `backend/google-credentials.json`. Alternatively, set `GOOGLE_CREDENTIALS_B64` to the base64-encoded JSON. Keep credentials and `.env` files out of commits.

The database connector also supports `MONGO_URI_STD`. Its direct-primary fallback uses `MONGO_USER`, `MONGO_PASS`, and `MONGO_DB` with hosts derived from that standard URI.

For AWS SES API delivery instead of SMTP, configure `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, and `EMAIL_FROM`, and leave `SMTP_HOST` unset.

### 3. Start both services

From `backend/`:

```bash
npm run dev
```

In another terminal, from the repository root:

```bash
npm start
```

Open **http://localhost:3000**. The API defaults to **http://localhost:4000**. Set `REACT_APP_API_URL` in the root `.env` to use another API address, then restart the frontend.

### 4. Check the workflow

```bash
curl http://localhost:4000/health
```

The health endpoint returns `{"ok":true}`. Then verify an account with your UCSD email, create a listing, search for it, and open a conversation with another account.

The frontend also provides `npm run build` and `npm test`. The backend package currently provides start/dev scripts and no automated test script.

## Main API surfaces

| Method and route | Purpose |
| --- | --- |
| `GET /health` | Health response |
| `POST /auth/signup/request-code` | Request a UCSD signup verification code |
| `POST /auth/login` | Log in |
| `GET /api/profile/me` | Retrieve the current profile |
| `POST /api/posts/find` or `/api/posts/loss` | Create a found or lost listing |
| `GET /api/posts` | Retrieve paginated listings |
| `GET /api/posts/search?q=...` | Search listings |
| `PATCH /api/posts/:id/resolve` | Resolve a listing |
| `POST /api/threads/open` | Open a conversation |
| `GET /api/threads/:id/messages` | Retrieve conversation messages |
| `POST /api/threads/:id/messages` | Send a message |
| `POST /api/upload` | Upload an image using multipart field `file` |
| `POST /api/vision/identify-item` | Identify an image using multipart field `file` |

Profile, listing, thread, and general upload routes require authentication. Listing creation uses multipart field `image`.

## Current scope

Vision supplies labels for item identification; it does not implement automated lost-to-found similarity matching. External credentials are required to exercise the complete workflow. The repository includes installed dependency and npm-cache directories; install from the package manifests when setting up a fresh environment.
