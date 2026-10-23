# Trail Buddy firmware: instructions for every session

You are building firmware for the Trail Buddy radio under a clean-room process.

- Before doing anything else in every session, run the four checks in `docs/clean-room/SESSION-CHECKS.md`; if any fails, stop and tell the user.
- Use only this repository: the `docs/` folder (including `docs/clean-room/POLICY.md`, `docs/clean-room/REQUIREMENTS.md` and the message spec), `spec/` once it exists, the tests, and the vendor and library documentation listed in `docs/clean-room/POLICY.md` Section 14.
- Do not search for, read, copy, port, translate or imitate code, documents or protocol definitions from any other mesh messaging project, and do not rely on your memory of how such projects work.
- If anything is missing or unclear, stop and add it to `OPEN-QUESTIONS.md`. Do not guess.
- Every change must map to a requirement ID in `docs/clean-room/REQUIREMENTS.md` and a test.
- Keep board code in `firmware/bsp/` and radio code in `firmware/radio/`. Board details come only from the manufacturer's datasheets, schematics and board support.
- Record every new dependency's licence and purpose in `SOURCE-REGISTER.md` before using it.
- Logging must never crash the radio: no arithmetic inside log statements, and widen unsigned values before subtracting.
- Code is merged only after automated tests pass and a clean human reviewer approves.

## Working with the owner
- The owner is not a developer. Use plain English and short reports.
- Never change the spec, POLICY.md or REQUIREMENTS.md.
- Before changing any other file, show the owner the change and wait for the owner's yes. Exception: new questions added to `OPEN-QUESTIONS.md` need no approval first; list them in the end-of-task report.
- After each approved change, save a snapshot with a clear message. Never save logs/ or .DS_Store files.
- End every task with: what you did, what changed, and what's waiting on the owner.
