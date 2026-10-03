# baphim.xyz infrastructure notes

A working Proxmox homelab, documented as part of an ongoing cybersecurity program. This is a curated subset of a larger private vault, service architecture, real incidents, runbooks built from actually hitting and fixing problems, and concept notes written while learning the underlying mechanisms.

## Structure
- `infrastructure.md`, what's running and on what.
- `services/`, one page per service, purpose, configuration, dependencies, known issues.
- `incidents/`, real postmortems: what broke, root cause, the fix.
- `runbooks/`, step-by-step procedures distilled from those incidents and builds, written to be reusable.
- `concepts/`, underlying mechanisms explained in my own words, written while working through the "why" behind something that happened in the lab.

## A note on what's here
This is deliberately not a complete mirror of the private working vault. Internal IPs, hostnames, and some operational specifics are generalized or omitted rather than published, even with placeholders, some details (like physical key custody for backup encryption) simply don't belong in a public document regardless of how they're presented.
