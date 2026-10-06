# 2026-10-06 — #948: Flow's generation record lost its `"CAE"` marker

**Question.** Three reporters (t2v `YhhmEf`, r2v `MZZa6b`, i2v `eb1hJf`) see a billed
migrated-host submit end in `WireFormatError: no generation record`. What does the reply
look like now?

**Instrument.** `scripts/dev/spike_948_submit_envelope.py`: one veo-lite t2v through
`FlowApiClient` on a profile served flow.google.com. Every submit/status frame is reduced
to a skeleton (UUIDs → `<uuid>`, URLs → `<url>`, strings → `<str>`, ints kept). Cost: one
veo-lite clip.

**Observed (1/1 run).** gflow raised the reported error. The skeletons:

```
YhhmEf [null, 770, [["<uuid>", null, null, ["<str>", [ts], null, null, "<uuid>", "<uuid>", [ts]], "<uuid>"]],
        [["<uuid>", "<uuid>", "<uuid>", null, null, [[ts], "<str>", …, [6], 1], null, [[…]]]]]
jwpduf [null, null, [["<uuid>", "<uuid>", "<uuid>", null, null, [[ts], "<str>", …, [2], 1], null, [[…]]]]]
```

- The record is **still present** in the submit reply, in the same wrapper slot as
  before (`[3][0]`). Only slot 3 changed: `"CAE"` became `null`.
- The status poll record has the same change. Status codes (6 submitted, 2 running) are
  in the same place.
- The `770` in the submit reply is the post-charge credit balance, as one reporter found.

**Reading.** The pre-registered first row held: the issue reproduces, and the envelope is
measured. A separate accept-envelope parser (one reporter's second change) is not needed,
because the full record is still in the reply. The fix is to `_is_record`
(`api/transports/batchexecute.py`). It now accepts `"CAE"` or `null` at slot 3 and requires
the DETAILS list at slot 5 in place of the lost marker.

**Follow-up runs with the fix, same day.**

- `spike_948_submit_envelope.py --project …` (veo-lite) logged every record on any rpc
  for our workflow: `YhhmEf` 6 at 12.3 s, then `jwpduf` 2 every 5 s, then `jwpduf` 3 at
  53.3 s, then `as29s` 3 with the video URL at 55.8 s. The run succeeded.
- `spike_948_project_records.py` ($0) opened the project and a finished clip's route.
  `as29s` carried the done record (null slot 3, status 3, URL, size), and the fixed
  `_is_record` matches it. That is the `gflow data download` path. The project-load rpc
  `Zzl0ze` also carries full records, but the driver does not need it: the live run got
  status 3 from `jwpduf`.
- `tests/e2e/test_migrated_host_e2e.py::test_e2e_t2v_runs_on_flow_google_com_by_default`
  with Flow's default model and length: **1 pass (47 s), 1 timeout (600 s).** In the
  timed-out run, the submit parsed and the clip later showed as done on `Zzl0ze`, but no
  terminal record reached the listener. That run's log was not kept, so the cause is
  **unmeasured**. If it recurs, capture the status rpcs with
  `spike_948_submit_envelope.py`.

**Not measured, originally.** `as29s` (now measured above). The run failed at submit, so the clip route was never opened.
The reporter's description (null at slot 3, the rest unchanged) is consistent with
`jwpduf`, and the e2e run after the fix exercises it. Also not measured: `MZZa6b` and
`eb1hJf` on our account. They share `generation_record`, and the reporters' captures show
the same shape.
