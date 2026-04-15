# Uncover

A 5-agent AI suite for SEO, GEO (Generative Engine Optimization), and AI search optimization.

Uncover tracks brand citations across AI answer engines (ChatGPT, Perplexity, Gemini, Claude, Google AI Overviews), simulates query fan-outs, audits LLM-readability, generates optimized content briefs, and mines Google Search Console + GA4 data for actionable insights.

## Status

Design phase. Not yet implemented.

## Design spec

See [docs/specs/2026-04-15-design.md](docs/specs/2026-04-15-design.md).

## The five agents

1. **GEO Citation Tracker** — monitors where your brand is cited across answer engines over time
2. **Query Fan-Out Simulator** — simulates realistic follow-up queries and checks whether your content surfaces
3. **LLM-Readability Auditor** — scores how well an LLM can extract and cite answers from a URL
4. **Content Brief Agent** — produces SEO + GEO-optimized briefs from a keyword
5. **Performance Insights Agent** — pulls GSC + GA4 data and outputs a prioritized action list

## Stack

- **Agents:** Python, LangGraph, Anthropic SDK
- **Frontend:** Next.js 15 (App Router)
- **Database:** Supabase Postgres with RLS
- **Hosting:** Modal (Python) + Vercel (Next.js)
- **Observability:** LangSmith
- **SERP data:** Serper.dev

## Cost target

$0–$5/month across all infrastructure and API spend.
