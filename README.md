## Fast_Source_Code
Backend service ringkas berbasis Express.js dengan arsitektur modular, dirancang untuk deployment serverless di Vercel maupun mode development lokal. Struktur project memisahkan concern secara jelas: server.js menangani inisialisasi aplikasi, session management, dan static file serving, sementara db.js mengelola koneksi PostgreSQL melalui connection pooling menggunakan library pg. Kompatibel dengan provider database seperti Neon dengan konfigurasi SSL otomatis. Sistem menerapkan session based authentication dengan cookie httpOnly untuk keamanan, CORS policy yang dapat dikonfigurasi, serta pemisahan mode production dan development yang eksplisit. Struktur folder mengikuti best practice separation of concern antara API layer, database layer, dan static assets, cocok untuk aplikasi web skala menengah yang membutuhkan performa stabil dan maintainability tinggi.

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

## source code 
```server.js 
import express from "express";
import cors from "cors";
import session from "express-session";
import dotenv from "dotenv";
import path from "path";
import { fileURLToPath } from "url";
import { cekKoneksiDatabase } from "./db.js";
import apiRouter from "./api.js";

dotenv.config();

const __dirname = path.dirname(fileURLToPath(import.meta.url));
const app = express();
const PORT = process.env.PORT;

app.use(cors({ origin: process.env.CORS_ORIGIN, credentials: true }));
app.use(express.json({ limit: "5mb" }));
app.use(
    session({
        secret: process.env.SESSION_SECRET,
        resave: false,
        saveUninitialized: false,
        cookie: {
            httpOnly: true,
            maxAge: 1000 * 60 * 60 * 8,
        },
    })
);

app.use("/api", apiRouter);

app.use(express.static(path.join(__dirname, "..", "public")));

app.get("/", (req, res) => {
    res.redirect("/app");
});

app.get("/app", (req, res) => {
    res.sendFile(path.join(__dirname, "..", "public", "index.html"));
});

app.get("/app/health", (req, res) => {
    res.json({ status: "ok" });
});

const MODE = process.env.NODE_ENV === "production" ? "production" : "development";

async function mulaiServer() {
    await cekKoneksiDatabase();
    console.log("Koneksi database berhasil");

    if (MODE === "development") {
        app.listen(PORT, () => {
            console.log(`Server berjalan di port ${PORT} (development)`);
        });
    } else {
        console.log("Server berjalan dalam mode production (serverless)");
    }
}

mulaiServer().catch((error) => {
    console.error("Gagal koneksi ke database:", error.message);
    if (MODE === "development") process.exit(1);
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
    ssl: process.env.DATABASE_URL?.includes("neon.tech")
        ? { rejectUnauthorized: false }
        : false,
});

export async function cekKoneksiDatabase() {
    const client = await pool.connect();
    try {
        await client.query("SELECT 1");
    } finally {
        client.release();
    }
}

export default pool;
```