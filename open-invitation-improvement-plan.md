# Open Invitation: Podcast Site Improvement Plan

## Context for Claude Code

This plan is for **Open Invitation** — a podcast site about Open Access efforts at the University of Idaho, hosted at `https://thecdil.github.io/open-invitation/`. The site is built with **CollectionBuilder-CSV** (Jekyll-based, using the Oral History as Data template). It currently has 3 episodes with transcripts, standard CB browse/subjects/map/timeline/data pages, and a homepage with studio chair artwork.

The GitHub repo is at `https://github.com/thecdil/open-invitation` (deployed via GitHub Pages). The site uses Jekyll with Liquid templating, data stored in CSV metadata files in `_data/`, and plain JavaScript. No Ruby plugins beyond what GitHub Pages supports natively.

**Goals:**
1. Generate a valid RSS 2.0 podcast feed from existing metadata so the podcast can be submitted to Apple Podcasts, Spotify, and other directories — without using a third-party podcast host
2. Improve the site design to feel more like a proper podcast platform while preserving the CollectionBuilder foundation
3. Add features that make the site a compelling standalone listening experience

---

## Existing Site Details (important for implementation)

- **Audio files** are already hosted at `cdil.lib.uidaho.edu` and referenced in the metadata CSV's **`object_location`** field. Example URL: `https://cdil.lib.uidaho.edu/open-invitation/objects/OI-S1_Bland-Ep-final.mp3`
- **`objectid`** serves as the RSS `<guid>` — no separate guid column needed.
- **Episode item pages already have an MP3 player** at the top (via the transcript display template). Do NOT add another player — just enhance the template with new metadata display.
- **Subject tags are already displayed** on episode pages.
- **The browse page (`/browse.html`) should be left as-is.**

---

## Part 1: RSS 2.0 Podcast Feed

### 1.1 New config values (`_config.yml`)

Add a `podcast` block to `_config.yml`:

```yaml
# Podcast Feed Settings
podcast:
  title: "Open Invitation"
  description: "Open Invitation seeks to highlight Open Access efforts from across the University of Idaho. Interviews feature U of I faculty, staff, and students discussing their cutting-edge research, as well as the benefits and challenges of navigating the open access landscape."
  author: "University of Idaho Library"
  email: "podcast-owner-email@uidaho.edu"  # MUST be a real email — Apple and Spotify send verification codes here
  language: "en-us"
  explicit: "false"
  type: "episodic"                          # or "serial"
  category: "Education"                     # Apple Podcasts category
  subcategory: "Higher Education"           # Apple Podcasts subcategory
  image: "/assets/img/podcast-cover.jpg"    # 3000x3000 recommended, 1400x1400 minimum
  link: "https://thecdil.github.io/open-invitation/"
  copyright: "University of Idaho Library"
  apple_url: ""      # fill in after Apple approval
  spotify_url: ""    # fill in after Spotify approval
  amazon_url: ""     # fill in after Amazon approval
```

### 1.2 Metadata CSV additions

The existing `object_location` field already contains the MP3 URL — **do not add a separate `audio_url` column**. The feed template reads directly from `object_location`.

Add these new columns to the metadata CSV:

| New Column | Example Value | Notes |
|---|---|---|
| `audio_length` | `34567890` | File size in bytes (required by RSS `<enclosure>`). Get via `curl -sI <url> \| grep -i content-length` |
| `duration` | `00:42:15` | Episode length in `HH:MM:SS` format |
| `episode_number` | `1` | Integer episode number |
| `season_number` | `1` | Integer season number (leave blank if not using seasons) |
| `episode_type` | `full` | `full`, `trailer`, or `bonus` |
| `guest` | `Dr. Jane Bland` | Guest name(s) for display on episode pages and homepage |

### 1.3 Feed template (`feed.xml`)

Create `feed.xml` in the project root:

```xml
---
layout: null
---
<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0"
     xmlns:itunes="http://www.itunes.com/dtds/podcast-1.0.dtd"
     xmlns:podcast="https://podcastindex.org/namespace/1.0"
     xmlns:atom="http://www.w3.org/2005/Atom"
     xmlns:content="http://purl.org/rss/1.0/modules/content/">
  <channel>
    <title>{{ site.podcast.title | xml_escape }}</title>
    <link>{{ site.podcast.link }}</link>
    <description>{{ site.podcast.description | xml_escape }}</description>
    <language>{{ site.podcast.language }}</language>
    <copyright>{{ site.podcast.copyright | xml_escape }}</copyright>
    <atom:link href="{{ site.url }}{{ site.baseurl }}/feed.xml" rel="self" type="application/rss+xml"/>
    <lastBuildDate>{{ site.time | date: "%a, %d %b %Y %H:%M:%S %z" }}</lastBuildDate>
    <itunes:author>{{ site.podcast.author | xml_escape }}</itunes:author>
    <itunes:summary>{{ site.podcast.description | xml_escape }}</itunes:summary>
    <itunes:type>{{ site.podcast.type }}</itunes:type>
    <itunes:owner>
      <itunes:name>{{ site.podcast.author | xml_escape }}</itunes:name>
      <itunes:email>{{ site.podcast.email }}</itunes:email>
    </itunes:owner>
    <itunes:explicit>{{ site.podcast.explicit }}</itunes:explicit>
    <itunes:category text="{{ site.podcast.category }}">
      {% if site.podcast.subcategory %}<itunes:category text="{{ site.podcast.subcategory }}"/>{% endif %}
    </itunes:category>
    <itunes:image href="{{ site.url }}{{ site.baseurl }}{{ site.podcast.image }}"/>
    <image>
      <url>{{ site.url }}{{ site.baseurl }}{{ site.podcast.image }}</url>
      <title>{{ site.podcast.title | xml_escape }}</title>
      <link>{{ site.podcast.link }}</link>
    </image>
    <podcast:locked>no</podcast:locked>

    {% assign episodes = site.data[site.metadata] | where_exp: "item", "item.object_location != nil and item.object_location != ''" | sort: "date" | reverse %}
    {% for item in episodes %}
    <item>
      <title>{{ item.title | xml_escape }}</title>
      <description><![CDATA[{{ item.description }}]]></description>
      <link>{{ site.url }}{{ site.baseurl }}/items/{{ item.objectid }}.html</link>
      <guid isPermaLink="false">{{ item.objectid }}</guid>
      <pubDate>{{ item.date | date: "%a, %d %b %Y 12:00:00 %z" }}</pubDate>
      <enclosure url="{{ item.object_location }}" length="{{ item.audio_length }}" type="audio/mpeg"/>
      <itunes:title>{{ item.title | xml_escape }}</itunes:title>
      <itunes:author>{{ site.podcast.author | xml_escape }}</itunes:author>
      <itunes:summary>{{ item.description | xml_escape }}</itunes:summary>
      <itunes:explicit>{{ site.podcast.explicit }}</itunes:explicit>
      <itunes:duration>{{ item.duration }}</itunes:duration>
      {% if item.episode_number %}<itunes:episode>{{ item.episode_number }}</itunes:episode>{% endif %}
      {% if item.season_number %}<itunes:season>{{ item.season_number }}</itunes:season>{% endif %}
      <itunes:episodeType>{{ item.episode_type | default: "full" }}</itunes:episodeType>
      {% if item.image_small %}
      <itunes:image href="{{ site.url }}{{ site.baseurl }}{{ item.image_small }}"/>
      {% endif %}
    </item>
    {% endfor %}
  </channel>
</rss>
```

**Key points:**
- Filters on `object_location` (not a new column) to find episodes with audio
- Uses `objectid` directly as the `<guid>`
- `<enclosure>` points to the `object_location` URL on `cdil.lib.uidaho.edu`

### 1.4 Feed autodiscovery

Add to the site's `<head>`. Check for an existing CB hook file first — CB often provides `_includes/head/extra-head.html` for custom head content. If that exists, add to it. If not, copy CB's head include locally and add:

```html
<link rel="alternate" type="application/rss+xml" title="{{ site.podcast.title }}" href="{{ site.url }}{{ site.baseurl }}/feed.xml">
```

### 1.5 Validate the feed

After building locally (`bundle exec jekyll serve`), validate at:
- https://www.castfeedvalidator.com/
- https://podba.se/validate/

Common issues: missing/blank `audio_length` values, dates not in RFC 822 format, artwork below 1400×1400.

---

## Part 2: Podcast Artwork

Square cover art required by all directories:

- **Minimum:** 1400 × 1400 pixels
- **Recommended:** 3000 × 3000 pixels
- **Format:** JPEG or PNG, RGB color space
- **Max file size:** 512KB recommended

Save as `/assets/img/podcast-cover.jpg`. Referenced in `_config.yml` under `podcast.image`. The current studio chair images could be adapted. This is a manual design step.

---

## Part 3: Design Improvements

### 3.1 Episode item page enhancements

The transcript template already has the MP3 player and subject tags. Modify `_layouts/item/transcript.html` to add three things:

**A. Prominent guest name** — below title, above player:

```html
{% if item.guest %}
<p class="episode-guest h5 text-muted mb-3">
  with <strong>{{ item.guest }}</strong>
</p>
{% endif %}
```

**B. Episode metadata line** — below the player:

```html
<div class="episode-meta text-muted mb-3">
  {% if item.season_number %}Season {{ item.season_number }}, {% endif %}
  {% if item.episode_number %}Episode {{ item.episode_number }}{% endif %}
  {% if item.duration %} · {{ item.duration }}{% endif %}
  {% if item.date %} · {{ item.date | date: "%B %d, %Y" }}{% endif %}
</div>
```

**C. Share button** — copy-link using vanilla JS:

```html
<div class="episode-share mb-4">
  <button class="btn btn-outline-secondary btn-sm" onclick="copyEpisodeLink(this)"
          data-url="{{ site.url }}{{ site.baseurl }}/items/{{ item.objectid }}.html">
    <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="currentColor" class="bi bi-link-45deg" viewBox="0 0 16 16">
      <path d="M4.715 6.542 3.343 7.914a3 3 0 1 0 4.243 4.243l1.828-1.829A3 3 0 0 0 8.586 5.5L8 6.086a1 1 0 0 0-.154.199 2 2 0 0 1 .861 3.337L6.88 11.45a2 2 0 1 1-2.83-2.83l.793-.792a4 4 0 0 1-.128-1.287z"/>
      <path d="M6.586 4.672A3 3 0 0 0 7.414 9.5l.775-.776a2 2 0 0 1-.896-3.346L9.12 3.55a2 2 0 1 1 2.83 2.83l-.793.792c.112.42.155.855.128 1.287l1.372-1.372a3 3 0 1 0-4.243-4.243z"/>
    </svg>
    Share Episode
  </button>
</div>

<script>
function copyEpisodeLink(btn) {
  var url = btn.getAttribute('data-url');
  navigator.clipboard.writeText(url).then(function() {
    btn.innerHTML = '✓ Link Copied!';
    setTimeout(function() {
      btn.innerHTML = '<svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="currentColor" class="bi bi-link-45deg" viewBox="0 0 16 16"><path d="M4.715 6.542 3.343 7.914a3 3 0 1 0 4.243 4.243l1.828-1.829A3 3 0 0 0 8.586 5.5L8 6.086a1 1 0 0 0-.154.199 2 2 0 0 1 .861 3.337L6.88 11.45a2 2 0 1 1-2.83-2.83l.793-.792a4 4 0 0 1-.128-1.287z"/><path d="M6.586 4.672A3 3 0 0 0 7.414 9.5l.775-.776a2 2 0 0 1-.896-3.346L9.12 3.55a2 2 0 1 1 2.83 2.83l-.793.792c.112.42.155.855.128 1.287l1.372-1.372a3 3 0 1 0-4.243-4.243z"/></svg> Share Episode';
    }, 2000);
  });
}
</script>
```

**Element order** at top of the episode content area should be:
1. Episode title (already exists)
2. Guest name (new)
3. Audio player (already exists)
4. Episode meta line (new)
5. Share button (new)
6. Transcript content (already exists)
7. Subject tags (already exist)

**Important:** Before modifying `_layouts/item/transcript.html`, check if it already exists locally. If not, copy it from the CB theme/gem first, then modify.

### 3.2 Homepage improvements

**A. Latest Episode card.** Create `_includes/feature/latest-episode.html`:

```html
{% assign latest = site.data[site.metadata] | where_exp: "item", "item.object_location != nil and item.object_location != ''" | sort: "date" | reverse | first %}
{% if latest %}
<div class="card mb-4 shadow-sm">
  <div class="card-body">
    <span class="badge bg-primary mb-2">Latest Episode</span>
    {% if latest.episode_number %}
    <span class="badge bg-secondary mb-2">Episode {{ latest.episode_number }}</span>
    {% endif %}
    <h4 class="card-title">{{ latest.title }}</h4>
    {% if latest.guest %}
    <p class="text-muted mb-2">with <strong>{{ latest.guest }}</strong></p>
    {% endif %}
    <p class="card-text">{{ latest.description | truncatewords: 40 }}</p>
    <audio controls preload="none" style="width:100%;">
      <source src="{{ latest.object_location }}" type="audio/mpeg">
    </audio>
    <div class="mt-3">
      <a href="{{ '/items/' | append: latest.objectid | append: '.html' | relative_url }}" class="btn btn-primary">
        Full Episode &amp; Transcript
      </a>
      {% if latest.duration %}
      <span class="text-muted ms-2">{{ latest.duration }}</span>
      {% endif %}
    </div>
  </div>
</div>
{% endif %}
```

**B. Subscribe badges.** Create `_includes/feature/subscribe-badges.html`:

```html
<div class="subscribe-section mb-4">
  <h5>Listen &amp; Subscribe</h5>
  <div class="subscribe-grid">
    {% if site.podcast.apple_url and site.podcast.apple_url != "" %}
    <a href="{{ site.podcast.apple_url }}" class="subscribe-badge" target="_blank" rel="noopener">
      Apple Podcasts
    </a>
    {% endif %}
    {% if site.podcast.spotify_url and site.podcast.spotify_url != "" %}
    <a href="{{ site.podcast.spotify_url }}" class="subscribe-badge" target="_blank" rel="noopener">
      Spotify
    </a>
    {% endif %}
    {% if site.podcast.amazon_url and site.podcast.amazon_url != "" %}
    <a href="{{ site.podcast.amazon_url }}" class="subscribe-badge" target="_blank" rel="noopener">
      Amazon Music
    </a>
    {% endif %}
    <a href="{{ site.url }}{{ site.baseurl }}/feed.xml" class="subscribe-badge">
      RSS Feed
    </a>
  </div>
</div>
```

**On the home page:**
- Replace the single "Listen and Subscribe on Spotify" link with `{% include feature/subscribe-badges.html %}`
- Add `{% include feature/latest-episode.html %}` below the hero description, above or replacing the "Top Subjects" section

### 3.3 Dedicated Listen page

Create `listen.md` (or `pages/listen.md` per CB convention):

```markdown
---
title: Listen & Subscribe
layout: page
permalink: /listen.html
---

## Subscribe to Open Invitation

Listen on your favorite podcast app:

{% include feature/subscribe-badges.html %}

## RSS Feed

Copy this URL into any podcast app to subscribe directly:

<div class="input-group mb-3" style="max-width:600px;">
  <input type="text" class="form-control" value="{{ site.url }}{{ site.baseurl }}/feed.xml" id="rss-url" readonly>
  <button class="btn btn-outline-secondary" onclick="document.getElementById('rss-url').select(); navigator.clipboard.writeText(document.getElementById('rss-url').value); this.textContent='Copied!'; setTimeout(()=>this.textContent='Copy',2000);">Copy</button>
</div>

## All Episodes

{% assign episodes = site.data[site.metadata] | where_exp: "item", "item.object_location != nil and item.object_location != ''" | sort: "date" | reverse %}
{% for item in episodes %}
### {{ item.title }}
{% if item.guest %}*with {{ item.guest }}*  {% endif %}
{{ item.date | date: "%B %d, %Y" }}{% if item.duration %} · {{ item.duration }}{% endif %}

{{ item.description | truncatewords: 30 }}

[Listen →]({{ '/items/' | append: item.objectid | append: '.html' | relative_url }})

---
{% endfor %}
```

Add "Listen" to navigation in `_data/config-nav.csv` (check existing format first).

### 3.4 CSS

Add to `assets/css/custom.scss` (create if needed — empty YAML front matter required):

```scss
---
---
// Episode guest name
.episode-guest { font-style: italic; }

// Episode metadata line
.episode-meta { font-size: 0.9rem; }

// Subscribe badges
.subscribe-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
}
.subscribe-badge {
  display: inline-flex;
  align-items: center;
  padding: 0.5rem 1rem;
  border: 1px solid #dee2e6;
  border-radius: 0.5rem;
  text-decoration: none;
  color: #212529;
  font-weight: 500;
  transition: background 0.2s, border-color 0.2s;
}
.subscribe-badge:hover {
  background: #f8f9fa;
  border-color: #adb5bd;
  color: #212529;
  text-decoration: none;
}
```

---

## Part 4: Submitting to Directories

After feed is live and validated:

| Directory | URL | Notes |
|---|---|---|
| Apple Podcasts | https://podcasters.apple.com/ | Submit RSS URL, verify via Apple ID. 24hrs–2wks approval. |
| Spotify | https://creators.spotify.com/ | Select "I have a podcast" → "Somewhere else". Verifies via email in feed. Hours to days. |
| Amazon Music | https://podcasters.amazon.com/ | RSS URL + Amazon account |
| Pocket Casts | https://pocketcasts.com/submit/ | RSS URL only |
| Podcast Index | https://podcastindex.org/add | Good for indie app discoverability |

After each approval, add the platform URL to `_config.yml` under `podcast.apple_url`, `podcast.spotify_url`, etc. so the subscribe badges auto-populate.

---

## Part 5: Implementation Order for Claude Code

### Phase 1: Feed infrastructure
1. Add `podcast:` config block to `_config.yml`
2. Add new columns to metadata CSV (`audio_length`, `duration`, `episode_number`, `season_number`, `episode_type`, `guest`)
3. Create `/feed.xml`
4. Add RSS `<link>` tag to site head (find CB's head include hook first)
5. Note: podcast cover art at `/assets/img/podcast-cover.jpg` is a manual design step

### Phase 2: Episode pages
6. Modify `_layouts/item/transcript.html` — add guest name, episode meta line, share button (player and subject tags already exist)

### Phase 3: Homepage
7. Create `_includes/feature/latest-episode.html`
8. Create `_includes/feature/subscribe-badges.html`
9. Add both to home page, replace single Spotify link with badges

### Phase 4: Listen page
10. Create `listen.md`
11. Add "Listen" to `_data/config-nav.csv`

### Phase 5: Styling
12. Add CSS to `assets/css/custom.scss`

### Phase 6: Validate
13. Build locally, validate `/feed.xml` at castfeedvalidator.com
14. Test audio, check mobile layout

---

## File Map

```
open-invitation/
├── _config.yml                            # MODIFY: add podcast: block
├── _data/
│   ├── metadata.csv                       # MODIFY: add audio_length, duration,
│   │                                      #   episode_number, season_number,
│   │                                      #   episode_type, guest columns
│   └── config-nav.csv                     # MODIFY: add Listen page
├── _includes/
│   ├── head/
│   │   └── (head.html or extra-head.html) # MODIFY: add RSS <link> tag
│   └── feature/
│       ├── latest-episode.html            # CREATE
│       └── subscribe-badges.html          # CREATE
├── _layouts/
│   └── item/
│       └── transcript.html                # MODIFY: add guest, meta, share
├── assets/
│   ├── css/
│   │   └── custom.scss                    # CREATE or MODIFY
│   └── img/
│       └── podcast-cover.jpg              # CREATE (manual, 3000x3000)
├── feed.xml                               # CREATE
└── listen.md                              # CREATE
```

---

## Technical Notes for Claude Code

- **`object_location`** is the existing metadata field with MP3 URLs (e.g., `https://cdil.lib.uidaho.edu/open-invitation/objects/OI-S1_Bland-Ep-final.mp3`). The feed uses this — no new audio URL column.
- **`objectid`** = RSS `<guid>`. Must never change after feed submission.
- **`site.data[site.metadata]`** accesses the metadata CSV. `site.metadata` is set in `_config.yml`.
- **Item pages** generate at `/items/{objectid}.html`. The `display_template` field picks the layout from `_layouts/item/`.
- **To override a CB include/layout**: copy from CB theme into the project's local directory, then modify. Jekyll resolves local files first.
- **Check for `_includes/head/extra-head.html`** first — it's CB's convention for custom head additions. If absent, look at `_includes/head.html` or `_includes/head/head.html`.
- **GitHub Pages has no custom plugin support** — the pure-Liquid feed.xml approach is essential.
- **`| xml_escape`** is critical in the feed to prevent XML parsing errors.
- **Date format**: `"%a, %d %b %Y %H:%M:%S %z"` for RFC 822. If `%z` doesn't produce a proper offset on GitHub Pages, hardcode `+0000`.
- **`audio_length`** must be a non-blank integer (bytes). Get values with `curl -sI <url> | grep -i content-length`.
