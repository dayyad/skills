---
name: x-following-audit
description: Audit who you follow on X (Twitter) and get a colour-coded spreadsheet of low-quality accounts to unfollow (engagement bait, follow-farming, AI slop, inactive). No paid API. You scroll your own Following page; Claude reads it, profiles each account and builds the sheet. Use when asked to audit, clean up or cull your X following list.
---

# X following audit

Result: a spreadsheet of the accounts the user follows, worst first, with a "Your call" column. **Claude never follows, unfollows, posts or clicks anything on the user's account. The user makes every call.**

Follow the steps in order. If a step's check fails, do its "If not" line and re-check. Don't skip ahead.

## Step 0. Check the browser is connected
Claude must be able to drive Chrome through the **Claude in Chrome** extension (works in Claude Code, the Claude desktop app and Cowork).
- Check: list your available tools. If `mcp__claude-in-chrome__*` tools exist (load them first if deferred), call `tabs_context_mcp` with `createIfEmpty: true`.
- If not, say this and stop until they confirm: "I can't see a Chrome connection. Install the **Claude in Chrome** extension from the Chrome Web Store, sign in to it with your Claude account, then tell me 'done'. If you already have it in another Chrome profile, open this chat's browser there."
- Do not fall back to another browser tool that isn't logged in to X.

## Step 1. Open X and confirm login
1. Open `https://x.com/home` in a new tab.
2. Read the page. If it shows a login or sign-up screen, say: "X isn't logged in in this browser. Please log in yourself (I won't touch your password), then tell me 'done'." Re-check after they reply. If the extension is in a different Chrome profile, tell them to switch to that one.
3. When logged in, read their handle from `a[data-testid="AppTabBar_Profile_Link"]` (its `href` is `/handle`). Say: "I can see your feed and you're logged in as @handle."
4. Go to `https://x.com/<handle>/following`. Read the profile's own "Following" count from `https://x.com/<handle>` first (link `a[href$="/following"]`) so you know the target number.

## Step 2. Install the read-only collector, then hand over to the user
Run this in the Following tab. It sends no requests; it only reads rows as they render (and ignores "Who to follow").
```js
window.__xa = window.__xa || new Map();
window.__xaGrab = () => document.querySelectorAll('[data-testid="primaryColumn"] [data-testid="UserCell"]').forEach(c => {
  const l = c.innerText.split('\n').map(s => s.trim()).filter(Boolean);
  const i = l.findIndex(s => /^@\w{1,15}$/.test(s)); if (i < 0) return;
  const k = l[i].toLowerCase(); if (window.__xa.has(k)) return;
  window.__xa.set(k, { n: window.__xa.size + 1, handle: l[i], name: l.slice(0, i).join(' '), follows_you: l.includes('Follows you') });
});
window.__xaGrab(); window.__xaObs?.disconnect();
window.__xaObs = new MutationObserver(window.__xaGrab);
window.__xaObs.observe(document.body, { childList: true, subtree: true });
window.__xa.size
```
Confirm it's working (`window.__xa.size` > 0 and the page shows the Following list), then say: "Everything's set. **You're good to start scrolling.** Scroll your Following list to the very bottom at a normal pace, and tell me when you're done."
Automated scrolling is not part of this skill (X's terms ban scraping; a human at the screen is the safe pattern). Only if the user explicitly asks you to scroll: mouse-wheel, 3-5 s pauses, one run, and warn them it's untested-risk.

When they say done, run `window.__xaGrab(); window.__xa.size`. If it's below the Following count from Step 1, say "I've got N of M. Scroll a bit further and tell me when you're at the bottom" and re-check (a few protected or suspended accounts may never show, so within ~3 of the total is fine).
Read the list out in chunks of ~30 (long output gets blocked) and save it to a scratch file:
```js
[...window.__xa.values()].slice(0, 30).map(x => `${x.n} ${x.handle} ${x.follows_you ? 'F' : '-'}`).join(' ; ')
```
`F` = follows you back. Then `window.__xaObs.disconnect()` and close the tab.

## Step 3. Ask before using the profile API (always ask)
Ask exactly this, and wait for the answer:
> "To judge each account I need its bio and latest posts. Can I look them up with **fxtwitter** (a free public mirror of X, no login, no cost)? It will see the list of handles you follow, but nothing from your X session. If you'd rather not, I can use web search instead. That works but is slower and less accurate. **Yes / No?**"
- **Yes** -> Step 4a. **No** -> Step 4b. Anything else -> ask again.

## Step 4a. Fetch with fxtwitter
`https://api.fxtwitter.com/2/profile/<handle>/statuses` returns the profile (`results[0].author`: bio, followers, following) and ~20 latest statuses; page with `?cursor=<cursor.bottom>`. Keep the first 5 **original** posts (skip items with `reposted_by` or `replying_to`, or a different author). If it fails, try `https://api.fxtwitter.com/<handle>` for profile only. Sleep ~1.2 s between calls, back off on HTTP 429, run it as one background script, and write one JSON file per handle so it can resume. A handle that doesn't resolve is renamed, suspended or protected: leave its cells blank.

## Step 4b. Fallback: web search
One WebSearch per account is too many (sessions are capped at ~200). Instead, hand batches of ~20 handles to a cheap subagent to search and write a one-line "what they post about" each. Mark every row "thin data" and cap its verdict at Review.

## Step 5. Signals (deterministic, per account)
followers, following, ratio = following/followers, follows-you, last post date, median likes/views, likes per 1k followers, hashtag/emoji/em-dash counts, and regex hits per post (hints, not proof):
```
reply_bait  (drop|comment|reply|share) (your|a|with|below|👇)|👇|what are you (building|working on)|tell me (your|what)
connect     let'?s connect|looking to connect|say hi|follow ?back|f4f|follow train|mutuals?
milestone   (away from|close to|help me (hit|reach)|road to) ?\d|\d+ ?k? followers
vote        \b(a or b|which one|this or that|agree\?|yes or no)\b
hype        game[- ]?changer|insane|\brip\b|is dead|changes everything|no one is talking
dm_funnel   dm me|comment ['"]?\w+['"]? (and|&) i'?ll|link in (bio|comments)
thread      bookmark|🧵|save this        listicle  ^\s*(\d+|top \d+)\s+(ai |free |best |tools|ways)
gm          ^(gm|good morning|gn)\b      day_n     \bday \d+\b
```
Follow-back fingerprint: ratio 0.7-1.4 with following >= 500. A follows-you account that also has farming flags is usually follow-for-follow residue.

## Step 6. Classify
Use cheap models: parallel Sonnet subagents at low effort, ~20 accounts each. Give each the rubric below, the fetched JSON and the signals; each writes JSONL `{"n","handle","verdict","quality","flags":[],"reason":"<=25 words citing what the posts/bio/ratio show","who":"<=15 words","posts_about":"<=20 words","evidence_url"}`. Validate every row (enums, quality 1-5, all accounts present) before building. Never invent: unknown stays blank; describe accounts, don't judge people.

**Flags** (any number): **engagement-bait** (reply/vote/like bait, keyword-DM funnels, curiosity-gap hooks, milestone begging) · **follow-farming** (follow-back asks, #connect/#buildinpublic stacks with no substance, follow-back fingerprint) · **ai-slop** (LLM voice, hype listicles, recycled news with no take, tiny engagement for the followers) · **low-signal** (gm/gn, vague motivation, link drops, only memes) · **inactive** (no original post in 6+ months).
**Positive:** specific technical/domain content, real numbers and shipped work, a recognisable voice, teaching, official accounts for tools the user uses. One bait post doesn't condemn an account.
**Verdict:** **Cull candidate** (a pattern dominates 3+ of 5 posts, two strong flags, or inactive + low-signal) · **Review** (mixed or thin data) · **Keep**. Quality 1-5.

## Step 7. Spreadsheet (openpyxl; make a venv if missing)
Sheet **Cull list**, sorted Cull -> Review -> Keep, then quality ascending. Columns: Your call · Verdict · Quality · Flags · Why · Handle (link) · Name · Follows you · Followers · Following · Following ÷ followers (formula) · Last post · Who · Posts about · Evidence post (link) · Follow order #.
- Your call: yellow input, dropdown Unfollow/Keep/Maybe (Unfollow red bold, Maybe amber, Keep green).
- Conditional formatting so colours follow edits: row tint by verdict; quality red->green scale; flag colour by priority follow-farming purple > engagement-bait orange > ai-slop blue-grey > inactive dark grey > low-signal light grey; followers data bar; amber ratio cell for the fingerprint; grey italic for stale last post.
- Freeze panes at C2, autofilter, wrap text, Arial.
Sheet **Summary**: COUNTIF counts by verdict, flag, follows-you and Your call, plus "following after cull". Sheet **Rubric**: the rubric text. Set `fullCalcOnLoad`.
Save to Desktop as `x-following-audit-<date>.xlsx`. If that name exists (or a `~$` lock file shows it's open in Excel), use a new name; never overwrite.

## Step 8. Report back
Counts by verdict and flag, how many cull candidates follow the user back, the 3-5 clearest examples and the pattern that condemns each, anything that didn't resolve, and the file path. Remind them: it's a 5-post snapshot, so skim the Review group before unfollowing, and do the unfollowing themselves.
