# reddit-mcp

A read-only MCP server for the Reddit Data API. It exposes four GET-backed
tools and **no tool that writes** — it cannot post, comment, vote, message,
subscribe or create anything, because no such code path exists.

That is the point. An agent reading Reddit is reading text written by
strangers, some of whom would like it to do something on their behalf. A
prompt that promises to stay read-only can be argued with. A server with no
write capability cannot.

## Tools

| Tool | Endpoint | What it answers |
|---|---|---|
| `subreddit_search(topic, limit)` | `/subreddits/search` | Where is this niche discussed? |
| `subreddit_about(name)` | `/r/{n}/about` + `/about/rules` | How big is it, how active, and what are its rules verbatim? |
| `search_posts(query, subreddit, time_filter, sort, limit)` | `/r/{n}/search` | Which threads match? |
| `thread_comments(post_id, limit)` | `/comments/{id}` | What did people actually say? |

## Two things it does differently

**Every payload is wrapped as untrusted data.** Responses come back inside
delimiters behind an `UNTRUSTED DATA` header, with `<<<` and `>>>` neutralised
in the body *and* the source URL json-encoded, so no comment, title or
permalink can close the wrapper early or forge a line inside the opening
marker. Retrieved content is data to quote, never instructions to follow.

**Threads are sampled from two passes, not one.** A thread can carry hundreds
of comments and the reader has a finite context, so the cut happens here. It
takes the 25 highest-scoring comments from a `sort=top` fetch **and** the 15
most recent from a separate `sort=new` fetch, merged by comment id. Drawing
both halves from a single top-sorted response — which is what the first
version did — means the "recent" comments are only the newest among the ones
Reddit already ranked highest, and the genuine complaint sitting at three
upvotes under the joke at nine hundred can never appear.

The response opens with `sampled: N of M comments`, where M is the thread's
own comment count. Quote M as the denominator of any claim about recurrence;
never estimate it. If the recency pass fails, the header says `a top pass
only` rather than claiming both.

## Access

Reddit's Data API is not self-service. Under the Responsible Builder Policy
you must request access through a form and be approved by a person *before*
creating an app, and unauthenticated traffic is blocked rather than
throttled. Reddit's definition of commercial use is broad — it names
subscription services and paywalled mobile apps outright — and a separate
clause forbids mining Reddit data, commercially or not, without written
approval. Read the policy and describe your use case accurately; misstating
it is sanctioned with token revocation and account suspension.

## Configuration

Three environment variables:

```
REDDIT_CLIENT_ID=...
REDDIT_CLIENT_SECRET=...
REDDIT_USER_AGENT=your-app/1.0 (by /u/your_handle)
```

The user agent is not decorative: Reddit throttles generic ones hard, and it
carries your handle.

Auth is OAuth `client_credentials` against `www.reddit.com/api/v1/access_token`,
which yields an app-only read-only token. Data requests go to
`oauth.reddit.com`. The token is cached for its own lifetime minus a margin,
and a 401 buys exactly one refresh and one retry — retrying a rejected client
id in a loop is how an app loses its access.

Requests are self-throttled to one every 0.7s. The free tier allows 100 per
minute per client id, averaged over ten minutes, so the ceiling is never
approached; the floor exists so an accidental loop cannot burn the client id
before anyone notices.

## Run

```bash
uv run --with mcp --with httpx python server.py
```

As an MCP stdio server, for example in `.mcp.json`:

```json
{
  "mcpServers": {
    "reddit": {
      "command": "uv",
      "args": ["run", "--with", "mcp", "--with", "httpx", "python", "/path/to/server.py"],
      "env": {
        "REDDIT_CLIENT_ID": "...",
        "REDDIT_CLIENT_SECRET": "...",
        "REDDIT_USER_AGENT": "your-app/1.0 (by /u/your_handle)"
      }
    }
  }
}
```

A caution worth stating: putting credentials in a config file that the model
can read is not the same as protecting them. If the agent holds a `Read`
tool and the config sits in its working directory, injected content can ask
it to read the file. Keep the config outside what the agent can reach, or
supply the credentials to the process another way.

## Tests

```bash
uv run --group dev pytest
```

No network: every test injects an `httpx.MockTransport`. The fake backend
honours `sort` and `limit` the way Reddit does, which matters — an earlier
version returned every comment regardless, and that let a test assert the
newest comment survived when the server could never have retrieved it.

Two of the tests are deliberately mutually confirming: one asserts the newest
comment is present when both fetches succeed, the other forces a 403 on the
second fetch and asserts it is absent. Remove the second fetch and the first
test fails.

## Errors

Every tool returns `"<tool> error: <message>"` as a string and never raises.
A refusal is never reported as an empty result — reporting zero results for a
403 would let a caller conclude nothing exists from a blocked request. 403
distinguishes "app not approved" from "private or quarantined subreddit"; 429
reports the reset window instead of retrying; an unknown subreddit is an
explicit error.

## License

MIT.
