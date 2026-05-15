# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Single-file static HTML site — a Japanese-language comparison page ranking furniture assembly services in Kobe, Japan. There is no build system, package manager, framework, or test suite. The entire site is `index.html`.

## Editing

Open `index.html` directly in a browser to preview. No compilation or server required.

## Architecture

`index.html` is self-contained: all CSS lives in a `<style>` block in `<head>`, all JavaScript is in a `<script>` block before `</body>`. There are no external JS dependencies; the only external resources are Google Fonts (loaded via `<link>`).

**CSS structure** — CSS custom properties (variables) are defined at `:root`. Sections are clearly delimited by `/* ===== SECTION NAME ===== */` comments. Responsive overrides are at the bottom inside `@media(max-width:640px)`.

**Page sections (in DOM order):**
1. Hero (`<header class="hero">`) — headline, maker chip badges
2. Update bar (`.upd-bar`) — last-updated notice
3. Intro (`.intro`) — lead paragraph
4. Ranking (`<main id="ranking">`) — five `.rcard` articles, each with `data-r="1"…"5"` for rank-specific gradient theming
5. Comparison table (`.ct-wrap`) — feature matrix across all five services
6. Tips (`.tips-wrap`) — dark-background tips grid
7. FAQ (`.faq-list`) — accordion items toggled by `classList.toggle('open')` via inline `onclick`
8. Footer + scroll-to-top button (`#scrollTop`)

**Rank card internals** — each `<article class="rcard" data-r="N">` contains `.rcard-head` (rank number, name, badge) and `.rcard-body` (scores, tags, description, optional price table / flow steps / detail grid, pros/cons, CTA link).

## Key details

- Language: Japanese (`lang="ja"`); all user-visible text is in Japanese.
- Canonical URL is set to `https://yourusername.github.io/furniture-assembly-kobe/` — update before deploying.
- Google Site Verification meta tag is present; keep it when editing `<head>`.
- Schema.org `ItemList` structured data in `<head>` mirrors the five ranked services; update it if rankings change.
- The scroll-reveal animation uses `IntersectionObserver`; elements need the `reveal` class to animate in.
