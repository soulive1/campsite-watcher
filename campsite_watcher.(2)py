"""
Campsite availability watcher for Cedar Point State Park (Clayton, NY).
Checks a ReserveAmerica search page every few minutes and texts you
when sites show up. It does NOT book anything; you do that yourself.

SETUP
  pip install playwright
  playwright install chromium

  Set these environment variables:
    EMAIL_TO         (the address that should get the alert)
    EMAIL_FROM       (a Gmail address used to send)
    EMAIL_APP_PASS   (Gmail app password: Google Account > Security >
                      2-Step Verification > App passwords)
    SEARCH_URL       (see step below)

GET SEARCH_URL
  1. On newyorkstateparks.reserveamerica.com, search Cedar Point State Park,
     arrival July 6, 11 nights, then set the filters: 30 amp electric,
     water, sewer, and equipment = trailer, length 30 ft.
  2. Copy the full URL of the results page into SEARCH_URL.

RUN
  python campsite_watcher.py            # watch forever
  python campsite_watcher.py --debug    # one check, saves page.png/page.html
                                        # so you can confirm it reads the page right
"""
import os
import sys
import time
import random
import smtplib
from email.message import EmailMessage
from playwright.sync_api import sync_playwright

CHECK_EVERY_SECONDS = 300  # 5 minutes; don't go much faster
NONE_PHRASES = [
    "no sites available",
    "no results",
    "0 results",
    "0 sites",
    "no availability",
]


def send_email(body):
    msg = EmailMessage()
    msg["Subject"] = "Campsite alert: Cedar Point SP, July 6-17"
    msg["From"] = os.environ["EMAIL_FROM"]
    msg["To"] = os.environ["EMAIL_TO"]
    msg.set_content(body)
    with smtplib.SMTP_SSL("smtp.gmail.com", 465) as s:
        s.login(os.environ["EMAIL_FROM"], os.environ["EMAIL_APP_PASS"])
        s.send_message(msg)


def check_once(page, url, debug=False):
    page.goto(url, wait_until="networkidle", timeout=60000)
    page.wait_for_timeout(3000)
    text = page.inner_text("body").lower()
    if debug:
        page.screenshot(path="page.png", full_page=True)
        open("page.html", "w", encoding="utf-8").write(page.content())
    if any(p in text for p in NONE_PHRASES):
        return False
    # Heuristic: results pages list sites with a "book"/"reserve" style link.
    return "available" in text and ("book" in text or "reserve" in text)


def main():
    url = os.environ["SEARCH_URL"]
    debug = "--debug" in sys.argv
    once = "--once" in sys.argv  # single check then exit (used by GitHub Actions)
    alerted = False
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        page = browser.new_page()
        while True:
            try:
                found = check_once(page, url, debug)
                print(time.strftime("%H:%M:%S"), "available" if found else "none")
                if found and not alerted:
                    send_email(
                        "Cedar Point SP: a site matching your filters may be "
                        f"open for July 6-17! Book now: {url}"
                    )
                    alerted = True
                elif not found:
                    alerted = False  # re-arm so you get texted again next time
            except Exception as e:
                print("check failed:", e)
            if debug or once:
                break
            time.sleep(CHECK_EVERY_SECONDS + random.randint(0, 60))
        browser.close()


if __name__ == "__main__":
    main()
