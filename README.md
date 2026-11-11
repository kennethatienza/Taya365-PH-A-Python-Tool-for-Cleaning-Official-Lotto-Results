# Taya365 PH: A Python Tool for Cleaning Official Lotto Results
> Draw tables may look neat, yet mixed dates and odd number marks can spoil later checks. **A short Python script can catch bad rows before they spread through shared lotto files.** It needs **checked PCSO draw records**, since clean text alone cannot prove where numbers came from.

<img width="1536" height="1024" alt="Python’s data comb for lotto results" src="https://github.com/user-attachments/assets/80f88c2c-578c-4e5d-8da0-3c6ba4bde7fc" />
<br>
<br>

The **[Taya365 PH](https://taya365ph.com/)** site has lottery topics, but **it is not the state results page**. Its links can lead to betting pages, so **those pages must not replace state draw data**. This guide helps **Filipino adults aged 21 and above** store draw files and check key fields.

## Start With the PCSO Record Instead of Screenshots

The **Philippine Charity Sweepstakes Office** has a search page for past draws, games, dates, and prizes. Its table shows **drawn balls, cash prizes, and winner counts** as fields that should stay apart. **Copy only rows seen on that page**, not claims shown in a chat or cropped image.

Open the **state draw search page**, then pick the game and dates for the needed rows. **Match each game name and draw date** before pasting a result in the local file. A wrong game name can make **a true number set fail checks** meant for another draw.

**This tool does not scrape pages, use a private feed, or give live draw updates.** A user makes a plain file from rows first checked on the state results page. **The script checks each row's shape**, but it cannot prove the source was truly checked.

## Save Five Clear Columns Before Running Python

Name the input **raw.csv**, then add **game, combination, draw_date, jackpot_php, and winners** as its five headers. Put **one draw on each line**, with no side notes or extra title rows in between. **Keep the same header order**, since the tool checks each name before it reads the rows.

Use **MM/DD/YYYY** for dates, and keep six balls in the source's shown order. Put **cash sums with commas inside quotes**, so one value does not split into two cells. **Keep a zero before short ball values**, since that helps match each line with its source.

```csv
game,combination,draw_date,jackpot_php,winners
Megalotto 6/45,29-27-12-11-43-05,11/09/2026,"15,000,000.00",0
```

## Check Six Number Games Against Their Limits

The tool checks **five PCSO six-ball games**, from **Lotto 6/42 through Ultra Lotto 6/58**. Each row needs **six distinct balls**, with each ball in range for its own named game. **The code stops on a lost, repeated, or high ball** rather than guessing its true value.

For **Lotto 6/42**, any ball above **42** breaks the rules, even if five balls are fine. **The tool keeps each ball in draw order** and pads short numbers with a leading zero. It does not sort the balls, since **that would change the source order seen by readers**.

**2D, 3D, 4D, and 6D draws need their own checks**, so this script leaves them out. Those games can have **the same digit more than once**, which makes this test unfit. **A game outside this tool is not a wrong PCSO result** merely because it gets refused.

## Run a Strict CSV Cleaner With Python

Save the code as **clean_lotto.py** in the same folder as the raw input file. It uses **tools that ship with Python**, so there is no need for added code packs. **A bad row stops the full run** instead of being changed with no clear source proof.

The script checks **header names, draw dates, six balls, cash sums, and winner counts** for faults. It also flags **the same game and date twice**, a sign one row was pasted again. **The script saves output only after all rows pass and blocks accidental source file replacement.**

```python
"""Clean user-copied PCSO six-number results; never fetch or predict draws.
Official source: https://www.pcso.gov.ph/SearchLottoResult.aspx
"""
import csv
import os
import re
import sys
import tempfile
from datetime import datetime
from decimal import Decimal
from pathlib import Path

GAMES = {
    "Lotto 6/42": 42,
    "Megalotto 6/45": 45,
    "Superlotto 6/49": 49,
    "Grand Lotto 6/55": 55,
    "Ultra Lotto 6/58": 58,
}
FIELDS = ["game", "combination", "draw_date", "jackpot_php", "winners"]


def clean(row, line):
    game = row["game"].strip()
    if game not in GAMES:
        raise ValueError(f"Line {line}: unsupported game {game!r}")
    date = datetime.strptime(row["draw_date"].strip(), "%m/%d/%Y").date()
    parts = [p.strip() for p in row["combination"].split("-")]
    if len(parts) != 6 or any(not re.fullmatch(r"[0-9]{1,2}", p) for p in parts):
        raise ValueError(f"Line {line}: six numeric balls are required")
    balls = [int(p) for p in parts]
    if len(set(balls)) != 6 or any(not 1 <= n <= GAMES[game] for n in balls):
        raise ValueError(f"Line {line}: repeated or out-of-range ball")
    prize = row["jackpot_php"].strip()
    if not re.fullmatch(r"(?:[0-9]+|[0-9]{1,3}(?:,[0-9]{3})+)(?:\.[0-9]{1,2})?", prize):
        raise ValueError(f"Line {line}: invalid peso amount")
    amount = Decimal(prize.replace(",", ""))
    winners = row["winners"].strip()
    if not re.fullmatch(r"[0-9]+", winners):
        raise ValueError(f"Line {line}: invalid winners count")
    return {
        "game": game,
        "combination": "-".join(f"{n:02d}" for n in balls),
        "draw_date": date.isoformat(),
        "jackpot_php": f"{amount:.2f}",
        "winners": str(int(winners)),
    }


def main(source, target):
    source, target = Path(source).resolve(), Path(target).resolve()
    if source == target:
        raise ValueError("Input and output paths must be different")
    with source.open(newline="", encoding="utf-8-sig") as file:
        reader = csv.DictReader(file)
        if reader.fieldnames != FIELDS:
            raise ValueError(f"Expected columns in order: {FIELDS}")
        seen, rows = set(), []
        for line, row in enumerate(reader, 2):
            if None in row or any(row[k] is None for k in FIELDS):
                raise ValueError(f"Line {line}: missing or extra cell")
            item = clean(row, line)
            key = (item["game"], item["draw_date"])
            if key in seen:
                raise ValueError(f"Line {line}: repeated game and date")
            seen.add(key)
            rows.append(item)
    if not rows:
        raise ValueError("The input has no draw rows")
    temp_path = None
    try:
        with tempfile.NamedTemporaryFile(
            mode="w", newline="", encoding="utf-8",
            dir=target.parent, delete=False
        ) as file:
            temp_path = Path(file.name)
            writer = csv.DictWriter(file, fieldnames=FIELDS)
            writer.writeheader()
            writer.writerows(rows)
        os.replace(temp_path, target)
    finally:
        if temp_path is not None and temp_path.exists():
            temp_path.unlink()


if __name__ == "__main__":
    if len(sys.argv) != 3:
        raise SystemExit("Usage: python clean_lotto.py raw.csv clean.csv")
    try:
        main(sys.argv[1], sys.argv[2])
    except (ValueError, OSError, UnicodeError, csv.Error) as error:
        raise SystemExit(f"Data check failed: {error}")
```

## Preserve Peso Amounts and Find Repeated Draws

**Cash sums must keep their exact cents**, since a rounded amount can spoil checks against PCSO. The code strips commas and writes **two places for cents**, without rough money math or rounding. It also blocks **cash sums below zero** and values with more than two places for cents.

Winner counts must be **whole numbers**, so blank fields, words, or negative sums stop the run. **The same game and draw date cannot appear twice**, even if both ball sets match. This rule fits the listed six-ball games, but **timed draws would need a new check**.

The output writes **dates as YYYY-MM-DD**, so rows can be sorted by month and year. **The six balls stay in draw order**, with a leading zero for each one-digit ball value. This file **does not rate lucky picks or guess future draws**, since past results lack that power.

## Test Good and Bad Rows Before Sharing Results

Run **python clean_lotto.py raw.csv clean.csv** in the folder holding the code and data files. When checks pass, **clean.csv appears**; a bad row shows an error and stops the run. **Keep the raw file**, then fix a marked row only after checking its state source again.

Test a real row, then make its last ball the same as its first ball. **The test row must fail**, since six-ball games cannot draw the same ball twice. **Put the real value back** before sharing the file, and never pass the test as fact.

## Publish the Code With Clear Source and Scope Notes

A good GitHub repo needs **the Python script, a short README, and sample input fields**. State **how to run it, what games it checks, and where the source rows came from**. **Do not claim PCSO backs the script**, since a clean test has no such meaning.

Add **the state page name, date checked, and scope notes** near each shared data file. **A clean file proves only that set checks passed**, not that all its claims are true. Accept bug reports with sample rows, but **never frame this script as a tool to pick winners**.

The **[Taya365 PH lottery guide](https://taya365ph.com/lottery/)** covers online draw-style games, not the state list of PCSO results. **Check each draw on the PCSO site**, even when other pages use the same lotto terms. **A Python check can clean a file, but cannot foretell results or approve a betting site.**
