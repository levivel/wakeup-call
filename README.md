# wakeup-call

The core architecture
Four pieces, regardless of how you build it:

Scheduling engine — decides when to text/call someone based on their goal and deadline
Messaging/calling layer — actually sends the SMS or places the call
AI layer — generates the message content / handles the call conversation
User + goal data store — who they are, what they're tracking, billing status

Build vs. buy for each piece
SMS + calls: don't build this yourself. Use Twilio (or Vonage as an alternative). Twilio gives you:

Programmable SMS (texting people)
Programmable Voice (placing calls, can use TTS or hook into an AI voice model)
This is the single biggest "buy don't build" decision — telephony compliance (10DLC registration, carrier filtering, opt-outs) is a real swamp you don't want to wade into yourself.

AI conversation layer:

For texts: Claude or GPT API generating personalized check-in messages based on goal + timeline + past responses
For calls: this is harder. You'd use Twilio's voice webhooks + a speech-to-text → LLM → text-to-speech pipeline, or a layer like Vapi or Bland.ai that's purpose-built for AI phone calls and sits on top of Twilio-like infra. At $5/month/user, starting with calls might not be financially viable — voice AI costs per minute add up fast. I'd seriously consider SMS-only for v1 and treat calls as a premium upsell later.

Scheduling logic:

This is genuinely your product's IP — the "intervals approaching and receding from the goal" idea. Worth building this yourself rather than outsourcing.
Simple version: a cron job / scheduled worker (e.g., on a service like Render, Railway, or AWS Lambda + EventBridge) that runs hourly, checks each user's goal deadline, and decides "is this a check-in moment?" based on a configurable curve (e.g., more frequent near the deadline, tapering off after).

Backend + database:

Postgres for user/goal data
A simple backend framework — given you're starting out, something like Node/Express or Python/FastAPI with Postgres will get you moving fast
Auth: Clerk or Auth0 to avoid building auth yourself

Billing:

Stripe for the $5/month subscription — handles recurring billing, dunning, cancellation flows out of the box

Suggested v1 build order

Landing page + waitlist (validate demand before writing backend code)
User signs up → sets a goal + deadline + phone number
Stripe subscription checkout
Backend cron job that calculates check-in times based on your interval curve
At each check-in time, call an LLM to generate a personalized message based on goal context + history, then send via Twilio SMS
User replies → Twilio webhook captures it → store as a check-in log → optionally feed into next message's context
Calls as a v2 feature once SMS engagement proves the core loop works

A few things worth deciding now, not later

Inbound reply handling: when someone texts back "I did it" or "I'm struggling," do you want the AI to respond conversationally, or just log it? Conversational is more compelling but increases AI cost and moderation surface area (someone venting about mental health to your bot needs careful handling).
Compliance: SMS marketing/automated messaging requires explicit opt-in language and easy opt-out (STOP keyword) — Twilio handles the mechanics but you need the right consent flow in your signup.
Unit economics: at $5/mo, your per-user costs (Twilio SMS ~$0.0079 each, LLM calls, any voice minutes) need to stay well under that. Worth modeling out a rough monthly cost per active user before committing to the price point.

