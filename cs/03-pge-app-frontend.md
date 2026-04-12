# 03 -- PGE App Frontend (CZ)

> Zdrojovy dokument: [../03-pge-app-frontend.md](../03-pge-app-frontend.md) (anglicky, AI-first)
> Posledni aktualizace: 2026-04-11

---

## Shrnuti

- **PGE App** = Vue 3 SPA bezici na app.pgepilot.cz:3060.
- **Technologie**: Vue 3 + TypeScript + Vite + Tailwind (CDN) + Chart.js.
- **9 aktualnich rout**: login, prehled, cp-detail, domeny, grafy, instalace, alerty, uzivatele, nastaveni.
- **Sdilena zakladni UI vrstva s mobilem**: `Prehled`, `Domeny`, `Grafy`, `Instalace`, `Alerty`, detail lokality a uzivatelsky kontext.
- **Aktualni `main`** nema samostatnou routu `predikce` ani hostname-based theming v `main.ts`.

---

> Pro uplne detaily viz anglicka verze: [../03-pge-app-frontend.md](../03-pge-app-frontend.md)
