# ScamShield Monitor

ScamShield Monitor is a lightweight cybersecurity dashboard built to surface scam activity, monitor suspicious patterns, and help users understand common scam behaviors through a structured intelligence interface.

## Project overview

This project is a demo-style monitoring app for scam intelligence. It brings together a set of curated scam feeds, category views, and knowledge entries so users can quickly understand the kinds of threats being reported and the patterns behind them.

The active experience focuses on:

- Live scam feed for recent activity and emerging reports
- Explore Scams section for reviewing scam types and campaign patterns
- Scam Encyclopedia for reference material on common scam tactics
- About & Data Sources page for context on how the mock data is structured

This is a front-end prototype intended for product demonstration and research workflows rather than a production security operations system.

## Tech stack

- Next.js App Router
- React + TypeScript
- Tailwind CSS
- Recharts for dashboard-style visualizations
- Lucide React icons

## Project structure

- `app/` — application routes and pages
- `components/` — reusable UI sections and dashboard layout
- `lib/` — mock intelligence data and helper utilities
- `public/` — static assets

## Getting started

Install dependencies:

```bash
npm install
```

Start the local development server:

```bash
npm run dev
```

Then open http://localhost:3000 in your browser.

## Notes

The dashboard currently uses mock data to demonstrate scam monitoring workflows. The interface is intentionally simplified for demonstration and does not represent a live production intelligence pipeline.

Note: This will be used as the database for scam search agent:
https://flexi-alpha.vercel.app/
