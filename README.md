# 🧠 Project Title

## Blogify: Server-Rendered Blogging Platform

# 📌 Overview

Blogify is a Node.js and Express-based blogging platform where users can create accounts, publish blog posts with cover images, and engage through comments.

It solves the common need for a simple, full-stack content publishing app with authentication, media uploads, and relational data display (author + comments), without requiring a frontend SPA framework.

This project is ideal for:
- Developers learning full-stack JavaScript with Express and MongoDB
- Students building portfolio-ready CRUD/auth projects
- Recruiters and reviewers evaluating backend fundamentals, MVC organization, and server-rendered UI skills

# 🚀 Features

- User signup/signin with secure password hashing and JWT-based session cookies
- Role-aware user model (USER/ADMIN enum in schema)
- Create blog posts with image uploads using Multer
- Homepage feed that lists all published blogs
- Blog detail page with author metadata and threaded comment list
- Comment creation flow tied to authenticated users
- Server-rendered views using EJS partials for reusable layout (head/nav/scripts)
- Static asset delivery for uploaded images and default profile images

# 🛠 Tech Stack

- Language: JavaScript (Node.js)
- Backend Framework: Express 5
- Database: MongoDB with Mongoose ODM
- Templating Engine: EJS
- Authentication: JSON Web Token (jsonwebtoken) + cookie-parser
- File Uploads: Multer (disk storage)
- Dev Tooling: Nodemon

# 📂 Project Structure

- `index.js`: App bootstrap, middleware registration, MongoDB connection, root feed route
- `routes/user.route.js`: Auth pages and signup/signin/logout flows
- `routes/blog.route.js`: Blog creation, blog details, comments, upload handling
- `middlewares/authentication.js`: Cookie token validation and req.user injection
- `services/auth.js`: JWT creation and verification utilities
- `models/user.model.js`: User schema, password hashing hook, login token generation method
- `models/blog.model.js`: Blog schema (title, body, cover image, author ref)
- `models/comment.model.js`: Comment schema (content, blog ref, author ref)
- `views/home.ejs`: Blog feed UI
- `views/blog.ejs`: Blog details + comment form + comment list
- `views/addBlog.ejs`: New blog form (multipart upload)
- `views/signin.ejs`, `views/signup.ejs`: Auth forms
- `views/partials/nav.ejs`: Auth-aware navigation
- `public/uploads`: Stored blog cover images
- `public/images`: Static images (including default profile image)

# ⚙️ Installation & Setup

1. Clone repository and move into the project folder.
2. Install dependencies.
3. Configure environment variables (see Environment Variables section).
4. Start MongoDB locally or provide a MongoDB URI.
5. Run the app.

```bash
npm install
npm run dev
```

Production start:

```bash
npm start
```

Default app URL:
- http://localhost:8000

# 💡 Usage

1. Open the app home page.
2. Create an account from the signup page.
3. Sign in to establish a token cookie session.
4. Click Add Blog to publish a post with a cover image.
5. Open a blog card to view full content and comments.
6. Add comments on blog detail pages while signed in.
7. Use logout from the navbar dropdown to end the session.

# 🧩 Key Implementation Insights

## 1) Password hashing with per-user salt via Mongoose pre-save

The user model uses a pre-save hook to transform plain passwords before writing to MongoDB. This avoids storing plaintext credentials and keeps authentication logic centralized in schema lifecycle hooks.

```js
const { createHmac, randomBytes } = require("node:crypto");

userSchema.pre("save", function (next) {
  const user = this;
  if (!user.isModified("password")) return next();

  const salt = randomBytes(16).toString("hex");
  const hashedPassword = createHmac("sha256", salt)
    .update(user.password)
    .digest("hex");

  user.salt = salt;
  user.password = hashedPassword;
  next();
});
```

## 2) Stateless auth using JWT in HTTP cookies

Signin creates a JWT from a compact user payload and stores it in a cookie. A global middleware validates the cookie token and attaches user data to `req.user` for downstream route/view access.

```js
// services/auth.js
function createTokenForUser(user) {
  return JWT.sign(
    {
      _id: user._id,
      email: user.email,
      profileImageURL: user.profileImageURL,
      role: user.role,
    },
    process.env.JWT_SECRET
  );
}

// middlewares/authentication.js
function checkForAuthenticationCookie(cookieName) {
  return (req, res, next) => {
    const token = req.cookies[cookieName];
    if (!token) return next();
    try {
      req.user = validateToken(token);
    } catch (_) {}
    next();
  };
}
```

## 3) File uploads with deterministic disk storage strategy

Blog images are uploaded with Multer and stored in `public/uploads`, then persisted as a URL path inside each blog document. This keeps render logic simple in EJS.

```js
const storage = multer.diskStorage({
  destination: (req, file, cb) => cb(null, path.resolve("./public/uploads/")),
  filename: (req, file, cb) => cb(null, `${Date.now()}-${file.originalname}`),
});

const upload = multer({ storage });

router.post("/", upload.single("coverImage"), async (req, res) => {
  const blog = await Blog.create({
    title: req.body.title,
    body: req.body.body,
    createdBy: req.user._id,
    coverImageURL: `/uploads/${req.file.filename}`,
  });
  res.redirect(`/blog/${blog._id}`);
});
```

## 4) Relational rendering using Mongoose populate

The blog detail route populates referenced user documents for both the blog author and comment authors in one server-render cycle, enabling rich template rendering without manual joins.

```js
router.get("/:id", async (req, res) => {
  const blog = await Blog.findById(req.params.id).populate("createdBy");
  const comments = await Comment.find({ blogId: req.params.id }).populate("createdBy");

  res.render("blog", { user: req.user, blog, comments });
});
```

Assumption note:
- The current repository appears to contain minor implementation issues (for example, hardcoded JWT secret and some model-level syntax/flow concerns). The snippets above show the intended production-safe pattern and can be used as a guide for cleanup.


Important:
- The current codebase uses hardcoded values for DB URI, port, and JWT secret. Move these to environment variables before production use.


# 📜 License

MIT License

Copyright (c) 2026

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
