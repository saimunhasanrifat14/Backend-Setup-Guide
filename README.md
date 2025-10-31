# MERN Backend Boilerplate (Updated Config Version)

A production-ready backend setup for MERN projects using Node.js, Express, MongoDB, Cloudinary, and Multer. This version uses your provided configuration pattern (with custom error handling, async handler, and API response classes).

---

## Step 1 — Project Setup

```bash
mkdir mern-backend && cd mern-backend
npm init -y

npm install express mongoose dotenv cors cookie-parser multer cloudinary bcryptjs jsonwebtoken

# Dev dependencies
npm install -D nodemon cross-env
```

---

## Step 2 — Folder Structure

```bash
mkdir -p src/{config,controllers,models,routes,middlewares,utilities,database,constants,uploads}

# Entry files
touch src/app.js src/server.js

# Config files
touch src/config/cloudinary.js

# Database file
touch src/database/db.js

# Utilities
touch src/utilities/{CustomError.js,GlobalErrorHandler.js,AsyncHandler.js,APIResponse.js}

# Example route/controller
touch src/routes/index.js src/controllers/health.controller.js

# Env files
touch .env .env.example .gitignore
```

---

## Step 3 — `.env.example`

```env
PORT=3000
BASE_URL=/api/v1
NODE_ENV=development
MONGODB_URL=mongodb+srv://<username>:<password>@cluster.mongodb.net
CLOUD_NAME=your_cloud_name
CLOUD_API_KEY=your_cloud_key
CLOUD_API_SECRECT=your_cloud_secret
dbName=mern_database
```

---

## Step 4 — `src/app.js`

```js
const express = require("express");
const cors = require("cors");
const cookieParser = require("cookie-parser");
const { GlobalErrorHandler } = require("./utilities/GlobalErrorHandler");

const app = express();
const apiVersion = process.env.BASE_URL || "/api/v1";

// Middleware
app.use(cors());
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(cookieParser());
app.use(express.static("public"));

// Routes
app.use(apiVersion, require("./routes/index"));

// Global Error Handler
app.use(GlobalErrorHandler);

module.exports = app;
```

---

## Step 5 — `src/server.js`

```js
require("dotenv").config();
const { connectDB } = require("./database/db");
const app = require("./app");

const port = process.env.PORT || 3000;

connectDB()
  .then(() => {
    app.listen(port, () => {
      console.log(`Server running on http://localhost:${port}`);
    });
  })
  .catch((err) => {
    console.log("DB Connection Failed", err);
  });
```

---

## Step 6 — Database Connection (`src/database/db.js`)

```js
require("dotenv").config();
const mongoose = require("mongoose");
const { dbName } = require("../constants/constant");

const connectDB = async () => {
  try {
    const connectionInstance = await mongoose.connect(
      `${process.env.MONGODB_URL}/${dbName}`
    );
    console.log(`MongoDB Connected: ${connectionInstance.connection.host}`);
  } catch (error) {
    console.log("MongoDB connection FAILED", error);
    process.exit(1);
  }
};

module.exports = { connectDB };
```

---

## Step 7 — Cloudinary Config (`src/config/cloudinary.js`)

```js
require("dotenv").config();
const cloudinary = require("cloudinary").v2;
const { CustomError } = require("../utilities/CustomError");
const fs = require("fs");

cloudinary.config({
  cloud_name: process.env.CLOUD_NAME,
  api_key: process.env.CLOUD_API_KEY,
  api_secret: process.env.CLOUD_API_SECRECT,
});

// Upload Image
exports.uploadImage = async (filePath) => {
  try {
    if (!filePath && !fs.existsSync(filePath)) {
      throw new CustomError(400, "File path is required");
    }

    const uploadedImage = await cloudinary.uploader.upload(filePath, {
      resource_type: "image",
      quality: "auto",
    });

    if (uploadedImage) {
      fs.unlinkSync(filePath);
      return { public_id: uploadedImage.public_id, url: uploadedImage.url };
    }
  } catch (error) {
    if (fs.existsSync(filePath)) fs.unlinkSync(filePath);
    console.log("Error from Cloudinary upload", error);
    throw new CustomError(500, "Cloudinary upload failed: " + error.message);
  }
};

// Delete Image
exports.deleteImage = async (public_id) => {
  try {
    if (!public_id) throw new CustomError(400, "Public id is required");
    const deleted = await cloudinary.uploader.destroy(public_id);
    if (deleted) return true;
  } catch (error) {
    console.log("Error deleting image:", error.message);
    throw new CustomError(500, "Failed to delete image: " + error.message);
  }
};
```

---

## Step 8 — Utilities

### `src/utilities/CustomError.js`

```js
class CustomError extends Error {
  constructor(statusCode, message = "Something went wrong", errors = [], stack = "") {
    super(message);
    this.statusCode = statusCode;
    this.data = null;
    this.success = false;
    this.errors = errors;

    if (stack) {
      this.stack = stack;
    } else {
      Error.captureStackTrace(this, this.constructor);
    }
  }
}

module.exports = { CustomError };
```

### `src/utilities/AsyncHandler.js`

```js
exports.AsyncHandler = (func) => {
  return async (req, res, next) => {
    try {
      await func(req, res);
    } catch (error) {
      next(error);
    }
  };
};
```

### `src/utilities/APIResponse.js`

```js
class APIResponse {
  constructor(statusCode, message = "Success", data) {
    this.statusCode = statusCode;
    this.message = message;
    this.data = data;
    this.success = statusCode < 400;
  }

  static success(res, statusCode, message, data) {
    return res.status(statusCode).json(new APIResponse(statusCode, message, data));
  }
}

module.exports = { APIResponse };
```

### `src/utilities/GlobalErrorHandler.js`

```js
require("dotenv").config();

const developmentResponse = (error, res) => {
  const statusCode = error.statusCode || 500;
  return res.status(statusCode).json({
    statusCode: error.statusCode,
    message: error.message,
    data: error.data,
    errorStack: error.stack,
  });
};

const productionResponse = (error, res) => {
  const statusCode = error.statusCode || 500;
  if (error.isOperationalError) {
    return res.status(statusCode).json({
      statusCode: error.statusCode,
      message: error.message,
    });
  } else {
    return res.status(statusCode).json({
      status: "!OK",
      message: "Something went wrong, please try again later!",
    });
  }
};

exports.GlobalErrorHandler = (error, req, res, next) => {
  if (process.env.NODE_ENV === "development") {
    developmentResponse(error, res);
  } else {
    productionResponse(error, res);
  }
};
```

---

## Step 9 — Example Route and Controller

### `src/routes/index.js`

```js
const express = require("express");
const router = express.Router();
const { healthCheck } = require("../controllers/health.controller");

router.get("/health", healthCheck);

module.exports = router;
```

### `src/controllers/health.controller.js`

```js
const { APIResponse } = require("../utilities/APIResponse");

exports.healthCheck = (req, res) => {
  APIResponse.success(res, 200, "Server is running", {
    uptime: process.uptime(),
    time: new Date().toISOString(),
  });
};
```

---

## Step 10 — Run the server

```bash
npm run dev
```

### Example script in `package.json`

```json
"scripts": {
  "dev": "nodemon src/server.js",
  "start": "node src/server.js"
}
```

---

✅ Now you have a **production-level MERN backend setup** with a clean modular structure, async handling, global error management, and Cloudinary upload utilities ready for future use.
