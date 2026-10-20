# Roll Call Vision

OpenCV 5 checks every board vote on video and refuses to approve the minutes until the count matches.

Volunteer boards (HOAs, school boards, nonprofits) approve minutes that record each vote, and those minutes become the legal record. Transcripts and notes often get the count wrong ("passed 5-2" when four hands went up). Roll Call Vision watches the meeting video during each vote window, counts raised hands per seat, compares that with what the transcript says, and then decides what to do: confirm, look again more carefully, or hold approval and ask a person.

Entry for the OpenCV AI Competition 2026, powered by AWS (Agentic Vision Award). Technical report: [REPORT.md](REPORT.md). Architecture: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) and [docs/architecture.png](docs/architecture.png).

All video in this repo is synthetic (stick-figure seats). No real meeting footage, no faces, no identities.

## How it works

1. **Perceive** (`rcv/vision.py`, OpenCV 5.0): for each seat region, grayscale, blur, `absdiff` against a median background from the frames before the vote, threshold, morphological open, external contours, largest-area test. A dark-fraction check flags a blocked seat. Per-frame results are pooled over the vote window (50% or more of frames = aye, 20 to 50% = unsure).
2. **Decide** (`rcv/agent.py`): if the count matches the transcript and no seat is unsure, confirm. Otherwise re-look at the same window with a denser, lower-threshold setting.
3. **Act**: still off after the second look means hold minutes approval and ask a human. A window with zero frames or an empty vote list never confirms. Every step goes into a JSON trace.

## Run it locally

Python 3.12 or newer.

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt pytest
.venv/bin/python -m pytest -q tests        # 13 tests
.venv/bin/python demo/run_demo.py          # renders 4 synthetic meetings to demo/out/ and prints each decision
.venv/bin/python eval/run_eval.py          # 32 synthetic meetings, 81 votes, seed 26
```

`demo/run_demo.py` covers four cases: clean (confirmed), transcript error (caught and held), occluded seat (held for a person), lighting drift (confirmed).

## Results (synthetic, reproducible)

| Metric | Result |
|---|---|
| Hand-count accuracy | 69/81 (85%) |
| Injected transcript errors caught | 4/4 |
| Unsafe confirms (approved a wrong transcript) | 0 |
| Holds on votes where the transcript was already right | 12/81 (8 half raises, 4 occlusions) |

Weak spots are in REPORT.md: half-height raises are undercounted (1/9) and fixed seat boxes are drawn by hand. In those cases the agent holds rather than confirming a wrong count. Nothing here is a claim about real rooms.

## AWS deployment

Deployed to AWS Lambda in us-east-2 as a zip function plus an OpenCV 5 layer (no container):

- Function `roll-call-vision`, Python 3.12, layer with `opencv-python-headless==5.0.0.93` and numpy.
- Function URL with AWS_IAM auth. It is not open to the public, so signed requests only. For judging we can run it live over a screen share, and the demo video shows live invocations.
- Request: `{"demo": "clean"}` or `{"demo": "transcript_error"}` for the bundled clips, or `{"s3_video": "s3://bucket/key.mp4", "transcript": {...}, "seats": 7}`.
- Response: per-vote `ayes_said`, `ayes_seen`, `action`, and `minutes_approvable`. Errors fail closed (`minutes_approvable: false`).
- The full decision trace is written to CloudWatch Logs on every call.

Build the zips with `deploy/build.sh` (no Docker needed). `deploy/iam-policy-rcv-deployer.json` is the least-privilege policy for the deploying user. `deploy/smoke.sh` sends a signed request to a deployed URL.

## Responsible use

Seats are counted, never identified. No face recognition. The system only flags; a person always makes the final call on a hold. Intended for meetings that are already public record and recorded with consent.

## License

MIT. See [LICENSE](LICENSE).
