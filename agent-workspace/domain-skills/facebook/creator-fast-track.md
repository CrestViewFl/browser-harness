# Facebook Creator Fast Track: eligibility and application handoff

## Entry points

- Current requirements: `https://creators.facebook.com/creator-fast-track`
- The landing page's **Apply on Facebook** anchor leads to
  `https://www.facebook.com/creator_programs/signup` with a `referral_code` query
  parameter. Extract the live href and preserve that parameter; do not hardcode
  a referral value from another run.

## Desktop and mobile browser boundaries

In an authenticated desktop Chrome session, the application route can display
**Continue on mobile**, a QR code, and **This link is only available on a mobile
device. Scan the QR code to begin.** This is a handoff screen, not an application
form or submission confirmation.

An iPhone viewport and user agent did not expose a web form. The route displayed
**Open the Facebook app to continue** and **The program is currently only
available in the Facebook app.** The URL could become the Facebook root while
the app handoff remained visible. Do not interpret that root URL as an ordinary
home feed or assume that a mobile browser can submit the application.

If device emulation was used, restore it on that tab before leaving:

```python
cdp("Emulation.clearDeviceMetricsOverride")
cdp("Emulation.setUserAgentOverride", userAgent="")
```

Return to the live application href and verify the QR screen. Report that
submission remains pending unless the application itself supplies a receipt.

## Eligibility evidence

Read the current landing page's **Eligibility for Creator Fast Track** and
**How to unlock your payout** sections. Launch announcements may omit later
follower tiers or differ from the current enrollment rules. Do not treat a
high-payment tier in the hero as the minimum follower requirement.

Keep program requirements, verified account facts, and unknown account facts
separate. A public profile follower count can establish a failed threshold;
public lifetime view counts cannot establish views received in a recent
eligibility window. Recent photos with music are not proof of recent Reels.

## Account identity and DOM traps

- Read the signed-in name or account menu. A company logo in the avatar does
  not establish that the active identity is a Page rather than its owner.
- A Page may show **Switch into ... Page to start managing it** while the
  signed-in identity remains the owner.
- Facebook can retain hidden home-feed and navigation nodes after a Page
  transition. Scope extraction to visible elements or the active main region;
  an unrestricted anchor scan can mix unrelated posts into the evidence.
- `wait_for_load()` can finish while the Page header is still a skeleton.
  Verify that the name and relevant fields have actually rendered before
  interpreting missing text or taking the final screenshot.
