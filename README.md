# baphim.xyz infrastructure notes

A working Proxmox homelab, documented as part of an ongoing cybersecurity program. This is a curated subset of a larger private vault: service architecture, real incidents, runbooks built from actually hitting and fixing problems, and concept notes written while learning the underlying mechanisms. The lab covers self-hosted services, SIEM monitoring with Wazuh, network detection with Suricata and Zeek, and local LLMs used to explain security alerts.

## Structure
The full list of pages is in the [index](index.md).

- `infrastructure.md`, what's running and on what.
- `services/`, one page per service, purpose, configuration, dependencies, known issues. Includes a reference page for the local models I have tested.
- `incidents/`, real postmortems: what broke, root cause, the fix, and what I would do differently. Some are still open and say so.
- `runbooks/`, step-by-step procedures distilled from those incidents and builds, written to be reusable. Values in angle brackets are placeholders to fill in.
- `concepts/`, underlying mechanisms explained in my own words, written while working through the "why" behind something that happened in the lab.

## A note on what's here
This is deliberately not a complete mirror of the private working vault. Internal IPs, hostnames, usernames, and some operational specifics are generalized or omitted rather than published. Runbooks use angle-bracket placeholders where a value has to be supplied, but the real values never appear in this repo. Some details, like physical key custody for backup encryption, simply don't belong in a public document regardless of how they're presented.