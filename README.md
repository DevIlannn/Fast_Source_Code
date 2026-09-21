## Fast Source Code
Backend service ringkas berbasis Express.js dengan arsitektur modular, dirancang untuk deployment serverless di Vercel maupun mode development lokal. Struktur project memisahkan concern secara jelas: server.js menangani inisialisasi aplikasi, session management, dan static file serving, sementara db.js mengelola koneksi PostgreSQL melalui connection pooling menggunakan library pg. Kompatibel dengan provider database seperti Neon dengan konfigurasi SSL otomatis. Sistem menerapkan session based authentication dengan cookie httpOnly untuk keamanan, CORS policy yang dapat dikonfigurasi, serta pemisahan mode production dan development yang eksplisit. Struktur folder mengikuti best practice separation of concern antara API layer, database layer, dan static assets, cocok untuk aplikasi web skala menengah yang membutuhkan performa stabil dan maintainability tinggi.

## Documentation
```
project/
├── src/
│   ├── engine_1.2.9/
│   ├── server.js
│   ├── db.js
│   ├── api.js
│   ├── auth.js
│   └── utils.js
│
├── public/
│   ├── assets/
│   └── index.html
│
├── .env.example
├── .gitignore
├── vercel.json
├── package.json
├── README.md
└── LICENSE
```

## Additional Modules
```env.example
PORT=3000
CORS_ORIGIN=true
SESSION_SECRET=KEY_SECRET_IN_HERE
NODE_ENV=development
DATABASE_URL=postgresql://neondb_owner:npg_XXXXXXX
```
```.gitignore
node_modules/
.env

dist/
build/
.vercel/

*.log
logs/

.DS_Store
.vscode/
.idea/

coverage/
.cache/
```
```vercel.json
{
    "version": 2,
    "builds": [
        {
            "src": "src/server.js",
            "use": "@vercel/node"
        }
    ],
    "routes": [
        {
            "src": "/(.*)",
            "dest": "src/server.js"
        }
    ]
}
```
```package.json
{
  "name": "aplikasi",
  "version": "1.0.0",
  "description": "Server Backend Aplikasi,
  "type": "module",
  "main": "src/server.js",
  "scripts": {
    "start": "node src/server.js",
    "dev": "node --watch src/server.js"
  },
  "dependencies": {
    "bcryptjs": "^3.0.3",
    "cors": "^2.8.5",
    "crypto": "^1.0.1",
    "dotenv": "^16.4.5",
    "express": "^4.21.1",
    "express-session": "^1.18.1",
    "pg": "^8.13.1"
  },
  "engines": {
    "node": ">=18.0.0"
  },
  "keywords": [],
  "devDependencies": {
    "nodemon": "^3.1.14"
  }
}
