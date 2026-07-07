All you need are logfiles.

Logfiles is the only reliable source of truth. We always use logfiles to investigate incidents anyway.
When application, container crashes the logs (stdout, stderr) are still preserved on the node and can be pushed over the network to central storage. If needed.

When other solutions push metrics, telemetry, errors:
- they assume the application will not crash because of the error and there will be enought time to submit data
- they assume the network will be available, some cloud solutions cut network before cutting pod
- OOM killer does not care about it
- Segfault does not care about it
- OOM killer because of buffered telemetry because of network errors does not care about

Logfiles stay. Or at least they are more reliable then other alternatives.

- Use structured logging, filter on extra data and calculate metrics
- Capture Exception/error messages and fill sentry
- Calculate start/end times, join on extra fields and fill telemetry
