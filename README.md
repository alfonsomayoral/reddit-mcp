# reddit-mcp

A read-only MCP server over the official Reddit Data API. It exposes five
GET-backed tools and no tool that writes. It cannot post, comment, vote, send
a message, subscribe or create anything, and it reads no user's history,
because no such code path exists.

That is the fact to weigh first. An agent reading Reddit is reading text
written by strangers, some of whom would like it to act on their behalf. A
promise a prompt makes can be argued with by the text it has just read. A
capability that is absent cannot be argued into existence.

Every payload the server returns is wrapped in delimiters behind an
`UNTRUSTED DATA` header, and the delimiter sequences are neutralised in the
retrieved text so that a title, a comment or a permalink cannot close the
wrapper early. Retrieved content is data to quote, never instructions to obey.

## Requirements

- Python 3.12 or later.
- `mcp` and `httpx`. There is nothing else at runtime.
- Reddit Data API credentials, which are not self-service. Read Access below
  before you plan around this server.

## Tools

### `subreddit_search(topic, limit)`

Searches the subreddit directory. Answers: **which communities discuss this
topic?** Returns, for each match, the community name, its subscriber count,
whether it is marked over 18, and its public description.

Use it when you know the subject but not the vocabulary or the venues. Then
use `subreddit_about` on the candidates.

### `subreddit_about(name)`

Reads one community's metadata and its rules. Answers: **how large and how
active is this community, and what does it actually permit?** Returns
subscribers, users currently active, creation date, description, and the
rules verbatim.

`name` accepts `python`, `r/python` or `/r/python`. The rules are a second
request and a soft failure: if they cannot be read, the rest is still
returned with the gap stated, rather than the whole call failing.

### `subreddit_posts(subreddit, sort, time_filter, limit, after)`

Lists a community's posts without a search query. Answers: **what is this
community posting?**

A search needs words to search for, and choosing those words is already an
assumption about what you will find. Browsing a listing makes no such
assumption, which matters when you do not yet know how a community phrases
things.

- `sort` — `hot`, `new`, `top` or `rising`.
- `time_filter` — narrows the window; Reddit applies it to `top`.
- `subreddit` — one name, or Reddit's `a+b+c` syntax to read several
  communities as a single listing.
- `after` — pagination cursor, see below.

### `search_posts(query, subreddit, time_filter, sort, limit, after)`

Searches posts across Reddit, or within one or more communities. Answers:
**which threads match these words?** Returns title, post id, community,
author handle, score, comment count, date, permalink and a trimmed body for
each post.

- `sort` — `relevance`, `hot`, `top`, `new` or `comments`.
- `time_filter` — `hour`, `day`, `week`, `month`, `year` or `all`.
- `subreddit` — optional. Empty searches all of Reddit; a name restricts the
  search to that community; `a+b+c` restricts it to several.
- `after` — pagination cursor, see below.

The post id in the results is what `thread_comments` takes.

### `thread_comments(post_id, limit)`

Reads one thread. Answers: **what did people actually say?** Accepts a bare
post id, a `t3_` fullname or a permalink. Returns the post itself and a mixed
sample of its comments, each with author handle, score, date and permalink.
How that sample is chosen is the first of the two decisions below.

### Pagination

`search_posts` and `subreddit_posts` accept an `after` cursor. Pass the
cursor from a previous response to receive the page that follows it. One call
returns one page and the server never walks pages on its own, so a caller
cannot spend quota it did not ask for.

## Two decisions worth explaining

These are the parts a reader tends to question, so here is the reasoning
rather than the rule.

### Comments are sampled from two fetches, not one

A busy thread carries more comments than a reader can hold, so something has
to choose which ones survive. `thread_comments` takes the highest-scoring
comments from a `sort=top` fetch **and** the most recent from a separate
`sort=new` fetch, then merges the two by comment id.

The second fetch is the entire point. Draw both halves from a single
top-sorted response and the "recent" half is merely the newest of the
comments Reddit had already ranked highest. The genuine complaint sitting at
three upvotes under the joke at nine hundred can then never surface — and
that comment is often the reason for opening the thread.

The recency pass is a soft failure. If it fails, the top pass is still
returned, and the response says which pass is missing.

### Every response states its own denominator

Each `thread_comments` response opens with a line of the form
`sampled: N of M comments`. M is the thread's own comment count as Reddit
reports it, not the number that survived the server's filters.

That distinction is what makes the figure usable. A caller writing "eleven
comments raise the same complaint" needs a real denominator; eleven of forty
and eleven of four hundred are different claims. Redefining M as "how many we
managed to read" would silently change the population the claim covers, and
estimating M is where such a claim quietly becomes false.

When the recency pass fails, that header says so instead of claiming both
passes ran. It is the one line a caller is most likely to repeat verbatim, so
it must not overstate what was read.

## Access

Reddit's Data API is not self-service, and this is the step most likely to
stop you.

Under the Responsible Builder Policy you request access through a form and a
person approves it **before** you create an app. Unauthenticated traffic is
blocked rather than throttled, so there is no unregistered path that merely
runs slowly.

Reddit's definition of commercial use is broad and names subscription
services and paywalled mobile apps outright. A separate clause forbids mining
Reddit data — commercially or not — without written approval.

Describe your use case accurately on the form. Misstating it is sanctioned
with token revocation and account suspension, and the sanction lands on the
account, not just the app.

## Configuration

The server reads three values:

```
REDDIT_CLIENT_ID=...
REDDIT_CLIENT_SECRET=...
REDDIT_USER_AGENT=your-app/1.0 (by /u/your_handle)
```

The user agent is not decorative. Reddit throttles generic ones hard, and it
carries the handle of the person answerable for the traffic.

Authentication is OAuth `client_credentials` against
`www.reddit.com/api/v1/access_token`, which yields an app-only, read-only
token. Data requests go to `oauth.reddit.com`. The token is cached for its own
lifetime less a margin, and a rejected token buys exactly one refresh and one
retry — past that the credentials are wrong rather than stale, and retrying a
rejected client id in a loop is how an app loses its access.

### `REDDIT_ENV_FILE`

Instead of the three variables, you may set `REDDIT_ENV_FILE` to the path of a
`KEY=value` file that contains them.

This exists because putting a secret into a configuration file that an AI
agent can read is not protecting it. If the agent holds a file-reading tool
and the configuration sits within its reach, retrieved text can ask it to
read that file, and the wrapper above makes such a request visible but does
not make the file unreadable. Passing a path rather than a secret keeps the
secret out of everything the caller writes, so it can live wherever the
operating system already protects it.

### `.mcp.json`

```json
{
  "mcpServers": {
    "reddit": {
      "command": "uv",
      "args": ["run", "--with", "mcp", "--with", "httpx", "python", "/path/to/server.py"],
      "env": {
        "REDDIT_ENV_FILE": "/home/you/.secrets/reddit.env"
      }
    }
  }
}
```

Swap the `env` block for the three variables directly if you would rather not
keep a file, and accept the trade-off described above.

## Running it

```bash
uv run --with mcp --with httpx python server.py
```

It speaks MCP over stdio and expects to be launched by a client rather than
used interactively.

## Rate limits

The free tier allows 100 queries per minute per client id, averaged over ten
minutes. The server self-throttles to one request every 0.7 seconds.

Ordinary use does not approach the ceiling. The floor exists so that an
accidental loop cannot burn the client id before anyone notices.

## Errors

Every tool returns a string, and no tool raises. Failures come back as
`"<tool> error: <message>"`.

A refusal is never reported as an empty result. Reporting zero results for a
403 would let a caller conclude that nothing exists when in fact the request
was blocked, and that conclusion is worse than the error, because it looks
like evidence. So a 403 says that the app may not be approved or that the
community is private, quarantined or banned; a 429 reports the reset window
rather than retrying; an unknown subreddit is an explicit error rather than an
empty listing.

## Tests

```bash
uv run --group dev pytest
uv run --group dev ruff check .
```

The tests need no network. Each injects an `httpx.MockTransport`, so the
whole suite runs against a fake Reddit that honours the parameters the real
one does. Both commands also run in CI on every push and every pull request,
so the result is something you can look at rather than something you have to
take on trust.

## Licence

MIT. The full text is in `LICENSE`.
