#!/usr/bin/env python3

from __future__ import annotations

import json
import re
import sys
from datetime import datetime, timezone
from pathlib import Path

import requests
from bs4 import BeautifulSoup


SOURCE_URL = "https://www.lcpdfr.com/downloads/gta5mods/g17media/7792-lspd-first-response/"
OUTPUT_FILE = Path("sources/lspdfr.json")

HEADERS = {
    "User-Agent": (
        "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
        "AppleWebKit/537.36 (KHTML, like Gecko) "
        "Chrome/151.0.0.0 Safari/537.36"
    ),
    "Accept": (
        "text/html,application/xhtml+xml,application/xml;q=0.9,"
        "image/avif,image/webp,*/*;q=0.8"
    ),
    "Accept-Language": "en-US,en;q=0.9,nl;q=0.8",
    "Cache-Control": "no-cache",
    "Pragma": "no-cache",
    "Referer": "https://www.lcpdfr.com/lspdfr/",
}

TITLE_PATTERNS = [
    re.compile(
        r"LSPD\s+First\s+Response\s+"
        r"(?P<version>[0-9]+(?:\.[0-9]+)+)\s*"
        r"\(Build\s+(?P<build>[0-9]+)\)\s*-\s*"
        r"GTA(?:\s+Build)?\s+(?P<gta>[0-9]+(?:\.[0-9]+)*)",
        re.IGNORECASE,
    ),
    re.compile(
        r"What[’']?s\s+New\s+in\s+Version\s+"
        r"(?P<version>[0-9]+(?:\.[0-9]+)+)\s*"
        r"\(Build\s+(?P<build>[0-9]+)\)\s*-\s*"
        r"GTA(?:\s+Build)?\s+(?P<gta>[0-9]+(?:\.[0-9]+)*)",
        re.IGNORECASE,
    ),
]


def utc_now() -> str:
    return datetime.now(timezone.utc).isoformat(timespec="seconds").replace("+00:00", "Z")


def load_existing() -> dict:
    if not OUTPUT_FILE.exists():
        return {}

    try:
        return json.loads(OUTPUT_FILE.read_text(encoding="utf-8"))
    except Exception:
        return {}


def normalize_space(value: str) -> str:
    return re.sub(r"\s+", " ", value or "").strip()


def extract_release(html: str) -> tuple[str, str, str]:
    soup = BeautifulSoup(html, "html.parser")

    candidates: list[str] = []

    og_title = soup.find("meta", attrs={"property": "og:title"})
    if og_title and og_title.get("content"):
        candidates.append(normalize_space(str(og_title["content"])))

    if soup.title and soup.title.string:
        candidates.append(normalize_space(soup.title.string))

    candidates.append(normalize_space(soup.get_text(" ", strip=True)))

    for candidate in candidates:
        for pattern in TITLE_PATTERNS:
            match = pattern.search(candidate)
            if match:
                return (
                    match.group("version"),
                    match.group("build"),
                    match.group("gta"),
                )

    raise RuntimeError(
        "LCPDFR reageerde, maar OPSNET kon versie/build/GTA-build niet betrouwbaar herkennen."
    )


def main() -> int:
    existing = load_existing()
    checked_at = utc_now()

    try:
        response = requests.get(
            SOURCE_URL,
            headers=HEADERS,
            timeout=20,
            allow_redirects=True,
        )

        response.raise_for_status()

        version, build, gta_build = extract_release(response.text)

        previous_signature = (
            str(existing.get("version", "")),
            str(existing.get("build", "")),
            str(existing.get("gta_build", "")),
        )
        current_signature = (version, build, gta_build)

        source_changed_at = existing.get("source_changed_at")
        if previous_signature != current_signature or not source_changed_at:
            source_changed_at = checked_at

        data = {
            "source": "lspdfr",
            "provider": "LCPDFR",
            "version": version,
            "build": build,
            "gta_build": gta_build,
            "source_url": SOURCE_URL,
            "checked_at": checked_at,
            "last_success_at": checked_at,
            "source_changed_at": source_changed_at,
            "status": "online",
        }

        OUTPUT_FILE.parent.mkdir(parents=True, exist_ok=True)
        OUTPUT_FILE.write_text(
            json.dumps(data, indent=2, ensure_ascii=False) + "\n",
            encoding="utf-8",
        )

        print("OPSNET OK: LSPDFR source succesvol uitgelezen.")
        print(f"Version: {version}")
        print(f"Build: {build}")
        print(f"GTA build: {gta_build}")
        return 0

    except Exception as exc:
        # Heartbeat/status blijven zichtbaar, maar we behouden de laatst bekende
        # succesvolle release-informatie wanneer die al bestaat.
        data = {
            "source": "lspdfr",
            "provider": "LCPDFR",
            "version": existing.get("version", ""),
            "build": existing.get("build", ""),
            "gta_build": existing.get("gta_build", ""),
            "source_url": SOURCE_URL,
            "checked_at": checked_at,
            "last_success_at": existing.get("last_success_at"),
            "source_changed_at": existing.get("source_changed_at"),
            "status": "degraded",
            "error": str(exc)[:500],
        }

        OUTPUT_FILE.parent.mkdir(parents=True, exist_ok=True)
        OUTPUT_FILE.write_text(
            json.dumps(data, indent=2, ensure_ascii=False) + "\n",
            encoding="utf-8",
        )

        print(f"OPSNET ERROR: {exc}", file=sys.stderr)
        return 1


if __name__ == "__main__":
    raise SystemExit(main())
