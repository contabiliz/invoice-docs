# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a monorepo for an **event-driven fiscal document processing system** (Brazilian SEFAZ invoices). It contains the API, processing pipeline, models, infrastructure, and documentation for the system.

## System Architecture Overview

The system processes fiscal documents (images, PDFs, XMLs) through an event-driven pipeline:

**Ingestion Layer**
- **Next.js frontend**: Uploads files directly to Supabase Object Storage, polls/SSE for status.
- **FastAPI (invoice-api)**: Receives file API requests, generates presigned URLs, and acts as the entry point. It publishes tasks to Redis.

**Message Broker**
- **Redis**: Task queue + caching. Used for rate limiting, high water mark control, and pub/sub between the API and pipeline workers.

**Worker Fleet (invoice-pipeline — Celery)**
Three processing branches, all writing results to Supabase PostgreSQL:
- **Image branch**: Download → decode → resize to 400KB → contrast enhancement (thermal paper) → Gemini API for structured extraction.
- **XML branch**: Download → lxml XSD validation against SEFAZ schema → XPath extraction (no AI). Validation failures write immediately to DB as failed, no retry.
- **PDF branch**: Download → pdfplumber text extraction → if above confidence threshold, extract directly; if below, convert pages to image frames via pdf2image and route into the image branch.

**Dead Letter Queue (DLQ)**
Separate worker monitoring DLQ. Updates document status to failed with reason, holds tasks for manual review — no auto-reprocess.

**External Services**
- **Gemini API**: Structured JSON extraction from preprocessed images. Rate limit failures are the primary retry/DLQ trigger.
- **Supabase Object Storage** (S3-compatible): Raw file storage by batch + document ID.
- **Supabase PostgreSQL**: System of record — batch metadata, document status, extracted data, error reasons.

## Repository Structure

- `invoice-api/` — FastAPI application serving as the integration point and frontend backend.
- `invoice-pipeline/` — Celery worker fleet for document ingestion and processing.
- `invoice-processors/` — Core processing logic (parsers, text extractors, image preprocessing).
- `invoice-models/` — Shared Pydantic models, schemas, and schemas used across the entire system.
- `invoice-infra/` — Infrastructure configurations (Docker, Supabase, migrations, scripts).
- `invoice-docs/` — System architecture and design documentation.

## Architecture Documentation

- `01. Architecture/event-driven.architecture.md` — Component descriptions and data flow
- `01. Architecture/architecture diagram.drawio` — Editable diagram source
- `01. Architecture/architecture.drawio.png` — Rendered architecture diagram
