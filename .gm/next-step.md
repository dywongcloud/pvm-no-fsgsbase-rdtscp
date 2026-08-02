# Next step

Dispatch the `instruction` verb to plugkit by writing `.gm/exec-spool/in/instruction/<N>.txt` (any unique N) with body `{}` (or `{"prompt":"<user request>"}` on the first dispatch of the turn). Read the response from `.gm/exec-spool/out/<N>.json` and follow the imperative in the `instruction` field.

This file is auto-rewritten by plugkit on every instruction dispatch.
