# Prompts

This file documents every prompt and model setting used in the **LinkedIn Post Qualifier & Comment Drafter** workflow. The sample company is fictional (LeapFlow) and is used only to demonstrate the workflow. To adapt the project, replace the four company context lines.

## Overview

| Step | n8n node | Purpose | Model | Temperature |
| 1 | Qualify Post | Score a post's relevance, return structured JSON | Google Gemini (Flash) | 0.2 |
| 2 | Draft Comments | Write 3 comment options for qualified posts | Google Gemini (Flash) | 0.7 |

Why two different temperatures: scoring should be consistent from run to run (low temperature), while comment drafts should vary in wording and tone (higher temperature).

---

## 1. Qualify Post

### User prompt (expression)

```
Post author: {{ $json['Author Name'] }}
Post text:
{{ $json['Post Text'] }}
```

### System message

```
You evaluate LinkedIn posts for a B2B team.

Company: LeapFlow, a B2B company that builds AI workflow automation (n8n, Make, LLM agents) for small and mid-size businesses.
Target audience: founders, COOs, operations managers, sales and marketing heads at SaaS, e-commerce, and agency companies with 10-200 employees.
Topics we care about: business process automation, AI agents, CRM and sales automation, customer support automation, lead generation, operational efficiency.
Ignore: job-seeker posts, hiring announcements, memes, politics, motivational quotes with no business content, personal life updates, giveaways.

Score the post from 1-10 for how worthwhile it is for us to engage with it.
8-10: directly relevant to our audience or topics and has a clear angle to comment on.
5-7: loosely relevant.
1-4: not relevant or no useful angle.
Category must be one of: hiring, thought leadership, product launch, industry news, personal story, other.
Return the JSON only. Base the score only on the post text.
```

### Output schema (Structured Output Parser, JSON example)

```json
{
  "relevant": true,
  "score": 8,
  "category": "thought leadership",
  "reason": "Short explanation in one sentence",
  "key_points": ["point one", "point two"]
}
```

### Design notes

- The four company context lines define what "relevant" means. Without them the model has to guess.
- A written scoring rubric (8-10, 5-7, 1-4) keeps scores consistent.
- Fixed category values make the output easy to filter and count.
- "Base the score only on the post text" reduces guessing about the author or company.
- The `reason` field is saved next to every score so a human can check the decision.
- Routing rule in the IF node (Check Score): score is 7 or higher AND relevant is true.

---

## 2. Draft Comments

Runs only for posts that pass the Check Score node.

### User prompt (expression)

```
Post text:
{{ $json.post_text }}

Key points: {{ $json.key_points }}
Why it matters to us: {{ $json.reason }}
```

### System message

```
Write 3 LinkedIn comments replying to the post below, as a human operations professional at LeapFlow, a company that builds AI workflow automation for small businesses.
draft_1: insightful, adds one useful idea or experience.
draft_2: ends with a thoughtful question.
draft_3: short and warm, under 20 words.
Rules: refer to a specific point from the post. No sales pitch, no links, no hashtags. Do not start with "Great post". At most one emoji. Under 50 words each except draft_3.
Return the JSON only.
```

### Output schema (Structured Output Parser, JSON example)

```json
{
  "draft_1": "insightful comment",
  "draft_2": "question-style comment",
  "draft_3": "short friendly comment"
}
```

### Design notes

- Three drafts with different styles give the reviewer a real choice instead of three near-copies.
- The rules block avoids the usual problems with AI comments: generic praise, sales pitches, links and hashtags.
- The qualification output (key points and reason) is passed in so the comment stays specific to the post.
- Drafts are never posted automatically. A person picks one, edits it if needed and posts it.

---

## Customizing for your own company

Replace only these four lines in the Qualify Post system message (and the company line in the Draft Comments system message):

```
Company: [what your company does]
Target audience: [roles and industries you sell to]
Topics we care about: [topics that should raise the score]
Ignore: [content that should get a low score]
```

## Tuning tips

| Problem | Fix |
| Good posts score below 7 | Add the missing topics to "Topics we care about", or lower the threshold in Check Score |
| Irrelevant posts score above 4 | Add those content types to "Ignore" |
| Comments sound generic | Add a rule such as "mention one specific detail from the post" or give one example comment |
| Scores change between runs | Lower the temperature further, or make the rubric more specific |
| Output parsing error | Check that the JSON example matches the field names and that Require Specific Output Format is on |

