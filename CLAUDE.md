# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a documentation repository for an **event-driven fiscal document processing system** (Brazilian SEFAZ invoices). It currently contains architecture documentation only — no runnable code.

## System Architecture Overview

The system processes fiscal documents (images, PDFs, XMLs) through an event-driven pipeline:

**Ingestion Layer (Vercel)**
- **Next.js frontend**: Uploads files directly to Supabase Object Storage, polls/SSE for status
- **Hono API**: Receives file URLs, runs the Classifier, publishes tasks to Redis
- **Classifier** (module inside Hono): Magic byte + MIME type inspection → tags each file as `img`, `xml`, or `pdf`

**Message Broker**
- **AWS ElastiCache (Redis)**: Task queue + Dead Letter Queue; pure infrastructure, no business logic

**Worker Fleet (AWS ECS — Celery)**
Three processing branches, all writing results to Supabase PostgreSQL:
- **Image branch**: Download → decode → resize to 400KB → contrast enhancement (thermal paper) → Gemini API for structured extraction
- **XML branch**: Download → lxml XSD validation against SEFAZ schema → XPath extraction (no AI). Validation failures write immediately to DB as failed, no retry.
- **PDF branch**: Download → pdfplumber text extraction → if above confidence threshold, extract directly; if below, convert pages to image frames via pdf2image and route into the image branch

**Dead Letter Queue Fleet (AWS ECS — Celery)**
Separate worker monitoring DLQ. Updates document status to failed with reason, holds tasks for manual review — no auto-reprocess.

**External Services**
- **Gemini API**: Structured JSON extraction from preprocessed images; rate limit failures are the primary retry/DLQ trigger
- **Supabase Object Storage** (S3-compatible): Raw file storage by batch + document ID
- **Supabase PostgreSQL**: System of record — batch metadata, document status, extracted data, error reasons

## Architecture Documentation

- `01. Architecture/event-driven.architecture.md` — Component descriptions and data flow
- `01. Architecture/architecture diagram.drawio` — Editable diagram source
- `01. Architecture/architecture.drawio.png` — Rendered architecture diagram
