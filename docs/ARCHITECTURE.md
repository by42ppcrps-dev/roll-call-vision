# Roll Call Vision: architecture

```mermaid
flowchart LR
  subgraph Planned["Planned, not deployed"]
    BC[Board Clerk MCP client<br>verify_vote_by_video]
    S3[(S3 meeting video input<br>SSE, 7-day lifecycle)]
  end
  subgraph AWS["AWS us-east-2 (deployed: zip + layer rcv-opencv5:1, v0.2.2)"]
    API[Lambda Function URL<br>AWS_IAM auth]
    L[Lambda roll-call-vision<br>Python 3.12 + OpenCV 5.0.0 layer<br>demo clips bundled]
    CW[CloudWatch Logs<br>agent trace JSON]
  end
  subgraph Agent["Agent loop inside the Lambda (rcv/agent.py)"]
    P1[Perceive: count_hands<br>stride 5, thresh 35]
    D{ayes said == seen<br>and no unsure seats?}
    P2[Re-perceive:<br>stride 1, thresh 25]
    D2{match?}
    A1[mark_vision_confirmed]
    A2[Hold minutes approval,<br>request human confirmation]
  end
  API --> L --> CW
  L --> P1 --> D
  D -- yes --> A1
  D -- no --> P2 --> D2
  D2 -- yes --> A1
  D2 -- no --> A2
  BC -.-> API
  S3 -.-> L
  A2 -.-> BC
```

OpenCV 5 pipeline per seat ROI: grayscale → Gaussian blur → absdiff vs a median baseline from the first 1.5 s of the meeting video → binary threshold → morphological open → external contours → largest-area test; dark-fraction occlusion flag. Per-frame votes aggregated over the vote window (>=50% of frames = aye; 20–50% = unsure).

Live endpoint (IAM): `https://hj6wfy2lilbbllikwriblbsx3m0thfif.lambda-url.us-east-2.on.aws/`, a Function URL with AWS_IAM auth (not public). Deploy path is zip function code + OpenCV layer (no container / ECR). Deployed today: the Function URL, the Lambda with the OpenCV 5 layer, and CloudWatch Logs. The demo clips ship inside the zip. The S3 input, its lifecycle rule and the Board Clerk client are planned and not deployed.
