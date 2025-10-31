# MERN Backend Boilerplate

A production-ready backend setup for MERN projects using Node.js, Express, MongoDB, Cloudinary, and Multer. This README contains:

* All shell commands to create the project, install dependencies, and create files/folders.
* A recommended file/folder structure.
* All essential configuration files and production-ready middlewares (DB, Cloudinary, Multer, logging, security, rate-limiting).
* A simple async handler to avoid repetitive `try/catch` blocks.
* A global error handler and not-found handler.
* `app.js` and `server.js` (index) that connect to DB first, then start the server.

> Use this document as a template for any MERN backend project. Copy the repository, update environment variables, and add routes/controllers/models as needed.

---

## Prerequisites

* Node.js (>= 18 recommended)
* npm or yarn
* MongoDB URI (Atlas or self-hosted)
* Cloudinary account (optional, required for file uploads)

---

## Step 1 — Create project & install dependencies

### Create project folder

```bash
# Create project directory and enter it
mkdir mern-backend-boilerplate && cd mern-backend-boilerplate

# Initialize git (optional)
git init
```

### Initialize npm and install dependencies

```bash
# Initialize package.json
npm init -y

# Install runtime dependencies
npm install express mongoose dotenv cors helmet compression express-rate-limit xss-clean hpp cookie-parser morgan winston bcryptjs jsonwebtoken cloudinary multer multer-storage-cloudinary

# Install developer dependencies
npm install -D nodemon cross-env eslint prettier eslint-config-prettier eslint-plugin-node
```

> The key packages:
>
> * `express`, `mongoose` — core server & DB
> * security: `helmet`, `xss-clean`, `hpp`, `express-rate-limit`
> * logging: `morgan`, `winston`
> * file uploads & Cloudinary: `multer`, `multer-storage-cloudinary`, `cloudinary`
> * `dotenv` for environment variables

---

## Step 2 — Create folders & files

Run these shell commands to create the standard file structure and empty files (works in Unix-like shells):

```bash
# Create folders
mkdir -p src/config src/controllers src/models src/routes src/middlewares src/utils src/validations src/services src/uploads

# Create entry files
touch src/server.js src/app.js

# Configs
touch src/config/db.js src/config/cloudinary.js src/config/multer.js

# Middleware
touch src/middlewares/errorHandler.js src/middlewares/asyncHandler.js src/middlewares/notFound.js src/middlewares/logger.js

# Utils
touch src/utils/logger.js

# Example index route
mkdir -p src/routes
touch src/routes/index.js

# Example controller (no API logic — just an example)
 touch src/controllers/health.controller.js

# Gitignore and env
touch .env .env.example .gitignore

# npm scripts helper
touch nodemon.json
```

You can add more files later (controllers, models, validations, services, etc.).

---

## Step 3 — `.env.example`

Create `.env.example` to show required environment variables (do not commit `.env` with secrets):

```env
# Server
PORT=5000
NODE_ENV=development

# MongoDB
MONGO_URI=mongodb+srv://<username>:<password>@cluster0.mongodb.net/myDatabase?retryWrites=true&w=majority

# Cloudinary
CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret

# Other
JWT_SECRET=ReplaceMeWithASecret
```

Add `.env` to `.gitignore`:

```
node_modules
.env
.DS_Store
/uploads
```

---

## Step 4 — Config files

### `src/config/db.js`

```js
// src/config/db.js
const mongoose = require('mongoose');
const logger = require('../utils/logger');

const connectDB = async (mongoURI) => {
  try {
    await mongoose.connect(mongoURI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    });
    logger.info('MongoDB connected');
  } catch (err) {
    logger.error('MongoDB connection error:', err);
    // Rethrow so caller can decide to exit
    throw err;
  }
};

module.exports = connectDB;
```

### `src/config/cloudinary.js`

```js
// src/config/cloudinary.js
const cloudinary = require('cloudinary').v2;

const configureCloudinary = ({ cloud_name, api_key, api_secret }) => {
  cloudinary.config({
    cloud_name,
    api_key,
    api_secret,
  });

  return cloudinary;
};

module.exports = configureCloudinary;
```

### `src/config/multer.js`

We'll configure Multer + `multer-storage-cloudinary` to directly upload to Cloudinary.

```js
// src/config/multer.js
const multer = require('multer');
const { CloudinaryStorage } = require('multer-storage-cloudinary');
const cloudinary = require('cloudinary').v2;

const getMulterUpload = () => {
  const storage = new CloudinaryStorage({
    cloudinary: cloudinary,
    params: {
      folder: 'mern_app_uploads',
      allowed_formats: ['jpg', 'jpeg', 'png', 'webp', 'gif'],
      transformation: [{ width: 1500, crop: 'limit' }],
    },
  });

  const parser = multer({
    storage,
    limits: { fileSize: 5 * 1024 * 1024 }, // 5MB
  });

  return parser;
};

module.exports = getMulterUpload;
```

> If you prefer streaming uploads (more control), you can use multer memoryStorage and `cloudinary.uploader.upload_stream()` in a service.

---

## Step 5 — Utilities: logger

### `src/utils/logger.js`

```js
// src/utils/logger.js
const { createLogger, format, transports } = require('winston');

const logger = createLogger({
  level: process.env.NODE_ENV === 'production' ? 'info' : 'debug',
  format: format.combine(
    format.timestamp({ format: 'YYYY-MM-DD HH:mm:ss' }),
    format.errors({ stack: true }),
    format.splat(),
    format.json()
  ),
  transports: [
    new transports.Console({
      format: format.combine(format.colorize(), format.simple()),
    }),
  ],
});

module.exports = logger;
```

---

## Step 6 — Async handler & middlewares

### `src/middlewares/asyncHandler.js`

A tiny helper to avoid repeating `try/catch` in controllers.

```js
// src/middlewares/asyncHandler.js
module.exports = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};
```

### `src/middlewares/notFound.js`

```js
// src/middlewares/notFound.js
module.exports = (req, res, next) => {
  res.status(404).json({
    success: false,
    message: `Not Found - ${req.originalUrl}`,
  });
};
```

### `src/middlewares/errorHandler.js`

A robust global error handler.

```js
// src/middlewares/errorHandler.js
const logger = require('../utils/logger');

module.exports = (err, req, res, next) => {
  // If headers already sent, delegate to default Express handler
  if (res.headersSent) {
    return next(err);
  }

  const statusCode = err.statusCode || 500;
  const response = {
    success: false,
    message: err.message || 'Internal Server Error',
  };

  // Provide detailed error stack in non-production
  if (process.env.NODE_ENV !== 'production') {
    response.stack = err.stack;
    response.meta = err.meta || null;
  }

  logger.error(err);

  res.status(statusCode).json(response);
};
```

---

## Step 7 — Security middlewares and core app

### `src/app.js`

```js
// src/app.js
const express = require('express');
const helmet = require('helmet');
const compression = require('compression');
const rateLimit = require('express-rate-limit');
const xss = require('xss-clean');
const hpp = require('hpp');
const cors = require('cors');
const morgan = require('morgan');
const cookieParser = require('cookie-parser');

const notFound = require('./middlewares/notFound');
const errorHandler = require('./middlewares/errorHandler');
const logger = require('./utils/logger');

const router = require('./routes'); // index route aggregator

const createApp = () => {
  const app = express();

  // Basic middlewares
  app.use(helmet());
  app.use(compression());
  app.use(express.json({ limit: '10kb' }));
  app.use(express.urlencoded({ extended: true }));
  app.use(cookieParser());
  app.use(xss());
  app.use(hpp());
  app.use(cors());

  // Logging
  if (process.env.NODE_ENV === 'development') {
    app.use(morgan('dev'));
  }

  // Rate limiter
  const limiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100, // limit each IP to 100 requests per windowMs
  });
  app.use('/api', limiter);

  // Routes
  app.use('/api', router);

  // 404
  app.use(notFound);

  // Global error handler
  app.use(errorHandler);

  return app;
};

module.exports = createApp;
```

---

## Step 8 — Route & controller examples

### `src/routes/index.js`

```js
// src/routes/index.js
const express = require('express');
const router = express.Router();
const { healthCheck } = require('../controllers/health.controller');

router.get('/health', healthCheck);

module.exports = router;
```

### `src/controllers/health.controller.js`

```js
// src/controllers/health.controller.js
exports.healthCheck = (req, res) => {
  res.json({ success: true, uptime: process.uptime(), time: new Date().toISOString() });
};
```

---

## Step 9 — Server entry: connect to DB, then start server

### `src/server.js`

```js
// src/server.js
require('dotenv').config();
const http = require('http');
const createApp = require('./app');
const connectDB = require('./config/db');
const configureCloudinary = require('./config/cloudinary');
const getMulterUpload = require('./config/multer');
const logger = require('./utils/logger');

const PORT = process.env.PORT || 5000;

const startServer = async () => {
  try {
    // 1. Configure Cloudinary early (so multer storage has config)
    if (process.env.CLOUDINARY_CLOUD_NAME) {
      configureCloudinary({
        cloud_name: process.env.CLOUDINARY_CLOUD_NAME,
        api_key: process.env.CLOUDINARY_API_KEY,
        api_secret: process.env.CLOUDINARY_API_SECRET,
      });
    }

    // 2. Connect to MongoDB
    await connectDB(process.env.MONGO_URI);

    // 3. Create express app
    const app = createApp();

    // 4. Create HTTP server
    const server = http.createServer(app);

    // 5. Start listening
    server.listen(PORT, () => {
      logger.info(`Server running in ${process.env.NODE_ENV} mode on port ${PORT}`);
    });

    // Graceful shutdown (example)
    process.on('unhandledRejection', (reason, p) => {
      logger.error('Unhandled Rejection at: Promise', p, 'reason:', reason);
    });

    process.on('uncaughtException', (err) => {
      logger.error('Uncaught Exception! Shutting down...');
      logger.error(err);
      process.exit(1);
    });
  } catch (err) {
    logger.error('Failed to start server:', err);
    process.exit(1);
  }
};

startServer();
```

---

## Step 10 — Additional production notes

* Use a process manager like `pm2` or run behind Docker + Kubernetes in production.
* Add rate-limiting, request logging to file, and central logging (ELK or a hosted log provider) for production.
* Use HTTPS with a reverse proxy (Nginx or cloud provider load balancer).
* Store secrets in a secret manager (AWS Secrets Manager, Google Secret Manager, or environment variables in CI/CD) — do not commit `.env`.
* Add tests (Jest + Supertest) for endpoints.

---

## Package.json example scripts

```json
{
  "scripts": {
    "dev": "cross-env NODE_ENV=development nodemon src/server.js",
    "start": "cross-env NODE_ENV=production node src/server.js",
    "lint": "eslint . --ext .js",
    "format": "prettier --write ."
  }
}
```

---

## GitHub README Tips

* Keep `.env.example` and instructions for setting required variables.
* Document how to run locally, how to run tests, and how to deploy (Docker/Heroku/Render/etc).
* Include examples of how to upload a file to the upload endpoint (when you add endpoints later).

---

## Final checklist (before commit)

* [ ] Remove any debug credentials from code
* [ ] Add `.env` to `.gitignore`
* [ ] Run `npm run lint` and `npm run format`
* [ ] Add Dockerfile and `docker-compose.yml` if needed
* [ ] Set up CI/CD (GitHub Actions example included if desired)

---

If you want, I can also:

* Generate a ready-to-commit repository (zipped) containing these files pre-filled.
* Add Dockerfile + docker-compose + GitHub Actions workflow.
* Add ESLint + Prettier config files.

Tell me which one you'd like next.
