## About

Total memes - **575948**

Scraped top 100 popular memes from [Imgflip](https://imgflip.com/) using [Scrapy](https://docs.scrapy.org/en/latest/)

Can be used with [Imgflip API](https://api.imgflip.com/) to caption memes. This datast is a bigger version of [Imgflip top 24 memes](https://www.kaggle.com/dylanwenzlau/imgflip-meme-text-samples-for-top-24-memes).

## Memes Dataset

Top 100 popular memes ```./dataset/popular_100_memes.csv```

Nr of memes / template ```./dataset/statistics.json```

- Templates ```./dataset/templates```

Template Example
```yaml
{
  "title": "10 Guy Meme Template",
  "template_url": "https://imgflip.com/s/meme/10-Guy.jpg",
  "alternative_names": "Really High Guy, Stoner Stanley, Brainwashed Bob, stoned guy, ten guy, stoned buzzed high dude bro",
  "template_id": "101440",
  "format": "jpg",
  "dimensions": "500x454 px",
  "file_size": "24 KB"
}
```

- Memes ```./dataset/memes```
  
  Meme example:
```yaml
 {
    "url": "https://i.imgflip.com/2cpxta.jpg",
    "post": "https://imgflip.com/i/2cpxta",
    "metadata": {
      "views": "2,426",
      "img-votes": "4",
      "title": "Watch out or it'll eat you whole",
      "author": "PLarsen985"
    },
    "boxes": [
      "I USED TO CODE WITH PYTHON",
      "BUT I QUIT AFTER THE FIRST TIME IT BIT ME"
    ]
  }
```


## How to run

A snapshot of the scraped data is already committed under `./dataset` and `./dataset1`
(see *Status* below for what each contains). You only need to run the spiders if you
want fresh data.

### Setup

All commands are run from the project root (the directory containing `scrapy.cfg`).

```sh
python3 -m venv .venv
. .venv/bin/activate          # Windows: .\.venv\Scripts\activate
pip install --upgrade pip
pip install Scrapy            # see the note under "Status" about requirements.txt
```

### Run the pipeline

The three spiders must run in order (each reads the output of the previous one),
then `statistics.py` aggregates counts:

1. `popular-memes` – scrapes the top-100 popular meme list into `dataset/popular_100_memes.csv`.
2. `templates` – scrapes template metadata + images into `dataset/templates/`.
3. `memes` – scrapes individual meme instances (captions/metadata) into `dataset/memes/*.json`.

**Linux / macOS**

```sh
./run.sh
```

**Windows (PowerShell)**

```sh
scrapy crawl -L INFO popular-memes; if ($?) { scrapy crawl -L INFO templates; if ($?) { scrapy crawl -L INFO memes; if ($?) { python statistics.py } } }
```

You can also run a single spider, e.g. `scrapy crawl -L INFO popular-memes`.

> Note: the spiders write into `dataset/` relative to the current working directory,
> so running them will overwrite the committed snapshot in `./dataset`.

## Status / what's incomplete

This project is a working-but-rough Scrapy crawler; it was left in a partially-finished
state. Assessment based on reading the code:

**What works**
- All three spiders load and run under a current Scrapy (verified against Scrapy 2.17 /
  Python 3.14). `popular-memes` was smoke-tested end-to-end against the live site and
  produces a correct 100-row CSV.
- `imgflip_scraper/{items,pipelines,middlewares}.py` are the default `scrapy startproject`
  boilerplate. They are **not wired in** (the `ITEM_PIPELINES` / `*_MIDDLEWARES` entries in
  `settings.py` are all commented out); each spider writes its own JSON/CSV files directly.
  This is intentional, not a bug, but means Scrapy's Item/Pipeline machinery is unused.

**Known rough edges (not fixed here — would change behaviour)**
- `templates_spider` saves its results and downloads template images from within
  `__del__`, which relies on garbage-collection timing and is unreliable (it did not flush
  in a short/limited run). It should save on the Scrapy `spider_closed` signal instead.
- `memes_spider` / `parse_meme` use broad bare `except:` blocks that swallow all errors,
  which can silently drop memes if the site markup changes.
- No automated tests exist.

**Requirements / environment**
- `requirements.txt` pins old versions (Scrapy 2.9.0, lxml 4.9.3, Twisted 22.10, etc.)
  that do **not** build/install on Python 3.13+ (e.g. lxml 4.9.3 has no wheels and fails
  to compile). Installing a current `Scrapy` works. The pins should be refreshed or the
  file regenerated from a working environment.

**Datasets in the repo**
- Two snapshots are committed: `dataset/` (~254 MB, 53 meme files, no `templates/`) and
  `dataset1/` (~194 MB, 99 meme files + 100 templates). They overlap and together bloat the
  git history (~109 MB `.git`). Consider consolidating to one and moving large data out of
  git (e.g. Git LFS or an external release) — left in place here to avoid destructive changes.

### Fix applied in this pass
- `templates_spider.save_template_img` used `urllib.request.URLopener`, which was removed in
  Python 3.14 and would raise `AttributeError`. Replaced with an equivalent
  `urllib.request.Request` + `urlopen` that preserves the custom `User-Agent`.
