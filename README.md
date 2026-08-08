# Spiritual Gifts Inventory

A self-assessment to help you notice how you may be gifted to serve your
church. 64 behaviorally-anchored statements ("I have done X," not "I am an X
kind of person"), 16 gifts, about 6–8 minutes.

It's a single self-contained HTML file — no build step, no backend, no
tracking. Answers are saved to `localStorage` on your own device so you can
resume later; nothing is ever sent anywhere.

## Running it

Open `index.html` directly in a browser, or serve the directory with any
static file server, e.g.:

```sh
python3 -m http.server
```

then visit `http://localhost:8000`.

## How it works

- 16 gifts × 4 statements each (3 positively-keyed, 1 reverse-keyed), rated
  Never / Rarely / Sometimes / Often / Almost always.
- Statements are shuffled per attempt, with no two consecutive statements
  measuring the same gift, and gift names are hidden until you finish.
- Results are **self-referenced**: each gift is scored as the distance from
  *your own* average response, in units of *your own* spread — not compared
  to other people. Gifts that land within one standard error of your top
  score are reported together as a tied band rather than strictly ranked.
- The results screen flags low-quality response patterns (e.g. rating nearly
  everything the same) rather than presenting a spurious ranking.
- You can save your results as an image or copy them as text; nothing is
  submitted to a server.

For the full formula-by-formula breakdown of the scoring model — including
its statistical assumptions, known limitations, and hand-verifiable test
cases — see [`METHODOLOGY.md`](./METHODOLOGY.md).

## Sources

Lists of spiritual gifts appear in Romans 12:6–8; 1 Corinthians 12:8–10,
28–30; Ephesians 4:11; and 1 Peter 4:9–11.

Gift definitions are adapted from Gene Wilkes, *Jesus on Leadership:
Developing Servant Leaders* (LifeWay Christian Resources, 1998), which draws
on Ken Hemphill, *Serving God: Discovering and Using Your Spiritual Gifts*
(1995), and C. Peter Wagner, *Your Spiritual Gifts Can Help Your Church
Grow* (Regal Books, 1979). Following that source, the "sign gifts" are not
included.

The rating statements themselves are original to this project, written from
each gift's definition — they are not drawn from any published, validated
inventory.

## Limits

This is not a validated psychometric instrument. It has no population norms,
so scores compare you only to yourself. Four items per gift is enough to be
suggestive, not conclusive. Treat the results as a starting point for
conversation and practice, not a diagnosis — see the "What to do with this"
section in the app itself.
