### Fast Source Code
Backend service ringkas berbasis Express.js dengan arsitektur modular, dirancang untuk deployment serverless di Vercel maupun mode development lokal. Struktur project memisahkan concern secara jelas: server.js menangani inisialisasi aplikasi, session management, dan static file serving, sementara db.js mengelola koneksi PostgreSQL melalui connection pooling menggunakan library pg. Kompatibel dengan provider database seperti Neon dengan konfigurasi SSL otomatis. Sistem menerapkan session based authentication dengan cookie httpOnly untuk keamanan, CORS policy yang dapat dikonfigurasi, serta pemisahan mode production dan development yang eksplisit. Struktur folder mengikuti best practice separation of concern antara API layer, database layer, dan static assets, cocok untuk aplikasi web skala menengah yang membutuhkan performa stabil dan maintainability tinggi.

### Documentation
Folder `src/` berisi seluruh logika sisi server: `server.js` sebagai titik masuk yang menginisialisasi Express, session, middleware, dan static file serving; `db.js` mengelola koneksi PostgreSQL melalui connection pool; `api.js` mendefinisikan route endpoint data; `auth.js` menangani registrasi, login, logout, dan verifikasi session; `utils.js` menyimpan fungsi bantu yang dipakai lintas modul; serta `schema_db/` untuk definisi skema dan migrasi tabel. Folder `public/` berisi seluruh aset sisi klien yang disajikan langsung ke browser: `index.html` sebagai halaman utama, `auth.html` sebagai halaman masuk dan daftar, `pages/` untuk halaman tambahan, `components/` untuk potongan UI yang dapat dipakai ulang, dan `assets/` untuk gambar, font, stylesheet, serta script klien.

```
project/
├── src/
│   ├── schema_db/
│   ├── server.js
│   ├── db.js
│   ├── api.js
│   ├── auth.js
│   └── utils.js
│
├── public/
│   ├── assets/
│   ├── components/
│   ├── pages/
│   ├── auth.html
│   └── index.html
│
├── .env.example
├── .gitignore
├── vercel.json
├── package.json
├── README.md
└── LICENSE
```

### Additional Modules
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
  "name": "application",
  "version": "1.0.0",
  "description": "Express backend server with PostgreSQL and session-based authentication",
  "type": "module",
  "main": "src/server.js",
  "private": true,
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js",
    "audit:cek": "npm audit --omit=dev"
  },
  "dependencies": {
    "bcryptjs": "^3.0.3",
    "compression": "^1.7.5",
    "connect-pg-simple": "^10.0.0",
    "cors": "^2.8.5",
    "dotenv": "^16.4.5",
    "express": "^4.21.1",
    "express-rate-limit": "^7.4.1",
    "express-session": "^1.18.1",
    "helmet": "^8.0.0",
    "morgan": "^1.10.0",
    "nanoid": "^5.0.9",
    "pg": "^8.13.1",
    "zod": "^3.24.1"
  },
  "devDependencies": {
    "nodemon": "^3.1.14"
  },
  "engines": {
    "node": ">=20.0.0"
  },
  "keywords": [
    "express",
    "postgresql",
    "session-auth",
    "backend"
  ]
}
```

### SOURCE CODE PSQL :
```server.js
import express from "express";
import { consola } from "consola";
import ora from "ora";
import boxen from "boxen";
import Table from "cli-table3";
import apiRouter from "./api.js";
import authRouter from "./auth.js";
import pool, { cekKoneksiDatabase } from "./db.js";
import { isProduction, port, publicFile, setupMiddleware, setupRoutes, setupErrorHandlers } from "./utils.js";

const app = express();
setupMiddleware(app, pool);
setupRoutes(app, { authRouter, apiRouter });

app.get("/", (req, res) => res.redirect("/app"));
app.get("/app", (req, res) => res.sendFile(publicFile("index.html")));
app.get("/app/health", (req, res) => res.json({ status: "ok" }));

setupErrorHandlers(app);

function showBanner() {
    const baseUrl = `http://localhost:${port}`;

    const table = new Table({
        head: ["Informasi", "Nilai"],
        style: { head: ["cyan"], border: ["gray"] },
    });

    table.push(
        ["Mode", process.env.NODE_ENV],
        ["Port", String(port)],
        ["URL", baseUrl],
        ["Health", `${baseUrl}/app/health`],
        ["Node", process.version]
    );

    console.log(
        boxen(table.toString(), {
            title: "Server Aktif",
            titleAlignment: "center",
            padding: 1,
            borderStyle: "round",
            borderColor: "cyan",
        })
    );
}

async function startServer() {
    const spinner = isProduction ? null : ora("Menghubungkan ke database").start();

    try {
        await cekKoneksiDatabase();
    } catch (error) {
        spinner?.fail("Koneksi database gagal");
        throw error;
    }

    if (spinner) spinner.succeed("Koneksi database berhasil");
    else consola.success("Koneksi database berhasil");

    if (isProduction) {
        consola.info("Server berjalan dalam mode production");
        return;
    }

    app.listen(port, showBanner);
}

startServer().catch((error) => {
    consola.error("Gagal memulai server:", error.message);
    if (!isProduction) process.exit(1);
});

export default app;
```
```db.js
import pg from "pg";
import dotenv from "dotenv";

dotenv.config();
const { Pool } = pg;

const pool = new Pool({
    connectionString: process.env.DATABASE_URL,
    max: Number(process.env.DATABASE_POOL_MAX),
    ssl: process.env.DATABASE_SSL === "true" ? { rejectUnauthorized: false } : false,
});

pool.on("error", (error) => {
    console.error("Kesalahan pada pool database:", error.message);
});

export async function cekKoneksiDatabase() {
    const client = await pool.connect();
    try {
        await client.query("SELECT 1");
    } finally {
        client.release();
    }
}

export async function query(text, params = []) {
    const result = await pool.query(text, params);
    return result.rows;
}

export async function queryOne(text, params = []) {
    const rows = await query(text, params);
    return rows[0] ?? null;
}

export async function execute(text, params = []) {
    const result = await pool.query(text, params);
    return result.rowCount;
}

export async function transaksi(callback) {
    const client = await pool.connect();
    try {
        await client.query("BEGIN");
        const result = await callback(client);
        await client.query("COMMIT");
        return result;
    } catch (error) {
        await client.query("ROLLBACK");
        throw error;
    } finally {
        client.release();
    }
}

export async function tutupKoneksiDatabase() {
    await pool.end();
}

export default pool;
```