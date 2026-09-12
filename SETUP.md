# KDPChimp cover landing page — setup

## Style

You picked **Direct Response**, and that's what `index.html` is.

`build.py` holds one copy of the content and three theme blocks, and still emits
all three variants so you can compare later. **Edit `build.py`, not the generated
files** — then run `python3 build.py`. `index.html` is rewritten from whichever
theme `CHOSEN` names.

## What's locked in

- **$499**, published on the page (a NN/g trust study recorded a user abandoning
  a service in 35 seconds because rates weren't shown — burying price costs more
  than it gains)
- **Unlimited revisions on your chosen direction** — fences the cost driver
  (direction changes) without withdrawing the promise your existing reviews
  already make
- **All four formats**: Kindle, paperback, hardcover, ACX — the strongest single
  line in the included list
- **10 projects a month**, in the announcement bar, the pricing block and its own
  FAQ answer. The bar reads *"Taking 10 cover projects in September."* — change
  `CONFIG.month` when the month turns. As September fills, set `CONFIG.slotsLeft`
  to a number and the bar adds *"3 left."*; set it to `0` and both places say the
  month is full and offer the next start date
- **No value stack** — clean included-list, per your call
- **The "That's The One" Guarantee** — named, boxed, beside the price
- One CTA label (`Order My Cover — $499`) repeated five times down the page
- A P.S. at the close, for skimmers

`landing_page_research.md` in project memory has the evidence behind these
choices and the popular claims that turned out to be unsupported.

## The framework section

Your six-point review is now its own section (`#review`), between the process and
the involvement block: the six criteria as a grid, the grayscale and blur tests
as a pair, and a simulated Amazon results row with your cover outlined among five
competitors. It's the strongest section on the page, because it's the only place
that answers "will it actually work?" with a method instead of an adjective. It
also appears in the included-list, the guarantee, and its own FAQ answer.

The competitor shelf currently reuses six portfolio covers as stand-ins. Swap in
real category screenshots when you have them and it gets sharper still.

## Team anonymity

No individual names on the page. Buyers get "a dedicated project manager who is
also a KDP expert" and "a dedicated professional graphic designer at your
disposal". Josiah is still named as founder, and in the Gold tier. If you ever
want to name the team, the strings are all in `build.py`.

## Split-testing — deliberately not on the page

Real customer split-testing (PickFu-style, or cheap targeted ads to the avatar)
would be a genuine differentiator and nobody else in this space offers it. It is
**not on the page**, because you described it as an idea rather than something
you deliver today, and a promise you can't fulfil on the first order costs more
than it wins. Two options when you're ready: make it a paid add-on (it carries
real ad spend, so it shouldn't be absorbed into $499), or make it the thing that
justifies a premium tier above $499. Say the word and I'll build the section.

---

## 1. The portfolio — done

The gallery is **16 real delivered covers**, chosen from your /10 quality ranking
in Airtable: all nine you scored 7, plus seven 6s picked to widen the niche
spread rather than repeat one. Files live in `assets/covers/`, 700px wide and
about 150KB each — 2.6MB for the lot, which is nothing.

They're ordered so no two adjacent tiles share a niche. The hero fan uses the
first three in the array, so reordering `CONFIG.covers` changes the hero too.

**The competitor shelf is real.** Eight covers: ours (*The Effortless
Anti-Inflammatory Diet Cookbook*, ORD-14306) outlined in orange, beside seven
actual competing titles pulled from that Amazon category in Sept 2026, in
`assets/shelf/`. Nothing in that row is cropped — no fixed aspect-ratio, no
`object-fit: cover` — because trimming a competitor to match our proportions
would rig the comparison. To swap the category later, replace the files in
`assets/shelf/` and point `CONFIG.shelf.yours` at the matching cover of ours.

**The gallery is masonry, not a fixed grid.** Your covers are 6x9, 8.5x11 and
1600x2560, and a fixed tile ratio crops the odd ones — on an 8.5x11 that means
slicing the title off the sides. Masonry columns let every cover keep its true
proportions. Four columns on desktop, three on tablet, two on phone.

To swap one: drop the new file in `assets/covers/`, change the `src` in
`CONFIG.covers` inside `build.py`, re-run `python3 build.py`. `genre:` is the
label that fades in on hover.

One thing worth knowing: **you have permission for exactly one of these covers.**
Vladimir gave it in writing for *Werewolf Sightings* (24 Mar 2026), which isn't
in the gallery. The 4 June 2025 campaign that converted 2/2 asked about A+
content, not covers, so neither grant carries over. `AUDIT-permissions-and-reviews.md`
has the email template — it's your own wording with the noun changed.

## 2. Payment links

Create both, then paste into `CONFIG` at the top of the `<script>` in `index.html`.

**Stripe** → Product catalogue → new product "Professional Book Cover Design",
$499 one-off → *Create payment link*.
- After payment: **redirect to** `https://covers.kdpchimp.com/thanks.html`
  (Stripe appends `?session_id=...` automatically — `thanks.html` reads it)
- Turn on: collect customer name, email, and phone
- Add a custom field: **Book title** (this lands in the webhook and saves Cara a round trip)
- Allow promotion codes → lets you run the $399 discount as a coupon instead of
  changing the price

**PayPal** → Pay Links & Buttons → new fixed-price link at $499 → set the return
URL to the same `thanks.html`.

```js
stripeLink : "https://buy.stripe.com/...",
paypalLink : "https://www.paypal.com/ncp/payment/...",
goldLink   : "",   // leave empty → the Gold button opens an email to you
```

Any link left empty falls back to a mailto, so the page is never broken.

**Gold pilot:** leave `goldLink` empty for now. It's an untested $499/mo offer —
let the applications come in by email, talk to the first few yourself, and only
build a subscription checkout once someone has actually said yes.

---

## 3. Intake form

Open `thanks.html` and set `BRIEF_FORM` to your existing brief form URL (Tally,
Fillout, or the Lovable form). The payment reference is appended as `?ref=` so
n8n can match the brief to the payment without anyone retyping an order number.

---

## 4. Deploy — live, and how to update it

- **Live:** https://covers.kdpchimp.com
- **Vercel project:** `coverslandingpage` (team `support-7807's projects`, Hobby)
- **Repo:** `KDPChimp/covers-landing-page`, branch `main` — connected to the
  project, so **every push to main deploys to production automatically**
- **DNS:** GoDaddy, CNAME `covers` -> `9f86d5233d43fa74.vercel-dns-017.com`
  (MX untouched: mail still on `smtp.google.com`)

### Making an update

1. Claude edits `build.py` and runs `python3 build.py` (regenerates `index.html`)
2. Claude commits
3. **You run one command, in your Mac's own Terminal:**

```
cd ~/Documents/Claude/Projects/KDPChimp/covers-landing-page && git push
```

Vercel builds from the commit and the domain updates itself. The push has to come
from your Terminal because that is where your GitHub credentials live — Claude's
sandbox can reach GitHub but has no way to authenticate as you, and cannot reach
vercel.com at all.

`.vercelignore` keeps `build.py`, `picker.html` and every `.md` in the repo for
history but out of the published site. `covers-site-deploy/` and
`covers-site.zip` are leftovers from the Vercel Drop era and can be deleted.

## Checks before you send it to the list

- [ ] Every portfolio image loads (open the deployed URL in an incognito window)
- [ ] Stripe checkout completes in **test mode** and lands on `thanks.html`
- [ ] PayPal does the same
- [ ] The n8n webhook created an Airtable order (see `N8N-AIRTABLE.md`)
- [ ] Read it on your phone — the hero, the gallery and the pricing card
- [ ] Testimonial wording matches kdpchimp.com/testimonials exactly (the `...`
      in several quotes marks where a longer quote was shortened — check them)
- [ ] The claim "30+ publishers have already experienced the new system" is
      lifted from your own site; confirm it's still true before it goes out
