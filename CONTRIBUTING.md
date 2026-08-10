# Contributing

Corrections and improvements are welcome.

## Before submitting a change

- Verify commands on the Ubuntu and desktop versions stated in the guide.
- Preserve an explicit rollback path for system, authentication, boot, and network changes.
- Distinguish observed results from assumptions or general recommendations.
- Keep package-owned files unchanged when a per-user override is sufficient.
- Use native resolution when evaluating display scaling.
- Keep page or document zoom at 100% when comparing application-scale factors.

## Privacy requirements

Do not commit:

- Passwords, tokens, cookies, API keys, or private SSH/GPG keys
- Biometric descriptors or photographs
- Personal email addresses or account identifiers without consent
- Private Tailnet details, hostnames, DNS suffixes, or node addresses
- Real SMB server addresses, usernames, or share names
- Machine IDs, serial numbers, MAC addresses, or exported credential stores

Use clear uppercase placeholders and explain what readers must replace.

## Markdown style

- Use descriptive headings and fenced code blocks with language identifiers where useful.
- Prefer commands that can be verified independently.
- State whether a restart, logout, or full application exit is required.
- Include the expected output when it materially helps validation.
- Keep examples reusable rather than tied to one account or private network.
